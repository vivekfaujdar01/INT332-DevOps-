# Docker Image Creation in Detail: The `docker build` Process

## Table of Contents

1. [Introduction](#introduction)
2. [What is a Docker Image?](#what-is-a-docker-image)
3. [The `docker build` Command](#the-docker-build-command)
4. [Build Context](#build-context)
5. [Step-by-Step Build Process](#step-by-step-build-process)
6. [How Docker Processes Each Instruction](#how-docker-processes-each-instruction)
7. [Build Cache Mechanism](#build-cache-mechanism)
8. [Multi-Stage Builds](#multi-stage-builds)
9. [Tagging Images](#tagging-images)
10. [BuildKit — The Modern Builder](#buildkit--the-modern-builder)
11. [`.dockerignore` File](#dockerignore-file)
12. [Build Arguments (`ARG`)](#build-arguments-arg)
13. [Best Practices for `docker build`](#best-practices-for-docker-build)
14. [Common `docker build` Flags](#common-docker-build-flags)
15. [Troubleshooting Build Failures](#troubleshooting-build-failures)
16. [Complete Example Walkthrough](#complete-example-walkthrough)

---

## Introduction

Building Docker images is one of the most fundamental operations in the Docker ecosystem. The `docker build` command reads instructions from a **Dockerfile** and assembles them into a reusable, portable **Docker image**. Understanding this process in depth is essential for creating efficient, secure, and optimized container images.

---

## What is a Docker Image?

A Docker image is a **read-only, layered filesystem** that contains everything needed to run an application:

- **Base operating system** (e.g., Ubuntu, Alpine)
- **Runtime and libraries** (e.g., Python, Node.js, Java)
- **Application source code**
- **Dependencies** (e.g., npm packages, pip modules)
- **Configuration files**
- **Metadata** (e.g., environment variables, exposed ports, entrypoint)

### Image vs Container

| Aspect       | Image                          | Container                          |
|--------------|--------------------------------|------------------------------------|
| Nature       | Read-only template             | Running instance of an image       |
| State        | Immutable                      | Mutable (has a writable layer)     |
| Creation     | Built with `docker build`      | Created with `docker run`          |
| Storage      | Stored in image registry       | Lives on the host machine          |
| Analogy      | A class in OOP                 | An object (instance) in OOP        |

---

## The `docker build` Command

### Basic Syntax

```bash
docker build [OPTIONS] PATH | URL | -
```

- **PATH** — The build context directory (usually `.` for the current directory).
- **URL** — A Git repository URL or a remote tarball.
- **`-`** — Read the Dockerfile from STDIN.

### Most Common Usage

```bash
docker build -t myapp:1.0 .
```

This tells Docker to:
1. Use the current directory (`.`) as the **build context**.
2. Look for a file named `Dockerfile` in that directory.
3. Build the image and tag it as `myapp:1.0`.

---

## Build Context

### What is Build Context?

The **build context** is the set of files and directories that Docker sends to the Docker daemon before building the image. When you run:

```bash
docker build .
```

Docker takes the **entire contents** of the current directory (`.`) and sends it to the daemon as a **tar archive**.

### Why Does Context Matter?

- Only files inside the build context can be used in `COPY` and `ADD` instructions.
- A large build context slows down the build because Docker must transfer all those files.
- Files outside the build context **cannot** be accessed during the build.

### Example

```
my-project/
├── Dockerfile
├── app/
│   ├── main.py
│   └── utils.py
├── requirements.txt
├── data/            ← Large data files
├── .git/            ← Git history
└── node_modules/    ← Heavy dependency folder
```

If you run `docker build .` from `my-project/`, **everything** — including `data/`, `.git/`, and `node_modules/` — is sent to the daemon. This is wasteful and can be avoided using a `.dockerignore` file.

### Build Context Transfer

```
Sending build context to Docker daemon  45.2MB
```

This message appears at the start of every build. If this number is unexpectedly large, you likely need a `.dockerignore` file.

---

## Step-by-Step Build Process

Here is what happens internally when you run `docker build`:

### Phase 1: Preparation

```
Client                          Docker Daemon
  |                                   |
  |-- Send build context (tar) ------>|
  |-- Send Dockerfile --------------->|
  |                                   |
  |                          Parse Dockerfile
  |                          Validate syntax
  |                          Resolve base image
```

1. **Docker client** reads the Dockerfile and packages the build context.
2. The entire context is sent to the **Docker daemon** (the background service).
3. The daemon **parses** the Dockerfile line by line.

### Phase 2: Layer-by-Layer Execution

Each instruction in the Dockerfile creates a new **layer** (or metadata entry):

```
Step 1/6 : FROM python:3.11-slim
 ---> a1b2c3d4e5f6

Step 2/6 : WORKDIR /app
 ---> Running in 7a8b9c0d1e2f
 ---> f6e5d4c3b2a1

Step 3/6 : COPY requirements.txt .
 ---> 1a2b3c4d5e6f

Step 4/6 : RUN pip install -r requirements.txt
 ---> Running in 2b3c4d5e6f7a
Successfully installed flask-2.3.0 ...
 ---> 3c4d5e6f7a8b

Step 5/6 : COPY . .
 ---> 4d5e6f7a8b9c

Step 6/6 : CMD ["python", "main.py"]
 ---> 5e6f7a8b9c0d
```

### Phase 3: Final Image Assembly

```
Successfully built 5e6f7a8b9c0d
Successfully tagged myapp:1.0
```

The final image ID is the hash of the **topmost layer**. All layers are stacked together to form the complete filesystem.

---

## How Docker Processes Each Instruction

Each Dockerfile instruction is processed differently:

### Instructions That Create New Filesystem Layers

These instructions **modify the filesystem** and produce a new layer:

| Instruction | What It Does                                    |
|-------------|--------------------------------------------------|
| `FROM`      | Sets the base image (first layer)                |
| `RUN`       | Executes a command inside a temporary container  |
| `COPY`      | Copies files from build context into the image   |
| `ADD`       | Like COPY, but also handles URLs and tar extraction |

**How `RUN` Works Internally:**

```
1. Docker creates a temporary container from the previous layer
2. The command is executed inside this container
3. The container's filesystem changes are captured as a new layer
4. The temporary container is removed
```

Example:

```dockerfile
RUN apt-get update && apt-get install -y curl
```

Internally:
```
Previous Layer (read-only)
        ↓
[Create temporary container]
        ↓
[Execute: apt-get update && apt-get install -y curl]
        ↓
[Capture filesystem diff → New Layer]
        ↓
[Remove temporary container]
```

### Instructions That Only Add Metadata

These instructions **do not create filesystem layers** — they only modify the image's configuration metadata:

| Instruction  | What It Sets                              |
|--------------|-------------------------------------------|
| `CMD`        | Default command to run                    |
| `ENTRYPOINT` | Fixed executable for the container        |
| `ENV`        | Environment variables                     |
| `EXPOSE`     | Documents which ports the app uses        |
| `WORKDIR`    | Sets the working directory                |
| `LABEL`      | Adds key-value metadata to the image      |
| `USER`       | Sets the user for subsequent instructions |
| `VOLUME`     | Declares a mount point                    |

---

## Build Cache Mechanism

Docker uses an **aggressive caching strategy** to speed up repeated builds.

### How Cache Works

For each instruction, Docker checks:

1. **Is there an existing layer from a previous build that matches this instruction?**
2. If yes → **Use the cached layer** (skip execution).
3. If no → **Execute the instruction** and create a new layer.

### Cache Rules

| Instruction | Cache Invalidation Rule                                           |
|-------------|-------------------------------------------------------------------|
| `RUN`       | Cache is invalid if the command string changes                    |
| `COPY`      | Cache is invalid if any source file's content (checksum) changes  |
| `ADD`       | Same as COPY, plus URL content or tar archive changes             |
| `FROM`      | Cache is invalid if the base image tag resolves to a new digest   |

### Cache Busting Example

```dockerfile
COPY requirements.txt .          # Cached if requirements.txt unchanged
RUN pip install -r requirements.txt  # Cached if previous layer cached
COPY . .                         # Cache BUSTED if ANY file in context changes
RUN python setup.py build        # Also rebuilds because cache is broken above
```

### The Cache Chain

Once a layer's cache is invalidated, **all subsequent layers must also be rebuilt**:

```
Layer 1: FROM python:3.11     ✅ Cached
Layer 2: COPY requirements.txt ✅ Cached
Layer 3: RUN pip install       ✅ Cached
Layer 4: COPY . .              ❌ Cache Miss (source code changed)
Layer 5: RUN python build      ❌ Must Rebuild (cache chain broken)
```

### Optimizing Cache Usage

**Bad Order** (cache busted on every code change):
```dockerfile
COPY . .
RUN pip install -r requirements.txt
```

**Good Order** (dependencies cached separately):
```dockerfile
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

---

## Multi-Stage Builds

Multi-stage builds allow you to use **multiple `FROM` statements** in a single Dockerfile. Each `FROM` starts a new **build stage**, and you can selectively copy artifacts from one stage to another.

### Why Use Multi-Stage Builds?

- **Smaller final images** — Build tools and dependencies stay in the build stage.
- **Security** — Source code and build tools are not shipped in the production image.
- **Clean separation** — Build environment vs. runtime environment.

### Example: Go Application

```dockerfile
# ===== Stage 1: Build =====
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

# ===== Stage 2: Runtime =====
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/myapp .
EXPOSE 8080
CMD ["./myapp"]
```

### How It Works Internally

```
Stage 1 (builder):
  golang:1.21 base → download deps → compile binary → /app/myapp exists

Stage 2 (final):
  alpine:3.18 base → copy /app/myapp FROM builder → final image

Result:
  Final image = Alpine (~5MB) + compiled binary (~10MB) = ~15MB
  Without multi-stage = Go SDK (~800MB) + source + binary = ~850MB
```

### Example: Node.js Application

```dockerfile
# Stage 1: Install dependencies and build
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production image
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY package*.json ./
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## Tagging Images

### What is a Tag?

A tag is a **human-readable label** attached to an image. The full image reference format is:

```
[registry/][repository:]name[:tag]
```

Examples:
```
myapp:1.0
myapp:latest
docker.io/library/python:3.11-slim
ghcr.io/myorg/myapp:v2.3.1
```

### Tagging During Build

```bash
# Single tag
docker build -t myapp:1.0 .

# Multiple tags
docker build -t myapp:1.0 -t myapp:latest .

# With registry prefix
docker build -t registry.example.com/myapp:1.0 .
```

### Tagging After Build

```bash
docker tag myapp:1.0 myapp:production
docker tag myapp:1.0 registry.example.com/myapp:1.0
```

### Tag Best Practices

| Practice                        | Example                  |
|---------------------------------|--------------------------|
| Use semantic versioning         | `myapp:1.2.3`            |
| Use Git commit hash             | `myapp:a1b2c3d`          |
| Avoid relying solely on latest  | `myapp:latest` is mutable|
| Include environment info        | `myapp:1.0-prod`         |

---

## BuildKit — The Modern Builder

**BuildKit** is the next-generation image builder for Docker, enabled by default since Docker 23.0.

### Enabling BuildKit

```bash
# Environment variable
DOCKER_BUILDKIT=1 docker build .

# Or set in Docker daemon config (/etc/docker/daemon.json)
{
  "features": {
    "buildkit": true
  }
}
```

### BuildKit Advantages Over Legacy Builder

| Feature                    | Legacy Builder       | BuildKit              |
|----------------------------|----------------------|-----------------------|
| Parallel stage execution   | ❌ Sequential        | ✅ Parallel           |
| Cache export/import        | ❌ No                | ✅ Yes                |
| Build secrets              | ❌ No                | ✅ `--mount=type=secret` |
| SSH forwarding             | ❌ No                | ✅ `--mount=type=ssh` |
| Output formats             | Image only           | Image, tar, local dir |
| Progress output            | Basic                | Rich, colored         |
| Garbage collection         | Manual               | Automatic             |

### BuildKit-Specific Features

**Secret Mounting** (keeps secrets out of layers):
```dockerfile
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
```

```bash
docker build --secret id=mysecret,src=./secret.txt .
```

**Cache Mounting** (persist package manager cache across builds):
```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

**SSH Mounting** (forward SSH agent for private repos):
```dockerfile
RUN --mount=type=ssh git clone git@github.com:private/repo.git
```

```bash
docker build --ssh default .
```

---

## `.dockerignore` File

The `.dockerignore` file tells Docker which files to **exclude** from the build context.

### Syntax

```
# Comments start with #
node_modules
.git
*.log
__pycache__
.env
dist/
build/
*.md
!README.md
```

### Pattern Rules

| Pattern         | Matches                                         |
|-----------------|--------------------------------------------------|
| `*.log`         | All `.log` files in root                         |
| `**/*.log`      | All `.log` files in any directory                |
| `temp?`         | `temp1`, `tempa`, etc.                           |
| `!README.md`    | Exception — do NOT ignore `README.md`            |
| `*/temp*`       | Files starting with `temp` in one subdirectory   |

### Impact on Build Performance

```
Without .dockerignore:
  Sending build context to Docker daemon  450MB
  Build time: 2m 30s

With .dockerignore:
  Sending build context to Docker daemon  5.2MB
  Build time: 45s
```

---

## Build Arguments (`ARG`)

Build arguments allow you to pass **variables at build time** that are not persisted in the final image.

### Usage

```dockerfile
ARG PYTHON_VERSION=3.11
FROM python:${PYTHON_VERSION}-slim

ARG APP_ENV=production
RUN echo "Building for $APP_ENV"
```

```bash
docker build --build-arg PYTHON_VERSION=3.12 --build-arg APP_ENV=staging .
```

### ARG vs ENV

| Feature     | ARG                              | ENV                              |
|-------------|----------------------------------|----------------------------------|
| Available   | During build only                | During build AND at runtime      |
| Set via     | `--build-arg` flag               | Dockerfile `ENV` or `docker run -e` |
| Persisted   | ❌ Not in final image            | ✅ In final image                |
| Use case    | Build-time configuration         | Runtime configuration            |

### ARG Scope

```dockerfile
ARG VERSION=1.0          # Available before FROM
FROM python:3.11
ARG VERSION              # Must re-declare after FROM to use
RUN echo $VERSION
```

> **Important:** `ARG` values declared before `FROM` are only available in the `FROM` instruction itself. To use them later, re-declare the `ARG` after `FROM`.

---

## Best Practices for `docker build`

### 1. Minimize the Number of Layers

```dockerfile
# Bad — 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good — 1 layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 2. Use Small Base Images

| Base Image          | Size     |
|---------------------|----------|
| `ubuntu:22.04`      | ~77 MB   |
| `debian:bookworm-slim` | ~74 MB |
| `python:3.11`       | ~920 MB  |
| `python:3.11-slim`  | ~120 MB  |
| `python:3.11-alpine`| ~50 MB   |
| `alpine:3.18`       | ~7 MB    |
| `scratch`           | 0 MB     |

### 3. Order Instructions by Change Frequency

```dockerfile
# Least frequently changing → Most frequently changing
FROM python:3.11-slim
RUN apt-get update && apt-get install -y libpq-dev
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .                          # Changes most often — keep it last
CMD ["python", "app.py"]
```

### 4. Use `.dockerignore`

Always include a `.dockerignore` to reduce context size.

### 5. Don't Run as Root

```dockerfile
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser
```

### 6. Use COPY Instead of ADD

Use `ADD` only when you need its special features (URL download, tar extraction). Use `COPY` for everything else.

### 7. Pin Versions

```dockerfile
FROM python:3.11.7-slim    # Not python:latest
RUN pip install flask==3.0.0  # Not flask
```

---

## Common `docker build` Flags

| Flag                          | Description                                      | Example                                              |
|-------------------------------|--------------------------------------------------|------------------------------------------------------|
| `-t, --tag`                   | Name and tag the image                           | `docker build -t myapp:1.0 .`                        |
| `-f, --file`                  | Specify Dockerfile path                          | `docker build -f docker/Dockerfile.prod .`           |
| `--build-arg`                 | Pass build-time variable                         | `docker build --build-arg ENV=prod .`                |
| `--no-cache`                  | Build without using cache                        | `docker build --no-cache .`                          |
| `--target`                    | Build a specific stage in multi-stage build      | `docker build --target builder .`                    |
| `--platform`                  | Build for a specific platform                    | `docker build --platform linux/arm64 .`              |
| `--progress`                  | Set output type (auto, plain, tty)               | `docker build --progress=plain .`                    |
| `--secret`                    | Expose a secret to the build (BuildKit)          | `docker build --secret id=key,src=key.pem .`         |
| `--ssh`                       | Forward SSH agent (BuildKit)                     | `docker build --ssh default .`                       |
| `--cache-from`                | Use an external cache source                     | `docker build --cache-from myapp:cache .`            |
| `--output`                    | Set build output destination (BuildKit)          | `docker build --output type=local,dest=./out .`      |
| `--squash`                    | Squash all layers into one (experimental)        | `docker build --squash .`                            |

---

## Troubleshooting Build Failures

### Common Errors and Solutions

#### 1. "COPY failed: file not found"
```
COPY failed: file not found in build context or excluded by .dockerignore
```
**Causes:**
- File is not in the build context directory.
- File is excluded by `.dockerignore`.
- File path is relative to build context, not the Dockerfile location.

**Fix:** Verify the file exists in the build context and is not in `.dockerignore`.

#### 2. "returned a non-zero code"
```
The command '/bin/sh -c apt-get install -y xyz' returned a non-zero code: 100
```
**Causes:**
- Package not found or misspelled.
- Missing `apt-get update` before `apt-get install`.
- Network issues during build.

**Fix:** Ensure `apt-get update` runs before `install`, and verify package names.

#### 3. "no space left on device"
```
Error: write /var/lib/docker/...: no space left on device
```
**Fix:**
```bash
docker system prune -a        # Remove unused data
docker builder prune           # Remove build cache
```

#### 4. Slow Builds
**Causes:**
- Large build context (no `.dockerignore`).
- Poor instruction ordering (cache not utilized).
- Downloading dependencies every time.

**Fix:** Add `.dockerignore`, reorder instructions, use cache mounts.

---

## Complete Example Walkthrough

Let's walk through a complete example of building a Python Flask application.

### Project Structure

```
flask-app/
├── Dockerfile
├── .dockerignore
├── requirements.txt
├── app/
│   ├── __init__.py
│   ├── main.py
│   └── templates/
│       └── index.html
└── tests/
    └── test_main.py
```

### `.dockerignore`

```
.git
__pycache__
*.pyc
.env
venv/
tests/
*.md
.dockerignore
```

### `Dockerfile`

```dockerfile
# ===== Stage 1: Build and Test =====
FROM python:3.11-slim AS builder

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies (cached separately)
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Copy application source
COPY app/ ./app/

# ===== Stage 2: Production =====
FROM python:3.11-slim AS production

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Set working directory
WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copy application
COPY --from=builder /app/app ./app/

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Run the application
CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]
```

### Building the Image

```bash
# Build with tag
docker build -t flask-app:1.0 .

# Build output:
# [+] Building 45.2s (14/14) FINISHED
#  => [internal] load build context                          0.1s
#  => [builder 1/5] FROM python:3.11-slim@sha256:abc123     5.2s
#  => [builder 2/5] WORKDIR /app                            0.1s
#  => [builder 3/5] RUN apt-get update && ...               8.3s
#  => [builder 4/5] COPY requirements.txt .                 0.1s
#  => [builder 5/5] RUN pip install --no-cache-dir ...     15.4s
#  => [builder 6/6] COPY app/ ./app/                        0.1s
#  => [production 1/4] RUN groupadd -r appgroup ...         1.2s
#  => [production 2/4] WORKDIR /app                         0.1s
#  => [production 3/4] COPY --from=builder /root/.local ... 2.1s
#  => [production 4/4] COPY --from=builder /app/app ...     0.1s
#  => exporting to image                                    1.3s
```

### Inspecting the Built Image

```bash
# View image details
docker inspect flask-app:1.0

# View image history (layers)
docker history flask-app:1.0

# View image size
docker images flask-app:1.0
# REPOSITORY   TAG   IMAGE ID       SIZE
# flask-app    1.0   5e6f7a8b9c0d   145MB

# Run the container
docker run -d -p 5000:5000 --name myflask flask-app:1.0
```

---

## Summary

The `docker build` process transforms a **Dockerfile** and a **build context** into a **layered, immutable Docker image** through the following pipeline:

```
Dockerfile + Build Context
        ↓
  Docker Client (packages context)
        ↓
  Docker Daemon (parses Dockerfile)
        ↓
  Layer-by-Layer Execution
  ├── FROM → Pull/use base image
  ├── RUN  → Execute in temp container → capture diff
  ├── COPY → Add files from context
  └── CMD  → Set metadata
        ↓
  Cache Check (reuse or rebuild)
        ↓
  Final Image (tagged and stored locally)
```

**Key Takeaways:**
- Every `RUN`, `COPY`, and `ADD` instruction creates a **new layer**.
- Docker uses a **build cache** to avoid redundant work — order your instructions wisely.
- Use **multi-stage builds** to keep production images small and secure.
- Always use a **`.dockerignore`** file to minimize build context.
- **BuildKit** provides significant improvements — parallel builds, secrets, and caching.
- **Pin versions** for reproducible builds.
- **Never run as root** in production containers.
