# Docker Image Tagging, Versioning & Inspecting Images

## Table of Contents

1. [Image Tagging](#1-image-tagging)
2. [Image Versioning](#2-image-versioning)
3. [Inspecting Images — History](#3-inspecting-images--history)
4. [Inspecting Images — Layers](#4-inspecting-images--layers)
5. [Practical Examples](#5-practical-examples)
6. [Best Practices](#6-best-practices)
7. [Key Docker Commands Reference](#7-key-docker-commands-reference)

---

## 1. Image Tagging

### What is an Image Tag?

A **tag** is a human-readable label attached to a specific Docker image. It acts as a pointer to a particular image ID (a SHA-256 digest). Tags allow developers and operations teams to identify, organize, and retrieve specific versions of an image without memorizing long hash strings.

The full format of a Docker image reference is:

```
[registry/][repository/]image_name[:tag]
```

| Component   | Example              | Description                                      |
|-------------|----------------------|--------------------------------------------------|
| Registry    | `docker.io`          | The server hosting the image (default: Docker Hub)|
| Repository  | `library/nginx`      | Namespace or user/organization                   |
| Image Name  | `nginx`              | The name of the image                            |
| Tag         | `1.25`               | The version or variant label                     |

**Full example:** `docker.io/library/nginx:1.25-alpine`

### The `latest` Tag

- If no tag is specified when pulling or building, Docker defaults to `latest`.
- **`latest` is NOT automatically the most recent image** — it is simply a tag name. A maintainer must explicitly push an image with the `latest` tag.
- Relying on `latest` in production is **discouraged** because it leads to unpredictable deployments.

```bash
# Both of these pull the same image (assuming latest exists)
docker pull nginx
docker pull nginx:latest
```

### Tagging an Image

You can tag an image at build time or after it has been built.

#### Tagging at Build Time

```bash
docker build -t myapp:1.0 .
docker build -t myapp:latest .
```

You can apply **multiple tags** in a single build:

```bash
docker build -t myapp:1.0 -t myapp:latest .
```

#### Tagging an Existing Image

Use `docker tag` to create a new tag pointing to an existing image:

```bash
docker tag <source_image>:<source_tag> <target_image>:<target_tag>
```

**Example:**

```bash
docker tag myapp:1.0 myapp:stable
docker tag myapp:1.0 myregistry.com/myapp:1.0
```

> **Key point:** `docker tag` does **not** create a copy of the image. Both tags point to the same underlying image ID (layers are shared).

### Listing Image Tags

```bash
# List all local images (with tags)
docker images

# Filter by image name
docker images myapp

# Show image IDs only
docker images -q myapp
```

### Removing a Tag

```bash
docker rmi myapp:stable
```

- If the image has other tags, only the specified tag is removed; the image data remains.
- If it is the **last** tag referencing that image ID, the image layers are deleted (unless used by another image or container).

---

## 2. Image Versioning

### Why Version Docker Images?

Versioning gives you:

- **Reproducibility** — deploy the exact same image to any environment.
- **Rollback capability** — quickly revert to a known-good version.
- **Auditability** — trace which code is running in production.
- **Parallel environments** — run different versions side-by-side for testing.

### Common Versioning Strategies

#### a) Semantic Versioning (SemVer)

Follows the `MAJOR.MINOR.PATCH` convention:

| Version Change | When to Increment                 | Example        |
|----------------|-----------------------------------|----------------|
| MAJOR          | Breaking / incompatible changes   | `2.0.0`        |
| MINOR          | New features, backward-compatible | `1.3.0`        |
| PATCH          | Bug fixes, backward-compatible    | `1.3.2`        |

```bash
docker build -t myapp:1.0.0 .
docker build -t myapp:1.0.1 .   # bug fix
docker build -t myapp:1.1.0 .   # new feature
docker build -t myapp:2.0.0 .   # breaking change
```

**Multi-level tagging** is a common practice — tag the same image with multiple granularity levels:

```bash
docker tag myapp:1.2.3 myapp:1.2
docker tag myapp:1.2.3 myapp:1
docker tag myapp:1.2.3 myapp:latest
```

This way, `myapp:1` always points to the latest `1.x.x` release.

#### b) Git Commit SHA

Tags the image with the short (or full) Git commit hash:

```bash
docker build -t myapp:a3f8b2c .
```

- **Pros:** Guarantees a 1-to-1 mapping between code and image.
- **Cons:** Not human-friendly; hard to identify "which version is newer" at a glance.

#### c) Date-Based Versioning

```bash
docker build -t myapp:2026-03-26 .
docker build -t myapp:20260326-142500 .   # with timestamp
```

Useful for nightly builds or CI pipelines where builds happen frequently.

#### d) Branch/Environment-Based Tags

```bash
docker build -t myapp:dev .
docker build -t myapp:staging .
docker build -t myapp:production .
```

These are **mutable** tags — they get overwritten on each deployment to that environment.

#### e) Combined Strategy (Recommended for Production)

```bash
docker build \
  -t myapp:1.2.3 \
  -t myapp:1.2.3-a3f8b2c \
  -t myapp:latest .
```

This combines SemVer with the Git SHA for full traceability.

### Image Digests (Immutable References)

Every image pushed to a registry gets an **immutable digest** (content-addressable hash):

```
myapp@sha256:abc123def456...
```

```bash
# Pull by digest (guarantees exact image)
docker pull myapp@sha256:abc123def456...

# View digest of a local image
docker inspect --format='{{index .RepoDigests 0}}' myapp:1.0
```

Digests are the **only truly immutable** way to reference an image — tags can be overwritten.

---

## 3. Inspecting Images — History

### What is Image History?

The `docker history` command shows the **sequence of instructions** that were used to build an image. Each row represents a layer or metadata change in the image.

### Using `docker history`

```bash
docker history <image_name>:<tag>
```

**Example:**

```bash
docker history nginx:latest
```

**Sample Output:**

```
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
a8758716bb6a   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   STOPSIGNAL SIGQUIT                              0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY file:xyz /docker-entrypoint.d/30-tune…     4.62kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY file:abc /docker-entrypoint.d/20-envo…     1.04kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY file:def /docker-entrypoint.d/10-list…     2.12kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY file:ghi /docker-entrypoint.sh             1.62kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   RUN /bin/sh -c set -x && apt-get update...      92.3MB    buildkit.dockerfile.v0
<missing>      2 weeks ago   ENV NGINX_VERSION=1.25.4                        0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      2 weeks ago   /bin/sh -c #(nop) ADD file:... in /              74.8MB
```

### Understanding the Output

| Column       | Description                                                                 |
|--------------|-----------------------------------------------------------------------------|
| `IMAGE`      | Layer ID. `<missing>` means the layer was built on another machine          |
| `CREATED`    | When the layer was created                                                  |
| `CREATED BY` | The Dockerfile instruction that created this layer                          |
| `SIZE`       | Size added by this layer (0B for metadata-only instructions)                |
| `COMMENT`    | Additional info (e.g., `buildkit.dockerfile.v0`)                            |

### Useful Flags

```bash
# Show full, untruncated commands
docker history --no-trunc nginx:latest

# Output in quiet mode (layer IDs only)
docker history -q nginx:latest

# Format output as JSON
docker history --format "{{json .}}" nginx:latest

# Custom format
docker history --format "table {{.ID}}\t{{.Size}}\t{{.CreatedBy}}" nginx:latest
```

### Key Observations

- **Metadata instructions** (`CMD`, `ENV`, `EXPOSE`, `LABEL`, `ENTRYPOINT`, `STOPSIGNAL`, `WORKDIR`, `USER`) produce **0B layers** — they modify the image configuration but add no filesystem content.
- **Filesystem-modifying instructions** (`RUN`, `COPY`, `ADD`) produce layers with actual size.
- The history is shown **top-down** (most recent layer first, base image at the bottom).
- `<missing>` in the IMAGE column is normal for layers built remotely or pulled from a registry.

---

## 4. Inspecting Images — Layers

### What are Image Layers?

A Docker image is composed of **stacked, read-only filesystem layers**. Each layer represents a set of filesystem changes (files added, modified, or deleted) produced by a single Dockerfile instruction.

```
┌─────────────────────────────┐
│  Layer 5: COPY app.js .     │  ← Topmost layer (most recent)
├─────────────────────────────┤
│  Layer 4: RUN npm install   │
├─────────────────────────────┤
│  Layer 3: WORKDIR /app      │  ← 0B metadata layer
├─────────────────────────────┤
│  Layer 2: RUN apt-get ...   │
├─────────────────────────────┤
│  Layer 1: Base Image (Ubuntu)│  ← Bottom layer
└─────────────────────────────┘
```

### Inspecting Layers with `docker inspect`

```bash
docker inspect <image_name>:<tag>
```

This returns a **JSON object** containing all metadata for the image. Key sections for layer inspection:

#### Viewing the Layer Stack (RootFS)

```bash
docker inspect --format='{{json .RootFS}}' nginx:latest | python3 -m json.tool
```

**Sample Output:**

```json
{
    "Type": "layers",
    "Layers": [
        "sha256:aaa111...",
        "sha256:bbb222...",
        "sha256:ccc333...",
        "sha256:ddd444...",
        "sha256:eee555...",
        "sha256:fff666...",
        "sha256:ggg777..."
    ]
}
```

Each entry is the **diff ID** (content hash) of that layer. Layers are listed base-to-top.

#### Viewing Image Configuration

```bash
# Full config
docker inspect nginx:latest

# Specific fields
docker inspect --format='{{.Os}}' nginx:latest                    # OS
docker inspect --format='{{.Architecture}}' nginx:latest          # Architecture
docker inspect --format='{{json .Config.Env}}' nginx:latest       # Environment variables
docker inspect --format='{{json .Config.Cmd}}' nginx:latest       # Default command
docker inspect --format='{{json .Config.ExposedPorts}}' nginx:latest  # Exposed ports
docker inspect --format='{{json .Config.Labels}}' nginx:latest    # Labels
docker inspect --format='{{.Config.WorkingDir}}' nginx:latest     # Working directory
docker inspect --format='{{json .Config.Entrypoint}}' nginx:latest    # Entrypoint
docker inspect --format='{{json .Config.Volumes}}' nginx:latest   # Volumes
docker inspect --format='{{.Created}}' nginx:latest               # Creation timestamp
docker inspect --format='{{.Size}}' nginx:latest                  # Total size in bytes
```

### Layer Sharing and Caching

Docker **deduplicates layers** across images. If two images share the same base layer, that layer is stored only once on disk.

```bash
# See how much space is actually used vs. shared
docker system df -v
```

**Example scenario:**

```
Image A (myapp:1.0):   [Ubuntu] → [apt-get] → [npm install] → [COPY v1]
Image B (myapp:1.1):   [Ubuntu] → [apt-get] → [npm install] → [COPY v2]
```

Layers 1–3 are **shared**. Only layer 4 differs — saving significant disk space.

### Exploring Layer Contents with `docker save`

You can export an image as a tar archive and examine each layer's filesystem changes:

```bash
# Save image to a tar file
docker save myapp:1.0 -o myapp.tar

# Extract the tar
mkdir myapp_layers && tar -xf myapp.tar -C myapp_layers

# Explore the extracted content
ls myapp_layers/
```

Inside, you'll find:
- `manifest.json` — maps tags to layer directories
- `<layer_hash>/layer.tar` — the actual filesystem diff for each layer
- `<image_config>.json` — full image configuration

```bash
# Inspect a specific layer's filesystem changes
tar -tf myapp_layers/<layer_hash>/layer.tar
```

### Using `dive` for Interactive Layer Exploration

[`dive`](https://github.com/wagoodman/dive) is a popular third-party tool for exploring Docker image layers interactively.

```bash
# Install dive
# On Ubuntu/Debian:
wget https://github.com/wagoodman/dive/releases/download/v0.12.0/dive_0.12.0_linux_amd64.deb
sudo dpkg -i dive_0.12.0_linux_amd64.deb

# Analyze an image
dive nginx:latest
```

**Dive provides:**
- Layer-by-layer filesystem diff view
- Size breakdown per layer
- Wasted space analysis
- Image efficiency score

---

## 5. Practical Examples

### Example 1: Building, Tagging, and Inspecting a Custom Image

**Dockerfile:**

```dockerfile
FROM node:18-alpine
LABEL maintainer="dev@example.com"
LABEL version="1.0"
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

**Build with multiple tags:**

```bash
docker build -t myapi:1.0.0 -t myapi:latest .
```

**Inspect the build history:**

```bash
docker history myapi:1.0.0
```

```
IMAGE          CREATED          CREATED BY                                SIZE
f1a2b3c4d5e6   10 seconds ago   CMD ["node" "server.js"]                  0B
<missing>      10 seconds ago   EXPOSE map[3000/tcp:{}]                   0B
<missing>      10 seconds ago   COPY dir:abc123 in .                      15.2kB
<missing>      11 seconds ago   RUN npm install --production              48.3MB
<missing>      15 seconds ago   COPY file:def456 in .                     520B
<missing>      15 seconds ago   WORKDIR /app                              0B
<missing>      15 seconds ago   LABEL version=1.0                         0B
<missing>      15 seconds ago   LABEL maintainer=dev@example.com          0B
<missing>      3 weeks ago      ...node:18-alpine base layers...          ...
```

**Inspect layers:**

```bash
docker inspect --format='{{json .RootFS.Layers}}' myapi:1.0.0 | python3 -m json.tool
```

### Example 2: Comparing Two Image Versions

```bash
# Compare sizes
docker images myapi

# Compare history
diff <(docker history --no-trunc myapi:1.0.0) <(docker history --no-trunc myapi:1.1.0)

# Compare layer counts
echo "v1.0.0 layers: $(docker inspect --format='{{len .RootFS.Layers}}' myapi:1.0.0)"
echo "v1.1.0 layers: $(docker inspect --format='{{len .RootFS.Layers}}' myapi:1.1.0)"
```

### Example 3: Tagging for a Private Registry

```bash
# Tag for pushing to a private registry
docker tag myapi:1.0.0 registry.example.com/team/myapi:1.0.0
docker tag myapi:1.0.0 registry.example.com/team/myapi:latest

# Push both tags
docker push registry.example.com/team/myapi:1.0.0
docker push registry.example.com/team/myapi:latest

# Pull from private registry
docker pull registry.example.com/team/myapi:1.0.0
```

---

## 6. Best Practices

### Tagging Best Practices

| Practice                                    | Reason                                                    |
|---------------------------------------------|-----------------------------------------------------------|
| Never rely solely on `latest`               | It is mutable and can cause unexpected deployments        |
| Use semantic versioning for releases        | Provides clear upgrade paths and compatibility signals    |
| Include Git SHA in CI/CD tags               | Enables code-to-image traceability                        |
| Use multi-level tags (`1.2.3`, `1.2`, `1`)  | Allows users to pin at different granularity levels       |
| Tag immutable releases in CI/CD             | Prevents accidental overwrites                            |
| Use digests in production Dockerfiles       | Guarantees reproducible builds: `FROM nginx@sha256:...`   |

### Inspection Best Practices

| Practice                                      | Reason                                                  |
|-----------------------------------------------|---------------------------------------------------------|
| Review history before deploying               | Catch unexpected layers or secrets                      |
| Monitor image sizes with `docker system df`   | Prevent disk space issues                               |
| Use `--no-trunc` for debugging                | Truncated commands hide important details               |
| Use `dive` for in-depth analysis              | Visual tool makes layer exploration easier              |
| Compare layer counts between versions         | Detect unnecessary layer additions                      |
| Check for secrets in layers                   | Sensitive data in any layer is permanently recoverable  |

### Security Considerations

- **Never embed secrets** (API keys, passwords) in image layers — even if deleted in a later layer, they remain in the image history.
- Use **multi-stage builds** to avoid leaking build-time secrets.
- Use `docker history --no-trunc` to audit images for accidentally included sensitive data.
- Prefer **minimal base images** (`alpine`, `distroless`) to reduce attack surface.

---

## 7. Key Docker Commands Reference

| Command                                          | Description                                      |
|--------------------------------------------------|--------------------------------------------------|
| `docker build -t name:tag .`                     | Build and tag an image                           |
| `docker tag source:tag target:tag`               | Create a new tag for an existing image           |
| `docker images`                                  | List all local images with tags                  |
| `docker rmi name:tag`                            | Remove a specific tag                            |
| `docker history name:tag`                        | Show the build history of an image               |
| `docker history --no-trunc name:tag`             | Show full, untruncated history                   |
| `docker inspect name:tag`                        | Show detailed JSON metadata for an image         |
| `docker inspect --format='...' name:tag`         | Extract specific fields from image metadata      |
| `docker save name:tag -o file.tar`               | Export an image as a tar archive                 |
| `docker system df -v`                            | Show disk usage with layer sharing details       |
| `docker pull name@sha256:...`                    | Pull an image by its immutable digest            |
| `dive name:tag`                                  | Interactively explore image layers (third-party) |

---

> **Summary:** Image tagging and versioning provide a structured way to manage different releases of your Docker images. Inspecting history and layers helps you understand what is inside an image, debug build issues, optimize image size, and ensure no sensitive data is accidentally included.
