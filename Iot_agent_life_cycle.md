# IoT AI Edge Agent Lifecycle & Architecture

## 1. Architecture Overview

The system consists of three primary actors:

AI Edge Agent: The intelligent node running on edge hardware (e.g., NVIDIA Jetson, Raspberry Pi, industrial gateway). It hosts the inference engine (TensorFlow Lite, ONNX, etc.).

Agent Manager: The central authority (Cloud/On-Prem) responsible for identity, policy, and model versioning.

Found MQTT Broker: A local message bus discovered dynamically on the LAN. It acts as the data source (sensors) and data sink (actuators/dashboards) for the agent.

## 2. Phase I: Birth & Provisioning (Zero-Touch Enrollment)
Goal: Establish trust and discover the local environment without hardcoded IP addresses.
Step 1: Secure Boot & Identity
Hardware Root of Trust: The agent boots. It reads a factory-provisioned certificate or TPM (Trusted Platform Module) key.
Call Home: The agent contacts the Agent Manager via a pre-configured HTTPS endpoint (e.g., provisioning.iot-cloud.com) using its factory certificate for mTLS authentication.
Step 2: Agent Manager Validation
The Agent Manager verifies the certificate against the whitelist.
Assignment: The Manager assigns the agent to a specific "Site" or "Customer Tenant."
Token Issuance: The Manager issues a temporary Session Token or a rotational Operational Certificate.
Step 3: Local "Found" Broker Discovery
The agent does not know the IP of the local data source. It uses mDNS (Multicast DNS) / DNS-SD to find it.
Scan: Agent queries the local network for _mqtt._tcp.local.
Resolve: The network responds with the Broker's IP (e.g., 192.168.1.50) and Port (1883 or 8883).
Connect: The agent connects to the found broker. If the broker requires auth, the agent tries a set of site-specific keys delivered during Step 2.

## 3. Phase II: Deployment & Configuration
Goal: Download the "Brain" (AI Models) and the "Instructions" (Config).
Step 1: Configuration Sync
The agent requests its desiredState from the Agent Manager.
Manifest: A JSON file listing:
Models: safety_vest_v1.2.tflite, anomaly_detection_v4.onnx
Topics: Input topics to listen to on the local broker.
Thresholds: Confidence intervals (e.g., "Alert if confidence > 85%").
Step 2: Model Hydration
The agent downloads model artifacts.
Verification: Checksums (SHA-256) are validated to ensure models haven't been tampered with in transit.
Warm-up: The agent loads the models into memory (GPU/NPU) and runs a dummy inference to verify stability.

## 4. Phase III: Operation (The Inference Loop)
Goal: Real-time processing of local data streams.
Interaction A: Inbound Data (From Found Broker)
The agent acts as a Subscriber on the local broker.
Subscription: site/production/line1/cameras/# or sensors/temperature/#
Buffering: High-frequency sensor data is buffered locally.
Interaction B: The AI Inference
Trigger: New MQTT message received OR Time-window full.
Process:
Decode payload (JSON/Binary image).
Pre-process (Normalize/Resize).
Inference: Pass to the loaded model.
Post-process (NMS, threshold filtering).
Interaction C: Outbound Actions (To Found Broker)
The agent acts as a Publisher to the local broker.
Topic: agent/{agent_id}/inference/result
Payload:
{
  "timestamp": 1715421200,
  "model": "safety_vest_v1.2",
  "inference_time_ms": 14,
  "detections": [
    { "class": "no_vest", "confidence": 0.92, "bbox": [10, 50, 200, 300] }
  ]
}


Interaction D: Heartbeat (To Agent Manager)
Every 60 seconds, the agent sends a heartbeat to the central Manager (via MQTT or HTTPS).
Payload:
Health: CPU Temp, RAM usage.
Model Performance: Avg inference time.
Drift Metrics: Distribution of output classes (e.g., "I am seeing 99% 'no_vest'—is the camera blocked?").

## 5. Phase IV: Updates & Evolution
Goal: Update the AI without downtime.
Scenario: The "Shadow" Deployment
The Agent Manager pushes a new model version (safety_vest_v1.3).
Download: Agent downloads v1.3 in the background.
Shadow Mode: The agent runs both v1.2 and v1.3 on incoming data.
v1.2 controls the actuators (Active).
v1.3 logs results silently (Passive).
Validation: The agent sends a comparison report to the Manager. "v1.3 matches v1.2 98% of the time but is 10ms faster."
Swap: The Manager sends a command: promote v1.3.
Cleanup: v1.2 is unloaded from memory.

## 6. Phase V: Retirement & Decommissioning
Goal: Securely kill the agent.
Step 1: Revocation
Administrator Action: Clicks "Retire" in the Agent Manager dashboard.
CRL Update: The agent's certificate serial number is added to the Certificate Revocation List.
Broker Ban: The Agent Manager instructs the Site Gateway (if managed) or rotates the keys, effectively locking the agent out of the local Found Broker.
Step 2: The Poison Pill
If the agent is still online, the Manager sends a wipe command.
Action: The agent deletes:
Stored AI models (IP protection).
Cached sensor data (Privacy protection).
WiFi/Network credentials.
The agent reverts to a factory-reset state or bricks itself depending on security policy.
Technical Reference: MQTT Topic Structure
A standardized topic structure ensures the Agent plays nicely with the Found Broker.

| Scope | Topic Pattern | Direction | Purpose |
| ----- | ------------ | --------- | ------- |
| Discovery | agent/{id}/status | Pub |  Retained message. {"state": "online", "ip": "192.168.1.55"}. Use LWT (Last Will & Testament) here. |
| Command | agent/{id}/cmd | Sub | Receive restart/update commands. |
| Data In | +/+/sensor/# | Sub | Wildcard subscription to local sensors.| 
| Data Out | agent/{id}/event | Pub | High-priority inferences (e.g., alarms).| 
| Telemetry| agent/{id}/telemetry | Pub | Low-priority stats (FPS, temperature). |


