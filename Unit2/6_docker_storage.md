# Docker Storage: Volumes, Bind Mounts & Copy-on-Write — A Complete Deep Dive

Understanding how Docker manages data persistence and storage is fundamental to building reliable containerized applications. Containers are **ephemeral by design** — when a container is removed, all data written inside it is lost. Docker provides multiple storage mechanisms to overcome this limitation. This document covers **everything** about Docker storage — from volumes and bind mounts to the underlying Copy-on-Write mechanism.

---

## 1. The Storage Problem — Why Containers Need Persistent Storage

By default, all files created inside a container are stored on a **writable container layer**. This presents several critical problems:

| Problem | Explanation |
|---|---|
| **Data is ephemeral** | When the container is removed (`docker rm`), the writable layer is deleted and all data is lost permanently |
| **Tight coupling** | Data inside the container layer is bound to that specific container and cannot be easily moved or shared |
| **Performance overhead** | Writing to the container layer goes through the storage driver (e.g., `overlay2`), which adds overhead compared to writing directly to the host filesystem |
| **No sharing** | Multiple containers cannot easily share data through the container layer |

Docker solves these problems with three storage options:

```
┌──────────────────────────────────────────────────────────────────┐
│                   Docker Storage Options                         │
├──────────────────┬──────────────────┬────────────────────────────┤
│     Volumes      │   Bind Mounts    │      tmpfs Mounts          │
│                  │                  │                            │
│  Managed by      │  Direct host     │  Stored in host            │
│  Docker daemon   │  path mapping    │  memory only               │
│                  │                  │                            │
│  Best for        │  Best for        │  Best for                  │
│  production      │  development     │  sensitive/temporary data  │
│  persistence     │  workflows       │                            │
└──────────────────┴──────────────────┴────────────────────────────┘
```

---

## 2. Docker Volumes — The Preferred Storage Mechanism

Volumes are **Docker-managed** storage areas that exist outside the container's Union File System. They are the **recommended way** to persist data in Docker.

### How Volumes Work

When you create a volume, Docker allocates a directory on the **host filesystem** (typically under `/var/lib/docker/volumes/`) and manages it entirely. The container sees this volume as a regular directory mounted at a specific path.

```
Host System                              Container
┌─────────────────────────┐              ┌───────────────────────┐
│                         │              │                       │
│  /var/lib/docker/       │              │   /app/data           │
│    volumes/             │   mounted    │   ├── database.db     │
│      mydata/            │ ──────────►  │   ├── config.json     │
│        _data/           │     as       │   └── logs/           │
│          ├── database.db│              │                       │
│          ├── config.json│              │                       │
│          └── logs/      │              │                       │
│                         │              │                       │
└─────────────────────────┘              └───────────────────────┘
```

### Creating and Managing Volumes

#### Create a Volume

```bash
# Create a named volume
$ docker volume create mydata

# Docker creates: /var/lib/docker/volumes/mydata/_data/
```

#### List All Volumes

```bash
$ docker volume ls

DRIVER    VOLUME NAME
local     mydata
local     postgres_data
local     redis_cache
```

#### Inspect a Volume (See Where It Lives on Host)

```bash
$ docker volume inspect mydata

[
    {
        "CreatedAt": "2025-03-28T10:15:30Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
        "Name": "mydata",
        "Options": {},
        "Scope": "local"
    }
]
```

> **Key:** The `Mountpoint` shows the actual directory on the host where data is stored.

#### Remove a Volume

```bash
# Remove a specific volume (must not be in use by any container)
$ docker volume rm mydata

# Remove ALL unused volumes (not attached to any container)
$ docker volume prune
```

### Using Volumes with Containers

#### Method 1: `--mount` Flag (Recommended — Explicit Syntax)

```bash
$ docker run -d \
    --name myapp \
    --mount type=volume,source=mydata,target=/app/data \
    myimage:latest
```

| Parameter | Meaning |
|---|---|
| `type=volume` | Specifies this is a Docker-managed volume |
| `source=mydata` | Name of the volume (created automatically if it doesn't exist) |
| `target=/app/data` | Path inside the container where the volume is mounted |

#### Method 2: `-v` / `--volume` Flag (Shorthand Syntax)

```bash
$ docker run -d \
    --name myapp \
    -v mydata:/app/data \
    myimage:latest
```

The `-v` syntax follows the pattern: `volume_name:container_path[:options]`

#### Read-Only Volumes

```bash
# Mount volume as read-only inside the container
$ docker run -d \
    --name myapp \
    --mount type=volume,source=mydata,target=/app/data,readonly \
    myimage:latest

# Or with -v shorthand
$ docker run -d -v mydata:/app/data:ro myimage:latest
```

### Anonymous Volumes

If you don't specify a volume name, Docker creates an **anonymous volume** with a random hash name:

```bash
$ docker run -d -v /app/data myimage:latest

$ docker volume ls
DRIVER    VOLUME NAME
local     7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b
```

> **Warning:** Anonymous volumes are hard to track and manage. Always use **named volumes** for important data.

### Volumes in Dockerfile

The `VOLUME` instruction in a Dockerfile declares that a specific path should be a mount point:

```dockerfile
FROM postgres:15

# Declares /var/lib/postgresql/data as a volume mount point
VOLUME /var/lib/postgresql/data
```

When a container is created from this image without explicitly mounting a volume, Docker will automatically create an **anonymous volume** and mount it at that path.

### Why Volumes Are Preferred

| Advantage | Detail |
|---|---|
| **Docker-managed** | Docker handles the lifecycle; easy to create, list, inspect, back up, and remove |
| **Platform-independent** | Work the same on Linux, macOS, and Windows — no host path issues |
| **Shareable** | Multiple containers can mount the same volume simultaneously |
| **Volume drivers** | Support remote/cloud storage (NFS, AWS EBS, Azure Files) via volume drivers |
| **Pre-populated** | If the mount target already has data in the image, it's copied into the volume on first use |
| **Performance** | On Docker Desktop (macOS/Windows), volumes perform significantly better than bind mounts |
| **Backup-friendly** | Easy to back up and restore without knowing host filesystem details |

---

## 3. Bind Mounts — Direct Host Path Mapping

Bind mounts map a **specific file or directory on the host** directly into the container. Unlike volumes, they are **not managed by Docker** — you are fully responsible for the host path.

### How Bind Mounts Work

```
Host System                              Container
┌─────────────────────────┐              ┌───────────────────────┐
│                         │              │                       │
│  /home/user/            │              │   /app                │
│    myproject/           │   bind       │   ├── src/            │
│      ├── src/           │ ──────────►  │   ├── package.json    │
│      ├── package.json   │   mounted    │   └── Dockerfile      │
│      └── Dockerfile     │              │                       │
│                         │              │                       │
│  (Any host path)        │              │  (Any container path) │
│                         │              │                       │
└─────────────────────────┘              └───────────────────────┘
```

The container sees the host directory's contents **as if they were its own files**. Changes in either direction are reflected immediately — the container and host share the **exact same files**.

### Using Bind Mounts with Containers

#### Method 1: `--mount` Flag (Recommended — Explicit Syntax)

```bash
$ docker run -d \
    --name devapp \
    --mount type=bind,source=/home/user/myproject,target=/app \
    node:18-alpine npm start
```

| Parameter | Meaning |
|---|---|
| `type=bind` | Specifies this is a bind mount (not a Docker volume) |
| `source=/home/user/myproject` | **Absolute path** on the host (must already exist) |
| `target=/app` | Path inside the container |

#### Method 2: `-v` / `--volume` Flag (Shorthand Syntax)

```bash
$ docker run -d \
    --name devapp \
    -v /home/user/myproject:/app \
    node:18-alpine npm start

# Using $(pwd) for the current directory
$ docker run -d \
    -v $(pwd):/app \
    node:18-alpine npm start
```

> **Important:** With `-v`, Docker distinguishes between volumes and bind mounts by checking if the source starts with `/` (absolute path = bind mount) or not (name = volume).

#### Read-Only Bind Mounts

```bash
$ docker run -d \
    --mount type=bind,source=/home/user/config,target=/app/config,readonly \
    myimage:latest

# Or with -v shorthand
$ docker run -d -v /home/user/config:/app/config:ro myimage:latest
```

### Bind Mounts for Development — The #1 Use Case

Bind mounts shine during development because they enable **live code reloading** without rebuilding the image:

```bash
# Development workflow — mount source code into the container
$ docker run -d \
    --name dev-server \
    -v $(pwd)/src:/app/src \
    -v $(pwd)/package.json:/app/package.json \
    -p 3000:3000 \
    node:18-alpine sh -c "npm install && npm run dev"
```

**What happens:**
1. You edit `src/App.js` on your host using your IDE.
2. The change is instantly visible inside the container at `/app/src/App.js`.
3. The dev server (e.g., Vite, Webpack) detects the change and hot-reloads.
4. You see the update in your browser — **no `docker build` needed**.

### Bind Mount Pitfalls and Risks

| Risk | Explanation |
|---|---|
| **Overwrites container content** | If the container path already has files, the bind mount **completely hides** them (they still exist in the image layer but are invisible) |
| **Host path must exist** | With `--mount`, Docker will error if the host path doesn't exist. With `-v`, Docker creates a directory automatically (which may not be what you want) |
| **Security risk** | The container can read/write arbitrary host files — a compromised container could damage host data |
| **Path portability** | Absolute paths differ between systems (`/home/user` vs `C:\Users\user`) — not portable across environments |
| **Permission issues** | User ID inside the container may not match the host, causing permission denied errors |

---

## 4. Volumes vs Bind Mounts — Complete Comparison

| Feature | Volume | Bind Mount |
|---|---|---|
| **Managed by** | Docker Engine | User / Host OS |
| **Storage location** | `/var/lib/docker/volumes/` | Anywhere on the host filesystem |
| **Created via** | `docker volume create` or automatically | Must pre-exist on the host (with `--mount`) |
| **Portable** | ✅ Works the same on any OS | ❌ Host paths differ across OS |
| **Can be named** | ✅ Named volumes are easy to reference | ❌ Must use full absolute path |
| **Supports volume drivers** | ✅ NFS, cloud storage, etc. | ❌ Local only |
| **Pre-populated** | ✅ Image content is copied to empty volume on first use | ❌ Host content always takes priority; image content is hidden |
| **Docker CLI management** | ✅ `docker volume ls/inspect/rm/prune` | ❌ Must manage via host OS commands |
| **Performance (Desktop)** | ✅ Optimized (runs inside the VM) | ⚠️ Slower (crosses the VM boundary) |
| **Performance (Linux)** | ✅ Near-native speed | ✅ Near-native speed |
| **Best for** | Production data, databases, shared secrets | Development, config injection, host tool access |
| **Shareable** | ✅ Multiple containers can use the same volume | ✅ Multiple containers can bind to the same host path |
| **Backup** | `docker run --rm -v mydata:/data -v $(pwd):/backup alpine tar czf /backup/data.tar.gz /data` | Standard host filesystem backup tools |

### When to Use Volumes

- ✅ Persisting **database data** (PostgreSQL, MySQL, MongoDB)
- ✅ Storing **application state** that must survive container restarts
- ✅ **Sharing data** between multiple containers
- ✅ Using **remote/cloud storage** backends
- ✅ **Production** deployments

### When to Use Bind Mounts

- ✅ **Local development** — live code syncing with the container
- ✅ Injecting **config files** from the host (e.g., `nginx.conf`, `.env`)
- ✅ Accessing **host tools or sockets** (e.g., Docker socket for Docker-in-Docker)
- ✅ **CI/CD pipelines** — mounting the repo into the build container
- ✅ Sharing **build artifacts** from the container back to the host

---

## 5. Backing Data on the Host — Where Docker Stores Data

Understanding the physical location of Docker's data on the host system is crucial for backup, debugging, and disaster recovery.

### Docker Root Directory

All Docker data lives under the **Docker root directory**, which defaults to:

```
/var/lib/docker/
```

You can check this with:

```bash
$ docker info | grep "Docker Root Dir"
Docker Root Dir: /var/lib/docker
```

### Directory Structure

```
/var/lib/docker/
├── overlay2/                 ← Storage driver layer data
│   ├── abc123.../            ← Individual layer directories
│   │   ├── diff/             ← The actual filesystem content of this layer
│   │   ├── link              ← Shortened identifier
│   │   ├── lower             ← Reference to parent layers
│   │   ├── merged/           ← Union mount of all layers (only for running containers)
│   │   └── work/             ← Internal scratch space for overlay2
│   └── l/                    ← Symbolic links to layer diffs
│
├── volumes/                  ← Docker-managed volumes
│   ├── mydata/
│   │   └── _data/            ← Actual volume data lives here
│   ├── postgres_data/
│   │   └── _data/
│   └── metadata.db           ← Volume metadata database
│
├── containers/               ← Container-specific data
│   └── abc123.../
│       ├── config.v2.json    ← Container configuration
│       ├── hostname          ← Container hostname
│       ├── resolv.conf       ← DNS configuration
│       ├── hosts             ← /etc/hosts file
│       └── mounts/           ← Mount information
│
├── image/                    ← Image metadata
│   └── overlay2/
│       ├── imagedb/          ← Image metadata and configs
│       ├── layerdb/          ← Layer metadata and relationships
│       └── repositories.json ← Repository and tag mappings
│
├── network/                  ← Network configuration
├── plugins/                  ← Installed plugins
├── buildkit/                 ← BuildKit cache and data
└── tmp/                      ← Temporary build files
```

### Accessing Volume Data on the Host

Since volumes live at a known host path, you can interact with them directly:

```bash
# Find where a volume's data is stored
$ docker volume inspect mydata --format '{{ .Mountpoint }}'
/var/lib/docker/volumes/mydata/_data

# List files in the volume (requires root/sudo on most systems)
$ sudo ls -la /var/lib/docker/volumes/mydata/_data

# Copy a file from the volume to the current directory
$ sudo cp /var/lib/docker/volumes/mydata/_data/important.db ./backup/
```

### Backing Up Volume Data

#### Method 1: Using a Helper Container (Recommended — No Root Required)

```bash
# Back up the "mydata" volume to a tar.gz file on the host
$ docker run --rm \
    -v mydata:/source:ro \
    -v $(pwd):/backup \
    alpine tar czf /backup/mydata_backup.tar.gz -C /source .
```

**What this does:**
1. Creates a temporary Alpine container.
2. Mounts the `mydata` volume as read-only at `/source`.
3. Mounts the current host directory at `/backup`.
4. Creates a compressed tarball of the volume's entire content.
5. Automatically removes the helper container when done.

#### Method 2: Restore a Volume from Backup

```bash
# Create a new volume and restore data into it
$ docker volume create mydata_restored

$ docker run --rm \
    -v mydata_restored:/target \
    -v $(pwd):/backup:ro \
    alpine sh -c "cd /target && tar xzf /backup/mydata_backup.tar.gz"
```

#### Method 3: Direct Host Copy (Requires Root)

```bash
# Back up
$ sudo tar czf mydata_backup.tar.gz -C /var/lib/docker/volumes/mydata/_data .

# Restore
$ sudo tar xzf mydata_backup.tar.gz -C /var/lib/docker/volumes/mydata/_data
```

### Backing Up Bind-Mounted Data

Since bind mounts use regular host directories, you can use **any standard backup tool**:

```bash
# Simple copy
$ cp -r /home/user/myproject/data ./backup/

# Using rsync
$ rsync -av /home/user/myproject/data/ ./backup/data/

# Compressed archive
$ tar czf project_data_backup.tar.gz /home/user/myproject/data
```

### Changing Docker's Data Location

If your root partition is small, you can move Docker's data directory to a different disk:

```bash
# 1. Stop Docker
$ sudo systemctl stop docker

# 2. Edit the Docker daemon config
$ sudo nano /etc/docker/daemon.json

# Add or modify:
{
    "data-root": "/mnt/large-disk/docker"
}

# 3. Move existing data
$ sudo rsync -aP /var/lib/docker/ /mnt/large-disk/docker/

# 4. Start Docker
$ sudo systemctl start docker

# 5. Verify
$ docker info | grep "Docker Root Dir"
Docker Root Dir: /mnt/large-disk/docker
```

---

## 6. tmpfs Mounts — Storing Data in Memory Only

Docker also supports **tmpfs mounts**, which store data in the host's **RAM** (never written to disk). The data disappears when the container stops.

### When to Use tmpfs

- Storing **sensitive secrets** that should never be written to disk
- **Temporary caches** or scratch data that doesn't need persistence
- **Performance-critical** writes where disk I/O is a bottleneck

### Using tmpfs Mounts

```bash
# Using --mount (recommended)
$ docker run -d \
    --name secure-app \
    --mount type=tmpfs,target=/app/secrets,tmpfs-size=100m \
    myimage:latest

# Using --tmpfs (shorthand)
$ docker run -d \
    --name secure-app \
    --tmpfs /app/secrets:size=100m \
    myimage:latest
```

### Storage Types — Complete Comparison

```
┌────────────────────────────────────────────────────────────────┐
│                Container Filesystem View                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   /app/data ─────────── Volume        → Host: /var/lib/docker/ │
│   /app/src ──────────── Bind Mount    → Host: /home/user/src/  │
│   /app/secrets ──────── tmpfs Mount   → Host: RAM (no disk)    │
│   /app/code ─────────── Container     → R/W layer (ephemeral)  │
│                          Layer                                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 7. Copy-on-Write (CoW) Mechanism — How Docker Handles File Writes

The Copy-on-Write mechanism is one of the most important concepts in Docker's storage architecture. It is the strategy Docker uses to manage the relationship between **read-only image layers** and the **writable container layer**.

### The Core Concept

Docker images are composed of multiple **read-only layers** stacked on top of each other. When a container starts, Docker adds a thin **writable layer** (called the **container layer**) on top. The CoW mechanism governs what happens when the container tries to read, modify, or delete files.

```
┌─────────────────────────────────────────┐
│  Container Layer (R/W)                  │ ← Thin writable layer
│  - Modified files are COPIED here       │
│  - New files are CREATED here           │
│  - Deleted files are MARKED here        │
├═════════════════════════════════════════┤
│  Image Layer 4 (R/O): COPY . .         │ ← Read-only
├─────────────────────────────────────────┤
│  Image Layer 3 (R/O): RUN npm install  │ ← Read-only
├─────────────────────────────────────────┤
│  Image Layer 2 (R/O): COPY package.json│ ← Read-only
├─────────────────────────────────────────┤
│  Image Layer 1 (R/O): FROM node:18     │ ← Read-only
└─────────────────────────────────────────┘
```

### How CoW Works — Three Scenarios

#### Scenario 1: Reading a File (No Copy Needed)

When a container reads a file, Docker searches for it from the **top layer down**:

```
Read request: /app/index.js

Step 1: Check Container Layer (R/W)  → Not found
Step 2: Check Image Layer 4          → FOUND! Return this version
         (No data is copied — the file is read in place)
```

> **Performance:** Reads are fast because no data is copied. The file is accessed directly from whatever layer it exists in.

#### Scenario 2: Modifying a File (Copy-on-Write Triggered)

When a container modifies a file that exists in a read-only layer, CoW kicks in:

```
Write request: Modify /app/index.js

Step 1: File found in Image Layer 4 (read-only)
Step 2: Cannot modify a read-only layer!
Step 3: COPY the entire file from Layer 4 → Container Layer (R/W)
Step 4: Apply the modification to the copy in the Container Layer
Step 5: Future reads of /app/index.js will find the copy in
        the Container Layer first (top-down search)
```

```
BEFORE modification:                    AFTER modification:
┌──────────────────────┐              ┌──────────────────────┐
│ Container Layer (R/W)│              │ Container Layer (R/W)│
│ (empty)              │              │ index.js (MODIFIED)  │ ← Copy lives here
├══════════════════════┤              ├══════════════════════┤
│ Layer 4 (R/O)        │              │ Layer 4 (R/O)        │
│ index.js (original)  │              │ index.js (original)  │ ← Still here, untouched!
├──────────────────────┤              ├──────────────────────┤
│ Layer 3 (R/O)        │              │ Layer 3 (R/O)        │
│ node_modules/        │              │ node_modules/        │
└──────────────────────┘              └──────────────────────┘
```

> **Key Insight:** The original file in the read-only layer is **never modified**. It still exists, but the modified copy in the container layer **shadows** it because Docker searches top-down.

#### Scenario 3: Deleting a File (Whiteout Files)

When a container deletes a file from a read-only layer, Docker cannot actually remove it (the layer is read-only). Instead, it creates a special **whiteout file** in the container layer:

```
Delete request: Remove /app/old-config.json

Step 1: File found in Image Layer 2 (read-only)
Step 2: Cannot delete from a read-only layer!
Step 3: Create a WHITEOUT FILE in the Container Layer:
        .wh.old-config.json
Step 4: The union filesystem sees the whiteout and hides the
        original file — it appears deleted to the container
```

```
┌──────────────────────────┐
│ Container Layer (R/W)    │
│ .wh.old-config.json      │ ← Whiteout marker (blocks the file below)
├══════════════════════════┤
│ Layer 2 (R/O)            │
│ old-config.json          │ ← Still physically exists, but hidden
└──────────────────────────┘
```

> **Important:** The original file is NOT freed from disk. It still exists in the read-only layer and contributes to the image size. This is why you should always clean up in the **same** `RUN` layer where files were created.

### The Copy-on-Write Process — Step by Step (overlay2)

Docker's default storage driver on Linux is `overlay2`, which implements CoW as follows:

```
overlay2 Filesystem Structure for a Running Container:

┌─────────────────────────────────────────────────────┐
│ merged/                                             │
│ (Union view — what the container process sees)      │
│ Combines all layers into one unified directory      │
├──────────────────────┬──────────────────────────────┤
│ upperdir/            │ lowerdir(s)                  │
│ (Container R/W)      │ (Image R/O layers)           │
│                      │                              │
│ - New files go here  │ - Layer 4: app code          │
│ - Copied-on-write    │ - Layer 3: npm packages      │
│   files land here    │ - Layer 2: package.json      │
│ - Whiteout files for │ - Layer 1: node base image   │
│   deleted items      │                              │
├──────────────────────┤                              │
│ workdir/             │                              │
│ (Scratch space for   │                              │
│  atomic operations)  │                              │
└──────────────────────┴──────────────────────────────┘
```

1. **`lowerdir`** — One or more read-only directories containing the image layers. These are stacked, with each layer overlaying the previous one.
2. **`upperdir`** — The writable container layer. All writes, modifications, and deletions happen here.
3. **`merged`** — The unified view presented to the container process. The overlay2 filesystem combines `lowerdir` and `upperdir` into this single directory.
4. **`workdir`** — Internal scratch space used by overlay2 for atomic copy-up operations.

### Verifying CoW in Action

```bash
# 1. Run a container
$ docker run -dit --name cowtest ubuntu:22.04 bash

# 2. Find the container's overlay2 directories
$ docker inspect cowtest --format '{{json .GraphDriver.Data}}' | python3 -m json.tool
{
    "LowerDir": "/var/lib/docker/overlay2/abc123/diff:/var/lib/docker/overlay2/def456/diff",
    "MergedDir": "/var/lib/docker/overlay2/xyz789/merged",
    "UpperDir": "/var/lib/docker/overlay2/xyz789/diff",
    "WorkDir": "/var/lib/docker/overlay2/xyz789/work"
}

# 3. Check the container's writable layer (UpperDir) — initially nearly empty
$ sudo ls /var/lib/docker/overlay2/xyz789/diff
# (mostly empty or minimal)

# 4. Modify a file inside the container
$ docker exec cowtest bash -c "echo 'modified' >> /etc/hostname"

# 5. Check UpperDir again — the modified file has been COPIED here
$ sudo ls /var/lib/docker/overlay2/xyz789/diff/etc/
hostname

# 6. The original in the lower layer is UNTOUCHED
$ sudo cat /var/lib/docker/overlay2/abc123/diff/etc/hostname
# (original content)
```

### Performance Implications of Copy-on-Write

| Operation | Performance | Notes |
|---|---|---|
| **Reading** unmodified files | ✅ Fast | No copy needed; read directly from image layer |
| **First write** to an existing file | ⚠️ Slower | Entire file must be copied up to container layer before modification |
| **Subsequent writes** to the same file | ✅ Fast | File is already in the container layer; direct write |
| **Writing new files** | ✅ Fast | Created directly in the container layer |
| **Deleting files** | ✅ Fast | Just creates a small whiteout marker |
| **Writing to a volume** | ✅ Fastest | Bypasses CoW entirely; direct filesystem write |

> **Critical Takeaway:** For **write-heavy workloads** (databases, logging, caching), ALWAYS use **volumes** instead of writing to the container layer. CoW adds overhead for every first-write, and all data is lost when the container is removed.

### Why CoW Matters for Image Sharing

CoW enables Docker's most powerful storage optimizations:

```
Image: ubuntu:22.04 (3 layers, 77MB)

Container A ──┐
              │     All 3 containers SHARE the same
Container B ──┤───► 3 read-only image layers (77MB total,
              │     NOT 77MB × 3 = 231MB)
Container C ──┘
              │
              ├──── Container A writable layer: 5MB
              ├──── Container B writable layer: 12MB
              └──── Container C writable layer: 3MB

Total disk usage: 77MB + 5MB + 12MB + 3MB = 97MB
(without CoW it would be: 231MB + 20MB = 251MB)
```

This means you can run **100 containers** from the same image and they all share the base layers. Only each container's unique changes consume additional disk space.

---

## 8. Docker Compose and Storage

In real-world applications, you typically define volumes and bind mounts in a `docker-compose.yml` file.

### Volumes in Docker Compose

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      # Bind mount — source code for development
      - ./html:/usr/share/nginx/html:ro
      # Named volume — for persistent logs
      - nginx_logs:/var/log/nginx

  database:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      # Named volume — persistent database storage
      - postgres_data:/var/lib/postgresql/data
      # Bind mount — initialization scripts
      - ./init-scripts:/docker-entrypoint-initdb.d:ro

  redis:
    image: redis:7-alpine
    volumes:
      # Named volume — redis persistence
      - redis_data:/data
    command: redis-server --appendonly yes

# Declare named volumes at the top level
volumes:
  postgres_data:           # Uses default local driver
  redis_data:              # Uses default local driver
  nginx_logs:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /var/log/nginx-host  # Custom host path for the volume
```

### Volume Lifecycle in Compose

```bash
# Create containers + volumes
$ docker compose up -d

# Volumes persist even after stopping and removing containers
$ docker compose down          # Containers removed, VOLUMES KEPT

# To remove volumes too, use --volumes flag
$ docker compose down --volumes  # ⚠️ Containers AND volumes removed
```

---

## 9. Best Practices Summary

### ✅ Do

1. **Use named volumes for persistent data** — never rely on the container layer for important data.
2. **Use bind mounts only for development** — live code syncing, config injection.
3. **Mount volumes as read-only when possible** — add `:ro` to prevent accidental writes.
4. **Back up volumes regularly** — use the helper container method for portability.
5. **Use volumes for write-heavy workloads** — databases, logging, caching bypass CoW overhead.
6. **Declare volumes in `docker-compose.yml`** — makes infrastructure reproducible.
7. **Clean up unused volumes** — `docker volume prune` to reclaim disk space.
8. **Use `--mount` over `-v`** — the explicit syntax prevents ambiguity (volume vs bind mount).

### ❌ Don't

1. **Don't store important data in the container layer** — it's lost when the container is removed.
2. **Don't use anonymous volumes for important data** — they're hard to find and manage.
3. **Don't bind mount host directories in production** — use Docker-managed volumes instead.
4. **Don't forget to back up volumes before `docker volume rm`** — data deletion is permanent.
5. **Don't write application logs to the container layer** — use volumes or log to stdout/stderr.
6. **Don't ignore file permission issues** — use matching UID/GID or `--user` flag with bind mounts.

---

## 10. Relevant Docker Commands

### Volume Commands

| Command | Description |
|---|---|
| `docker volume create <name>` | Create a new named volume |
| `docker volume ls` | List all volumes |
| `docker volume inspect <name>` | Show detailed volume info (mountpoint, driver, etc.) |
| `docker volume rm <name>` | Remove a specific volume |
| `docker volume prune` | Remove all unused volumes |
| `docker volume prune --filter "label!=persist"` | Remove unused volumes that don't have a "persist" label |

### Running Containers with Storage

| Command | Description |
|---|---|
| `docker run -v mydata:/app/data <image>` | Mount a named volume |
| `docker run -v /host/path:/container/path <image>` | Create a bind mount |
| `docker run -v /container/path <image>` | Create an anonymous volume |
| `docker run --mount type=volume,src=mydata,dst=/app/data <image>` | Mount a volume (explicit syntax) |
| `docker run --mount type=bind,src=/host/path,dst=/app <image>` | Bind mount (explicit syntax) |
| `docker run --mount type=tmpfs,dst=/app/tmp <image>` | tmpfs mount (memory-only) |
| `docker run -v mydata:/app/data:ro <image>` | Mount as read-only |

### Inspection and Debugging

| Command | Description |
|---|---|
| `docker inspect <container> --format '{{json .Mounts}}'` | Show all mounts for a container |
| `docker inspect <container> --format '{{json .GraphDriver.Data}}'` | Show overlay2 layer paths |
| `docker diff <container>` | Show filesystem changes (A=added, C=changed, D=deleted) in the container layer |
| `docker system df` | Show disk usage by images, containers, volumes |
| `docker system df -v` | Verbose disk usage breakdown |

---

## 11. Visual Summary — Docker Storage Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     Docker Storage Architecture                    │
│                                                                    │
│  ┌─────────────────────────────────────────────┐   ┌────────────┐ │
│  │            Running Container                │   │            │ │
│  │                                             │   │   Volume   │ │
│  │  ┌───────────────────────────────────────┐  │   │  /var/lib/ │ │
│  │  │    Container R/W Layer (CoW)          │  │   │  docker/   │ │
│  │  │    - New files created here           │  │   │  volumes/  │ │
│  │  │    - Modified files copied here       │  │   │  mydata/   │ │
│  │  │    - Deleted files marked (whiteout)  │  │   │   _data/   │ │
│  │  ├═══════════════════════════════════════┤  │   │            │ │
│  │  │    Image Layer N (R/O)                │  │   └──────┬─────┘ │
│  │  ├───────────────────────────────────────┤  │          │       │
│  │  │    Image Layer 2 (R/O)                │  │    mount │       │
│  │  ├───────────────────────────────────────┤  │          │       │
│  │  │    Image Layer 1 (R/O)                │  │   /app/data      │
│  │  └───────────────────────────────────────┘  │  (inside ctr)   │
│  │                                             │                  │
│  │  /app/src ——————— Bind Mount ——————► /home/user/myproject     │
│  │  /app/tmp ——————— tmpfs Mount ————► Host RAM (no disk)        │
│  │                                             │                  │
│  └─────────────────────────────────────────────┘                  │
│                                                                    │
│  Storage Driver (overlay2) manages the layer stacking             │
│  Copy-on-Write ensures image layers are never modified            │
│  Volumes bypass the storage driver for best performance           │
└────────────────────────────────────────────────────────────────────┘
```

- **Image layers** — Read-only, shared across all containers using the same image.
- **Container layer** — Read-write, unique per container, managed by CoW. Ephemeral.
- **Volumes** — Bypass CoW, persist independently, Docker-managed. Best for data.
- **Bind mounts** — Map host paths directly. Best for development.
- **tmpfs** — Memory-only, never touches disk. Best for secrets and temp data.

---

*This document is part of the INT332 Unit 2 — Docker Core Concepts series.*
