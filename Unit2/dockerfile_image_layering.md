# Dockerfile Core Concepts: Image Layering — A Complete Deep Dive

Understanding how Docker builds images layer by layer is essential for writing efficient, optimized Dockerfiles. Every instruction you write has a direct impact on image size, build speed, and caching behavior. This document covers **everything** about image layering from the ground up.

---

## 1. What Is a Docker Image?

A Docker image is **not** a single monolithic file. It is an **ordered collection of filesystem layers**, each representing a set of file changes. These layers are stacked on top of each other using a Union File System (typically `overlay2`) to present a single, unified filesystem to the container.

Think of it like transparent sheets stacked on top of each other — each sheet adds, modifies, or hides content from the sheets below.

### Key Properties of a Docker Image

| Property | Detail |
|---|---|
| **Immutable** | Once built, an image layer cannot be changed |
| **Content-addressable** | Each layer is identified by a SHA256 hash   of its content |
| **Shareable** | Multiple images can share the same base layers |
| **Portable** | Images can be pushed/pulled from registries across machines |

---

## 2. How Dockerfile Instructions Create Layers

Every instruction in a Dockerfile is processed sequentially (top to bottom). However, **not every instruction creates a new filesystem layer**. This distinction is critical.

### Layer-Creating Instructions (Modifies the filesystem)

These instructions produce a **new layer** because they change files on disk:

| Instruction | What It Does | Creates Layer? |
|---|---|---|
| `FROM` | Sets the base image (all its layers become the foundation) | ✅ Yes |
| `RUN` | Executes a command inside the image (install packages, compile code, etc.) | ✅ Yes |
| `COPY` | Copies files/directories from the build context into the image | ✅ Yes |
| `ADD` | Like `COPY`, but also handles URLs and auto-extracts tar archives | ✅ Yes |

### Metadata-Only Instructions (No filesystem change)

These instructions only add **configuration metadata** to the image — they do NOT create new filesystem layers:

| Instruction | What It Does | Creates Layer? |
|---|---|---|
| `CMD` | Default command to run when a container starts | ❌ No |
| `ENTRYPOINT` | Configures the container to run as an executable | ❌ No |
| `ENV` | Sets environment variables | ❌ No |
| `EXPOSE` | Documents which ports the container listens on | ❌ No |
| `LABEL` | Adds metadata (key-value pairs) to the image | ❌ No |
| `VOLUME` | Creates a mount point for external storage | ❌ No |
| `WORKDIR` | Sets the working directory for subsequent instructions | ❌ No |
| `USER` | Sets the user/UID for subsequent instructions | ❌ No |
| `ARG` | Defines a build-time variable | ❌ No |
| `STOPSIGNAL` | Sets the system call signal for stopping the container | ❌ No |
| `HEALTHCHECK` | Tells Docker how to test the container's health | ❌ No |
| `SHELL` | Overrides the default shell used for `RUN` commands | ❌ No |

> **Important:** Even though metadata instructions don't create filesystem layers, they are still cached as intermediate steps in the build process.

---

## 3. The Anatomy of Image Layering — Step by Step

Let's trace through exactly what happens when Docker builds an image.

### Example Dockerfile

```dockerfile
# Layer 1: Pull the base image (all Ubuntu layers become the foundation)
FROM ubuntu:22.04

# Metadata only — sets environment variable, no new layer
ENV APP_HOME=/app

# Metadata only — sets working directory
WORKDIR $APP_HOME

# Layer 2: Update package lists + install curl (filesystem changes)
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Layer 3: Copy requirements file
COPY requirements.txt .

# Layer 4: Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Layer 5: Copy entire application source code
COPY . .

# Metadata only — expose port
EXPOSE 8000

# Metadata only — set the startup command
CMD ["python", "app.py"]
```

### What Docker Does Internally

```
┌─────────────────────────────────────────────────┐
│   Step 1: FROM ubuntu:22.04                     │
│   → Pull base image layers (Layer 1a, 1b, 1c…) │
│   → These become the read-only foundation       │
├─────────────────────────────────────────────────┤
│   Step 2: ENV APP_HOME=/app                     │
│   → Store metadata; no new filesystem layer     │
├─────────────────────────────────────────────────┤
│   Step 3: WORKDIR $APP_HOME                     │
│   → Create /app directory if it doesn't exist   │
│   → Minimal layer (just a new empty directory)   │
├─────────────────────────────────────────────────┤
│   Step 4: RUN apt-get update && install curl    │
│   → Spin up temp container on top of base       │
│   → Execute command inside it                   │
│   → Capture filesystem diff → NEW LAYER         │
│   → Destroy temp container                      │
├─────────────────────────────────────────────────┤
│   Step 5: COPY requirements.txt .               │
│   → Add file to /app → NEW LAYER               │
├─────────────────────────────────────────────────┤
│   Step 6: RUN pip install -r requirements.txt   │
│   → Install packages → NEW LAYER               │
├─────────────────────────────────────────────────┤
│   Step 7: COPY . .                              │
│   → Add all source code → NEW LAYER            │
├─────────────────────────────────────────────────┤
│   Step 8: EXPOSE 8000 + CMD                     │
│   → Metadata only; no layer                     │
└─────────────────────────────────────────────────┘
```

Each layer only stores the **diff** (the filesystem changes) from the layer below it — not a full copy of the filesystem.

---

## 4. Layer Caching — Docker's Build Acceleration Engine

Docker uses an aggressive caching mechanism to avoid re-executing unchanged steps. This is one of the most powerful (and easily misused) features of Docker.

### How the Build Cache Works

1. Docker starts at the first instruction and checks if it has a cached result.
2. For `RUN` instructions, Docker checks if the **exact same command string** was previously executed on the **exact same parent layer**.
3. For `COPY`/`ADD` instructions, Docker computes a **checksum (hash) of every file** being copied and compares it against the cache.
4. If the cache is valid → Docker reuses the existing layer (instant — no execution needed).
5. If the cache is **invalidated** at any step → **ALL subsequent layers are also rebuilt from scratch**.

### Cache Invalidation Chain

```
FROM ubuntu:22.04           ✅ Cached
RUN apt-get update          ✅ Cached
COPY requirements.txt .     ❌ CACHE BUST (file changed!)
RUN pip install ...         ❌ Must rebuild (parent changed)
COPY . .                    ❌ Must rebuild (parent changed)
```

> **Key Insight:** A single cache miss cascades — everything below it rebuilds. This is why **instruction ordering matters enormously**.

### The Golden Rule of Layer Ordering

```
Put instructions that change LEAST FREQUENTLY at the top.
Put instructions that change MOST FREQUENTLY at the bottom.
```

**Good ordering:**
```dockerfile
FROM node:18-alpine          # Rarely changes
RUN npm install -g pm2       # Rarely changes
COPY package.json .          # Changes when dependencies change
RUN npm install              # Rebuilds only when package.json changes
COPY . .                     # Changes every code commit (at bottom!)
```

**Bad ordering:**
```dockerfile
FROM node:18-alpine
COPY . .                     # 💥 Changes every commit — busts cache early!
RUN npm install              # Forced to reinstall EVERY time
RUN npm install -g pm2       # Forced to rebuild EVERY time
```

---

## 5. Layer Size Optimization — Keeping Images Lean

Every layer permanently adds to the final image size. Even if you delete a file in a later layer, the file still exists in the earlier layer (because layers are immutable). Understanding this is critical to building small images.

### Problem: The "Delete Doesn't Actually Delete" Trap

```dockerfile
RUN apt-get update && apt-get install -y build-essential   # +200MB
RUN make && make install                                    # +5MB
RUN apt-get remove -y build-essential && apt-get autoremove # ⚠️ Still 200MB!
```

Even though `build-essential` is removed in Layer 3, it still physically exists in Layer 1. The final image carries all three layers.

### Solution: Combine Commands in a Single `RUN` Layer

```dockerfile
RUN apt-get update && \
    apt-get install -y build-essential && \
    make && make install && \
    apt-get remove -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
```

Now all the changes happen in **one layer**. The build tools are installed, used, and removed within the same layer — so the final layer diff doesn't include them.

### Common Techniques for Smaller Layers

| Technique | Example |
|---|---|
| Chain `RUN` commands with `&&` | `RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*` |
| Use `--no-cache-dir` with pip | `RUN pip install --no-cache-dir -r requirements.txt` |
| Use `--no-install-recommends` | `RUN apt-get install -y --no-install-recommends curl` |
| Clean up in the same layer | `RUN yum install -y gcc && make && yum remove -y gcc && yum clean all` |
| Use `.dockerignore` | Prevent `node_modules/`, `.git/`, logs from being sent to the build context |
| Use slim/alpine base images | `FROM python:3.11-slim` instead of `FROM python:3.11` |

---

## 6. Inspecting Image Layers

Docker provides several commands to inspect the layers of an image.

### `docker history <image>`

Shows every layer in the image, the instruction that created it, and its size:

```bash
$ docker history myapp:latest

IMAGE          CREATED        CREATED BY                                      SIZE
a1b2c3d4e5f6   2 min ago      CMD ["python" "app.py"]                         0B
b2c3d4e5f6a7   2 min ago      COPY . .                                        4.2kB
c3d4e5f6a7b8   3 min ago      RUN pip install --no-cache-dir -r requireme…    28.5MB
d4e5f6a7b8c9   3 min ago      COPY requirements.txt .                         58B
e5f6a7b8c9d0   5 min ago      RUN apt-get update && apt-get install -y …      85.3MB
f6a7b8c9d0e1   2 weeks ago    /bin/sh -c #(nop) CMD ["bash"]                  0B
a7b8c9d0e1f2   2 weeks ago    /bin/sh -c #(nop) ADD file:abc123… in /        77.8MB
```

- Layers with `0B` size are metadata-only instructions.
- Large layers indicate opportunities for optimization.

### `docker inspect <image>`

Returns full JSON metadata about the image, including the ordered list of layer digests:

```bash
$ docker inspect myapp:latest --format '{{json .RootFS.Layers}}' | python3 -m json.tool

[
    "sha256:aaa111...",
    "sha256:bbb222...",
    "sha256:ccc333...",
    "sha256:ddd444...",
    "sha256:eee555..."
]
```

### `docker image ls` (Check Image Size)

```bash
$ docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
myapp        latest    a1b2c3d4e5f6   5 min ago      195MB
```

### `dive` (Third-Party Tool)

[`dive`](https://github.com/wagoodman/dive) is a popular open-source CLI tool that lets you interactively explore each layer, see exactly what files were added/modified/removed, and identify wasted space.

```bash
$ dive myapp:latest
```

---

## 7. Multi-Stage Builds — The Ultimate Layering Strategy

Multi-stage builds let you use multiple `FROM` statements in a single Dockerfile. Each `FROM` starts a **completely new image** with its own set of layers. You can selectively copy artifacts from one stage to another, leaving behind all the build dependencies.

### Why Multi-Stage Builds Matter

Without multi-stage builds, you face a dilemma:
- **Development image:** Has compilers, debuggers, source code → Very large
- **Production image:** Only needs the compiled binary → Should be tiny

Multi-stage builds solve this by letting you build in one stage and deploy from another.

### Example: Go Application

```dockerfile
# ========== Stage 1: Build ==========
FROM golang:1.21 AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .

# ========== Stage 2: Production ==========
FROM alpine:3.18

RUN apk --no-cache add ca-certificates
WORKDIR /app

# Copy ONLY the compiled binary from Stage 1
COPY --from=builder /app/server .

EXPOSE 8080
CMD ["./server"]
```

### Size Comparison

| Stage | Base Image | Approximate Size |
|---|---|---|
| Builder (Stage 1) | `golang:1.21` | ~850MB |
| Production (Stage 2) | `alpine:3.18` | ~8MB |

The final image is based only on Stage 2. All of Stage 1's layers (Go compiler, source code, intermediate build files) are completely absent from the final image.

### Example: Node.js Application

```dockerfile
# Stage 1: Install dependencies & build
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production image with only built assets
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 8. The `.dockerignore` File — Controlling the Build Context

Before Docker even starts reading your Dockerfile, it sends the **entire build context** (usually the directory where you run `docker build`) to the Docker daemon. A `.dockerignore` file filters what gets sent.

### Why It Matters for Layering

- Large build contexts slow down `docker build` because more data must be transferred to the daemon.
- Files in the build context that end up in `COPY . .` instructions bloat layers unnecessarily.
- Sensitive files (`.env`, secrets) should never make it into any layer.

### Example `.dockerignore`

```
# Version control
.git
.gitignore

# Dependencies (will be installed inside the image)
node_modules
__pycache__
*.pyc

# Build artifacts
dist
build

# Docker files
Dockerfile
docker-compose*.yml
.dockerignore

# IDE and OS files
.vscode
.idea
*.swp
.DS_Store

# Environment and secrets
.env
.env.*
*.pem
*.key

# Documentation and tests
README.md
docs/
tests/
```

---

## 9. Best Practices Summary — Efficient Image Layering

### ✅ Do

1. **Order instructions by change frequency** — least changing at the top, most changing at the bottom.
2. **Combine related `RUN` commands** — use `&&` and `\` to chain commands into a single layer.
3. **Use multi-stage builds** — separate build-time dependencies from runtime.
4. **Use slim/alpine base images** — start small (`python:3.11-slim` over `python:3.11`).
5. **Leverage `.dockerignore`** — exclude unnecessary files from the build context.
6. **Clean up in the same `RUN` layer** — remove caches, temp files, and package managers in the same instruction.
7. **Pin base image versions** — use `FROM node:18.17-alpine` not `FROM node:latest`.
8. **Copy dependency files first** — e.g., `COPY package.json` before `COPY . .` to cache `npm install`.

### ❌ Don't

1. **Don't delete files in a separate layer** — the original layer still retains them.
2. **Don't use `ADD` when `COPY` suffices** — `ADD` has implicit behaviors (URL fetch, tar extraction) that introduce unpredictability.
3. **Don't run `apt-get update` alone** — always pair it with `apt-get install` in the same `RUN` to avoid stale package lists from cache.
4. **Don't use `latest` tag for base images** — builds become non-reproducible.
5. **Don't include secrets in any layer** — use build secrets (`--secret`) or multi-stage builds instead.

---

## 10. Relevant Docker Commands

| Command | Description |
|---|---|
| `docker build -t <name>:<tag> .` | Build an image from a Dockerfile in the current directory |
| `docker build --no-cache -t <name>:<tag> .` | Build without using any cached layers |
| `docker build --target <stage> -t <name> .` | Build only up to a specific stage in a multi-stage Dockerfile |
| `docker history <image>` | Show the layer history of an image with sizes |
| `docker inspect <image>` | Show detailed metadata including layer SHA digests |
| `docker image ls` | List all local images with their sizes |
| `docker image prune` | Remove dangling (untagged) images to free disk space |
| `docker image prune -a` | Remove ALL unused images (not just dangling) |
| `docker system df` | Show disk usage by images, containers, and volumes |
| `docker save <image> -o image.tar` | Export an image as a tar archive (all layers included) |
| `docker load -i image.tar` | Import an image from a tar archive |

---

## 11. Visual Summary — How Layers Stack

```
┌──────────────────────────────────────┐
│         Container R/W Layer          │  ← Runtime writes go here
│     (Ephemeral — deleted with rm)    │
├══════════════════════════════════════┤
│    Layer 5: COPY . .  (4.2 kB)      │  ← App source code
├──────────────────────────────────────┤
│    Layer 4: RUN pip install (28 MB)  │  ← Python packages
├──────────────────────────────────────┤
│    Layer 3: COPY requirements.txt    │  ← Dependency list
├──────────────────────────────────────┤
│    Layer 2: RUN apt-get install      │  ← System packages
├──────────────────────────────────────┤
│    Layer 1: FROM ubuntu:22.04        │  ← Base OS
│    (multiple sub-layers)             │
└──────────────────────────────────────┘
         ▲ Read-Only Image Layers
```

- All **image layers** (1–5) are **read-only** and **shared** across containers.
- The **container layer** at the top is **read-write** and **unique** to each running container.
- Deleting the container (`docker rm`) removes only the top R/W layer.
- The image layers remain intact and reusable.

---

*This document is part of the INT332 Unit 2 — Dockerfile Core Concepts series.*
