#LINUXES FOR DOCKER ON SMALL COMPUTER#

The requirements for a small computer running our IOT management evironment:

1. multiple services need to be run
2. restart when powered up
3. container management system to allow for quick upgrade of software with tested containers

The following will be a list of tested harware with Linux with tested load and startups:

 | Board	| Recommended Distro	| Why? |
 | -------- | --- | --- |
| RPi 3	| DietPi (64-bit)	| RPi 3 has limited RAM (1GB). DietPi uses ~30-50MB RAM at idle, leaving the maximum possible room for Docker containers.|
| RPi 4	| Raspberry Pi OS Lite (64-bit)	| Perfectly stable, optimized kernel, and the most extensive community support for troubleshooting Docker-specific GPIO/hardware issues. |
| RPi 5 |	Ubuntu Server |	RPi 5 is powerful enough to run Ubuntu comfortably. Ubuntu provides the most modern Docker/Containerd packages and 16k page size kernel support. |
| Odroid (N2/M1/H3)	| Armbian |	Hardkernel’s official images often lag. Armbian provides refined, modern kernels and a "minimal" build specifically for server/Docker use. |

For test purposes, the intial install will be DietPi for all of the hardware.
This will give a base line comparison and give and idea of the capability of running docker on all the hardware.  Also, this will become the simplist way to test new containers and new hardware.

Comparative Benchmarking Tests
To compare how Docker performs on different deployments, you should test four key areas: CPU Throughput, Disk I/O (the biggest SBC bottleneck), Memory Latency, and Container Overhead.

1. CPU & Memory: sysbench
Run this inside a Docker container to see the actual performance available to your apps.

Bash

# Test CPU performance
docker run --rm silvios/sysbench cpu --cpu-max-prime=20000 run

# Test Memory speed
docker run --rm silvios/sysbench memory --memory-block-size=1K --memory-total-size=10G run
2. Disk I/O: fio (Flexible I/O Tester)
Since most SBCs run on SD cards or NVMe, disk latency often causes Docker containers to "hang" during startup.

Bash

# Test Random Read/Write performance inside a container
docker run --rm -v $(pwd):/tmp -it martinvw/fio --name=test --rw=randrw --direct=1 --ioengine=libaio --bs=4k --numjobs=1 --size=512m --group_reporting
3. Network Throughput: iperf3
Crucial if you are running a media server (Plex/Jellyfin) or a NAS in Docker.

Bash

# On the SBC (Server)
docker run  -it --rm -p 5201:5201 networkstatic/iperf3 -s

# On your PC (Client)
iperf3 -c <SBC_IP_ADDRESS>

Possible reporting structure:
| Metric	| RPi 3 (DietPi)	| RPi 5 (Ubuntu)	| Odroid N2+ (Armbian) |
| --- | --- | --- | --- |
| Idle RAM Usage |	~45MB	| ~180MB	| ~90MB |
| CPU Prime Time |	Higher (Slower) |	Lower (Faster) |	Lowest (Fastest) |
| Random 4K Write	| (Depends on SD)	| (SSD Recommended)	| (eMMC is faster) |


The "64-bit" Rule: Always use 64-bit (aarch64) OS versions, even on the Pi 3.Docker performance and image availability are significantly better than on 32-bit (armhf).

Thermal Throttling: Run vcgencmd measure_temp (on Pi) during your benchmarks. A distro that doesn't manage thermals well will cause the CPU to throttle, skewing your results.

I/O Wait: Use htop while your containers are running. If you see a high %wa (IO Wait), your OS/SD card is the bottleneck, not the Docker engine itself.