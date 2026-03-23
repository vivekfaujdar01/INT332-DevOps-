# Docker Layering & Filesystem Deep Dive

To efficiently manage storage and build times, Docker employs a highly optimized, layered filesystem architecture. This allows different images to share common data and ensures running containers can operate securely without heavily burdening the host machine's physical storage.

Understanding this mechanism is key to writing efficient `Dockerfiles`.

---

## 1. The Union File System (UnionFS)

At the core of Docker's storage magic is the **Union File System (UnionFS)** (commonly implemented using the `overlay2` storage driver on modern Linux distributions).

A traditional filesystem maps a single directory tree to a physical disk partition. A UnionFS, however, is capable of taking multiple separate directories (the "layers") and magically merging them together so they appear to the user (the container process) as a single, unified directory structure.

*   If two layers contain a file with the exact same name and path, the file in the *higher* layer obscures (hides) the file in the *lower* layer. The container only sees the topmost version.

---

## 2. Image Layers (Read-Only)

When you look at a Docker image, you are actually looking at a stack of read-only layers.

*   **Dockerfile Instructions:** Every command in a `Dockerfile` that modifies the filesystem (like `RUN`, `COPY`, or `ADD`) generates a brand-new, independent layer. 
*   **The Build Process:**
    ```dockerfile
    FROM ubuntu:20.04           # Layer 1: Base Ubuntu OS
    RUN apt-get update          # Layer 2: APT package lists
    RUN apt-get install python3 # Layer 3: Python binaries
    COPY . /app                 # Layer 4: Your application code
    ```
    Docker executes these from top to bottom. It takes Layer 1, spins up a temporary container, runs Layer 2's command inside it, captures the resulting filesystem changes, and saves those changes as Layer 2. It repeats this process until finished.
*   **Sharing and Caching:** Because these layers are read-only and immutable, they can be safely shared. If you build two completely different applications that both start with `FROM ubuntu:20.04`, the host machine only stores the Ubuntu base layer once. Furthermore, if you change your application code (Layer 4) and rebuild, Docker intelligently reuses the cached versions of Layers 1, 2, and 3, making the build incredibly fast.

---

## 3. The Container Layer (Read-Write)

We know images are read-only, but applications frequently need to write logs, upload files, or alter configuration at runtime. How does Docker handle this?

When you use `docker run` to start a container from an image, the Docker Engine adds a very thin, **Read-Write (R/W) layer** to the very top of the underlying read-only image stack.

All changes made to the running container—writing a new file, modifying an existing one, or deleting a file—are written exclusively to this thin, ephemeral topmost layer.

*   When you stop a container, the R/W layer persists (you can restart it later).
*   When you **delete** a container using `docker rm`, this top R/W layer is permanently destroyed. The underlying image layers remain perfectly untouched.

---

## 4. The Copy-on-Write (CoW) Strategy

What happens if an application inside a container wants to modify a file that exists deep down in one of the read-only image layers (for example, tweaking an `/etc/nginx/nginx.conf` file provided by the base image)?

Docker handles this using a highly efficient strategy called **Copy-on-Write (CoW)**:

1.  **Search:** The storage driver searches down through the image layers, starting from the top, until it finds the file that needs to be modified.
2.  **Copy Up:** Once found, the driver copies the *entire file* up from that lower read-only layer into the very top Read-Write container layer. (This happens the very first time the file is written to).
3.  **Modify:** The container then modifies the newly copied version of the file in the top layer.
4.  **Obscure:** Because the UnionFS always shows the top-most version of a file, the container now only sees the heavily modified copy in the R/W layer. The original file sits untouched in the read-only layer below.

### Why Volumes are Important for Performance
While Copy-on-Write is brilliant for preserving image integrity, it introduces performance overhead. 

If your application does a lot of heavy I/O—like a busy PostgreSQL database constantly rewriting large data files—the constant "copying up" of files into the container's R/W layer will cause severe performance degradation. Furthermore, increasing the size of the container's R/W layer makes the container heavier on disk.

This is exactly why you use **Docker Volumes** for databases or heavy logs. A volume completely bypasses the UnionFS and Copy-on-Write mechanism, writing directly to the host machine's filesystem at native speeds.

---

## Relevant Docker Commands

*   **`docker diff <container>`**: Inspects changes (additions, modifications, deletions) made to files or directories on a container's topmost read-write filesystem compared to the underlying image.
*   **`docker volume create <volume>`**: Explicitly creates a new named volume on the host system to be used for persistent data.
*   **`docker volume ls`**: Lists all Docker volumes currently managed by the local Docker daemon.
*   **`docker volume inspect <volume>`**: Displays detailed information about a volume, including its exact mount point on the host filesystem.
*   **`docker volume rm <volume>`**: Permanently deletes a Docker volume and all the persistent data within it.
*   **`docker run -v <volume_name>:<container_path> <image>`** (or `--mount`): Runs a container and mounts a volume into its filesystem, bypassing the UnionFS for that specific directory path.
