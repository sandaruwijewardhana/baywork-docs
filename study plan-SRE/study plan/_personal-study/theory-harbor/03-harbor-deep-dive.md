# 03 — Harbor Deep Dive

> **Goal:** Understand every component inside a Harbor instance, what robot accounts and projects are, and how vulnerability scanning works.

---

## 1. What Is Harbor?

Harbor is an **open-source enterprise container registry** built by VMware (now CNCF). It wraps the standard OCI Distribution registry with:

- **Role-based access control (RBAC)** — who can push/pull what
- **Projects** — logical namespaces for images
- **Vulnerability scanning** — Trivy scans every pushed image for CVEs
- **Robot accounts** — service accounts for CI/CD
- **Replication** — sync images between registries
- **Audit logs** — who pulled/pushed what, when

---

## 2. Harbor Architecture — All Components

When this system deploys Harbor via Helm, it creates **7 components** in the tenant's namespace:

```
                        External Traffic
                              │
                    ┌─────────▼──────────┐
                    │    nginx (proxy)    │  ← single entry point
                    │  :80 / :443        │    routes to core or registry
                    └─────┬──────┬───────┘
                          │      │
            ┌─────────────▼──┐  ┌▼──────────────────┐
            │   Core         │  │  Registry          │
            │ - Auth         │  │  (OCI Distribution)│
            │ - RBAC         │  │  - push/pull blobs │
            │ - API          │  │  - store layers    │
            │ - Events       │  └──────────┬─────────┘
            │ - Webhooks     │             │
            └──┬─────────────┘             │ blob storage
               │                    ┌──────▼──────┐
        ┌──────▼─────────┐          │  PVC (disk)  │
        │   JobService   │          │  /storage    │
        │ - scan tasks   │          └─────────────┘
        │ - replication  │
        │ - GC jobs      │
        └──┬─────────────┘
           │ scans
    ┌──────▼──────────┐
    │  Trivy Adapter  │
    │  - CVE database │
    │  - scan images  │
    └─────────────────┘

Supporting services:
    ┌─────────────────┐     ┌─────────────────┐
    │  PostgreSQL      │     │  Redis          │
    │  - projects      │     │  - sessions     │
    │  - users         │     │  - job queue    │
    │  - audit log     │     │  - cache        │
    │  - tags/labels   │     └─────────────────┘
    └─────────────────┘

    ┌─────────────────┐
    │  Portal (UI)    │
    │  - web dashboard│
    │  - React SPA    │
    └─────────────────┘
```

---

## 3. Component Deep Dives

### nginx (Reverse Proxy)
The **front door** for all traffic. Every request hits nginx first:

```
Request: GET /v2/library/myapp/manifests/latest
    nginx decides: this is a registry API call (/v2/ prefix)
    → routes to Registry component

Request: GET /api/v2.0/projects
    nginx decides: this is the Harbor API
    → routes to Core component

Request: GET /
    nginx decides: this is the portal UI
    → routes to Portal component
```

### Core
The **brain** of Harbor. Handles:
- Authentication (validates credentials against PostgreSQL)
- Authorization (is this user allowed to push to this project?)
- The full Harbor REST API (`/api/v2.0/...`)
- Event processing (image pushed → notify JobService to scan)

```go
// In bootstrap.go — the system calls the Core API
harborClient.Configure(ctx)         // PUT /api/v2.0/configurations
harborClient.CreateProject(ctx)     // POST /api/v2.0/projects
harborClient.CreateRobotAccount(ctx) // POST /api/v2.0/robots
```

### Registry (OCI Distribution)
The **actual storage engine** for container images. Implements the OCI Distribution Spec:

```
Push flow:
  docker push → POST /v2/library/myapp/blobs/uploads/
              → PUT  /v2/library/myapp/blobs/uploads/{uuid}  (layer data)
              → PUT  /v2/library/myapp/manifests/v1          (link layers)

Pull flow:
  docker pull → GET /v2/library/myapp/manifests/v1  (what layers?)
              → GET /v2/library/myapp/blobs/{sha256}  (download each layer)
```

Blobs (image layers) are stored on the **PersistentVolumeClaim** (disk). In this system, the size is set by the tenant plan:
- Starter: 20 GiB
- Professional: 50 GiB
- Enterprise: 200 GiB

### JobService
A **background job runner**. When an image is pushed, Core sends an event to JobService which queues:
- Vulnerability scan (runs Trivy)
- Replication (if configured)
- Garbage collection (cleans up unreferenced blobs)

```
Image pushed event
       │
       ▼
JobService queue (Redis)
       │
       ▼
scan job: fetch manifest → download layers → run trivy → POST results to Core
```

### Trivy Adapter
Wraps the **Trivy vulnerability scanner**. Trivy downloads a CVE database (a list of known vulnerabilities) and checks image layers against it.

```
Image layer bytes
       │
       ▼
Trivy: extract OS packages (apt, apk), language packages (pip, npm, go.sum)
       │
       ▼
Compare against CVE database
       │
       ▼
Report: {
  "critical": 2,
  "high": 5,
  "medium": 12,
  ...
  "vulnerabilities": [
    { "pkg": "openssl", "version": "1.1.1", "cve": "CVE-2023-..." }
  ]
}
```

### PostgreSQL (Harbor's own)
**Every Harbor instance has its own PostgreSQL**. It stores:
- Projects and their settings
- Users and RBAC assignments
- Image tags and metadata
- Audit log (who pulled what)
- Robot accounts

This is **separate** from the platform PostgreSQL (which stores deployment state).

### Redis
Used as:
- **Session store** — remember logged-in users
- **Job queue** — JobService reads from this
- **Rate limit cache** — prevent abuse

---

## 4. Projects — The Access Boundary

A **project** is a namespace for images within Harbor. Access control is per-project:

```
Harbor Instance for tenant-acme
├── library/                     ← the "library" project (created by bootstrap)
│   ├── myapp:v1.0               ← anyone with project access can push/pull
│   ├── myapp:v2.0
│   └── nginx:alpine
├── infra/                       ← another project (if created later)
│   └── base-image:1.0
└── ...
```

Project visibility:
- **Private** — only authenticated users with access can pull
- **Public** — anyone can pull (no auth needed for pulls)

This system creates the project as **private** during bootstrap:

```go
// From backend/internal/harbor/bootstrap.go
func (c *Client) CreateProject(ctx context.Context) error {
    body := map[string]interface{}{
        "project_name": "library",
        "public":       false,      // ← private
        "metadata": map[string]string{
            "auto_scan": "true",    // ← auto-scan on push
        },
    }
    return c.post(ctx, "/api/v2.0/projects", body, nil)
}
```

---

## 5. Robot Accounts — Service Identities

A robot account represents a **machine identity** (CI pipeline, deployment script):

```
Human user:
  username: sandaruw@wso2.com
  auth: OIDC / username+password
  permissions: manage everything in the project
  expires: never (until deactivated)

Robot account:
  name: robot$ci-default        (auto-prefixed with "robot$")
  secret: <256-bit random token>
  permissions: push+pull to "library" project only
  expires: 365 days
```

How the system creates it:

```go
// From backend/internal/harbor/bootstrap.go
func (c *Client) CreateRobotAccount(ctx context.Context) (*RobotAccount, error) {
    body := map[string]interface{}{
        "name":     "ci-default",
        "duration": 365,           // ← expires in 365 days
        "level":    "system",
        "permissions": []map[string]interface{}{
            {
                "kind":      "project",
                "namespace": "library",   // ← only the library project
                "access": []map[string]string{
                    {"resource": "repository", "action": "push"},
                    {"resource": "repository", "action": "pull"},
                    {"resource": "artifact",   "action": "read"},
                    {"resource": "tag",        "action": "create"},
                    {"resource": "scan",       "action": "create"},
                },
            },
        },
    }
    var robot RobotAccount
    c.post(ctx, "/api/v2.0/robots", body, &robot)
    return &robot, nil  // robot.Secret ← the token
}
```

The returned `robot.Secret` is then **AES-256-GCM encrypted** and stored in PostgreSQL (never in plain text).

---

## 6. Harbor Bootstrap Flow

The bootstrap is Step 5 in the deploy worker. It runs after all pods are `Ready`:

```
Harbor pods are Ready
        │
        ▼
Step 1: WaitReady
  Poll GET /api/v2.0/ping every 10s
  Wait up to 3 minutes for 200 OK
        │
        ▼
Step 2: Configure
  PUT /api/v2.0/configurations
  - auth_mode: db_auth (use Harbor's own user DB, not LDAP)
  - project_creation_restriction: adminonly (only admins create projects)
  - robot_token_duration: 365 days
  - self_registration: false (users can't sign up themselves)
  - scan_all_policy: run Trivy scans every night at 2am
        │
        ▼
Step 3: CreateProject
  POST /api/v2.0/projects
  - name: "library"
  - public: false
  - auto_scan: true
        │
        ▼
Step 4: CreateRobotAccount
  POST /api/v2.0/robots
  Returns: { name: "robot$ci-default", secret: "eyJ..." }
        │
        ▼
Return robot account to worker
Worker encrypts and stores in DB
```

---

## 7. Vulnerability Scanning with Trivy

When you push an image:

```
docker push registry.tenant.lkdc.wso2.com/library/myapp:v1
                        │
                        ▼
              Harbor Core receives push event
                        │
                        ▼
              JobService creates scan job
                        │
                        ▼
              Trivy downloads/updates CVE database
                        │
                        ▼
              Trivy unpacks image layers
                        │
                        ▼
        ┌───────────────────────────────┐
        │  Check each package:          │
        │  - OS packages (apk, apt)     │
        │  - Python pip packages        │
        │  - Node npm packages          │
        │  - Go binaries (go.sum)       │
        │  - Java JARs                  │
        └───────────────────────────────┘
                        │
                        ▼
              Results stored in Harbor DB
                        │
                        ▼
              Visible in Harbor portal UI
              Badge shown on image tag:
              🔴 CRITICAL: 2  🟠 HIGH: 5  🟡 MEDIUM: 12
```

This system enables auto-scan on push by setting `"auto_scan": "true"` when creating the project.

---

## 8. The Harbor Helm Chart

This system deploys Harbor using its official Helm chart (`harbor/harbor` version 1.14.0). The chart creates all 7 components above.

Key values generated per-tenant:

```yaml
# Generated by backend/internal/helm/values_generator.go
expose:
  ingress:
    hosts:
      core: registry.{tenantId}.lkdc.wso2.com  # ← unique URL per tenant

harborAdminPassword: "{randomly generated 32-char password}"

persistence:
  persistentVolumeClaim:
    registry:
      size: 50Gi    # ← based on tenant plan (starter/professional/enterprise)
    database:
      size: 5Gi

core:
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
```

Each tenant's Harbor is completely independent — different URL, different passwords, different storage, different namespace.

---

## 🏋️ Exercises

### Exercise 1 — Explore the Harbor Portal (When Running)
When Harbor is fully deployed to a cluster, visit `https://registry.{tenant}.lkdc.wso2.com`:
- Log in as admin
- Create a project
- Push a test image
- View the vulnerability scan results

### Exercise 2 — Trace a Push Through Components
For the request `docker push registry.example.com/library/nginx:test`, write down which Harbor component handles each step:
1. TLS termination and routing
2. Authentication check
3. Blob storage
4. Scan trigger
5. Metadata storage

### Exercise 3 — Read the Bootstrap Code
Open [backend/internal/harbor/bootstrap.go](../backend/internal/harbor/bootstrap.go):
- How does it authenticate to Harbor?
- What HTTP method does `Configure` use?
- What happens if `CreateProject` fails?

### Exercise 4 — Robot Account Permissions
Look at the robot account permissions in `CreateRobotAccount`. The robot can do:
- `push` to repository ✓
- `pull` from repository ✓

What is it **NOT** allowed to do?
- Delete images? 
- Create new projects?
- View user list?

Why is this principle called **least privilege**?

### Exercise 5 — Understand the Values Template
Open [backend/internal/helm/values_generator.go](../backend/internal/helm/values_generator.go).
- What is the Trivy storage size?
- What `logLevel` is configured? Why not `debug`?
- What happens if you request a plan called `"ultra"`? Trace through the code.

---

## Summary

| Component | Role |
|-----------|------|
| **nginx** | Reverse proxy — routes traffic to the right component |
| **Core** | API server — auth, RBAC, projects, events |
| **Registry** | OCI Distribution — stores/serves image layers |
| **JobService** | Background jobs — scanning, replication, GC |
| **Trivy** | CVE vulnerability scanner |
| **PostgreSQL** | Harbor's own DB — projects, users, metadata |
| **Redis** | Session store + job queue |
| **Portal** | Web UI |
| **Robot Account** | Machine identity for CI/CD pipelines |
| **Project** | RBAC namespace — "library" by default |

**Next:** [04 — Kubernetes Fundamentals →](./04-kubernetes-fundamentals.md)
