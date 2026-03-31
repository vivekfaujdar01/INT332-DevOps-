# DNS Inside Docker, Container Linking & Port Mapping

## Table of Contents

1. [DNS Inside Docker](#dns-inside-docker)
2. [Linking Containers](#linking-containers)
3. [Port Mapping Between Host and Container](#port-mapping-between-host-and-container)
4. [Practical Examples](#practical-examples)
5. [Summary](#summary)

---

## DNS Inside Docker

### What is DNS?

**DNS (Domain Name System)** is a system that translates human-readable domain names (like `web-server` or `google.com`) into IP addresses (like `172.18.0.2` or `142.250.190.46`). Without DNS, you would need to remember the IP address of every service you want to connect to.

In Docker, DNS plays a critical role in enabling **service discovery** — the ability for containers to find and communicate with each other **by name** instead of by IP address.

### Why DNS Matters in Docker

Container IP addresses are **dynamic and unpredictable**. Every time a container is created, destroyed, or restarted, it may receive a different IP address. Hardcoding IP addresses would make container communication fragile and unreliable.

DNS solves this by allowing containers to refer to each other using **stable names**, while Docker's internal DNS server automatically resolves those names to the correct, current IP addresses.

```
Without DNS:
  Container A → must know 172.18.0.5 → Container B
  (If Container B restarts, IP may change to 172.18.0.9 — connection breaks!)

With DNS:
  Container A → resolves "container-b" → always gets current IP → Container B
  (If Container B restarts, DNS returns the new IP — connection works!)
```

### Docker's Embedded DNS Server

Docker runs an **embedded DNS server** inside every container at the IP address **`127.0.0.11`**. This DNS server is automatically configured in every container's `/etc/resolv.conf` file.

```bash
# Check the DNS configuration inside a container
$ docker run --rm alpine cat /etc/resolv.conf
nameserver 127.0.0.11
options ndots:0
```

This embedded DNS server handles name resolution in the following order:

1. **Container Name Resolution**: Resolves container names and network aliases to their IP addresses within the same Docker network.
2. **Service Name Resolution** (Swarm mode): Resolves service names to the Virtual IP (VIP) or individual container IPs.
3. **External DNS Forwarding**: If the name is not a Docker-managed name, the query is forwarded to the host's configured external DNS servers (e.g., `8.8.8.8`, `1.1.1.1`).

```
┌─────────────────────────────────────────────────────────────┐
│                        Container                             │
│                                                              │
│  Application → DNS Query ("web-server")                      │
│                      │                                       │
│                      ▼                                       │
│         ┌────────────────────────┐                           │
│         │  Embedded DNS Server   │                           │
│         │     127.0.0.11         │                           │
│         └────────┬───────────────┘                           │
│                  │                                            │
└──────────────────┼───────────────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         │                    │
    Docker Name?         Not Docker Name?
         │                    │
         ▼                    ▼
  ┌──────────────┐   ┌────────────────┐
  │ Resolve from  │   │ Forward to     │
  │ Docker's      │   │ Host's External│
  │ internal      │   │ DNS server     │
  │ name store    │   │ (8.8.8.8 etc.) │
  └──────────────┘   └────────────────┘
```

### DNS on Default Bridge vs. User-Defined Bridge

One of the most important distinctions in Docker networking is the DNS behavior between the **default bridge** (`docker0`) and **user-defined bridge** networks:

#### Default Bridge Network — No Automatic DNS

The default `docker0` bridge does **not** provide automatic DNS resolution. Containers on the default bridge cannot resolve each other's names.

```bash
# Start two containers on the default bridge
$ docker run -d --name container-a alpine sleep 3600
$ docker run -d --name container-b alpine sleep 3600

# Try to ping by name — FAILS
$ docker exec container-a ping container-b
# ping: bad address 'container-b'

# Must use IP address instead
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container-b
# 172.17.0.3

$ docker exec container-a ping 172.17.0.3
# PING 172.17.0.3: 56 data bytes
# 64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.089 ms
```

#### User-Defined Bridge Network — Automatic DNS ✓

User-defined bridge networks provide **automatic DNS resolution**. Containers can reach each other by name.

```bash
# Create a user-defined bridge network
$ docker network create my-app-network

# Start two containers on the user-defined network
$ docker run -d --name web-server --network my-app-network nginx
$ docker run -d --name app-server --network my-app-network alpine sleep 3600

# Ping by name — WORKS
$ docker exec app-server ping web-server
# PING web-server (172.19.0.2): 56 data bytes
# 64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.076 ms

# DNS lookup
$ docker exec app-server nslookup web-server
# Server:    127.0.0.11
# Address:   127.0.0.11:53
#
# Non-authoritative answer:
# Name:      web-server
# Address:   172.19.0.2
```

### Network Aliases

Docker allows you to assign **network aliases** to containers. An alias acts as an additional DNS name that resolves to the container's IP address. Multiple containers can share the same alias, in which case Docker performs **DNS round-robin** across them.

```bash
# Run a container with a network alias
$ docker run -d \
  --name postgres-primary \
  --network my-app-network \
  --network-alias db \
  postgres:15

# Another container can reach it using either name:
$ docker exec app-server ping postgres-primary   # works
$ docker exec app-server ping db                  # also works

# Multiple containers can share the same alias (round-robin DNS)
$ docker run -d \
  --name postgres-replica \
  --network my-app-network \
  --network-alias db \
  postgres:15

# Now "db" resolves to BOTH containers (round-robin)
$ docker exec app-server nslookup db
# Name:      db
# Address:   172.19.0.3
# Address:   172.19.0.4
```

### Configuring Custom DNS Settings

Docker allows you to override DNS settings at both the **daemon level** and the **container level**.

#### Container-Level DNS Configuration

```bash
# Set a custom DNS server for a container
$ docker run -d --name my-app --dns 8.8.8.8 nginx

# Set multiple DNS servers
$ docker run -d --name my-app --dns 8.8.8.8 --dns 8.8.4.4 nginx

# Set a DNS search domain
$ docker run -d --name my-app --dns-search example.com nginx
# Now "api" automatically resolves as "api.example.com"

# Set a custom hostname
$ docker run -d --name my-app --hostname my-custom-host nginx

# Add a manual host-to-IP mapping (like /etc/hosts)
$ docker run -d --name my-app --add-host db-server:10.0.0.50 nginx

# Verify inside the container
$ docker exec my-app cat /etc/hosts
# 127.0.0.1       localhost
# 10.0.0.50       db-server
# 172.19.0.5      my-custom-host
```

#### Daemon-Level DNS Configuration

You can set default DNS servers for all containers in the Docker daemon configuration file `/etc/docker/daemon.json`:

```json
{
  "dns": ["8.8.8.8", "8.8.4.4"],
  "dns-search": ["example.com"],
  "dns-opts": ["ndots:5"]
}
```

```bash
# After editing daemon.json, restart Docker
$ sudo systemctl restart docker
```

### DNS Resolution Flow (Complete)

```
Container makes a DNS query for "web-server"
                    │
                    ▼
     ┌──────────────────────────┐
     │  Step 1: Check /etc/hosts │
     │  (manual --add-host       │
     │   entries checked first)  │
     └─────────────┬────────────┘
                   │ Not found
                   ▼
     ┌──────────────────────────────┐
     │  Step 2: Query Embedded DNS   │
     │  Server at 127.0.0.11        │
     │                               │
     │  Is "web-server" a container  │
     │  name or alias on the same    │
     │  Docker network?              │
     └─────────┬─────────┬──────────┘
               │ Yes     │ No
               ▼         ▼
     ┌────────────┐  ┌──────────────────────┐
     │ Return     │  │ Step 3: Forward to    │
     │ container  │  │ external DNS servers  │
     │ IP address │  │ (from --dns flag or   │
     │            │  │  host's resolv.conf)  │
     └────────────┘  └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────┐
                     │ Return resolved   │
                     │ IP or NXDOMAIN   │
                     │ (not found)      │
                     └──────────────────┘
```

### DNS in Docker Compose

Docker Compose automatically creates a user-defined bridge network for each project, so all services defined in the same `docker-compose.yml` can resolve each other by **service name**.

```yaml
# docker-compose.yml
version: "3.8"

services:
  web:
    image: nginx
    ports:
      - "8080:80"

  api:
    image: node:18
    environment:
      - DB_HOST=database    # Uses DNS name to connect

  database:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
```

In this setup:
- `web` can reach `api` and `database` by name.
- `api` can reach `web` and `database` by name.
- `database` can reach `web` and `api` by name.

Docker Compose creates a network named `<project-name>_default`, and all three services are automatically placed on it with full DNS resolution.

---

## Linking Containers

### What is Container Linking?

**Container linking** is a mechanism that allows containers to discover and communicate with each other. Docker provides two approaches:

1. **Legacy Linking** (`--link` flag) — An older method that injects environment variables and `/etc/hosts` entries. **Deprecated** but still supported.
2. **Modern Linking** (User-defined networks + DNS) — The recommended approach using Docker networks with automatic DNS-based service discovery.

### Legacy Linking (`--link`)

#### How `--link` Works

The `--link` flag creates a unidirectional connection from one container (the source) to another (the target). When you link Container A to Container B, Docker:

1. Adds an entry to Container A's `/etc/hosts` file mapping Container B's name to its IP address.
2. Injects **environment variables** into Container A with information about Container B's exposed ports and IP address.
3. Allows Container A to communicate with Container B, but **not** the reverse (unless you link B to A separately).

```
┌──────────────────────────────────────────────────────────┐
│                     Docker Host                           │
│                                                           │
│  ┌──────────────┐         ┌──────────────┐               │
│  │  Container A  │ --link→ │  Container B  │              │
│  │  (web)        │         │  (database)   │              │
│  │               │         │               │              │
│  │ /etc/hosts:   │         │               │              │
│  │ 172.17.0.3 db │         │ IP:172.17.0.3 │              │
│  │               │         │ Port: 5432    │              │
│  │ Env Variables:│         │               │              │
│  │ DB_PORT=5432  │         │               │              │
│  │ DB_ADDR=...   │         │               │              │
│  └──────────────┘         └──────────────┘               │
│                                                           │
│         A can reach B ✓    B cannot reach A ✗             │
└──────────────────────────────────────────────────────────┘
```

#### Using `--link`

```bash
# Step 1: Start the target container (the one being linked TO)
$ docker run -d --name my-database postgres:15

# Step 2: Start the source container and link it to the target
$ docker run -d --name my-web --link my-database:db nginx
#                                    ↑             ↑
#                              target container   alias
```

The syntax is `--link <container-name>:<alias>`, where:
- `<container-name>` is the name of the target container.
- `<alias>` is the name used to refer to the target inside the source container.

#### What `--link` Provides

**1. `/etc/hosts` Entry:**

```bash
$ docker exec my-web cat /etc/hosts
# 127.0.0.1       localhost
# 172.17.0.3      db a1b2c3d4e5f6 my-database
# 172.17.0.4      my-web
```

The target container gets multiple entries: its alias (`db`), its container ID (`a1b2c3d4e5f6`), and its container name (`my-database`).

**2. Environment Variables:**

```bash
$ docker exec my-web env | grep DB_
# DB_PORT=tcp://172.17.0.3:5432
# DB_PORT_5432_TCP=tcp://172.17.0.3:5432
# DB_PORT_5432_TCP_ADDR=172.17.0.3
# DB_PORT_5432_TCP_PORT=5432
# DB_PORT_5432_TCP_PROTO=tcp
# DB_NAME=/my-web/db
# DB_ENV_POSTGRES_PASSWORD=secret
```

The environment variables follow the pattern:
- `<ALIAS>_PORT` — The full URL of the first exposed port.
- `<ALIAS>_PORT_<port>_TCP_ADDR` — The IP address of the linked container.
- `<ALIAS>_PORT_<port>_TCP_PORT` — The port number of the exposed port.
- `<ALIAS>_PORT_<port>_TCP_PROTO` — The protocol (TCP or UDP).
- `<ALIAS>_ENV_<variable>` — Any environment variables set in the target container.

**3. Connectivity:**

```bash
# The web container can now reach the database by alias
$ docker exec my-web ping db
# PING db (172.17.0.3): 56 data bytes
# 64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.085 ms

# Or by container name
$ docker exec my-web ping my-database
# PING my-database (172.17.0.3): 56 data bytes
# 64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.079 ms
```

#### Limitations of Legacy Linking

| Limitation | Description |
|---|---|
| **Deprecated** | Docker officially discourages `--link` in favor of user-defined networks. |
| **Default bridge only** | `--link` only works on the default `docker0` bridge, not on user-defined networks. |
| **Unidirectional** | Linking is one-way. A→B does not mean B→A. |
| **No dynamic updates** | If the target container is restarted and gets a new IP, the `/etc/hosts` entry in the source is **not updated**. |
| **Startup order dependency** | The target container must be started **before** the source container. |
| **Cannot link across hosts** | Works only on a single Docker host. |
| **No network isolation** | All linked containers share the default bridge with no segmentation. |

### Modern Linking — User-Defined Networks (Recommended)

The modern and recommended way to connect containers is through **user-defined networks** with **automatic DNS resolution**. This approach replaces `--link` entirely.

#### How Modern Linking Works

```
┌──────────────────────────────────────────────────────────────┐
│                         Docker Host                           │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐│
│  │              User-Defined Bridge Network                  ││
│  │              (Embedded DNS: 127.0.0.11)                   ││
│  │                                                           ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   ││
│  │  │   web        │  │   api        │  │   database   │   ││
│  │  │  172.19.0.2  │  │  172.19.0.3  │  │  172.19.0.4  │   ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘   ││
│  │                                                           ││
│  │  web ↔ api ↔ database  (all can reach each other by name)││
│  └──────────────────────────────────────────────────────────┘│
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

#### Using User-Defined Networks

```bash
# Step 1: Create a user-defined network
$ docker network create app-network

# Step 2: Run containers on the same network
$ docker run -d --name database --network app-network postgres:15
$ docker run -d --name api --network app-network node:18
$ docker run -d --name web --network app-network nginx

# Step 3: Containers can now communicate by name (bidirectional)
$ docker exec web ping api
# PING api (172.20.0.3): 56 data bytes — WORKS ✓

$ docker exec api ping database
# PING database (172.20.0.4): 56 data bytes — WORKS ✓

$ docker exec database ping web
# PING web (172.20.0.2): 56 data bytes — WORKS ✓
```

#### Advantages Over Legacy Linking

| Feature | `--link` (Legacy) | User-Defined Networks (Modern) |
|---|---|---|
| **DNS resolution** | `/etc/hosts` only | Full embedded DNS server |
| **Direction** | Unidirectional | Bidirectional |
| **Dynamic IP updates** | No (stale on restart) | Yes (DNS always current) |
| **Startup order** | Target must start first | No order requirement |
| **Network isolation** | No (shared default bridge) | Yes (separate networks) |
| **Multiple networks** | Not supported | Container can join many |
| **Aliases** | Single alias per link | Multiple aliases per container |
| **Cross-host support** | No | Yes (with overlay networks) |
| **Connect/disconnect live** | No | Yes (`docker network connect/disconnect`) |
| **Docker Compose support** | Deprecated field | Native support |

#### Connecting Containers to Multiple Networks

A single container can be connected to **multiple networks**, enabling it to act as a bridge between isolated groups:

```bash
# Create two separate networks
$ docker network create frontend-network
$ docker network create backend-network

# Run a database on the backend-only network
$ docker run -d --name database --network backend-network postgres:15

# Run the API server on BOTH networks
$ docker run -d --name api --network backend-network node:18
$ docker network connect frontend-network api

# Run the web server on the frontend-only network
$ docker run -d --name web --network frontend-network nginx
```

```
┌──────────────────────────────────────────────────────────────────┐
│                           Docker Host                             │
│                                                                   │
│   Frontend Network                    Backend Network             │
│  ┌────────────────────────┐    ┌────────────────────────────┐    │
│  │  ┌──────┐  ┌────────┐ │    │ ┌────────┐  ┌────────────┐ │    │
│  │  │ web  │  │  api   │ │    │ │  api   │  │  database  │ │    │
│  │  │      │  │(also   │ │    │ │(also   │  │            │ │    │
│  │  │      │  │ here)  │ │    │ │ here)  │  │            │ │    │
│  │  └──────┘  └────────┘ │    │ └────────┘  └────────────┘ │    │
│  └────────────────────────┘    └────────────────────────────┘    │
│                                                                   │
│   web → api  ✓ (frontend)      api → database  ✓ (backend)      │
│   web → database  ✗            database → web  ✗                 │
│                                                                   │
│   Result: "api" acts as a gateway between the two networks       │
└──────────────────────────────────────────────────────────────────┘
```

In this setup:
- `web` can reach `api` (they share the frontend network).
- `api` can reach `database` (they share the backend network).
- `web` **cannot** reach `database` directly (they are on separate networks).
- This provides **network segmentation** and improved security.

---

## Port Mapping Between Host and Container

### What is Port Mapping?

**Port mapping** (also called **port publishing** or **port forwarding**) is the mechanism that makes a containerized application accessible from outside the Docker host. By default, containers are isolated — their internal ports are not accessible from the host's network. Port mapping creates a connection between a port on the **host machine** and a port inside the **container**.

```
Without Port Mapping:
  External User → Host:8080 → ✗ (nothing listening)
  Container runs nginx on port 80, but nobody outside can reach it.

With Port Mapping (-p 8080:80):
  External User → Host:8080 → Docker NAT → Container:80 → nginx responds ✓
```

### How Port Mapping Works Internally

When you publish a port, Docker uses **iptables** (Linux firewall/NAT rules) to redirect traffic from the host port to the container port:

```
┌──────────────────────────────────────────────────────────────┐
│                        Docker Host                            │
│                     Host IP: 192.168.1.100                    │
│                                                               │
│  External Request → 192.168.1.100:8080                       │
│                          │                                    │
│                          ▼                                    │
│              ┌────────────────────────┐                       │
│              │    iptables NAT Rules   │                      │
│              │                         │                      │
│              │  DNAT: 8080 → 172.17.0.2:80                   │
│              └───────────┬────────────┘                       │
│                          │                                    │
│                          ▼                                    │
│              ┌────────────────────────┐                       │
│              │   docker-proxy process  │                      │
│              │  (userland proxy on     │                      │
│              │   host port 8080)       │                      │
│              └───────────┬────────────┘                       │
│                          │                                    │
│                          ▼                                    │
│              ┌─────────────────────┐                          │
│              │      Container       │                         │
│              │   IP: 172.17.0.2     │                         │
│              │   nginx on port 80   │                         │
│              └─────────────────────┘                          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

Docker sets up two paths for traffic forwarding:
1. **iptables DNAT rules**: Kernel-level packet rewriting for high-performance forwarding.
2. **docker-proxy**: A userland process that listens on the host port and forwards traffic. Used as a fallback, especially for connections originating from the host itself.

### The `-p` Flag — Publishing Ports

The `-p` (or `--publish`) flag is used to map ports when running a container.

#### Syntax

```bash
$ docker run -p [host_ip:]<host_port>:<container_port>[/protocol] IMAGE
```

| Component | Description | Required |
|---|---|---|
| `host_ip` | Host interface to bind to (default: `0.0.0.0` = all interfaces) | Optional |
| `host_port` | Port number on the host machine | Yes |
| `container_port` | Port number inside the container | Yes |
| `protocol` | `tcp` or `udp` (default: `tcp`) | Optional |

#### Basic Port Mapping Examples

```bash
# Map host port 8080 to container port 80
$ docker run -d --name web -p 8080:80 nginx
# Access at: http://localhost:8080

# Map host port 3307 to container port 3306
$ docker run -d --name mysql-db -p 3307:3306 mysql:8
# Connect at: localhost:3307

# Map host port 6380 to container port 6379
$ docker run -d --name redis-cache -p 6380:6379 redis:7
# Connect at: localhost:6380

# Same port on host and container
$ docker run -d --name web -p 80:80 nginx
# Access at: http://localhost:80
```

#### Binding to a Specific Host Interface

By default, Docker binds to **all network interfaces** (`0.0.0.0`), making the port accessible from anywhere that can reach the host. You can restrict this by specifying a host IP:

```bash
# Bind to localhost only (accessible only from the host itself)
$ docker run -d --name web -p 127.0.0.1:8080:80 nginx
# Accessible at:    http://127.0.0.1:8080      ✓
# NOT accessible at: http://192.168.1.100:8080  ✗

# Bind to a specific network interface
$ docker run -d --name web -p 192.168.1.100:8080:80 nginx
# Accessible only from the 192.168.1.100 interface

# Bind to all interfaces (default behavior, explicit)
$ docker run -d --name web -p 0.0.0.0:8080:80 nginx
```

#### Random Host Port Assignment

If you don't specify a host port, Docker assigns a **random available port** from the ephemeral port range (usually 32768–60999):

```bash
# Let Docker choose the host port
$ docker run -d --name web -p 80 nginx

# Find the assigned port
$ docker port web
# 80/tcp -> 0.0.0.0:32769

# Or use docker ps
$ docker ps
# PORTS                   NAMES
# 0.0.0.0:32769->80/tcp   web
```

#### The `-P` Flag — Publish All Exposed Ports

The uppercase `-P` (or `--publish-all`) flag automatically publishes **all ports** that are declared with the `EXPOSE` instruction in the Dockerfile:

```bash
# Nginx Dockerfile contains: EXPOSE 80
$ docker run -d --name web -P nginx

$ docker port web
# 80/tcp -> 0.0.0.0:32770
```

The ports are mapped to random host ports.

#### Multiple Port Mappings

A single container can have **multiple port mappings**:

```bash
# Map multiple ports
$ docker run -d --name my-app \
  -p 8080:80 \
  -p 8443:443 \
  -p 9090:9090 \
  my-web-server

# Verify all mappings
$ docker port my-app
# 80/tcp  -> 0.0.0.0:8080
# 443/tcp -> 0.0.0.0:8443
# 9090/tcp -> 0.0.0.0:9090
```

#### UDP Port Mapping

By default, port mapping uses **TCP**. To map UDP ports, append `/udp`:

```bash
# Map a UDP port
$ docker run -d --name dns-server -p 5353:53/udp my-dns-server

# Map both TCP and UDP on the same port
$ docker run -d --name dns-server \
  -p 5353:53/tcp \
  -p 5353:53/udp \
  my-dns-server
```

### EXPOSE vs. `-p`: Understanding the Difference

A common point of confusion is the difference between the `EXPOSE` instruction in a Dockerfile and the `-p` flag at runtime:

| Feature | `EXPOSE` (Dockerfile) | `-p` (Runtime) |
|---|---|---|
| **Purpose** | Documentation — declares which ports the container listens on | Actually publishes ports and makes them accessible |
| **Effect** | Does **not** open or publish any port | Creates a real port mapping on the host |
| **When used** | At image build time (`docker build`) | At container run time (`docker run`) |
| **Accessibility** | Container port is NOT accessible from host | Container port IS accessible from host |
| **Required for `-p`?** | No — you can map any port with `-p`, even without `EXPOSE` | N/A |

```dockerfile
# In Dockerfile:
EXPOSE 80          # Just documentation — port 80 is NOT accessible yet

# At runtime:
$ docker run -p 8080:80 myimage   # NOW port 80 is accessible via host:8080
```

### Port Mapping in Docker Compose

Docker Compose uses the `ports` key to define port mappings:

```yaml
# docker-compose.yml
version: "3.8"

services:
  web:
    image: nginx
    ports:
      - "8080:80"              # host:container
      - "8443:443"             # HTTPS

  api:
    image: node:18
    ports:
      - "3000:3000"            # Same port

  database:
    image: postgres:15
    ports:
      - "127.0.0.1:5433:5432"  # Localhost only, custom host port
    environment:
      - POSTGRES_PASSWORD=secret

  redis:
    image: redis:7
    ports:
      - "6379"                  # Random host port
```

#### Long Syntax in Docker Compose

Docker Compose also supports a more explicit **long syntax** for port mappings:

```yaml
services:
  web:
    image: nginx
    ports:
      - target: 80          # Container port
        published: 8080      # Host port
        protocol: tcp        # Protocol
        mode: host           # "host" or "ingress" (Swarm)
```

### Port Conflicts and How to Handle Them

A port on the host can only be used by **one container** at a time. If you try to bind two containers to the same host port, the second one will fail:

```bash
$ docker run -d --name web1 -p 8080:80 nginx
# Success ✓

$ docker run -d --name web2 -p 8080:80 nginx
# Error: Bind for 0.0.0.0:8080 failed: port is already allocated ✗
```

**Solutions:**

```bash
# Solution 1: Use different host ports
$ docker run -d --name web1 -p 8080:80 nginx
$ docker run -d --name web2 -p 8081:80 nginx

# Solution 2: Bind to different interfaces
$ docker run -d --name web1 -p 127.0.0.1:8080:80 nginx
$ docker run -d --name web2 -p 192.168.1.100:8080:80 nginx

# Solution 3: Let Docker assign random ports
$ docker run -d --name web1 -p 80 nginx
$ docker run -d --name web2 -p 80 nginx
```

### Viewing Port Mappings

```bash
# View ports for a specific container
$ docker port my-container
# 80/tcp -> 0.0.0.0:8080

# View ports in the container list
$ docker ps --format "table {{.Names}}\t{{.Ports}}"
# NAMES       PORTS
# web         0.0.0.0:8080->80/tcp
# api         0.0.0.0:3000->3000/tcp
# database    127.0.0.1:5433->5432/tcp

# Detailed port info from inspect
$ docker inspect --format '{{json .NetworkSettings.Ports}}' my-container
# {"80/tcp":[{"HostIp":"0.0.0.0","HostPort":"8080"}]}
```

### Security Considerations for Port Mapping

| Risk | Description | Mitigation |
|---|---|---|
| **Publishing to all interfaces** | Default `0.0.0.0` exposes the port to the entire network | Bind to `127.0.0.1` for local-only access |
| **Docker bypasses UFW/firewalld** | Docker modifies iptables directly, bypassing host firewall rules | Use `DOCKER-USER` iptables chain for custom rules |
| **Running as root** | Binding to ports below 1024 requires root privileges | Use high ports (>1024) or rootless Docker |
| **Unnecessary exposure** | Publishing ports that don't need external access | Only publish ports that external clients need |

```bash
# Securing with the DOCKER-USER chain
# (This chain is preserved across Docker restarts)
$ sudo iptables -I DOCKER-USER -i eth0 -p tcp --dport 8080 \
  -s 10.0.0.0/8 -j ACCEPT
$ sudo iptables -A DOCKER-USER -i eth0 -p tcp --dport 8080 -j DROP
# Only allows access to port 8080 from the 10.0.0.0/8 subnet
```

---

## Practical Examples

### Example 1: Multi-Container Web Application

This example sets up a complete web application stack with DNS resolution and proper port mapping:

```bash
# Create the application network
$ docker network create webapp-network

# Start the database (no port mapping — only internal access needed)
$ docker run -d \
  --name postgres-db \
  --network webapp-network \
  --network-alias db \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  postgres:15

# Start the API server (internal access + published port for debugging)
$ docker run -d \
  --name api-server \
  --network webapp-network \
  -p 127.0.0.1:3000:3000 \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
  node:18

# Start the web server (published to all interfaces)
$ docker run -d \
  --name web-server \
  --network webapp-network \
  -p 80:80 \
  nginx
```

```
┌──────────────────────────────────────────────────────────────────┐
│                          Docker Host                              │
│                                                                   │
│  ┌──────────────── webapp-network ────────────────────────────┐  │
│  │                                                             │  │
│  │  ┌────────────┐   ┌────────────┐   ┌──────────────────┐   │  │
│  │  │ web-server │   │ api-server │   │   postgres-db    │   │  │
│  │  │  Port: 80  │──→│  Port:3000 │──→│  Port: 5432      │   │  │
│  │  │            │   │            │   │  Alias: "db"     │   │  │
│  │  └────────────┘   └────────────┘   └──────────────────┘   │  │
│  │                                                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Published Ports:                                                 │
│    0.0.0.0:80      → web-server:80     (public)                  │
│    127.0.0.1:3000  → api-server:3000   (localhost only)          │
│    (none)          → postgres-db:5432  (internal only)           │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example 2: Docker Compose with Full DNS & Port Mapping

```yaml
# docker-compose.yml
version: "3.8"

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # No external access

services:
  nginx:
    image: nginx
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend
    depends_on:
      - api

  api:
    image: node:18
    networks:
      - frontend        # Reachable by nginx
      - backend          # Can reach database
    environment:
      - DB_HOST=postgres  # DNS name of the database service
      - REDIS_HOST=redis  # DNS name of the Redis service

  postgres:
    image: postgres:15
    networks:
      - backend          # Only reachable from backend network
    environment:
      - POSTGRES_PASSWORD=secret

  redis:
    image: redis:7
    networks:
      - backend          # Only reachable from backend network
```

In this setup:
- **DNS**: `api` can resolve `postgres` and `redis` by name (they share the backend network). `nginx` can resolve `api` by name (they share the frontend network).
- **Port Mapping**: Only `nginx` has published ports (80, 443). All other services are internal only.
- **Network Isolation**: `postgres` and `redis` are on an internal network — they cannot be reached from outside, even by the host.

---

## Summary

| Concept | Purpose | Key Points |
|---|---|---|
| **DNS Inside Docker** | Enables containers to find each other by name instead of IP address | Docker runs an embedded DNS server at `127.0.0.11`; automatic DNS works only on **user-defined networks**, not the default bridge; network aliases allow multiple DNS names for a container. |
| **Legacy Linking (`--link`)** | Older method to connect containers on the default bridge | Injects `/etc/hosts` entries and environment variables; unidirectional; **deprecated** — do not use in new projects. |
| **Modern Linking (Networks)** | Recommended way to connect containers using user-defined networks and DNS | Bidirectional, dynamic IP updates, network isolation, supports multiple networks, works with Docker Compose natively. |
| **Port Mapping (`-p`)** | Makes container ports accessible from outside the Docker host | Syntax: `-p host_port:container_port`; uses iptables and docker-proxy; bind to specific interfaces with `host_ip:host_port:container_port`; `-P` publishes all `EXPOSE`d ports to random host ports. |

### Quick Reference Commands

```bash
# DNS: Check DNS config inside a container
$ docker exec CONTAINER cat /etc/resolv.conf

# DNS: Lookup a container/service name
$ docker exec CONTAINER nslookup SERVICE_NAME

# Linking: Create a user-defined network (modern approach)
$ docker network create my-network

# Linking: Run containers on the same network
$ docker run -d --name app --network my-network IMAGE

# Port Mapping: Map host port to container port
$ docker run -d -p HOST_PORT:CONTAINER_PORT IMAGE

# Port Mapping: View mappings for a container
$ docker port CONTAINER_NAME

# Port Mapping: Localhost-only binding
$ docker run -d -p 127.0.0.1:HOST_PORT:CONTAINER_PORT IMAGE
```
