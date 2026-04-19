# Docker Compose: Build vs. Image & Service Dependency Ordering

When defining services in a `docker-compose.yml` file, two critical decisions you typically make are how to source the container image (using `build` or `image`) and how to manage the startup sequence of your services (using `depends_on`).

---

## 1. The `image` vs. `build` Fields

Every service in Docker Compose needs a container image to run. You can provide this image using either the `image` field, the `build` field, or both.

### The `image` Field
The `image` field tells Docker Compose to pull a pre-built image from a Docker registry (like Docker Hub, AWS ECR, or a private registry) or use a pre-existing image on your local machine.

**Usage Scenario:** 
- For standard, third-party services like databases (`postgres`, `mysql`), caches (`redis`), or web servers (`nginx`).
- When your application's image has already been built and pushed to a registry by a CI/CD pipeline.

**Example:**
```yaml
services:
  db:
    image: postgres:14-alpine
```

### The `build` Field
The `build` field instructs Docker Compose to build a new image from source code using a `Dockerfile`.

**Usage Scenario:**
- For your own custom applications (APIs, frontends) where you are actively iterating on the code.
- When you want Compose to automatically build the application environment directly from the repository.

**Example (Short Syntax):**
If the directory containing the `docker-compose.yml` also has the `Dockerfile`:
```yaml
services:
  api:
    build: .
```

**Example (Long Syntax):**
For more complex setups, you can specify the build context (path), the specific Dockerfile name, and even pass build arguments.
```yaml
services:
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
      args:
        - NODE_ENV=development
```

### Using Both `image` and `build` Together
If you specify both, Docker Compose will build the image based on the `build` configuration and then **tag** the resulting image with the name specified in the `image` field.

```yaml
services:
  webapp:
    build: ./frontend
    image: mycompany/frontend-app:v1.0
```
This is extremely useful for building an image locally and naming it correctly before pushing it to a registry.

---

## 2. Service Dependency Ordering (`depends_on`)

In a multi-container application, some services naturally depend on others. For instance, a backend API cannot function properly if its database isn't running yet.

The `depends_on` field controls the order of startup and shutdown. 

### Basic Usage
By default, `depends_on` simply waits until the specified service has **started** (i.e., the container process has begun) before starting the current service.

```yaml
services:
  db:
    image: postgres:14

  backend:
    build: ./backend
    depends_on:
      - db
```
In this example:
1. Compose starts `db` first.
2. Once the `db` container is "up", Compose immediately starts `backend`.
3. When bringing down the stack, Compose stops `backend` *before* stopping `db`.

### The Problem with Basic `depends_on`
A container being "started" does *not* mean the service inside it is "ready". A PostgreSQL container might start quickly, but it takes several more seconds for the database system to initialize and accept connections. If the `backend` tries to connect during this window, it will crash.

### Advanced Usage: Ensuring the Service is Healthy
To solve this, Docker Compose allows you to use `depends_on` with specific conditions, commonly combined with a `healthcheck`.

You define a `healthcheck` on the database to determine when it's actually ready to accept connections. Then, you use the `condition: service_healthy` in your dependent service.

```yaml
services:
  db:
    image: postgres:14
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secretpassword
      POSTGRES_DB: appdb
    # 1. Define how to check if the DB is ready
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build: ./backend
    # 2. Only start backend when db passes the healthcheck
    depends_on:
      db:
        condition: service_healthy
```

**Key Conditions for `depends_on`:**
- `service_started`: (Default) The dependent service container has been started.
- `service_healthy`: The dependent service is completely ready, as determined by its `<healthcheck>`.
- `service_completed_successfully`: The dependent service container ran its task and exited with a successful status code (0) before the current unit is started. Useful for initialization scripts or database migration containers.
