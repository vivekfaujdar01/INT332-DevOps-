# Docker Registries: Storage & Distribution of Container Images

A **Docker registry** is a server-side application that stores and distributes Docker images. Registries are central to the container workflow — they act as the bridge between **building** an image locally and **deploying** it on remote servers, CI/CD pipelines, or collaborators' machines.

---

## 1. Docker Hub

**Docker Hub** (`hub.docker.com`) is the default public registry used by Docker. When you run `docker pull nginx`, Docker automatically contacts Docker Hub to fetch the image.

### Key Features

| Feature | Description |
|---|---|
| **Public Repositories** | Unlimited free public repos; anyone can pull images |
| **Private Repositories** | One free private repo on the free tier; paid plans for more |
| **Official Images** | Curated, security-scanned images maintained by Docker (e.g., `nginx`, `python`, `node`) |
| **Verified Publisher Images** | Images from trusted software vendors (e.g., `bitnami/redis`) |
| **Automated Builds** | Link a GitHub/Bitbucket repo to auto-build images on push |
| **Webhooks** | Trigger external services when a new image is pushed |
| **Teams & Organizations** | Role-based access control for collaborative workflows |

### Common Commands

```bash
# Log in to Docker Hub
docker login

# Pull an image from Docker Hub (default registry)
docker pull ubuntu:22.04

# Tag a local image for Docker Hub
docker tag my-app:latest myusername/my-app:v1.0

# Push the tagged image
docker push myusername/my-app:v1.0

# Search for images on Docker Hub
docker search python
```

### Image Naming Convention

```
[registry/]<namespace>/<repository>:<tag>

# Docker Hub examples (registry is implicit)
ubuntu:22.04                  # Official image — no namespace needed
myusername/my-app:v1.0        # User namespace
myorg/backend-service:latest  # Organization namespace
```

- **Official images** live in the top-level namespace (e.g., `nginx`, `postgres`).
- **User/org images** are prefixed with the account name (e.g., `myusername/my-app`).

### Rate Limits

Docker Hub enforces pull rate limits:

| Account Type | Limit |
|---|---|
| Anonymous (unauthenticated) | 100 pulls per 6 hours per IP |
| Free authenticated | 200 pulls per 6 hours |
| Pro / Team / Business | Higher or unlimited pulls |

> **Tip:** Always authenticate with `docker login` in CI/CD pipelines to avoid hitting anonymous rate limits.

---

## 2. GitHub Container Registry (GHCR)

**GHCR** (`ghcr.io`) is GitHub's native container registry, tightly integrated with GitHub repositories, actions, and permissions.

### Key Features

| Feature | Description |
|---|---|
| **GitHub Integration** | Images are linked to repositories; visibility inherits from repo settings |
| **Free for Public Images** | Public packages are free with unlimited bandwidth |
| **GitHub Actions Support** | Seamless authentication using `GITHUB_TOKEN` in workflows |
| **Fine-Grained Permissions** | Package-level visibility (public/private) independent of the repo |
| **OCI Compliant** | Supports OCI image manifests and artifacts beyond Docker images |

### Pushing an Image to GHCR

```bash
# 1. Authenticate with a Personal Access Token (PAT)
echo $GHCR_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# 2. Build and tag the image with the ghcr.io prefix
docker build -t ghcr.io/USERNAME/my-app:v1.0 .

# 3. Push the image
docker push ghcr.io/USERNAME/my-app:v1.0

# 4. Pull the image (from any machine)
docker pull ghcr.io/USERNAME/my-app:v1.0
```

### Using GHCR in GitHub Actions

```yaml
# .github/workflows/publish.yml
name: Publish Docker Image

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write          # Required for pushing to GHCR

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # Auto-provided token

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

### Image Naming Convention for GHCR

```
ghcr.io/<owner>/<image-name>:<tag>

# Examples
ghcr.io/faujdar/backend-api:v2.1
ghcr.io/my-org/frontend:latest
```

### Docker Hub vs GHCR — Quick Comparison

| Aspect | Docker Hub | GHCR |
|---|---|---|
| Default registry | Yes (`docker pull`) | No (must prefix `ghcr.io/`) |
| Free private repos | 1 | Unlimited (with GitHub plan limits) |
| CI/CD integration | Generic webhooks | Native GitHub Actions support |
| Auth mechanism | Docker Hub credentials / PAT | GitHub PAT / `GITHUB_TOKEN` |
| Official images | Yes | No |
| OCI artifact support | Limited | Full |

---

## 3. Private Registries

A **private registry** is a self-hosted or cloud-managed registry restricted to your organization. It gives you full control over storage, access, and network proximity to your infrastructure.

### Why Use a Private Registry?

- **Security** — Images never leave your network; sensitive code stays internal.
- **Performance** — Faster pulls when the registry is colocated with your cluster.
- **Compliance** — Meet regulatory requirements (HIPAA, GDPR) for data residency.
- **Cost** — Avoid per-pull charges and rate limits of public registries.

### Option A: Docker's Official Registry (Self-Hosted)

Docker provides an open-source registry server image called `registry:2`.

```bash
# Run a local private registry on port 5000
docker run -d \
  --name my-registry \
  -p 5000:5000 \
  --restart always \
  registry:2

# Tag an image for the private registry
docker tag my-app:latest localhost:5000/my-app:v1.0

# Push to the private registry
docker push localhost:5000/my-app:v1.0

# Pull from the private registry
docker pull localhost:5000/my-app:v1.0
```

#### Adding TLS (Production Setup)

For production, always enable TLS:

```bash
docker run -d \
  --name secure-registry \
  -p 443:5000 \
  -v /certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  --restart always \
  registry:2
```

#### Listing Images in a Private Registry

```bash
# List all repositories
curl -X GET https://myregistry.example.com/v2/_catalog

# List tags for a specific image
curl -X GET https://myregistry.example.com/v2/my-app/tags/list
```

### Option B: Cloud-Managed Private Registries

Major cloud providers offer managed registries:

| Provider | Service | Registry URL Pattern |
|---|---|---|
| **AWS** | Elastic Container Registry (ECR) | `<account-id>.dkr.ecr.<region>.amazonaws.com` |
| **Google Cloud** | Artifact Registry | `<region>-docker.pkg.dev/<project>/<repo>` |
| **Azure** | Azure Container Registry (ACR) | `<registry-name>.azurecr.io` |
| **DigitalOcean** | Container Registry | `registry.digitalocean.com/<registry>` |

#### Example: Pushing to AWS ECR

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag my-app:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0

docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
```

### Option C: Third-Party Registries

| Tool | Description |
|---|---|
| **Harbor** | Open-source enterprise registry with vulnerability scanning, RBAC, and replication |
| **JFrog Artifactory** | Universal artifact manager supporting Docker and many other formats |
| **GitLab Container Registry** | Built into GitLab, uses `registry.gitlab.com` |
| **Quay.io** | Red Hat's registry, supports image scanning and geo-replication |

### Configuring Docker to Trust a Private Registry

If your private registry uses a self-signed certificate or runs over HTTP (not recommended for production):

```json
// /etc/docker/daemon.json
{
  "insecure-registries": ["myregistry.local:5000"]
}
```

```bash
# Restart Docker after modifying daemon.json
sudo systemctl restart docker
```

> **Warning:** Never use `insecure-registries` in production. Always configure proper TLS certificates.

---

## 4. Authentication & Access Tokens

Securing registry access is critical. Without proper authentication, anyone could pull proprietary images or push malicious ones.

### 4.1 Docker Login — The Basics

```bash
# Interactive login (prompts for password)
docker login

# Login to a specific registry
docker login ghcr.io
docker login myregistry.example.com

# Non-interactive login (for CI/CD)
echo $MY_TOKEN | docker login registry.example.com -u username --password-stdin
```

> **Security Tip:** Always use `--password-stdin` instead of `-p` flag. The `-p` flag exposes the password in shell history and process listings.

### 4.2 How Docker Stores Credentials

After `docker login`, credentials are saved in `~/.docker/config.json`:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="    // Base64-encoded username:password
    },
    "ghcr.io": {
      "auth": "dXNlcm5hbWU6Z2hwX3Rva2VuMTIz"
    }
  }
}
```

> **Warning:** By default, credentials are stored as **Base64-encoded plaintext** — not encrypted! Use a **credential helper** for secure storage.

### 4.3 Credential Helpers

Credential helpers store tokens securely using the OS keychain instead of plaintext files.

| Helper | Platform | Config Value |
|---|---|---|
| `docker-credential-osxkeychain` | macOS | `"credsStore": "osxkeychain"` |
| `docker-credential-secretservice` | Linux (GNOME Keyring) | `"credsStore": "secretservice"` |
| `docker-credential-wincred` | Windows | `"credsStore": "wincred"` |
| `docker-credential-pass` | Linux (pass) | `"credsStore": "pass"` |
| `docker-credential-ecr-login` | AWS ECR | `"credHelpers": {"<ecr-url>": "ecr-login"}` |

#### Setting Up a Credential Helper

```json
// ~/.docker/config.json
{
  "credsStore": "secretservice"
}
```

After this, `docker login` will store credentials in your system keychain instead of the config file.

### 4.4 Personal Access Tokens (PATs)

Most registries recommend using **tokens** instead of passwords for authentication. Tokens offer:

- **Scoped permissions** — Grant only the access needed (read, write, delete).
- **Revocability** — Revoke a token without changing your password.
- **Expiration** — Set tokens to expire automatically.
- **Auditability** — Track which token performed which action.

#### Docker Hub — Access Tokens

1. Go to [hub.docker.com](https://hub.docker.com) → **Account Settings** → **Security** → **Access Tokens**.
2. Click **New Access Token**.
3. Choose permissions: **Read**, **Read/Write**, or **Read/Write/Delete**.
4. Copy the token and use it in place of your password:

```bash
echo $DOCKERHUB_TOKEN | docker login -u myusername --password-stdin
```

#### GitHub (GHCR) — Personal Access Tokens

1. Go to **GitHub Settings** → **Developer Settings** → **Personal Access Tokens** → **Fine-grained tokens**.
2. Create a token with the `read:packages` and/or `write:packages` scope.
3. Use the token to authenticate:

```bash
echo $GHCR_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
```

#### In GitHub Actions — `GITHUB_TOKEN`

GitHub Actions automatically provides a `GITHUB_TOKEN` secret scoped to the repository. No manual PAT creation needed:

```yaml
- name: Log in to GHCR
  run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
```

### 4.5 Robot Accounts & Service Accounts

For automated systems (CI/CD, Kubernetes), use dedicated service accounts instead of personal credentials:

| Registry | Mechanism |
|---|---|
| Docker Hub | Service accounts via Teams (paid plans) |
| GHCR | GitHub App tokens or machine user accounts |
| AWS ECR | IAM roles / instance profiles |
| Harbor | Robot accounts with scoped permissions |
| GCP Artifact Registry | Service account keys or Workload Identity |

### 4.6 Kubernetes Image Pull Secrets

When Kubernetes needs to pull images from a private registry, you create an **imagePullSecret**:

```bash
# Create the secret
kubectl create secret docker-registry my-registry-secret \
  --docker-server=ghcr.io \
  --docker-username=USERNAME \
  --docker-password=$GHCR_TOKEN \
  --docker-email=user@example.com

# Reference it in a Pod spec
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: ghcr.io/myorg/my-app:v1.0
  imagePullSecrets:
    - name: my-registry-secret
```

### 4.7 Authentication Best Practices

| Practice | Why |
|---|---|
| **Never hardcode credentials** | Use environment variables or secret managers |
| **Use short-lived tokens** | Reduce blast radius if a token leaks |
| **Scope tokens minimally** | Read-only tokens for pull-only services |
| **Rotate tokens regularly** | Automated rotation reduces long-term risk |
| **Use credential helpers** | Avoid plaintext storage on disk |
| **Audit access logs** | Detect unauthorized pulls or pushes |
| **Use `--password-stdin`** | Prevents password leakage in shell history |

---

## Summary Diagram: Registry Workflow

```
                ┌──────────────┐
                │  Developer   │
                │   Machine    │
                └──────┬───────┘
                       │
              docker build & tag
                       │
                       ▼
            ┌─────────────────────┐
            │    docker push      │──── Auth (PAT / Token) ────┐
            └─────────────────────┘                            │
                       │                                       ▼
          ┌────────────┼────────────────┐          ┌───────────────────┐
          │            │                │          │    Registry       │
          ▼            ▼                ▼          │  ┌─────────────┐  │
    ┌──────────┐ ┌──────────┐  ┌──────────────┐   │  │ Docker Hub  │  │
    │ Docker   │ │  GHCR    │  │   Private    │   │  │ GHCR        │  │
    │  Hub     │ │ ghcr.io  │  │  Registry   │   │  │ ECR / ACR   │  │
    └──────────┘ └──────────┘  └──────────────┘   │  │ Harbor      │  │
          │            │                │          │  └─────────────┘  │
          └────────────┼────────────────┘          └───────────────────┘
                       │
              docker pull (+ Auth)
                       │
                       ▼
            ┌─────────────────────┐
            │   Production /      │
            │   Kubernetes /      │
            │   CI-CD Pipeline    │
            └─────────────────────┘
```

---

## Quick Reference Commands

```bash
# === Docker Hub ===
docker login                                          # Login to Docker Hub
docker push myuser/myimage:tag                        # Push to Docker Hub
docker pull myuser/myimage:tag                        # Pull from Docker Hub

# === GHCR ===
echo $TOKEN | docker login ghcr.io -u USER --password-stdin   # Login to GHCR
docker push ghcr.io/user/image:tag                            # Push to GHCR
docker pull ghcr.io/user/image:tag                            # Pull from GHCR

# === Private Registry ===
docker run -d -p 5000:5000 registry:2                 # Start local registry
docker push localhost:5000/myimage:tag                 # Push to local registry
docker pull localhost:5000/myimage:tag                 # Pull from local registry

# === Credential Management ===
docker login <registry>                                # Store credentials
docker logout <registry>                               # Remove credentials
cat ~/.docker/config.json                              # View stored credentials
```

---

> **Key Takeaway:** Registries are the distribution mechanism for Docker images. Choose Docker Hub for public sharing, GHCR for GitHub-centric workflows, and private registries for enterprise security. Always authenticate with scoped, short-lived tokens rather than passwords.
