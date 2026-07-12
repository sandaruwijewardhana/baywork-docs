# 12 — Full Codebase Tour

This file walks through every file in the provisioner backend, explaining what each piece does and which Go concepts from the earlier files it uses.

---

## The Big Picture

```
A tenant calls:  POST /api/v1/tenants/acme/registry  {"plan":"starter"}

1. Gin router routes to handlers/registry.go → Create()
2. Create() writes PENDING record to PostgreSQL → returns 202 immediately
3. Background worker polls DB every 5s → finds PENDING
4. Worker runs 7 steps on Harvester cluster (via kubeconfig in a Secret):
   Step 1: Create namespace "acme-management"
   Step 2: Generate Helm values (admin password, db password)
   Step 3: Helm install Harbor chart
   Step 4: Wait for all Harbor pods to be Running
   Step 5: Call Harbor REST API to set up projects and robot accounts
   Step 6: Encrypt credentials with AES-256-GCM → store in PostgreSQL
   Step 7: Update status to READY, store registry URL
5. Tenant polls: GET /api/v1/tenants/acme/registry → gets READY + URL
```

---

## `cmd/main.go` — Entry Point

**What it does:** Creates all dependencies and wires them together. Starts the HTTP server and background worker. Handles graceful shutdown.

**Key Go concepts:**
- `func main()` — program entry point
- `func init()` — runs before main, creates temp directories
- Dependency injection — creates each component with its dependencies
- Goroutines — `go deployWorker.Run(...)` and `go httpServer.ListenAndServe()`
- Channels — `quit := make(chan os.Signal, 1)` + `<-quit` for shutdown
- `defer` — `defer store.Close()`, `defer logger.Sync()`, `defer cancel()`
- `context.WithTimeout` — graceful shutdown has a 30s deadline

**Read this line by line:**

```go
// Line 26: logger first — everything else needs it
logger, _ := zap.NewProduction()
defer logger.Sync()   // ← flush buffered logs on exit

// Line 29: load config from env vars — panics if required vars missing
cfg, err := config.Load()

// Line 35: DB connection — everything depends on this
store, err := db.New(cfg.DB, logger)
defer store.Close()

// Line 41: run SQL migrations to create/update tables
store.Migrate()

// Line 46: read encryption key from mounted K8s Secret file
masterKey, _ := os.ReadFile(cfg.Encryption.MasterKeyPath)
cipher, _ := crypto.NewCipher(masterKey)

// Line 58: create client that talks to HARVESTER cluster
harvesterK8sClient, _ := k8s.NewHarvesterClient(cfg.Kubernetes.HarvesterKubeconfigPath)

// Line 70: Helm deployer also targets Harvester
helmDeployer, _ := helm.NewDeployer(cfg.Helm, cfg.Kubernetes.HarvesterKubeconfigPath, logger)

// Line 83: background worker — runs as a goroutine
deployWorker := worker.New(store, helmDeployer, harvesterK8sClient, ...)
go deployWorker.Run(context.Background())

// Line 86: HTTP server
srv := api.NewServer(cfg, store, cipher, auditLog, reg, logger)
httpServer := &http.Server{Addr: ":8080", Handler: srv.Router()}
go func() { httpServer.ListenAndServe() }()

// Line 103: wait for shutdown signal
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit   // blocks here

// Line 108: graceful shutdown — wait up to 30s for requests to finish
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
httpServer.Shutdown(ctx)
```

---

## `config/config.go` — Configuration

**What it does:** Reads all environment variables into typed structs. Panics at startup if required variables are missing.

**Key concepts:**
- Nested structs: `Config` contains `ServerConfig`, `DBConfig`, etc.
- `mustEnv()` — panics if env var not set (fail-fast at startup)
- `envStr()` / `envInt()` / `envBool()` — read with defaults
- `strconv.Atoi`, `strconv.ParseBool` — string to typed values

**Why panic on missing config:**
```go
func mustEnv(key string) string {
    v := os.Getenv(key)
    if v == "" {
        panic(fmt.Sprintf("required environment variable %s is not set", key))
    }
    return v
}
```
If `DATABASE_URL` is missing, the pod should crash immediately with a clear error — not run for 5 minutes then fail mysteriously when it first tries to use the DB.

---

## `db/postgres.go` — Database Layer

**What it does:** All PostgreSQL interaction. Defines the data structs that map to DB tables. Provides CRUD operations for deployments, credentials, and audit log.

**Key concepts:**
- `type RegistryStatus string` — custom type prevents invalid status values
- `sql.Open` + `db.PingContext` — connect and verify
- `sql.NullString`, `sql.NullTime` — handle nullable columns
- `row.Scan(&field, &field, ...)` — read query result into variables
- `ExecContext` vs `QueryRowContext` vs `QueryContext`
- `json.Marshal` / `json.Unmarshal` — progress JSONB column
- `FOR UPDATE SKIP LOCKED` — prevents two worker replicas picking the same row

**The critical `GetOldestPending` function:**

```go
func (s *Store) GetOldestPending(ctx context.Context) (*RegistryDeployment, error) {
    hostname, _ := os.Hostname()
    row := s.db.QueryRowContext(ctx, `
        UPDATE registry_deployments           ← atomically claim the row
        SET worker_lock = $1,                 ← mark this pod is handling it
            status = 'DEPLOYING',
            updated_at = now()
        WHERE tenant_id = (
            SELECT tenant_id FROM registry_deployments
            WHERE status = 'PENDING'
            AND worker_lock IS NULL            ← only unclaimed rows
            ORDER BY created_at ASC           ← oldest first
            LIMIT 1
            FOR UPDATE SKIP LOCKED            ← skip rows locked by other workers
        )
        RETURNING ...
    `, hostname)
    return scanDeployment(row)
}
```

This is atomic — two workers running simultaneously will each pick a DIFFERENT PENDING row because `FOR UPDATE SKIP LOCKED` makes one wait and the other proceed.

---

## `crypto/aes.go` — Encryption

**What it does:** AES-256-GCM authenticated encryption for storing robot tokens and admin passwords securely.

**Key concepts:**
- `[]byte` everywhere — crypto works on bytes, not strings
- `io.ReadFull(rand.Reader, nonce)` — cryptographically secure random nonce
- `c.aesGCM.Seal` — encrypt + authenticate
- `c.aesGCM.Open` — decrypt + verify (fails if tampered)
- `hex.DecodeString` — support hex-encoded keys
- Named return values: `(ciphertext, nonce []byte, err error)`

**Why a nonce:**
```
Nonce = Number used ONCE
Every encryption generates a fresh random nonce.
Same plaintext + different nonce = completely different ciphertext.
Both ciphertext AND nonce must be stored to decrypt later.
```

**Why GCM (Galois/Counter Mode):**
GCM is "authenticated encryption" — it not only encrypts but also detects tampering. If someone modifies the ciphertext in the database, `Open` returns an error. This prevents an attacker from flipping bits in an encrypted token to forge privileges.

---

## `k8s/client.go` — Kubernetes Client

**What it does:** Creates and manages namespaces on the Harvester cluster. Applies NetworkPolicies. Waits for pods to become Ready.

**Key concepts:**
- `k8s.io/client-go` — official Go SDK for Kubernetes API
- `clientcmd.BuildConfigFromFlags` — build kubeconfig from file path
- `kubernetes.NewForConfig` — create the typed client
- `clientset.CoreV1().Namespaces().Create(...)` — create K8s objects
- `clientset.CoreV1().Pods().List(...)` — list pods
- Polling loop with context timeout — wait for readiness

**How it targets Harvester:**
```go
func NewHarvesterClient(kubeconfigPath string) (*Client, error) {
    // Load the kubeconfig file (mounted from a K8s Secret)
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfigPath)
    if err != nil {
        return nil, fmt.Errorf("build kubeconfig: %w", err)
    }
    // Create the typed client targeting HARVESTER API server
    clientset, err := kubernetes.NewForConfig(config)
    ...
}
```

---

## `helm/deployer.go` — Helm Deployment

**What it does:** Downloads the Harbor Helm chart and installs it into a tenant namespace on Harvester. Checks if already installed before installing.

**Key concepts:**
- `helm.sh/helm/v3/pkg/action` — Helm Go SDK (equivalent of `helm install`)
- `action.NewInstall` → configure → `install.RunWithContext` → runs `helm install`
- `action.NewHistory` → check if release exists (skip if already deployed)
- Environment settings (`cli.EnvSettings`) for chart cache directories

**Critical skip-if-exists logic:**
```go
func (d *Deployer) Install(ctx context.Context, tenantID, namespace string, values []byte) error {
    // Check if already deployed — don't reinstall (would regenerate passwords)
    hist := action.NewHistory(d.cfg)
    hist.Max = 1
    if _, err := hist.Run(releaseName); err == nil {
        d.logger.Info("helm release already exists, skipping install")
        return nil   // ← skip — Harbor is already running with correct passwords
    }

    // Pull chart from registry
    pull := action.NewPullWithOpts(...)
    pull.RepoURL = d.cfg.HarborRepoURL
    pull.Version = d.cfg.HarborChartVer
    pull.Run("harbor")   // downloads harbor-1.14.0.tgz to /tmp/helm-charts/

    // Install
    install := action.NewInstall(d.cfg)
    install.ReleaseName = releaseName
    install.Namespace = namespace
    install.RunWithContext(ctx, chart, vals)
}
```

---

## `harbor/bootstrap.go` — Harbor API Setup

**What it does:** After Harbor pods are running, calls Harbor's REST API to:
1. Wait for the API to be reachable (`/api/v2.0/ping`)
2. Update the admin password
3. Create the `library` project (default image repository)
4. Create a robot account with push/pull access
5. Return the robot credentials

**Key concepts:**
- `http.Client` with custom TLS config (insecure for dev)
- HTTP request with `Authorization: Basic` header (base64-encoded)
- `c.Request.Context()` passed through for cancellation
- Wait-and-retry loop with exponential backoff

---

## `worker/deploy_worker.go` — The Orchestrator

**What it does:** The core logic — polls the database and drives the 7-step deploy flow.

**Key concepts:**
- `time.NewTicker(5 * time.Second)` — poll interval
- `select { case <-ctx.Done(): ... case <-ticker.C: ... }` — clean shutdown
- `go w.deploy(...)` — each deploy runs in its own goroutine
- Local `fail` function (closure) — avoids repeating error handling code
- `context.WithTimeout` — timeout for each phase

**The `fail` closure:**
```go
func (w *Worker) deploy(ctx context.Context, d *db.RegistryDeployment) {
    // ...

    // Define a helper that handles any step failure uniformly
    fail := func(step string, err error) {
        msg := fmt.Sprintf("step %s failed: %v", step, err)
        w.logger.Error("provision failed",
            zap.String("tenant", tenantID),
            zap.String("step", step),
            zap.Error(err))
        w.store.UpdateDeploymentStatus(ctx, tenantID, db.StatusFailed, msg)
        w.metrics.ProvisionResult(tenantID, "failure")
    }

    // Step 1: namespace
    if err := w.k8s.EnsureNamespace(ctx, namespace, tenantID); err != nil {
        fail("namespace", err)
        return   // ← stop this deploy
    }
    // ... repeat for each step
```

`fail` is a closure — it captures `tenantID`, `w`, and `ctx` from the surrounding scope. This avoids repeating 4 lines of error handling for each of the 7 steps.

---

## `api/server.go` — HTTP Server

**What it does:** Sets up the Gin router with all routes, middleware, and the health check endpoints.

**Key concepts:**
- `gin.New()` — empty router (vs `gin.Default()` which includes logger+recovery)
- `r.Group(...)` — prefix groups with shared middleware
- Middleware chain ordering matters — applied in the order you add them

---

## `api/handlers/registry.go` — Request Handlers

**What it does:** 6 handlers for the 6 API endpoints. Each one: reads input → validates → does business logic → writes response.

**Key patterns:**
- `c.ShouldBindJSON(&req)` → validate request body
- `c.Request.Context()` → pass to DB/K8s for cancellation
- `switch existing.Status` → handle all deployment lifecycle states
- Early returns after errors — never fall through to success path
- `h.auditLog.Log(...)` — append-only audit trail after every mutation

---

## `api/middleware/` — Cross-Cutting Concerns

**`auth.go`:** JWT validation or DEV_SKIP_AUTH bypass. Stores claims in Gin context.

**`logging.go`:** Logs every request with method, path, status, duration, client IP.

**`ratelimit.go`:** One provision per tenant per 10 minutes. Uses in-memory map + mutex.

---

## `audit/logger.go` — Audit Log

**What it does:** Non-blocking audit log writer. Uses a goroutine to write to DB asynchronously so audit log failures don't block the API response.

---

## `metrics/prometheus.go` — Prometheus Metrics

**What it does:** Counters and histograms tracked with labels:
- `provision_total{tenant, result}` — count of provisions (success/failure)
- `provision_duration_seconds{tenant}` — how long each provision took
- `credential_fetch_total{tenant}` — how often credentials are read

Gin's `/metrics` endpoint exposes these for Prometheus to scrape.

---

## Reading Checklist

Go through these files in this order, with the concepts in mind:

| File | Focus On |
|------|---------|
| `config/config.go` | Struct nesting, env reading, mustEnv pattern |
| `db/postgres.go` | Custom types, sql.Scan, GetOldestPending query |
| `crypto/aes.go` | []byte, io.ReadFull, named returns |
| `cmd/main.go` | DI wiring order, goroutines, channels, defer |
| `worker/deploy_worker.go` | select loop, closures, context.WithTimeout |
| `api/server.go` | Gin router, Group, Use, middleware chain |
| `api/handlers/registry.go` | ShouldBindJSON, switch on status, early returns |
| `api/middleware/auth.go` | Middleware pattern, c.Set/c.Get, AbortWithStatus |
| `harbor/bootstrap.go` | HTTP client, retry loop, context cancellation |
| `helm/deployer.go` | Helm SDK, skip-if-exists logic |
| `k8s/client.go` | client-go, kubeconfig from file, polling loop |

---

## Final Exercise — Trace a Full Request

Trace `POST /api/v1/tenants/acme/registry {"plan":"starter"}` through every file:

1. Which Gin middleware runs before the handler? In what order?
2. Which function validates `"plan": "starter"`?
3. What row gets written to PostgreSQL? What SQL is executed?
4. When does the background worker pick it up? How soon?
5. What does `GetOldestPending` return? How does it prevent double-deploy?
6. What files/secrets does Step 2 (generate values) need?
7. Which Helm functions run in Step 3?
8. How does Step 4 know all pods are ready?
9. What HTTP calls does Step 5 make to Harbor?
10. Where are the credentials stored? In what form?
11. What does Step 7 write to PostgreSQL?
12. What does the tenant see when they poll `GET /api/v1/tenants/acme/registry`?

Answer each question by pointing to the file and line number.

---

## 🧠 Retention — lock the whole tutorial in (this is your exam chapter)

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). This chapter integrates all 11 before it — if you can do the Final Exercise from memory, the tutorial stuck.

### Recall questions (no peeking)
1. List the 7 deploy steps in order, from memory.
2. How does `GetOldestPending` stop two worker replicas from deploying the same tenant?
3. Why are credentials stored encrypted, and what does the **GCM** in AES-256-GCM add over plain encryption?
4. Why does `helm.Install` skip when the release already exists?
5. What is the `fail` closure, and which variables does it capture from the surrounding scope?

### Make these Anki cards (front → back)
- 7-step deploy → namespace → generate values → helm install → wait pods ready → Harbor bootstrap → encrypt + store creds → mark READY
- `GetOldestPending` → atomic `UPDATE … WHERE … FOR UPDATE SKIP LOCKED` claims exactly one row per worker
- AES-**GCM** → authenticated encryption: `Open` **fails** if the ciphertext was tampered with
- Helm skip-if-exists → avoids regenerating Harbor's admin/db passwords on an already-running release
- `fail` closure → captures `tenantID`, `w`, `ctx`; sets status `FAILED` uniformly for any step

### Spaced-repetition schedule — this is the recurring Phase exam
- [ ] **Day 1:** make the Anki cards; attempt the **Final Exercise (trace a full request, 12 Qs)** with the code open.
- [ ] **Day 7 (Friday redo):** redo the **Final Exercise from memory** — file + line for each of the 12 questions. Whatever you can't answer is your re-read list for the week.
- [ ] **Weekly thereafter:** repeat the Final Exercise until you can answer all 12 cold. That fluency is the prerequisite for the **Source-Code Track** in the SRE curriculum (reading Harbor/Rancher controllers is the same skill, scaled up).
- [ ] **Capstone:** teach the full request flow to someone (or to Claude) with no notes — the Feynman test. Gaps = your next study targets.
