# 02 — Container Registries

> **Goal:** Understand what a registry is, how push/pull works, and why this project provisions a private Harbor registry per tenant instead of using Docker Hub.

---

## 1. What Is a Container Registry?

A container registry is a **server that stores and serves container images**. Think of it like GitHub, but instead of storing source code, it stores compiled Docker images.

```
Developer                    Registry                    Kubernetes
─────────                    ────────                    ──────────
  │                             │                             │
  │  docker push myapp:1.2  ───►│  stores image               │
  │                             │  myapp:1.2                  │
  │                             │  ┌─────────────────────┐    │
  │                             │  │ layers:              │    │
  │                             │  │  sha256:abc123...    │    │
  │                             │  │  sha256:def456...    │    │
  │                             │  └─────────────────────┘    │
  │                             │                             │
  │                             │◄── docker pull myapp:1.2 ───│
  │                             │         (to run a pod)      │
```

---

## 2. The Push/Pull Cycle

### Push (uploading an image)

```bash
# 1. Tag the image with the registry address
docker tag myapp:latest registry.mytenant.lkdc.wso2.com/library/myapp:latest

# 2. Log in to the registry
docker login registry.mytenant.lkdc.wso2.com \
  --username robot$ci-default \
  --password <token>

# 3. Push the image
docker push registry.mytenant.lkdc.wso2.com/library/myapp:latest
```

What happens during push:
```
docker push registry.example.com/library/myapp:latest
  │
  ├── checks what layers already exist on the server
  ├── uploads only the MISSING layers (deduplication!)
  ├── uploads the manifest (JSON describing the image)
  └── updates the "latest" tag to point to this manifest
```

### Pull (downloading an image)

```bash
docker pull registry.mytenant.lkdc.wso2.com/library/myapp:latest
```

What happens during pull:
```
docker pull registry.example.com/library/myapp:latest
  │
  ├── fetches the manifest (which layers make up this image?)
  ├── checks local cache — skips layers already downloaded
  └── downloads missing layers
```

---

## 3. Image Naming — Anatomy of a Tag

```
registry.mytenant.lkdc.wso2.com / library / myapp : latest
│─────────────────────────────────│ │───────│ │─────│ │──────│
         Registry host               Project   Name    Tag
         (where to push/pull)        (namespace)       (version)
```

| Part | Example | Meaning |
|------|---------|---------|
| Registry host | `registry.mytenant.lkdc.wso2.com` | The server to connect to |
| Project | `library` | A grouping of images (Harbor calls these "projects") |
| Name | `myapp` | The image name |
| Tag | `latest`, `v1.2.3`, `abc1234` | A human-readable pointer to a specific version |

> **Important:** `latest` is just a tag like any other. It doesn't automatically update. Explicitly pushing `:latest` is a convention, not a guarantee.

---

## 4. Docker Hub vs. Private Registry

| Feature | Docker Hub (public) | Harbor (private, per-tenant) |
|---------|--------------------|-----------------------------|
| **Cost** | Free for public, paid for private | Self-hosted — you pay for compute |
| **Privacy** | Public images are visible to everyone | Completely private |
| **Vulnerability scanning** | Basic (paid) | Built-in Trivy scanner (free) |
| **Access control** | Basic | Fine-grained RBAC |
| **Rate limiting** | Yes (100 pulls/6h for free) | No limit |
| **Network** | Goes over internet | Stays inside your cluster |
| **Compliance** | Images leave your infrastructure | Images never leave your cluster |

**Why per-tenant Harbor instead of one shared Harbor?**

```
SHARED HARBOR (bad for multi-tenant SaaS):
  ┌────────────────────────────────────────┐
  │          One Harbor instance           │
  │  tenant-A/project-1/myapp:v1           │
  │  tenant-A/project-2/backend:v2         │
  │  tenant-B/project-1/myapp:v1           │  ← tenant-B can misconfigure
  │  tenant-B/project-2/frontend:v5        │     and accidentally see
  │                                        │     tenant-A's images
  └────────────────────────────────────────┘
  Problems: one outage affects all tenants,
            RBAC bugs leak across tenants,
            one noisy tenant consumes all storage

PER-TENANT HARBOR (this system):
  ┌────────────────────┐  ┌────────────────────┐
  │  tenant-A Harbor   │  │  tenant-B Harbor   │
  │  Namespace:        │  │  Namespace:        │
  │  tenantA-mgmt      │  │  tenantB-mgmt      │
  │  URL: registry     │  │  URL: registry     │
  │  .tenantA          │  │  .tenantB          │
  │  .lkdc.wso2.com  │  │  .lkdc.wso2.com  │
  └────────────────────┘  └────────────────────┘
  Completely isolated: separate DB, separate storage,
  separate credentials, separate K8s namespace
```

---

## 5. How Harbor Stores Images

Harbor uses the **OCI Distribution Spec** (Open Container Initiative). Under the hood:

```
PUT /v2/library/myapp/blobs/uploads/      ← start upload session
PUT /v2/library/myapp/blobs/uploads/<id>  ← upload layer bytes
PUT /v2/library/myapp/manifests/latest    ← link layers + metadata

GET /v2/library/myapp/manifests/latest    ← pull: get manifest
GET /v2/library/myapp/blobs/<sha256>      ← pull: download each layer
```

The actual blob storage is backed by a filesystem or object storage (S3, GCS, etc.). In this system it uses Kubernetes PersistentVolumeClaims.

---

## 6. What Happens When You Push to Your Harbor

```
CI/CD Pipeline                Harbor                        Storage (PVC)
─────────────                 ──────                        ─────────────
docker push ──────────────►  nginx (reverse proxy)
                                  │
                              ┌───▼────────────────────┐
                              │  Registry component     │
                              │  (OCI Distribution API) │──────► /var/lib/
                              └───┬────────────────────┘         registry/
                                  │
                              ┌───▼───────────────────┐
                              │  Core component        │
                              │  - auth check          │
                              │  - event notification  │
                              └───┬───────────────────┘
                                  │
                              ┌───▼───────────────────┐
                              │  JobService            │
                              │  - trigger Trivy scan  │
                              └───────────────────────┘
```

---

## 7. Robot Accounts — The CI/CD Key

A **robot account** is a service account for automated systems (CI/CD pipelines, Kubernetes pods). It gets:
- A **username** like `robot$ci-default`
- A **token** (long random string) used as password
- **Specific permissions** (push, pull, scan) on specific projects

```
Human admin         Robot account
────────────        ─────────────
username: admin     username: robot$ci-default+library
password: ***       token: eyJ...long random string...
full access         only push/pull to "library" project
expires: never      expires: 365 days (configurable)
```

This system creates a robot account during the **bootstrap step** (Step 5 of 7 in the deploy worker). The token is then **encrypted** and stored in PostgreSQL.

---

## 8. Pull Secrets in Kubernetes

When Kubernetes needs to pull an image from a **private** registry, it needs credentials. These are stored as a special Secret type called `kubernetes.io/dockerconfigjson`:

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/dockerconfigjson
metadata:
  name: registry-pull-secret
data:
  .dockerconfigjson: <base64 encoded JSON>
```

The JSON inside looks like:
```json
{
  "auths": {
    "registry.mytenant.lkdc.wso2.com": {
      "username": "robot$ci-default",
      "password": "the-robot-token",
      "auth": "<base64(username:password)>"
    }
  }
}
```

Then a Pod references it:
```yaml
spec:
  imagePullSecrets:
    - name: registry-pull-secret
  containers:
    - image: registry.mytenant.lkdc.wso2.com/library/myapp:v1
```

The `GET /api/v1/tenants/{id}/registry/pull-secret` endpoint in this system generates and returns this YAML so developers can paste it into their Kubernetes manifests.

---

## 🏋️ Exercises

### Exercise 1 — Understand Tags
```bash
# Pull the same image with different tags — notice which layers are shared
docker pull alpine:3.18
docker pull alpine:3.19
docker images alpine  # compare sizes — layers are reused!
```

### Exercise 2 — Inspect a Manifest
```bash
# Pull the nginx image and inspect its manifest
docker pull nginx:alpine
docker inspect nginx:alpine | python3 -m json.tool | head -60

# What layers does it have?
# What is the architecture?
# What is the entry command?
```

### Exercise 3 — Push to the Dev Registry (Conceptual)
If Harbor were running, you would:
```bash
# 1. Log in
docker login registry.dev-tenant.lkdc.wso2.com \
  --username robot$ci-default \
  --password <token from GET /credentials>

# 2. Tag a local image
docker tag nginx:alpine registry.dev-tenant.lkdc.wso2.com/library/nginx:test

# 3. Push
docker push registry.dev-tenant.lkdc.wso2.com/library/nginx:test

# 4. Verify it's in Harbor by visiting the Harbor portal
```

### Exercise 4 — Explore the API Endpoint
```bash
# Hit the pull-secret endpoint (returns the Kubernetes YAML)
curl -s http://localhost:8080/api/v1/tenants/dev-tenant/registry/pull-secret

# What does it return? (Will return 404 since no registry exists yet — that's fine)
```

### Exercise 5 — Think Through Isolation
Draw on paper (or ASCII) the difference between:
- 100 tenants sharing ONE Harbor instance
- 100 tenants each having their OWN Harbor instance

Consider: storage, network, failure blast radius, RBAC complexity.

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Registry** | A server that stores and serves container images |
| **Push** | Uploading an image (layers + manifest) to a registry |
| **Pull** | Downloading an image from a registry |
| **Tag** | A name pointing to a specific image version (e.g., `v1.2.3`) |
| **Project** | A namespace within Harbor that groups images |
| **Robot account** | A service account with a token for automated systems |
| **Pull secret** | A Kubernetes secret containing registry credentials |

**Next:** [03 — Harbor Deep Dive →](./03-harbor-deep-dive.md)
