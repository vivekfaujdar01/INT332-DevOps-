# Comprehensive Docker Commands Guide

This file serves as a consolidated reference for all the key Docker commands associated with the concepts covered in Unit 1, alongside additional essential commands for daily Docker usage and debugging.

---

## 1. General & Introductory Commands

*   **`docker --version`**: Shows the installed version of Docker on your system, verifying that the CLI is available.
*   **`docker info`**: Displays rich, system-wide information about your Docker installation, including the number of containers/images, the storage driver, and kernel versions.
*   **`docker run hello-world`**: A classic verification command. It pulls a minimal test image from Docker Hub, runs it in a container, prints a confirmation message, and then exits, proving that the runtime is functioning perfectly.

## 2. Client, Server, and Registry Architecture

*   **`docker version`**: Unlike `docker --version`, this command shows detailed version information for *both* the Docker Client (CLI) and the Docker Server (Daemon), confirming they can communicate successfully over the API.
*   **`docker login`**: Authenticates the Docker client with a container registry (defaults to Docker Hub). This creates a credential file so you can pull private images or push your own images.
*   **`docker logout`**: Logs out the client from a container registry and removes your stored credentials.
*   **`docker search <term>`**: Queries the Docker Hub (or configured registry) directly from your terminal to find images matching the search term, displaying their names, descriptions, and star counts.

## 3. Core Objects Management (Containers & Networks)

*   **`docker ps`** (or `docker container ls`): Lists all currently running containers.
*   **`docker ps -a`**: Lists all containers, including those that are stopped or have exited.
*   **`docker run <image>`**: Creates and starts a new container from a specified image. Useful flags include `-d` (detached mode), `-p` (port mapping), and `--name` (assigning a specific name).
*   **`docker start <container>`**: Starts one or more stopped containers without recreating them.
*   **`docker stop <container>`**: Gracefully stops one or more running containers.
*   **`docker restart <container>`**: Restarts a running or stopped container.
*   **`docker rm <container>`**: Permanently removes one or more stopped containers. Use `-f` to forcefully stop and remove a running container.
*   **`docker pause <container>`** / **`docker unpause <container>`**: Suspends / resumes all processes within one or more containers.
*   **`docker rename <old_name> <new_name>`**: Renames an existing container.
*   **`docker network create <network>`**: Creates a new, isolated Docker network.
*   **`docker network ls`**: Lists all networks managed by the Docker daemon.
*   **`docker network connect <network> <container>`**: Connects a running container to an existing network.
*   **`docker network inspect <network>`**: Displays detailed configuration and connection information about a Docker network.

## 4. Container Interaction and Debugging

*   **`docker exec -it <container> /bin/bash`** (or `sh`): Runs a new command in an already running container. The `-it` flags together give you an interactive terminal session inside the container.
*   **`docker logs <container>`**: Fetches the output (logs) produced by the main process running inside the container. Add the `-f` flag to follow the logs as they populate live.
*   **`docker port <container>`**: Lists the port mappings for a specific container, showing which host ports translate to which container ports.
*   **`docker cp <local_path> <container>:<path>`**: Copies a file or folder from your local host machine directly into a container's filesystem. Note that this works both ways (you can copy from a container to the host).

## 5. Container Images and Registries

*   **`docker pull <image>`**: Pulls a container image or a repository from a registry (e.g., Docker Hub) to your local machine.
*   **`docker push <image>`**: Pushes an image or a repository from your local machine up to a registry.
*   **`docker build -t <tag> .`**: Builds a new Docker image from a `Dockerfile` located in the current directory (`.`), and tags it with a specific name/version.
*   **`docker images`** or **`docker image ls`**: Lists all the container images currently downloaded and stored on your local machine.
*   **`docker history <image>`**: Shows the history (the layers) of an image, displaying how it was built step-by-step.
*   **`docker rmi <image>`**: Removes one or more images from your local machine.
*   **`docker commit <container> <new_image>`**: Creates a brand new image directly from the current state (changes) of a container.
*   **`docker tag <source_image> <target_image>`**: Assigns an additional tag (like a new version number or registry URL alias) to an existing image.
*   **`docker save -o <file.tar> <image>`**: Exports an image to a `.tar` file, useful for transferring an image directly between machines without using a registry.
*   **`docker load -i <file.tar>`**: Loads an image from a `.tar` archive previously created by `docker save`.

## 6. Layering, Filesystems, and Volumes

*   **`docker diff <container>`**: Inspects changes (additions, modifications, deletions) made to files or directories on a container's topmost read-write filesystem compared to the underlying image.
*   **`docker volume create <volume>`**: Explicitly creates a new named volume on the host system to be used for persistent data.
*   **`docker volume ls`**: Lists all Docker volumes currently managed by the local Docker daemon.
*   **`docker volume inspect <volume>`**: Displays detailed information about a volume, including its exact mount point on the host filesystem.
*   **`docker volume rm <volume>`**: Permanently deletes a Docker volume and all the persistent data within it.
*   **`docker run -v <volume>:<container_path> <image>`** (or `--mount`): Runs a container and mounts a volume into its filesystem, bypassing the UnionFS for that specific directory path.

## 7. Container Internals, Namespaces, and Cgroups

*   **`docker run --cpus=<value> --memory=<value> <image>`**: Runs a container while applying cgroup resource limits. For example, `--cpus=0.5` restricts the container to half a CPU core, and `--memory=512m` limits its RAM usage to 512 Megabytes.
*   **`docker inspect <container>`**: Returns detailed, low-level JSON information about a Docker object. You can use it to see the specific namespace and cgroup configurations applied to a running container.
*   **`docker stats`**: Displays a live, continuously updating stream of resource usage statistics (CPU, memory, net I/O, block I/O) for all running containers, reflecting the limits enforced by cgroups.
*   **`docker top <container>`**: Displays the running processes *inside* a specific container, showcasing the PID namespace isolation at work.

## 8. Cleanup and Maintenance

*   **`docker system prune`**: Removes all unused containers, networks, images (both dangling and unreferenced), and optionally, volumes. Highly useful for freeing up disk space.
*   **`docker container prune`**: Removes all stopped containers.
*   **`docker image prune`**: Removes unused (dangling) images.
