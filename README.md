# VXLAN: Multi-Host Container Networking

## Introduction

VXLAN (Virtual Extensible LAN) is a network virtualization technology that addresses the limitations of traditional VLANs in cloud and container environments. This document explains VXLAN packet structure, implementation, and provides a step-by-step guide for configuring VXLAN tunnels between Docker containers on different hosts.

## VXLAN Packet Structure

![packet encapsulation](https://raw.githubusercontent.com/Raihan-009/vxlan-multi-host-container-networking/refs/heads/main/frame-after-encapsulation.png)

VXLAN uses encapsulation to create Layer 2 overlay networks on Layer 3 infrastructure:

1. **Outer Ethernet Header**: Contains the source MAC (Host1) and destination MAC (Host2) of the physical hosts.
2. **Outer IP Header**: Contains the source IP (10.0.1.30) and destination IP (10.0.1.40) of the VTEPs.
3. **Outer UDP Header**: Uses source port (dynamic) and destination port (4789, standard VXLAN port).
4. **VXLAN Header**: 8 bytes containing flags, reserved fields, and the VNI (100 in this example).
5. **Original Ethernet Frame**: The entire original frame becomes the payload.

The original Ethernet frame becomes completely encapsulated as the payload of the VXLAN packet, allowing it to traverse Layer 3 networks while maintaining Layer 2 connectivity between containers.

## Original Ethernet Frame

![packet encapsulation](https://raw.githubusercontent.com/Raihan-009/vxlan-multi-host-container-networking/refs/heads/main/original-frame-before-encapsulation.png)

Before VXLAN encapsulation, a standard Ethernet frame from a container consists of:

- **Ethernet Header**: Source MAC (Container 172.18.0.11) and Destination MAC (Docker Bridge)
- **IP Header**: Source IP (172.18.0.11) and Destination IP (172.18.0.12)
- **TCP/UDP Header**: Source and Destination ports for the application
- **Payload**: Application data
- **FCS**: Frame Check Sequence for error detection

This standard frame works for communication within a single host but cannot traverse between hosts on different networks without encapsulation.

## VTEP Function

![packet encapsulation](https://raw.githubusercontent.com/Raihan-009/vxlan-multi-host-container-networking/refs/heads/main/vtep.png)

VXLAN Tunnel Endpoints (VTEPs) handle the encapsulation and decapsulation process:

1. Container A (172.18.0.11) on Host 1 (10.0.1.30) sends a packet to Container D (172.18.0.12) on Host 2 (10.0.1.40)
2. The packet goes through the Docker Bridge (172.18.0.1/16) on Host 1
3. The VTEP on Host 1:
   - Recognizes the destination is on another host
   - Encapsulates the original frame
   - Adds VXLAN header with VNI 100
   - Adds outer UDP header with port 4789
   - Adds outer IP header (10.0.1.30 → 10.0.1.40)
   - Adds outer Ethernet header
4. The encapsulated packet traverses the physical network to Host 2
5. The VTEP on Host 2:
   - Receives the encapsulated packet
   - Recognizes the VXLAN header and VNI 100
   - Removes all outer headers
   - Forwards the original frame to the Docker Bridge
6. The Docker Bridge delivers the packet to Container D (172.18.0.12)

This entire process is transparent to the containers, which operate as if they are on the same Layer 2 network.

## Implementation Steps

### 1. Host Setup & Docker Installation

**On both hosts:**

```bash
# Update and install Docker
sudo apt update
sudo apt install -y docker.io
```

### 2. Create Docker Bridge Networks

**On Host 1 (10.0.1.30):**

```bash
# Create a custom bridge network
sudo docker network create --subnet 172.18.0.0/16 vxlan-net
```

**On Host 2 (10.0.1.40):**

```bash
# Create identical network
sudo docker network create --subnet 172.18.0.0/16 vxlan-net
```

### 3. Launch Containers

**On Host 1:**

```bash
# Launch container with static IP
sudo docker run -d --net vxlan-net --ip 172.18.0.11 ubuntu sleep 3000
```

**On Host 2:**

```bash
# Launch container with static IP
sudo docker run -d --net vxlan-net --ip 172.18.0.12 ubuntu sleep 3000
```

### 4. Setup Container Testing Environment

**On both hosts:**

```bash
# Enter the container
sudo docker exec -it CONTAINER_ID bash

# Install networking tools
apt-get update
apt-get install -y net-tools iputils-ping
```

### 5. Create VXLAN Interfaces

**On Host 1 (10.0.1.30):**

```bash
# Create VXLAN interface
sudo ip link add vxlan-demo type vxlan \
  id 100 \
  remote 10.0.1.40 \
  dstport 4789 \
  dev eth0

# Bring interface up
sudo ip link set vxlan-demo up

# Get docker bridge name
ip link | grep 172.18.0.1

# Add VXLAN interface to docker bridge
sudo ip link set vxlan-demo master DOCKER_BRIDGE_NAME
```

**On Host 2 (10.0.1.40):**

```bash
# Create VXLAN interface
sudo ip link add vxlan-demo type vxlan \
  id 100 \
  remote 10.0.1.30 \
  dstport 4789 \
  dev eth0

# Bring interface up
sudo ip link set vxlan-demo up

# Add VXLAN interface to docker bridge
sudo ip link set vxlan-demo master DOCKER_BRIDGE_NAME
```

### 6. Test Container Connectivity

**On Host 1:**

```bash
# Enter container
sudo docker exec -it CONTAINER_ID bash

# Ping container on Host 2
ping 172.18.0.12 -c 2
```

**On Host 2:**

```bash
# Enter container
sudo docker exec -it CONTAINER_ID bash

# Ping container on Host 1
ping 172.18.0.11 -c 2
```

## Monitoring VXLAN Traffic

To monitor VXLAN traffic and troubleshoot connectivity issues:

```bash
# Watch VXLAN traffic on the physical interface
sudo tcpdump -i eth0 udp port 4789 -vv
```

## Conclusion

VXLAN provides an elegant solution for extending Layer 2 networks across Layer 3 boundaries, enabling container communication across different hosts. The encapsulation technique allows containers to communicate as if they were on the same local network, regardless of their physical location.
