# Docker Networking: Bridge, Host & Overlay Networks

## Table of Contents

1. [Introduction to Docker Networking](#introduction-to-docker-networking)
2. [How Docker Networking Works](#how-docker-networking-works)
3. [Docker Network Drivers](#docker-network-drivers)
4. [Bridge Network](#1-bridge-network)
5. [Host Network](#2-host-network)
6. [Overlay Network](#3-overlay-network)
7. [Comparison of Network Types](#comparison-of-network-types)
8. [Docker Networking Commands](#docker-networking-commands)
9. [Best Practices](#best-practices)

---

## Introduction to Docker Networking

Docker networking is a fundamental feature that enables containers to communicate with each other, with the host system, and with external networks. When Docker is installed, it automatically creates a networking subsystem that allows containers to be isolated or connected as needed.

### Why Docker Networking Matters

- **Container Isolation**: Each container can have its own network stack, providing security and separation.
- **Inter-Container Communication**: Containers running on the same or different hosts can communicate seamlessly.
- **Service Discovery**: Docker provides built-in DNS-based service discovery for containers on the same network.
- **Port Management**: Docker handles port mapping and network address translation (NAT) automatically.
- **Scalability**: Networking features support scaling applications across multiple hosts and clusters.

### Docker Networking Architecture

Docker uses a pluggable networking architecture called the **Container Network Model (CNM)**. The CNM defines three building blocks:

1. **Sandbox**: An isolated network stack containing a container's network configuration (interfaces, routing table, DNS settings).
2. **Endpoint**: A virtual network interface that connects a sandbox to a network. Each endpoint belongs to exactly one network and one sandbox.
3. **Network**: A group of endpoints that can communicate directly with each other. Networks are implemented by network drivers.

```
┌──────────────────────────────────────────────────────┐
│                    Docker Host                        │
│                                                       │
│  ┌─────────────┐    ┌─────────────┐                  │
│  │  Container A │    │  Container B │                 │
│  │  ┌────────┐  │    │  ┌────────┐  │                │
│  │  │Sandbox │  │    │  │Sandbox │  │                │
│  │  │  eth0  │  │    │  │  eth0  │  │                │
│  │  └───┬────┘  │    │  └───┬────┘  │                │
│  └──────┼───────┘    └──────┼───────┘                │
│         │ Endpoint          │ Endpoint               │
│    ┌────┴───────────────────┴────┐                   │
│    │        Docker Network       │                   │
│    │       (Network Driver)      │                   │
│    └─────────────────────────────┘                   │
│                                                       │
└──────────────────────────────────────────────────────┘
```

---

## How Docker Networking Works

When Docker is installed, three default networks are automatically created:

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
f6e5d4c3b2a1   host      host      local
1a2b3c4d5e6f   none      null      local
```

- **bridge**: The default network for containers. All containers connect to this network unless specified otherwise.
- **host**: Removes network isolation between the container and the Docker host.
- **none**: Disables all networking for the container.

### Network Namespace

Docker uses Linux **network namespaces** to provide isolation. Each container gets its own network namespace, which includes:

- Its own IP address
- Its own routing table
- Its own network interfaces
- Its own firewall rules (iptables)

This means containers are completely isolated from each other at the network level unless explicitly connected through a Docker network.

### Virtual Ethernet (veth) Pairs

Docker uses **veth pairs** (virtual Ethernet pairs) to connect containers to networks. A veth pair acts like a virtual cable:

- One end is placed inside the container's network namespace (appears as `eth0`).
- The other end is attached to the Docker network bridge on the host.

---

## Docker Network Drivers

Docker supports several network drivers, each designed for different use cases:

| Driver      | Description                                         | Scope   |
|-------------|-----------------------------------------------------|---------|
| **bridge**  | Default driver for standalone containers            | Local   |
| **host**    | Shares host's network namespace                     | Local   |
| **overlay** | Multi-host networking for Docker Swarm              | Swarm   |
| **macvlan** | Assigns a MAC address, making container appear physical | Local |
| **none**    | Disables networking entirely                        | Local   |
| **ipvlan**  | Gives containers control over IPv4/IPv6 addressing  | Local   |

This document focuses on the three most important drivers: **Bridge**, **Host**, and **Overlay**.

---

## 1. Bridge Network

### What is a Bridge Network?

A **bridge network** is the default and most commonly used network driver in Docker. It creates an isolated internal network on the Docker host, and containers connected to the same bridge network can communicate with each other. Containers on different bridge networks are isolated from each other.

A bridge network works at **Layer 2 (Data Link Layer)** of the OSI model and uses a Linux bridge (software-based switch) to forward traffic between containers.

### How Bridge Network Works

When Docker is installed, it automatically creates a default bridge network called `bridge` (also referred to as `docker0`):

```
┌──────────────────────────────────────────────────────────────┐
│                       Docker Host                             │
│                                                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │ Container1 │  │ Container2 │  │ Container3 │             │
│  │ 172.17.0.2 │  │ 172.17.0.3 │  │ 172.17.0.4 │             │
│  │   eth0     │  │   eth0     │  │   eth0     │             │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘             │
│        │ veth          │ veth          │ veth                │
│  ┌─────┴───────────────┴───────────────┴──────┐             │
│  │           docker0 Bridge (172.17.0.1)       │             │
│  └──────────────────────┬─────────────────────┘             │
│                         │ NAT (iptables)                     │
│  ┌──────────────────────┴─────────────────────┐             │
│  │              Host Network (eth0)            │             │
│  └──────────────────────┬─────────────────────┘             │
└─────────────────────────┼─────────────────────────────────── ┘
                          │
                    External Network
```

**Step-by-step process:**

1. Docker creates a virtual bridge interface called `docker0` on the host.
2. Each container gets a **veth pair** — one end inside the container (`eth0`), the other attached to `docker0`.
3. Containers on the same bridge can communicate via their internal IP addresses.
4. Outbound traffic from containers is NATed through the host's IP address using `iptables`.
5. Inbound traffic from external networks reaches containers via **port mapping** (`-p` flag).

### Default Bridge vs. User-Defined Bridge

Docker provides two types of bridge networks:

#### Default Bridge (`docker0`)

- Created automatically when Docker is installed.
- Containers must use IP addresses to communicate (no DNS resolution).
- All containers connect here by default if no network is specified.
- Limited configuration options.

#### User-Defined Bridge

- Created manually by the user.
- Provides **automatic DNS resolution** — containers can communicate by name.
- Better isolation and security.
- Containers can be connected/disconnected dynamically.
- Supports custom subnets, gateways, and IP ranges.

### Creating and Using Bridge Networks

#### Creating a User-Defined Bridge Network

```bash
# Create a simple bridge network
$ docker network create my-bridge-network

# Create a bridge network with custom configuration
$ docker network create \
  --driver bridge \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --ip-range 192.168.1.128/25 \
  --opt com.docker.network.bridge.name=my-bridge \
  my-custom-bridge
```

#### Running Containers on a Bridge Network

```bash
# Run a container on the user-defined bridge network
$ docker run -d --name web-server --network my-bridge-network nginx

# Run another container on the same network
$ docker run -d --name app-server --network my-bridge-network node:18

# Now app-server can reach web-server by name:
$ docker exec app-server ping web-server
# PING web-server (192.168.1.2): 56 data bytes
# 64 bytes from 192.168.1.2: seq=0 ttl=64 time=0.102 ms
```

#### Connecting and Disconnecting Containers

```bash
# Connect a running container to a network
$ docker network connect my-bridge-network existing-container

# Disconnect a container from a network
$ docker network disconnect my-bridge-network existing-container
```

#### Port Mapping with Bridge Networks

Since bridge networks are isolated, you need to expose container ports to the host:

```bash
# Map container port 80 to host port 8080
$ docker run -d --name web -p 8080:80 --network my-bridge-network nginx

# Map container port 80 to a random host port
$ docker run -d --name web -P --network my-bridge-network nginx

# Map to a specific host interface
$ docker run -d --name web -p 127.0.0.1:8080:80 --network my-bridge-network nginx
```

### Inspecting a Bridge Network

```bash
$ docker network inspect my-bridge-network
[
    {
        "Name": "my-bridge-network",
        "Id": "a1b2c3d4e5f6...",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Containers": {
            "container-id-1": {
                "Name": "web-server",
                "IPv4Address": "172.18.0.2/16"
            },
            "container-id-2": {
                "Name": "app-server",
                "IPv4Address": "172.18.0.3/16"
            }
        }
    }
]
```

### Advantages of Bridge Network

- **Isolation**: Containers are isolated from the host and from containers on other networks.
- **DNS Service Discovery** (user-defined bridges): Containers can find each other by name.
- **Port Control**: Only explicitly published ports are accessible from outside.
- **Simple Setup**: Default network requires no configuration.
- **Custom Subnets**: User-defined bridges allow full control over IP addressing.

### Limitations of Bridge Network

- **Single Host Only**: Bridge networks work only on a single Docker host — no cross-host communication.
- **NAT Overhead**: Network Address Translation adds a small performance overhead.
- **No DNS on Default Bridge**: The default `docker0` bridge does not support DNS-based discovery.
- **Port Conflicts**: Multiple containers cannot bind to the same host port.

### Use Cases

- Development and testing environments
- Running multiple related services on a single host (e.g., web server + database)
- Microservices that need isolated but interconnected networking
- CI/CD pipelines running on a single machine

---

## 2. Host Network

### What is Host Network?

The **host network** driver removes the network isolation between the Docker container and the Docker host. When a container uses the host network, it directly uses the host machine's network stack. The container does not get its own IP address — instead, it shares the host's IP address and ports.

### How Host Network Works

```
┌──────────────────────────────────────────────────────┐
│                    Docker Host                        │
│                  IP: 192.168.1.100                    │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │              Host Network Namespace              │ │
│  │                                                   │ │
│  │  ┌────────────┐  ┌────────────┐                 │ │
│  │  │ Container1 │  │ Container2 │                 │ │
│  │  │ Port: 80   │  │ Port: 3000 │                 │ │
│  │  └────────────┘  └────────────┘                 │ │
│  │                                                   │ │
│  │  Host eth0: 192.168.1.100                        │ │
│  │  (Shared by host and all host-network containers)│ │
│  └──────────────────────┬──────────────────────────┘ │
└─────────────────────────┼────────────────────────────┘
                          │
                    External Network
```

**Key behavior:**

1. The container does **not** get its own network namespace.
2. The container shares the host's network interfaces, IP address, and port space.
3. There is **no NAT** (Network Address Translation) — traffic goes directly through the host's network stack.
4. The container can bind to any port on the host directly.
5. If the container runs a service on port 80, it is immediately accessible at `host-ip:80` without port mapping.

### Using Host Network

```bash
# Run a container with host networking
$ docker run -d --name my-nginx --network host nginx

# The nginx server is now accessible at:
# http://<host-ip>:80  (no port mapping needed)

# Verify — the container shares the host's IP
$ docker exec my-nginx hostname -I
# Output: 192.168.1.100  (same as host IP)

# Compare with bridge network container:
$ docker run -d --name my-nginx-bridge nginx
$ docker exec my-nginx-bridge hostname -I
# Output: 172.17.0.2  (different, container-specific IP)
```

### Important Notes on Host Network

```bash
# The -p (port mapping) flag is IGNORED with host network
$ docker run -d --name my-app --network host -p 8080:80 nginx
# WARNING: Published ports are discarded when using host network mode
# The container will still listen on port 80 (not 8080)
```

### Inspecting Host Network

```bash
$ docker network inspect host
[
    {
        "Name": "host",
        "Id": "host-network-id...",
        "Scope": "local",
        "Driver": "host",
        "IPAM": {
            "Config": []
        },
        "Containers": {
            "container-id": {
                "Name": "my-nginx",
                "IPv4Address": "",
                "IPv6Address": ""
            }
        }
    }
]
# Note: No IP address is listed because the container shares the host's IP
```

### Performance Benefits

The host network driver provides the **best network performance** because:

1. **No NAT overhead**: Packets do not go through Docker's NAT layer.
2. **No bridge overhead**: There is no virtual bridge to traverse.
3. **No veth pair overhead**: No virtual Ethernet interfaces are created.
4. **Direct access**: The container directly uses the host's network interfaces.

This makes the host network ideal for **network-intensive applications** where every millisecond of latency matters.

### Advantages of Host Network

- **Maximum Performance**: No network virtualization overhead (no NAT, no bridge, no veth pairs).
- **Simplified Networking**: No need for port mapping — services are directly accessible.
- **Full Network Access**: Container has full access to host's network interfaces and configuration.
- **Lower Latency**: Direct network stack access means minimal latency.
- **Service Discovery**: Container can use the host's network for service registration (e.g., mDNS).

### Limitations of Host Network

- **No Network Isolation**: The container shares the host's network, so it can see all host network traffic.
- **Port Conflicts**: Multiple containers cannot use the same port since they all share the host's port space.
- **Security Risk**: Full host network access can be a security concern in production.
- **Linux Only**: Host networking only works on Linux. On Docker Desktop (macOS/Windows), it runs inside a Linux VM, so "host" refers to the VM, not the actual host OS.
- **No DNS-based Discovery**: Unlike user-defined bridge networks, containers on host network cannot use container names for communication.
- **Not Supported in Docker Swarm Services**: Cannot be used directly as a Swarm service network mode (must use `endpoint_mode: dnsrr`).

### Use Cases

- **Performance-critical applications**: Real-time data processing, high-frequency trading systems.
- **Network monitoring tools**: Applications like Wireshark, tcpdump that need to see host network traffic.
- **Applications binding to many ports**: Services that dynamically open ports (e.g., FTP servers).
- **Development and debugging**: Quick testing without port mapping configuration.
- **Legacy applications**: Applications that expect to run directly on the host network.

---

## 3. Overlay Network

### What is an Overlay Network?

An **overlay network** creates a distributed network across multiple Docker hosts (nodes). It enables containers running on different physical or virtual machines to communicate as if they were on the same local network. Overlay networks are the **default network driver for Docker Swarm** services.

The overlay driver encapsulates container traffic inside a **VXLAN (Virtual Extensible LAN)** tunnel, which allows Layer 2 frames to be transmitted over a Layer 3 network (the underlying physical network).

### How Overlay Network Works

```
┌───────────────────────────┐        ┌───────────────────────────┐
│        Docker Host 1       │        │        Docker Host 2       │
│      (Swarm Manager)       │        │      (Swarm Worker)        │
│                             │        │                             │
│  ┌──────────┐ ┌──────────┐ │        │ ┌──────────┐ ┌──────────┐ │
│  │Container │ │Container │ │        │ │Container │ │Container │ │
│  │    A     │ │    B     │ │        │ │    C     │ │    D     │ │
│  │10.0.0.2  │ │10.0.0.3  │ │        │ │10.0.0.4  │ │10.0.0.5  │ │
│  └────┬─────┘ └────┬─────┘ │        │ └────┬─────┘ └────┬─────┘ │
│       │             │       │        │      │             │       │
│  ┌────┴─────────────┴────┐  │        │ ┌────┴─────────────┴────┐ │
│  │   Overlay Network     │  │        │ │   Overlay Network     │ │
│  │   (VXLAN Tunnel)      │  │        │ │   (VXLAN Tunnel)      │ │
│  └───────────┬───────────┘  │        │ └───────────┬───────────┘ │
│              │               │        │             │              │
│  ┌───────────┴───────────┐  │        │ ┌───────────┴───────────┐ │
│  │    Host eth0          │  │        │ │    Host eth0          │ │
│  │  192.168.1.10         │  │◄──────►│ │  192.168.1.20         │ │
│  └───────────────────────┘  │ VXLAN  │ └───────────────────────┘ │
│                              │Tunnel  │                            │
└──────────────────────────────┘        └────────────────────────────┘
                    │                              │
              Physical Network (Underlay)
```

**Step-by-step process:**

1. Docker creates a VXLAN tunnel between Docker hosts participating in the Swarm.
2. Each overlay network gets its own **subnet** (e.g., `10.0.0.0/24`).
3. Containers on the overlay network get IP addresses from this subnet, regardless of which host they run on.
4. When Container A (Host 1) sends traffic to Container C (Host 2):
   - The packet is encapsulated in a **VXLAN header**.
   - The encapsulated packet is sent over the **underlay network** (physical network) to Host 2.
   - Host 2 de-encapsulates the packet and delivers it to Container C.
5. Docker uses a **distributed key-value store** (built into Swarm) to share network state across hosts.

### VXLAN Encapsulation

VXLAN (Virtual Extensible LAN) is the tunneling technology that overlay networks use:

```
Original Container Packet:
┌──────────────────────────────────────────┐
│  Ethernet │  IP Header  │  TCP/UDP │ Data│
│  Header   │  src→dst    │  Header  │     │
└──────────────────────────────────────────┘

After VXLAN Encapsulation:
┌────────────────────────────────────────────────────────────────────┐
│ Outer    │ Outer IP   │ Outer   │ VXLAN  │ Original Container     │
│ Ethernet │ (Host IPs) │ UDP     │ Header │ Packet (as payload)    │
│ Header   │            │ Port    │ (VNI)  │                        │
│          │            │ 4789    │        │                        │
└────────────────────────────────────────────────────────────────────┘
```

- **VNI (VXLAN Network Identifier)**: A 24-bit identifier that uniquely identifies each overlay network (supports up to 16 million networks).
- **UDP Port 4789**: The default port used for VXLAN traffic.

### Prerequisites for Overlay Networks

Before creating overlay networks, the following requirements must be met:

1. **Docker Swarm Mode**: Overlay networks require Docker Swarm to be initialized.
2. **Open Ports**: The following ports must be open between Docker hosts:
   - **TCP port 2377**: Cluster management communication.
   - **TCP/UDP port 7946**: Node-to-node communication.
   - **UDP port 4789**: Overlay network (VXLAN) traffic.
3. **Key-Value Store**: Docker Swarm uses an internal distributed store (Raft consensus), but standalone overlay networks may require an external store (Consul, etcd).

### Setting Up Docker Swarm

```bash
# Initialize Swarm on the manager node
$ docker swarm init --advertise-addr 192.168.1.10

# Output includes a join token for workers:
# docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377

# On each worker node, join the swarm:
$ docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377

# Verify nodes
$ docker node ls
ID              HOSTNAME    STATUS    AVAILABILITY    MANAGER STATUS
abc123 *        manager1    Ready     Active          Leader
def456          worker1     Ready     Active
ghi789          worker2     Ready     Active
```

### Creating and Using Overlay Networks

#### Creating an Overlay Network

```bash
# Create a basic overlay network
$ docker network create --driver overlay my-overlay-network

# Create with custom configuration
$ docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  --gateway 10.0.9.1 \
  --opt encrypted \
  my-secure-overlay

# Create an attachable overlay (allows standalone containers to join)
$ docker network create --driver overlay --attachable my-attachable-overlay
```

#### Deploying Services on Overlay Network

```bash
# Create a service on the overlay network
$ docker service create \
  --name web-service \
  --network my-overlay-network \
  --replicas 3 \
  --publish published=80,target=80 \
  nginx

# Create another service on the same overlay
$ docker service create \
  --name api-service \
  --network my-overlay-network \
  --replicas 2 \
  node:18

# api-service containers can reach web-service by name:
# curl http://web-service:80
```

#### Connecting Standalone Containers (Attachable Networks)

```bash
# Create an attachable overlay network
$ docker network create --driver overlay --attachable my-attachable

# Now standalone containers can connect:
$ docker run -d --name standalone-app --network my-attachable nginx
```

### Service Discovery in Overlay Networks

Docker Swarm provides built-in service discovery for overlay networks using an **internal DNS server**:

```bash
# Containers can discover services by name
$ docker exec container-a nslookup web-service
# Server:    127.0.0.11
# Address:   127.0.0.11:53
#
# Name:      web-service
# Address:   10.0.0.5  (VIP - Virtual IP)

# Docker uses Virtual IP (VIP) load balancing by default
# All replicas of web-service are accessible through a single VIP
```

**DNS Round-Robin mode:**

```bash
# Use DNS round-robin instead of VIP
$ docker service create \
  --name web-service \
  --network my-overlay-network \
  --endpoint-mode dnsrr \
  --replicas 3 \
  nginx

# Now DNS returns all replica IPs instead of a single VIP
```

### Overlay Network Encryption

By default, overlay network traffic is **not encrypted**. You can enable encryption to protect data in transit:

```bash
# Create an encrypted overlay network
$ docker network create \
  --driver overlay \
  --opt encrypted \
  my-encrypted-overlay
```

When encryption is enabled:
- Docker uses **IPsec (AES-GCM)** to encrypt all traffic between nodes.
- Encryption keys are automatically managed and rotated by the Swarm manager.
- There is a **performance overhead** due to encryption/decryption.

### Inspecting Overlay Networks

```bash
$ docker network inspect my-overlay-network
[
    {
        "Name": "my-overlay-network",
        "Id": "overlay-id...",
        "Scope": "swarm",
        "Driver": "overlay",
        "IPAM": {
            "Config": [
                {
                    "Subnet": "10.0.1.0/24",
                    "Gateway": "10.0.1.1"
                }
            ]
        },
        "Peers": [
            {
                "Name": "manager1",
                "IP": "192.168.1.10"
            },
            {
                "Name": "worker1",
                "IP": "192.168.1.20"
            }
        ],
        "Containers": {
            "container-id-1": {
                "Name": "web-service.1.task-id",
                "IPv4Address": "10.0.1.3/24"
            },
            "container-id-2": {
                "Name": "web-service.2.task-id",
                "IPv4Address": "10.0.1.4/24"
            }
        }
    }
]
```

### Ingress Network

Docker Swarm automatically creates a special overlay network called **ingress**:

```bash
$ docker network ls --filter driver=overlay
NETWORK ID     NAME      DRIVER    SCOPE
abc123def456   ingress   overlay   swarm
```

The **ingress network** handles:

- **Routing Mesh**: Incoming traffic on any Swarm node (even nodes not running the service) is routed to the correct container.
- **Load Balancing**: Traffic is distributed across all replicas of a service.

```
External Request → Any Swarm Node:80
                         │
              ┌──────────┴──────────┐
              │    Ingress Network   │
              │   (Routing Mesh)     │
              └──┬───────┬───────┬──┘
                 │       │       │
           Replica 1  Replica 2  Replica 3
           (Host 1)  (Host 2)   (Host 3)
```

### Advantages of Overlay Network

- **Multi-Host Networking**: Containers on different hosts communicate seamlessly.
- **Service Discovery**: Built-in DNS resolves service names to IPs automatically.
- **Load Balancing**: Automatic load balancing across service replicas.
- **Encryption Support**: Optional IPsec encryption for secure communication.
- **Routing Mesh**: Any Swarm node can accept traffic for any service.
- **Transparent to Containers**: Containers see a flat network — they don't know about the VXLAN tunneling.
- **Scalable**: Can span across data centers and cloud providers.

### Limitations of Overlay Network

- **Requires Docker Swarm**: Must initialize Swarm mode (unless using standalone with `--attachable`).
- **VXLAN Overhead**: Encapsulation adds approximately 50 bytes of overhead per packet.
- **Performance**: Slightly lower throughput compared to bridge or host networks due to encapsulation.
- **Complexity**: More complex to debug than bridge networks.
- **Port Requirements**: Requires specific ports (2377, 7946, 4789) to be open between hosts.
- **MTU Considerations**: VXLAN encapsulation may require MTU adjustments to avoid fragmentation.

### Use Cases

- **Microservices Architecture**: Multiple services running across a cluster of Docker hosts.
- **Docker Swarm Deployments**: Default networking for Swarm-based services.
- **Multi-Host Communication**: Any scenario requiring containers on different hosts to communicate.
- **Distributed Applications**: Databases, message queues, and application servers spread across hosts.
- **Blue-Green Deployments**: Run two versions of a service on different hosts with shared networking.
- **Hybrid/Multi-Cloud Deployments**: Connect containers across different cloud providers.

---

## Comparison of Network Types

| Feature                     | Bridge                        | Host                         | Overlay                              |
|-----------------------------|-------------------------------|------------------------------|--------------------------------------|
| **Scope**                   | Single host                   | Single host                  | Multi-host (Swarm)                   |
| **Isolation**               | High (own network namespace)  | None (shares host namespace) | High (VXLAN encapsulation)           |
| **Performance**             | Good (minor NAT overhead)     | Best (no overhead)           | Good (VXLAN overhead)                |
| **Port Mapping**            | Required (`-p` flag)          | Not needed                   | Handled by routing mesh              |
| **DNS Service Discovery**   | Yes (user-defined bridges)    | No                           | Yes (built-in)                       |
| **IP Address**              | Container gets its own IP     | Shares host IP               | Container gets overlay IP            |
| **Multi-Host Support**      | No                            | No                           | Yes                                  |
| **Docker Swarm Required**   | No                            | No                           | Yes (or `--attachable` for standalone) |
| **Encryption**              | Not applicable                | Not applicable               | Optional (IPsec)                     |
| **Load Balancing**          | Manual (external)             | Manual (external)            | Built-in (VIP or DNSRR)             |
| **Best For**                | Single-host development       | Performance-critical apps    | Production multi-host clusters       |
| **Security**                | Good (isolated)               | Low (shared with host)       | Good (isolated + optional encryption)|
| **Complexity**              | Low                           | Low                          | Medium-High                          |

---

## Docker Networking Commands

### Network Management Commands

```bash
# List all networks
$ docker network ls

# Create a network
$ docker network create [OPTIONS] NETWORK_NAME

# Inspect a network (detailed info)
$ docker network inspect NETWORK_NAME

# Remove a network
$ docker network rm NETWORK_NAME

# Remove all unused networks
$ docker network prune

# Connect a container to a network
$ docker network connect NETWORK_NAME CONTAINER_NAME

# Disconnect a container from a network
$ docker network disconnect NETWORK_NAME CONTAINER_NAME
```

### Useful Inspection Commands

```bash
# List networks with custom format
$ docker network ls --format "table {{.ID}}\t{{.Name}}\t{{.Driver}}\t{{.Scope}}"

# Filter networks by driver
$ docker network ls --filter driver=bridge

# See which containers are on a network
$ docker network inspect --format '{{range .Containers}}{{.Name}} {{end}}' NETWORK_NAME

# See a container's network settings
$ docker inspect --format '{{json .NetworkSettings.Networks}}' CONTAINER_NAME

# Check a container's IP address
$ docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' CONTAINER_NAME
```

### Troubleshooting Commands

```bash
# Test connectivity between containers
$ docker exec container1 ping container2

# Check DNS resolution
$ docker exec container1 nslookup service-name

# View network interfaces inside a container
$ docker exec container1 ip addr show

# Check routing table inside a container
$ docker exec container1 ip route

# Monitor network traffic (using a debug container)
$ docker run --rm --net container:target-container nicolaka/netshoot tcpdump -i eth0

# Check iptables rules on the host (for bridge networks)
$ sudo iptables -t nat -L -n -v
```

---

## Best Practices

### General Networking Best Practices

1. **Always use user-defined bridge networks** instead of the default `docker0` bridge for better DNS support and isolation.
2. **Use overlay networks** for multi-host communication in production Swarm deployments.
3. **Avoid host network** in production unless performance is absolutely critical and security trade-offs are acceptable.
4. **Enable encryption** on overlay networks when handling sensitive data across hosts.
5. **Limit exposed ports** — only publish the ports that external clients need to access.

### Security Best Practices

1. **Segment networks**: Place different application tiers (frontend, backend, database) on separate networks.
2. **Use internal networks** for backend services that don't need external access:
   ```bash
   $ docker network create --internal backend-network
   ```
3. **Restrict inter-container communication** on the default bridge:
   ```bash
   # In Docker daemon config (/etc/docker/daemon.json):
   { "icc": false }
   ```
4. **Use network aliases** to decouple container names from service names:
   ```bash
   $ docker run --network my-net --network-alias db postgres
   ```

### Performance Best Practices

1. **Use host network** for latency-sensitive or high-throughput applications.
2. **Set appropriate MTU values** for overlay networks to avoid fragmentation:
   ```bash
   $ docker network create --driver overlay --opt com.docker.network.driver.mtu=1400 my-overlay
   ```
3. **Monitor network performance** using tools like `iperf3` between containers.
4. **Place communicating containers on the same host** when possible to avoid overlay overhead.

---

## Summary

Docker networking provides flexible options for connecting containers:

- **Bridge Network**: The default choice for single-host setups. Provides isolation, port mapping, and DNS discovery (on user-defined bridges). Best for development and single-host production.

- **Host Network**: Removes all network isolation for maximum performance. The container shares the host's network stack directly. Best for performance-critical and network-monitoring applications.

- **Overlay Network**: Enables multi-host networking through VXLAN tunneling. Provides built-in service discovery, load balancing, and optional encryption. Essential for Docker Swarm and distributed production deployments.

Choosing the right network type depends on your specific requirements for **isolation**, **performance**, **security**, and **scale**. For most production deployments, a combination of user-defined bridge networks (for single-host communication) and overlay networks (for cross-host communication) provides the best balance of features and performance.
