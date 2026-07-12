# 06 — Go Language & REST APIs

> **Goal:** Understand Go's key concepts (goroutines, channels, error handling, packages) and how the Gin framework builds the REST API in this project.

---

## 1. Why Go?

Go was chosen for the provisioner because:

| Property | Why It Matters Here |
|----------|-------------------|
| **Compiled to a single binary** | Easy to put in a distroless container with no runtime |
| **Built-in concurrency** | The deploy worker runs alongside the HTTP server effortlessly |
| **Fast startup** | Container starts in milliseconds |
| **Strong typing** | Catch bugs at compile time, not in production |
| **Great K8s/Helm SDKs** | `client-go` and `helm.sh/helm/v3` are Go-native |

---

## 2. Go Fundamentals

### Variables and Types

```go
// Declare and assign
name := "tenant-a"          // type inferred as string
port := 8080                // type inferred as int
isReady := false            // bool

// Explicit type
var count int = 0
var status string = "PENDING"
```

### Structs — Go's version of classes

```go
// Define a struct
type RegistryDeployment struct {
    TenantID  string
    Namespace string
    Status    RegistryStatus    // a custom type (string underneath)
    Plan      string
    CreatedAt time.Time
}

// Create an instance
d := RegistryDeployment{
    TenantID:  "tenant-a",
    Namespace: "tenant-a-management",
    Status:    StatusPending,
}

// Access fields
fmt.Println(d.TenantID)   // "tenant-a"
```

### Methods on Structs

```go
// Method with a value receiver (gets a copy)
func (d RegistryDeployment) IsReady() bool {
    return d.Status == StatusReady
}

// Method with a pointer receiver (can modify the struct)
func (s *Store) Ping() error {
    return s.db.Ping()
}
```

### Error Handling — No Exceptions

Go has **no exceptions**. Errors are regular values returned from functions:

```go
// Every function that can fail returns an error as the last value
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil    // nil means "no error"
}

// Caller must check the error
result, err := divide(10, 2)
if err != nil {
    log.Fatal("division failed:", err)
}
fmt.Println(result)  // 5.0
```

Real example from this codebase:

```go
// From backend/cmd/main.go
store, err := db.New(cfg.DB, logger)
if err != nil {
    logger.Fatal("failed to connect to database", zap.Error(err))
    // Fatal = log the error + os.Exit(1) — process stops
}
```

### Error Wrapping — Adding Context

```go
// Good: wrap errors with context so the caller knows where it came from
if _, err := install.RunWithContext(ctx, chart, vals); err != nil {
    return fmt.Errorf("helm install: %w", err)
    //                                   ^^ %w wraps the original error
}

// Caller sees: "helm install: dial tcp: connection refused"
// instead of just: "dial tcp: connection refused"
```

---

## 3. Interfaces — Flexible Abstractions

An interface defines a **behaviour** (a set of methods). Any type that has those methods satisfies the interface automatically:

```go
// Define an interface
type Deployer interface {
    Install(ctx context.Context, tenantID, namespace string, values []byte) error
    Uninstall(ctx context.Context, tenantID, namespace string) error
}

// HelmDeployer satisfies Deployer (has both methods)
type HelmDeployer struct { ... }
func (d *HelmDeployer) Install(...) error { ... }
func (d *HelmDeployer) Uninstall(...) error { ... }

// MockDeployer also satisfies Deployer (for tests!)
type MockDeployer struct { called bool }
func (d *MockDeployer) Install(...) error { d.called = true; return nil }
func (d *MockDeployer) Uninstall(...) error { return nil }
```

---

## 4. Goroutines — Lightweight Threads

A **goroutine** is a function running concurrently. Start one with the `go` keyword:

```go
// This runs in the SAME goroutine (blocking)
helmDeployer.Install(ctx, tenantID, namespace, values)   // waits here...

// This runs in a NEW goroutine (non-blocking)
go helmDeployer.Install(ctx, tenantID, namespace, values) // returns immediately
```

In `main.go`, the provisioner runs two things simultaneously:

```go
// Goroutine 1: background worker (polls DB every 5s)
go deployWorker.Run(context.Background())

// Goroutine 2: HTTP server (handles API requests)
go func() {
    httpServer.ListenAndServe()
}()

// Main goroutine: wait for shutdown signal
<-quit   // blocks here until SIGTERM received
```

The goroutines run **at the same time** — the HTTP server serves requests while the worker is deploying Harbor in the background.

### Channels — Goroutine Communication

Channels pass values between goroutines safely:

```go
quit := make(chan os.Signal, 1)       // create a channel of type os.Signal
signal.Notify(quit, syscall.SIGTERM) // OS sends SIGTERM into this channel

// In main goroutine:
<-quit    // BLOCKS until a signal arrives, then continues

// After this line: gracefully shutdown the HTTP server
```

---

## 5. Context — Cancellation and Deadlines

`context.Context` passes **cancellation signals** and **deadlines** through call chains:

```go
// Create a context with a timeout
podCtx, cancel := context.WithTimeout(ctx, 8*time.Minute)
defer cancel()   // ← always call cancel to free resources

// Pass to the function — it will stop if the timeout expires
err = w.k8s.WaitForAllReady(podCtx, namespace, progressCb)
// If 8 minutes pass: podCtx.Done() fires, WaitForAllReady returns an error
```

```
main context (from os.Signal)
  └── deploy context
        └── podCtx (8 min timeout)
              └── bootstrapCtx (3 min timeout)
```

If the main context is cancelled (shutdown), all child contexts are cancelled too — cleanly stopping all ongoing work.

---

## 6. The Gin Framework — HTTP Server

**Gin** is a fast HTTP framework for Go. It handles routing, middleware, and request/response.

### Router Setup

```go
// From backend/internal/api/server.go
r := gin.New()

// Global middleware (runs for every request)
r.Use(middleware.Recovery(s.logger))       // recover from panics
r.Use(middleware.RequestLogger(s.logger))  // log every request
r.Use(cors.New(...))                       // allow browser requests

// Route groups
api := r.Group("/api/v1")          // prefix: /api/v1
api.Use(middleware.ByTenant(...))  // rate limiting for this group
api.Use(middleware.DevAuthBypass()) // auth (dev mode)

tenants := api.Group("/tenants/:tenantId")  // URL parameter
tenants.Use(middleware.TenantGuard())

registry := tenants.Group("/registry")
registry.POST("",    h.Create)             // POST /api/v1/tenants/{id}/registry
registry.GET("",     h.Get)                // GET  /api/v1/tenants/{id}/registry
registry.DELETE("",  h.Delete)             // DELETE ...
registry.GET("/credentials",   h.GetCredentials)
registry.POST("/credentials/rotate", h.RotateCredentials)
registry.GET("/pull-secret",   h.GetPullSecret)
```

### Handler — Processing a Request

```go
// From backend/internal/api/handlers/registry.go (conceptual)
func (h *Handler) Create(c *gin.Context) {
    // 1. Read the URL parameter
    tenantID := c.Param("tenantId")

    // 2. Parse the JSON request body
    var req CreateRegistryRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "INVALID_BODY"})
        return
    }

    // 3. Business logic
    err := h.store.CreateDeployment(c.Request.Context(), &db.RegistryDeployment{
        TenantID:  tenantID,
        Namespace: tenantID + "-management",
        Status:    db.StatusPending,
        Plan:      req.Plan,
    })
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "DB_ERROR"})
        return
    }

    // 4. Respond
    c.JSON(http.StatusAccepted, gin.H{
        "tenantId": tenantID,
        "status":   "PENDING",
    })
}
```

### Middleware Pattern

Middleware is a function that wraps every request. It can:
- **Inspect/modify** the request (add claims, validate auth)
- **Short-circuit** (return 401 before the handler runs)
- **Run after** the handler (log timing)

```
Request arrives
     │
     ▼
Recovery middleware    ← catch panics, return 500
     │
     ▼
RequestLogger          ← log the request
     │
     ▼
CORS middleware        ← add CORS headers
     │
     ▼
RateLimiter            ← check rate limit
     │
     ▼
DevAuthBypass          ← inject fake claims (dev) OR validate JWT (prod)
     │
     ▼
TenantGuard            ← check JWT tenantId matches URL param
     │
     ▼
Handler (Create/Get/Delete...)
     │
     ▼
Response sent
     │
     ▼
RequestLogger after    ← log duration and status code
```

```go
// Middleware structure
func SomeMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // ← runs BEFORE the handler
        doSomethingBefore()

        c.Next()   // ← call the next middleware/handler

        // ← runs AFTER the handler
        doSomethingAfter()
    }
}
```

---

## 7. Packages — Code Organisation

Go organises code into **packages**. Each directory = one package:

```
backend/
  cmd/
    main.go                  package main    ← entry point
  internal/
    api/
      server.go              package api     ← HTTP server
      handlers/
        registry.go          package handlers
      middleware/
        auth.go              package middleware
    db/
      postgres.go            package db
    helm/
      deployer.go            package helm
      values_generator.go   package helm    ← same package, different file
    k8s/
      client.go              package k8s
    crypto/
      aes.go                 package crypto
    harbor/
      bootstrap.go           package harbor
    worker/
      deploy_worker.go       package worker
```

The `internal/` directory means these packages **cannot be imported** by code outside this module — a built-in way to hide implementation details.

---

## 8. Dependency Injection — Wiring Everything Together

In `main.go`, all dependencies are created and wired manually:

```go
func main() {
    // Create dependencies bottom-up
    store, _   := db.New(cfg.DB, logger)         // DB first (no deps)
    cipher, _  := crypto.NewCipher(masterKey)    // Crypto (no deps)
    k8sClient, _ := k8s.NewClient(cfg.Kubernetes) // K8s client (no deps)
    helmDeployer, _ := helm.NewDeployer(cfg.Helm, logger)

    auditLog  := audit.NewLogger(store, logger)
    reg       := metrics.NewRegistry()

    // Worker needs: store, helm, k8s, cipher, auditLog, metrics
    deployWorker := worker.New(store, helmDeployer, k8sClient,
                               cipher, cfg.Helm, auditLog, reg, logger)

    // API server needs: store, cipher, auditLog, metrics
    srv := api.NewServer(cfg, store, cipher, auditLog, reg, logger)
}
```

Each component receives its dependencies through the **constructor** (`New...`). This makes testing easy — you can pass a mock database or mock Helm deployer.

---

## 9. Structured Logging with Zap

The project uses `go.uber.org/zap` for structured logging (key-value pairs instead of plain text):

```go
// Plain text logging (hard to search/filter in production)
log.Printf("Harbor deployment failed for tenant %s: %v", tenantID, err)
// Output: 2024/01/15 10:30:00 Harbor deployment failed for tenant acme: dial tcp: ...

// Structured logging (searchable, parseable)
logger.Error("provision failed",
    zap.String("tenant", tenantID),
    zap.String("step", "helm_install"),
    zap.Error(err),
)
// Output (JSON): {"level":"error","msg":"provision failed","tenant":"acme","step":"helm_install","error":"dial tcp: ..."}
```

In production, JSON logs are ingested by tools like Loki, Elasticsearch, or Datadog for searching and alerting.

---

## 🏋️ Exercises

### Exercise 1 — Read main.go Top-to-Bottom
Open [backend/cmd/main.go](../backend/cmd/main.go). Write down in plain English what happens in order:
1. Logger created
2. Config loaded
3. ... (continue for all steps)

### Exercise 2 — Trace an API Request
Manually trace `GET http://localhost:8080/api/v1/tenants/dev-tenant/registry`:
- Which middleware runs first?
- Which handler processes it?
- What does it return if no registry exists?
- What does it return if one exists?

```bash
curl -s http://localhost:8080/api/v1/tenants/dev-tenant/registry | python3 -m json.tool
```

### Exercise 3 — Error Handling Pattern
In Go, the pattern `if err != nil { return ..., err }` appears everywhere. Count how many times it appears in [backend/internal/worker/deploy_worker.go](../backend/internal/worker/deploy_worker.go).

Why does Go prefer this over exceptions (try/catch)?

### Exercise 4 — Add a Simple Route
Conceptually, add a route `GET /api/v1/version` that returns `{"version": "1.0.0"}`.
Where in [backend/internal/api/server.go](../backend/internal/api/server.go) would you add it?
Write the handler code.

### Exercise 5 — Goroutine Understanding
In `main.go`:
```go
go deployWorker.Run(context.Background())

go func() {
    logger.Info("registry provisioner starting", ...)
    httpServer.ListenAndServe()
}()

<-quit
```

- How many goroutines are running after these 3 lines?
- What happens if the HTTP server crashes? Does the deploy worker stop?
- What triggers `<-quit` to unblock?

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Struct** | A named collection of fields (like a class without inheritance) |
| **Method** | A function attached to a struct |
| **Interface** | A set of method signatures — satisfied implicitly |
| **error** | A regular return value — not an exception |
| **Goroutine** | A lightweight concurrent function (`go func()`) |
| **Channel** | A pipe between goroutines (`<-`, `chan`) |
| **Context** | Carries cancellation/timeouts through call chains |
| **Gin** | HTTP framework: routing, middleware, JSON I/O |
| **Middleware** | A function that wraps every request in a route group |
| **Package** | A directory of Go files — the unit of organisation |
| **Zap** | Structured JSON logging library |

**Next:** [07 — PostgreSQL & Database Patterns →](./07-postgresql-and-database.md)
