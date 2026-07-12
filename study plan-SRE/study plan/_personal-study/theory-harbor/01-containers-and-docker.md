# 01 — Containers & Docker

> **Goal:** Understand what a container is, how Docker works, and why this project packages its Go server as a Docker image.

---

## 1. The Problem Containers Solve

Before containers, deploying software meant:

```
Developer: "works on my machine" 🤷
Server:    crashes — missing dependency, wrong OS version, config differs
```

### Old Way vs Container Way

```
OLD WAY                          CONTAINER WAY
─────────────────────────        ─────────────────────────
App needs Python 3.11            App ships WITH Python 3.11
  └── you install on server        └── baked into the image
  └── hope versions match          └── always the right version

App needs libssl.so.1.1          libssl.so.1.1 is inside
  └── might not be on server        the container image
  └── version conflicts            └── no conflict possible
```

A container is a **process that thinks it's on its own OS** — it has its own filesystem, its own network interface, its own view of running processes — but it actually shares the host's Linux kernel.

---

## 2. Virtual Machines vs Containers

```
VIRTUAL MACHINE                  CONTAINER
────────────────                 ──────────
┌──────────────────┐             ┌──────────────────┐
│   Your App       │             │   Your App       │
├──────────────────┤             ├──────────────────┤
│   Guest OS       │             │   Libs/Deps      │
│  (full Linux,    │             │   (just what     │
│   ~1-2 GB)       │             │    you need,     │
├──────────────────┤             │    ~50-200 MB)   │
│   Hypervisor     │             ├──────────────────┤
├──────────────────┤             │   Container      │
│   Host OS        │             │   Runtime        │
├──────────────────┤             ├──────────────────┤
│   Hardware       │             │   Host OS        │
└──────────────────┘             ├──────────────────┤
                                 │   Hardware       │
Boots in: ~30-60 seconds         └──────────────────┘
Size: GB                         
                                 Starts in: ~1 second
                                 Size: MB
```

Containers are faster and smaller because they **share the host kernel** instead of emulating an entire computer.

---

## 3. How Docker Works

Docker is the most popular tool for building and running containers.

```
┌───────────────────────────────────────────────────────┐
│                     Docker                            │
│                                                       │
│   Dockerfile  ──build──►  Image  ──run──►  Container │
│                                                       │
│   (recipe)              (snapshot)        (running    │
│                                            process)   │
└───────────────────────────────────────────────────────┘
```

### The Dockerfile (a recipe for an image)

```dockerfile
# Start from an existing image as the base
FROM golang:1.22-alpine AS builder

# Set the working directory inside the build container
WORKDIR /app

# Copy dependency files first (for layer caching)
COPY go.mod ./
RUN go mod download && go mod tidy

# Copy the rest of the code
COPY . .

# Compile the Go binary
RUN CGO_ENABLED=0 GOOS=linux go build -o /registry-provisioner ./cmd/main.go

# --- Second stage: tiny production image ---
FROM gcr.io/distroless/static-debian12:nonroot

# Copy only the compiled binary from the builder stage
COPY --from=builder /registry-provisioner /registry-provisioner

# Tell Docker what port the app uses (documentation only)
EXPOSE 8080

# The command to run when the container starts
ENTRYPOINT ["/registry-provisioner"]
```

> This is the **actual Dockerfile** from `backend/Dockerfile`. Notice it has two stages — this is called a **multi-stage build**.

---

## 4. Image Layers

Images are built in **layers**, like a stack of pancakes. Each `RUN`, `COPY`, `ADD` instruction creates a new layer.

```
Layer 5: COPY binary           ← tiny, changes every build
Layer 4: RUN go build          ← large, but cached if code unchanged
Layer 3: COPY . .              ← changes with code
Layer 2: RUN go mod download   ← cached until go.mod changes
Layer 1: FROM golang:1.22      ← cached after first pull
```

**Why this matters:** Docker caches layers. If `go.mod` hasn't changed, `go mod download` doesn't re-run. That's why `go.mod` is copied **before** the code — so the slow download step is cached.

---

## 5. Multi-Stage Builds

The project uses a two-stage build to get a **tiny, secure final image**:

```
Stage 1 (builder):               Stage 2 (final):
┌────────────────────────┐        ┌────────────────────────┐
│ golang:1.22-alpine     │        │ distroless/static      │
│  ~300 MB               │        │  ~2 MB                 │
│                        │        │                        │
│  - Go compiler         │        │  - Just the binary     │
│  - Build tools         │        │  - CA certificates     │
│  - Source code         │  copy  │  - No shell (!)        │
│  - Dependencies        │ ─────► │  - No package manager  │
│  - go.sum              │ binary │  - Non-root user       │
└────────────────────────┘        └────────────────────────┘
          ↓                                 ↓
  Used to compile              Used in production
  the binary                   Final image: ~8 MB
```

Benefits:
- **Smaller attack surface** — no shell means attackers can't run commands even if they break in
- **Smaller size** — ships nothing except what's needed
- **Non-root** — the process runs as user 65534 (nobody)

---

## 6. Running Containers with Docker Compose

For local development, instead of typing long `docker run` commands, we use `docker-compose.dev.yml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine        # Use this pre-built image
    environment:
      POSTGRES_DB: registry_provisioner  # Pass env vars into container
    ports:
      - "5432:5432"                  # host_port:container_port
    volumes:
      - postgres-data:/var/lib/postgresql/data  # Persist data

  provisioner:
    build:
      context: ./backend             # Build from this folder's Dockerfile
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy   # Don't start until postgres is ready
```

```
Host Machine                     Docker Network
────────────────                 ──────────────────────────────────
localhost:8080  ◄──────────────► provisioner:8080
localhost:5432  ◄──────────────► postgres:5432
localhost:3000  ◄──────────────► ui:3000
                                         │
                                    Internal: provisioner
                                    connects to postgres:5432
                                    (using service name as hostname)
```

---

## 7. Container Lifecycle

```
                ┌─────────┐
           ─────►  Created │  docker create / docker compose up
                └────┬────┘
                     │ docker start
                ┌────▼────┐
           ─────►  Running │  process is executing
                └────┬────┘
               ┌─────┴──────┐
               │            │
          ┌────▼────┐  ┌────▼────┐
          │ Paused  │  │ Stopped │  SIGTERM sent, process exits
          └────┬────┘  └────┬────┘
               │            │ docker rm
               │       ┌────▼────┐
               └──────►│ Deleted │
                        └─────────┘
```

When Kubernetes manages containers, it handles this lifecycle automatically — restarting crashed containers, stopping them when updated, etc.

---

## 8. How This Project Uses Docker

```
developer laptop:
  docker compose -f docker-compose.dev.yml up --build
        │
        ├── builds provisioner image (from backend/Dockerfile)
        ├── pulls postgres:16-alpine (pre-built)
        ├── pulls node:20-alpine (for UI)
        └── starts all 3 containers connected on same network

production (Kubernetes):
  - provisioner image is pushed to a registry
  - Kubernetes pulls and runs it as a Deployment
  - Harbor images run in tenant namespaces (managed by this system!)
```

---

## 🏋️ Exercises

### Exercise 1 — Read a Dockerfile
Open [backend/Dockerfile](../backend/Dockerfile). Answer:
- How many stages does it have?
- What is the base image of the final stage?
- What port does it expose?
- Why is `go.mod` copied before the rest of the code?

### Exercise 2 — Run a Container Manually
```bash
# Pull and run a simple container
docker run --rm alpine:3.19 echo "Hello from inside a container"

# Run interactively (get a shell inside)
docker run --rm -it alpine:3.19 sh

# Inside the shell, try:
ls /           # see the filesystem
cat /etc/os-release  # what OS is this?
ps aux         # what processes are running?
exit
```

### Exercise 3 — Inspect the Running Provisioner
```bash
# See all running containers
docker ps

# Get a shell inside the provisioner (distroless has no shell, but try postgres)
docker exec -it implementation-of-auto-harbor-deployment-postgres-1 sh

# Inside postgres container:
psql -U provisioner -d registry_provisioner
\dt    # show tables
\q     # quit
exit
```

### Exercise 4 — Understand Layers
```bash
# Inspect the layers of the provisioner image
docker history implementation-of-auto-harbor-deployment-provisioner

# What's the total size? How many layers?
```

### Exercise 5 — Build the Image Yourself
```bash
cd '/home/sandaruw/Service as a registry/implementation-of-auto-harbor-deployment/backend'
docker build -t my-provisioner:test .

# How long does it take?
# Now run it again — is it faster? Why?
docker build -t my-provisioner:test .
```

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Container** | An isolated process with its own filesystem, shares host kernel |
| **Image** | A read-only snapshot used to create containers |
| **Dockerfile** | A recipe that describes how to build an image |
| **Layer** | Each instruction in a Dockerfile creates a cached layer |
| **Multi-stage build** | Build in a big image, copy only the output to a small final image |
| **Docker Compose** | Tool to run multiple containers together (used for local dev) |
| **Distroless** | A minimal base image with no shell or package manager |

**Next:** [02 — Container Registries →](./02-container-registry.md)
