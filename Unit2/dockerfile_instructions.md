# Dockerfile Writing & Basic Instructions

## Table of Contents
- [What is a Dockerfile?](#what-is-a-dockerfile)
- [Dockerfile Syntax Rules](#dockerfile-syntax-rules)
- [Basic Instructions](#basic-instructions)
  - [FROM](#1-from)
  - [RUN](#2-run)
  - [COPY](#3-copy)
  - [ADD](#4-add)
  - [CMD](#5-cmd)
  - [ENTRYPOINT](#6-entrypoint)
  - [WORKDIR](#7-workdir)
  - [ENV](#8-env)
  - [EXPOSE](#9-expose)
  - [VOLUME](#10-volume)
  - [LABEL](#11-label)
  - [ARG](#12-arg)
  - [USER](#13-user)
  - [HEALTHCHECK](#14-healthcheck)
  - [SHELL](#15-shell)
- [CMD vs ENTRYPOINT](#cmd-vs-entrypoint)
- [COPY vs ADD](#copy-vs-add)
- [Complete Dockerfile Example](#complete-dockerfile-example)
- [Best Practices](#best-practices)

---

## What is a Dockerfile?

A **Dockerfile** is a text-based script that contains a series of **instructions** and **arguments** used to automate the creation of a Docker image. Each instruction in a Dockerfile creates a new **layer** in the image, and Docker caches these layers to speed up subsequent builds.

### Key Characteristics:
- The file is named `Dockerfile` (no extension) by default.
- Instructions are **not case-sensitive**, but by convention they are written in **UPPERCASE** to distinguish them from arguments.
- Docker processes instructions **sequentially** from top to bottom.
- Each instruction creates a **read-only layer** in the final image.
- Comments start with `#`.

### Basic Structure:
```dockerfile
# Comment
INSTRUCTION arguments
```

---

## Dockerfile Syntax Rules

| Rule | Description |
|------|-------------|
| **File name** | Must be named `Dockerfile` (default) or specified with `-f` flag |
| **Comments** | Lines starting with `#` are treated as comments |
| **Order matters** | Instructions are executed top-to-bottom |
| **First instruction** | Must be `FROM` (or `ARG` before `FROM`) |
| **Case convention** | Instructions in UPPERCASE, arguments in lowercase |
| **Line continuation** | Use `\` to break long commands across multiple lines |

---

## Basic Instructions

### 1. FROM

**Purpose:** Sets the **base image** for the build. Every Dockerfile **must** begin with a `FROM` instruction (except `ARG` which can precede it).

**Syntax:**
```dockerfile
FROM <image>[:<tag>] [AS <name>]
```

**How it works:**
- Pulls the specified image from Docker Hub (or a specified registry).
- The `tag` specifies the version; if omitted, `latest` is used.
- The `AS` keyword is used in **multi-stage builds** to name a build stage.

**Examples:**
```dockerfile
# Use Ubuntu 22.04 as base
FROM ubuntu:22.04

# Use official Node.js image
FROM node:18-alpine

# Use Python slim variant
FROM python:3.11-slim

# Multi-stage build with named stages
FROM node:18 AS builder
# ... build steps ...
FROM nginx:alpine AS production
# ... production steps ...
```

**Key Points:**
- A Dockerfile can have **multiple** `FROM` instructions for multi-stage builds.
- Always prefer **specific tags** (e.g., `node:18`) over `latest` for reproducibility.
- Use **slim** or **alpine** variants to reduce image size.

---

### 2. RUN

**Purpose:** Executes commands **during the image build process**. Used to install packages, compile code, create directories, etc.

**Syntax:**
```dockerfile
# Shell form (runs in /bin/sh -c)
RUN <command>

# Exec form
RUN ["executable", "param1", "param2"]
```

**How it works:**
- Each `RUN` instruction creates a **new layer** in the image.
- Shell form runs the command through a shell (`/bin/sh -c` on Linux).
- Exec form runs the command directly without a shell.

**Examples:**
```dockerfile
# Install packages
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    git \
    && rm -rf /var/lib/apt/lists/*

# Create a directory
RUN mkdir -p /app/data

# Run multiple commands (combined to reduce layers)
RUN pip install --no-cache-dir flask gunicorn && \
    mkdir -p /var/log/app && \
    chmod 755 /var/log/app

# Exec form
RUN ["pip", "install", "flask"]
```

**Key Points:**
- **Combine related commands** with `&&` to reduce the number of layers.
- Always clean up caches (`rm -rf /var/lib/apt/lists/*`) to keep the image small.
- `RUN` executes at **build time**, not at container runtime.

---

### 3. COPY

**Purpose:** Copies files and directories from the **host machine** (build context) into the image filesystem.

**Syntax:**
```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--from=<name>] <src>... <dest>
```

**How it works:**
- Copies files from the build context (the directory where `docker build` is run).
- Paths are relative to the build context.
- Destination is an absolute path or relative to `WORKDIR`.

**Examples:**
```dockerfile
# Copy a single file
COPY package.json /app/package.json

# Copy multiple files
COPY package.json package-lock.json /app/

# Copy an entire directory's contents
COPY . /app

# Copy with ownership
COPY --chown=node:node . /app

# Copy from a build stage (multi-stage)
COPY --from=builder /app/dist /usr/share/nginx/html
```

**Key Points:**
- Preferred over `ADD` for simple file copying.
- Does **not** extract archives or download remote URLs.
- Supports wildcard patterns: `COPY *.json /app/`
- Respects `.dockerignore` file to exclude files from the build context.

---

### 4. ADD

**Purpose:** Similar to `COPY`, but with **additional features** — it can extract tar archives and download files from remote URLs.

**Syntax:**
```dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
```

**How it works:**
- If `<src>` is a **local tar archive** (e.g., `.tar`, `.tar.gz`), it is automatically extracted into the destination.
- If `<src>` is a **URL**, Docker downloads the file and places it at the destination.
- Otherwise, behaves like `COPY`.

**Examples:**
```dockerfile
# Auto-extract a tar archive
ADD app.tar.gz /app

# Download a file from a URL
ADD https://example.com/config.json /app/config.json

# Simple file copy (use COPY instead for this)
ADD index.html /var/www/html/
```

**Key Points:**
- Use `COPY` unless you specifically need tar extraction or URL downloading.
- Remote URL downloads do **not** get cached like regular layers.
- Docker recommends `COPY` for transparency and simplicity.

---

### 5. CMD

**Purpose:** Specifies the **default command** to run when a container starts. It can be **overridden** by providing a command at `docker run`.

**Syntax:**
```dockerfile
# Exec form (preferred)
CMD ["executable", "param1", "param2"]

# Shell form
CMD command param1 param2

# As default parameters to ENTRYPOINT
CMD ["param1", "param2"]
```

**How it works:**
- Only the **last** `CMD` instruction in a Dockerfile takes effect.
- If the user provides a command in `docker run`, it **overrides** `CMD`.
- When used with `ENTRYPOINT`, `CMD` provides default arguments.

**Examples:**
```dockerfile
# Run a Python application
CMD ["python", "app.py"]

# Start a web server
CMD ["nginx", "-g", "daemon off;"]

# Using shell form
CMD echo "Hello, Docker!"

# Default arguments for ENTRYPOINT
ENTRYPOINT ["python"]
CMD ["app.py"]
```

**Key Points:**
- Use the **exec form** `["executable", "param"]` to avoid shell processing issues.
- There can be only **one** `CMD`; if multiple are specified, the last one wins.
- `CMD` is executed at **runtime**, not at build time.

---

### 6. ENTRYPOINT

**Purpose:** Configures the container to run as an **executable**. Unlike `CMD`, it is **not easily overridden** by `docker run` arguments.

**Syntax:**
```dockerfile
# Exec form (preferred)
ENTRYPOINT ["executable", "param1", "param2"]

# Shell form
ENTRYPOINT command param1 param2
```

**How it works:**
- Sets the main command that always runs when the container starts.
- Arguments passed via `docker run` are **appended** to the `ENTRYPOINT` command.
- Can be overridden only with `docker run --entrypoint`.

**Examples:**
```dockerfile
# Container always runs Python
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myimage         → runs "python app.py"
# docker run myimage test.py → runs "python test.py"

# Container always runs Node
ENTRYPOINT ["node"]
CMD ["server.js"]

# Container as an executable
ENTRYPOINT ["curl"]
CMD ["https://example.com"]
```

**Key Points:**
- Use when you want the container to always run a specific executable.
- Combine with `CMD` to provide default arguments that users can override.
- The exec form is preferred to ensure proper signal handling.

---

### 7. WORKDIR

**Purpose:** Sets the **working directory** inside the container for subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD` instructions.

**Syntax:**
```dockerfile
WORKDIR /path/to/directory
```

**How it works:**
- If the directory doesn't exist, Docker **creates** it automatically.
- Can be used multiple times; each sets a new working directory.
- Supports environment variable substitution.

**Examples:**
```dockerfile
# Set working directory
WORKDIR /app

# Subsequent COPY is relative to /app
COPY . .

# Can change multiple times
WORKDIR /app
RUN npm install
WORKDIR /app/src
RUN npm run build

# Using environment variables
ENV APP_HOME=/application
WORKDIR $APP_HOME
```

**Key Points:**
- Always use `WORKDIR` instead of `RUN cd /path` — the `cd` in `RUN` only affects that single `RUN` layer.
- Use **absolute paths** for clarity.
- Sets the default directory when the container starts.

---

### 8. ENV

**Purpose:** Sets **environment variables** inside the container that persist in the running container.

**Syntax:**
```dockerfile
ENV <key>=<value>
ENV <key> <value>          # legacy syntax
```

**How it works:**
- Environment variables are available during build and at runtime.
- Can be used in subsequent Dockerfile instructions via `$VARIABLE` or `${VARIABLE}`.
- Can be overridden at runtime with `docker run -e`.

**Examples:**
```dockerfile
# Set single variable
ENV NODE_ENV=production

# Set multiple variables
ENV APP_HOME=/app \
    APP_PORT=3000 \
    APP_VERSION=1.0.0

# Use in subsequent instructions
ENV APP_DIR=/opt/myapp
WORKDIR $APP_DIR
COPY . $APP_DIR

# Database configuration
ENV DB_HOST=localhost \
    DB_PORT=5432 \
    DB_NAME=mydb
```

**Key Points:**
- Environment variables persist in the running container.
- Use `ARG` instead of `ENV` for build-time-only variables.
- Can be overridden at runtime: `docker run -e NODE_ENV=development myimage`.

---

### 9. EXPOSE

**Purpose:** Documents which **ports** the container listens on at runtime. It does **not** actually publish the port.

**Syntax:**
```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

**How it works:**
- Acts as **documentation** between the image builder and the container runner.
- Does not make the port accessible from outside the container.
- To actually publish the port, use `-p` flag with `docker run`.

**Examples:**
```dockerfile
# Expose a single port
EXPOSE 80

# Expose multiple ports
EXPOSE 80 443

# Expose with protocol specification
EXPOSE 80/tcp
EXPOSE 53/udp

# Common web application ports
EXPOSE 3000    # Node.js
EXPOSE 8080    # Alternative HTTP
EXPOSE 5432    # PostgreSQL
```

**Key Points:**
- `EXPOSE` is purely **informational** / documentational.
- You still need `docker run -p 8080:80` to map host port to container port.
- Use `docker run -P` to publish all exposed ports to random host ports.

---

### 10. VOLUME

**Purpose:** Creates a **mount point** and marks it as holding externally mounted volumes or data that should persist beyond the container's lifecycle.

**Syntax:**
```dockerfile
VOLUME ["/data"]
VOLUME /data /logs
```

**How it works:**
- Creates a mount point inside the container.
- Data written to this path is stored outside the container's writable layer.
- Docker manages the volume automatically if no explicit mount is provided at `docker run`.

**Examples:**
```dockerfile
# Single volume
VOLUME /app/data

# Multiple volumes
VOLUME ["/var/log", "/app/uploads"]

# Database data directory
VOLUME /var/lib/mysql

# Application logs and uploads
VOLUME /app/logs
VOLUME /app/uploads
```

**Key Points:**
- Data in volumes **persists** even after the container is removed.
- Use `docker run -v /host/path:/container/path` for explicit mount binding.
- Changes to volume directories in the Dockerfile **after** the `VOLUME` instruction are **discarded**.

---

### 11. LABEL

**Purpose:** Adds **metadata** to an image in the form of key-value pairs.

**Syntax:**
```dockerfile
LABEL <key>=<value> <key>=<value> ...
```

**Examples:**
```dockerfile
LABEL maintainer="vivek@example.com"
LABEL version="1.0"
LABEL description="My web application"

# Multiple labels in one instruction
LABEL maintainer="vivek@example.com" \
      version="1.0" \
      description="My web application" \
      org.opencontainers.image.source="https://github.com/user/repo"
```

---

### 12. ARG

**Purpose:** Defines **build-time variables** that users can pass at build time with `docker build --build-arg`.

**Syntax:**
```dockerfile
ARG <name>[=<default value>]
```

**Examples:**
```dockerfile
# With default value
ARG NODE_VERSION=18

FROM node:${NODE_VERSION}-alpine

# Without default (must be provided)
ARG API_KEY

# Used during build
ARG APP_ENV=production
RUN echo "Building for $APP_ENV"
```

**Key Points:**
- `ARG` values are **not** available in the running container (unlike `ENV`).
- Can be used **before** `FROM` to parameterize the base image.
- Pass at build time: `docker build --build-arg NODE_VERSION=20 .`

---

### 13. USER

**Purpose:** Sets the **user** (and optionally group) for running subsequent `RUN`, `CMD`, and `ENTRYPOINT` instructions.

**Syntax:**
```dockerfile
USER <user>[:<group>]
```

**Examples:**
```dockerfile
# Create and switch to non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# Switch to specific UID
USER 1001
```

**Key Points:**
- Running containers as **non-root** is a security best practice.
- The user must exist in the image (create with `RUN adduser`).

---

### 14. HEALTHCHECK

**Purpose:** Tells Docker how to **check** if a container is still working properly.

**Syntax:**
```dockerfile
HEALTHCHECK [OPTIONS] CMD command
HEALTHCHECK NONE   # disables inherited healthcheck
```

**Examples:**
```dockerfile
# Check if web server is running
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Check database connectivity
HEALTHCHECK --interval=60s --timeout=5s \
  CMD pg_isready -U postgres || exit 1
```

| Option | Default | Description |
|--------|---------|-------------|
| `--interval` | 30s | Time between health checks |
| `--timeout` | 30s | Max time for a check to complete |
| `--retries` | 3 | Consecutive failures before "unhealthy" |
| `--start-period` | 0s | Grace period for container startup |

---

### 15. SHELL

**Purpose:** Overrides the **default shell** used for the shell form of `RUN`, `CMD`, and `ENTRYPOINT`.

**Syntax:**
```dockerfile
SHELL ["executable", "parameters"]
```

**Examples:**
```dockerfile
# Use bash instead of sh
SHELL ["/bin/bash", "-c"]
RUN echo "Now using bash"

# On Windows, use PowerShell
SHELL ["powershell", "-Command"]
RUN Write-Host "Hello from PowerShell"
```

---

## CMD vs ENTRYPOINT

Understanding the difference between `CMD` and `ENTRYPOINT` is critical:

| Feature | CMD | ENTRYPOINT |
|---------|-----|------------|
| **Purpose** | Default command/arguments | Main executable |
| **Override** | Easily overridden by `docker run` args | Requires `--entrypoint` flag to override |
| **Multiple** | Only last one takes effect | Only last one takes effect |
| **Combination** | Provides default args to ENTRYPOINT | Defines the fixed executable |
| **Use case** | Default behavior that users may change | Container should always run a specific program |

### Interaction Table:

| | No ENTRYPOINT | ENTRYPOINT exec_entry p1_entry | ENTRYPOINT ["exec_entry", "p1_entry"] |
|---|---|---|---|
| **No CMD** | Error, not allowed | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry |
| **CMD ["exec_cmd", "p1_cmd"]** | exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry exec_cmd p1_cmd |
| **CMD exec_cmd p1_cmd** | /bin/sh -c exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd |

---

## COPY vs ADD

| Feature | COPY | ADD |
|---------|------|-----|
| **Local files** | ✅ Yes | ✅ Yes |
| **Remote URLs** | ❌ No | ✅ Yes |
| **Auto-extract tar** | ❌ No | ✅ Yes |
| **Transparency** | ✅ Clear intent | ⚠️ Implicit behavior |
| **Recommendation** | ✅ Preferred | Use only when extraction needed |

---

## Complete Dockerfile Example

Here's a complete, production-ready Dockerfile for a Node.js application:

```dockerfile
# ============================================
# Stage 1: Build
# ============================================
FROM node:18-alpine AS builder

# Metadata
LABEL maintainer="vivek@example.com"
LABEL description="Production Node.js Application"

# Build arguments
ARG NODE_ENV=production
ENV NODE_ENV=$NODE_ENV

# Set working directory
WORKDIR /app

# Copy dependency files first (for layer caching)
COPY package.json package-lock.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application source code
COPY . .

# Build the application
RUN npm run build

# ============================================
# Stage 2: Production
# ============================================
FROM node:18-alpine AS production

# Set environment variables
ENV NODE_ENV=production \
    PORT=3000

# Set working directory
WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy built files from builder stage
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

# Create volume for logs
VOLUME /app/logs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

# Switch to non-root user
USER appuser

# Start the application
ENTRYPOINT ["node"]
CMD ["dist/server.js"]
```

### Build and Run:
```bash
# Build the image
docker build -t my-node-app:1.0 .

# Run the container
docker run -d \
  --name my-app \
  -p 8080:3000 \
  -v ./logs:/app/logs \
  -e DB_HOST=db.example.com \
  my-node-app:1.0

# Check health
docker inspect --format='{{.State.Health.Status}}' my-app
```

---

## Best Practices

### 1. Minimize Layers
```dockerfile
# ❌ Bad: Creates 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good: Creates 1 layer
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

### 2. Use Multi-Stage Builds
Separate build and runtime environments to reduce final image size.

### 3. Leverage Build Cache
```dockerfile
# ✅ Copy dependency files first, then source code
COPY package.json package-lock.json ./
RUN npm install
COPY . .
# This way, npm install is cached unless package files change
```

### 4. Use `.dockerignore`
```
node_modules
.git
.env
*.md
Dockerfile
docker-compose.yml
```

### 5. Use Specific Base Image Tags
```dockerfile
# ❌ Avoid
FROM node:latest

# ✅ Prefer
FROM node:18.17-alpine
```

### 6. Run as Non-Root User
Always create and switch to a non-root user for security.

### 7. Use COPY Over ADD
Unless you specifically need tar extraction or URL downloading.

### 8. One Process Per Container
Each container should run a single process following the microservices principle.

---

> **Summary:** A Dockerfile is a blueprint for building Docker images. Mastering its instructions — `FROM`, `RUN`, `COPY`, `ADD`, `CMD`, `ENTRYPOINT`, `WORKDIR`, `ENV`, `EXPOSE`, and `VOLUME` — gives you full control over how your application is containerized, from base image selection to runtime behavior.
