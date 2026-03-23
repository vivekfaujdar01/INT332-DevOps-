# Container Images and Distribution

A container runtime needs instructions and files to run an application. This is where **Container Images**, **Layers**, and **Registries** come into play. They form the packaging and distribution mechanism of the container ecosystem.

---

## 1. Container Images & Layers

A container image is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries, and settings.

Crucially, an image is **read-only**. Once built, it cannot be modified. If you want to change an image, you must build a new one.

### The Layered Architecture
Container images are not single, massive files. They are constructed from a series of individual **layers**. 

When you define an image using a tool like a `Dockerfile`, every instruction (like `RUN`, `COPY`, `ADD`) creates a new layer.

*   **Base Layer:** An image usually starts with a base layer (e.g., a minimal Ubuntu or Alpine Linux distribution).
*   **Sequential Stacking:** Subsequent instructions add new layers on top. For instance, if you run a command to install Python, that installation becomes a new, separate layer stacked on top of the base OS layer. If you then copy your application code, that becomes another layer on top.

### Union File Systems (UnionFS)
How does the runtime present all these separate layers to the container as a single, coherent file system? It uses a **Union File System** (e.g., OverlayFS, AUFS).

A UnionFS takes multiple directories (the individual layers) and transparently overlays them onto a single directory. The layers remain separate on the host's hard drive, but the container perceives them as one unified filesystem.

### The Writable Container Layer
Because images are read-only, how can an application running inside a container write logs, save files, or modify its environment?

When a container is started from an image, the container engine adds a very thin, **read-write layer** on top of the stack of read-only image layers. 
*   Any files the container creates or modifies are written to this top, writable layer.
*   If the container tries to modify a file that exists in a lower, read-only layer, the runtime performs a "copy-on-write" operation. It copies the file up to the writable layer and modifies the copy, leaving the original read-only layer completely untouched.
*   When a container is deleted, this writable layer is also deleted. The underlying image layers remain perfectly intact.

### Benefits of Layers
*   **Storage Efficiency:** Different images can share exactly the same underlying layers. If you have 10 Node.js applications that all use the same base Node image, the host system only stores that base image layer once.
*   **Faster Downloads/Builds:** When pulling an image from a registry, you only download the layers you don't already have. When rebuilding an image, the build tool caches existing layers, only rebuilding from the point where a change occurred.

---

## 2. Image Registries & Distribution

An image registry is a highly scalable storage and distribution system for container images. It is the centralized hub where developers push their built images and where production servers pull images from to run them.

### Repositories and Tags
Inside a registry, images are organized into **repositories**. A repository holds all the versions of a specific image. These versions are identified by **tags**.

*   Example: `nginx:1.21.0`
    *   `nginx` is the repository name.
    *   `1.21.0` is the tag indicating the version.
    *   `latest` is a special, commonly used tag that usually points to the most recent stable build (though this is just a convention, not a strict rule).

### Image Digests
While tags are mutable (the image associated with `ubuntu:latest` changes over time), you can also reference an image by its **digest**—a cryptographic hash (usually SHA256) of the image's contents. Pulling by digest (e.g., `ubuntu@sha256:45b23fac...`) guarantees you get the exact, immutable byte-for-byte image, ensuring ultimate deployment consistency.

### Types of Registries
1.  **Public Registries:**
    *   **Docker Hub:** The default, largest public registry. Anyone can search for and download thousands of official and community-contributed images (like `nginx`, `postgres`, `node`).
2.  **Private Registries:**
    *   Organizations use private registries to host their proprietary application images securely.
    *   **Managed Cloud Registries:** Amazon Elastic Container Registry (ECR), Google Artifact Registry (GAR), Azure Container Registry (ACR).
    *   **Self-Hosted:** Tools like Harbor, Sonatype Nexus, or the open-source Docker Registry can be run on an organization's own infrastructure.

### The Distribution Workflow
1.  **Build:** A developer writes a `Dockerfile` and runs `docker build` (or similar). This creates the localized image layers on their machine.
2.  **Push:** They run `docker push registry.example.com/myapp:v1`. The engine checks which layers the registry already has. It only uploads the new, unique layers to the registry.
3.  **Pull:** A Kubernetes node needs to run the application. It executes a `containerd` command to pull the image. The node authenticates with the registry, checks its local cache, and downloads only the layers it is missing. Once assembled via the UnionFS, the container starts.
