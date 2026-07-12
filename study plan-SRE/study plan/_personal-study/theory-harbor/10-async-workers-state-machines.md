# 10 — Async Workers & State Machines

> **Goal:** Understand why Harbor deployment is asynchronous, how the polling worker works, what a state machine is, and how the 7-step deploy flow runs.

---

## 1. Why Asynchronous?

Deploying a Harbor instance takes **5–10 minutes**. What if the API call waited for it?

```
SYNCHRONOUS (bad):
  Client: POST /registry
  Server: ...deploying Harbor... (10 minutes)
  Client: timeout after 30 seconds → error! (connection dropped)
  Server: still deploying... but client already gave up
  Result: half-deployed Harbor, confused client

ASYNCHRONOUS (this system):
  Client: POST /registry
  Server: INSERT INTO DB (status=PENDING) → return 202 Accepted immediately
  Server: background worker picks it up → deploys over 10 minutes
  Client: polls GET /registry every 5 seconds
  Client: sees status change: PENDING → DEPLOYING → READY
  Total client wait: 0ms for initial response; polls to see progress
```

---

## 2. The Queue Pattern

The database acts as a **job queue**:

```
                        Database
                        ────────
HTTP handler            tenant_a: PENDING   ← job waiting
(API server)     ──►    tenant_b: DEPLOYING ← being processed
INSERT new job           tenant_c: READY    ← done

                            ▲
                            │ polls every 5s
                        ┌───┴──────────────┐
                        │   Deploy Worker  │
                        │  (goroutine)     │
                        └──────────────────┘
```

---

## 3. The Worker Loop

```go
// From backend/internal/worker/deploy_worker.go

func (w *Worker) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)  // fires every 5 seconds
    defer ticker.Stop()

    w.logger.Info("deploy worker started")

    for {
        select {
        case <-ctx.Done():
            // Context cancelled (server shutting down) → stop
            w.logger.Info("deploy worker stopped")
            return
        case <-ticker.C:
            // Every 5 seconds: look for a PENDING job
            w.processPending(ctx)
        }
    }
}

func (w *Worker) processPending(ctx context.Context) {
    // Atomically claim the oldest PENDING deployment
    deployment, err := w.store.GetOldestPending(ctx)
    if err != nil || deployment == nil {
        return  // no PENDING jobs → do nothing this tick
    }

    w.logger.Info("picked up pending deployment",
        zap.String("tenant", deployment.TenantID))

    // Run the full 7-step deploy in its own goroutine
    // so the worker loop keeps ticking every 5s
    go w.deploy(context.Background(), deployment)
}
```

```
Timeline (two tenants request registries at the same time):

t=0s:   tenant-a requests registry → DB: tenant-a PENDING
t=1s:   tenant-b requests registry → DB: tenant-b PENDING
t=5s:   worker tick → GetOldestPending → picks tenant-a (oldest)
        → go w.deploy(tenant-a)   [goroutine A started]
t=10s:  worker tick → GetOldestPending → picks tenant-b
        → go w.deploy(tenant-b)   [goroutine B started, A still running]
t=15s:  worker tick → GetOldestPending → no more PENDING → do nothing
        (goroutines A and B are still deploying)
t=8m:   goroutine A finishes → tenant-a: READY
t=9m:   goroutine B finishes → tenant-b: READY
```

---

## 4. State Machine

A **state machine** is a system with a finite set of states and rules for which transitions are allowed.

```
Harbor Deployment State Machine:

         ┌─────────────────────────────────────┐
         │                                     │
  ┌──────▼──────┐                         (retry: create new)
  │  NO_REGISTRY│  ← tenant has no registry
  └──────┬──────┘
         │ POST /registry
         ▼
  ┌──────────────┐
  │   PENDING    │  ← row created, worker hasn't picked it up yet
  └──────┬───────┘
         │ worker picks it up
         ▼
  ┌──────────────┐
  │  DEPLOYING   │  ← worker is running the 7 steps
  └──────┬───────┘
         │              │
    success             │ any step fails
         │              ▼
         │        ┌──────────┐
         │        │  FAILED  │  ← error stored in DB
         │        └──────────┘
         ▼
  ┌──────────────┐
  │    READY     │  ← Harbor is live, credentials available
  └──────┬───────┘
         │ DELETE /registry
         ▼
  ┌──────────────┐
  │  DELETING    │  ← Helm uninstall in progress
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │   DELETED    │
  └──────────────┘
```

The state is enforced by:
1. PostgreSQL `CHECK` constraint — rejects invalid status values
2. The worker only processes `PENDING` rows
3. Code only transitions in the correct direction

---

## 5. The 7-Step Deploy Flow

```go
func (w *Worker) deploy(ctx context.Context, d *db.RegistryDeployment) {
    // Start a Prometheus timer (measures how long deployment takes)
    timer := w.metrics.StartProvisionTimer(d.TenantID)
    defer timer.ObserveDuration()  // records duration when function returns

    fail := func(step string, err error) {
        w.store.UpdateDeploymentStatus(ctx, tenantID, db.StatusFailed, errorMsg)
        w.metrics.ProvisionResult(tenantID, "failure")
    }
```

### Step 1: Ensure Namespace

```go
w.setProgress(ctx, tenantID, "namespace", "STARTING")

// Create: tenant-a-management namespace with labels
w.k8s.EnsureNamespace(ctx, namespace, tenantID)

// Apply network policy (tenant isolation)
w.k8s.ApplyNetworkPolicy(ctx, namespace)

w.setProgress(ctx, tenantID, "namespace", "READY")
```

```
Kubernetes:
  Namespace "tenant-a-management" created
    Labels: lkdc.wso2.com/tenant: tenant-a
  NetworkPolicy "deny-cross-tenant" created
    → blocks all cross-tenant traffic
```

### Step 2: Generate Helm Values

```go
plan, _ := helm.PlanFor(d.Plan)  // get resource sizes for this plan

// Generate cryptographically random passwords
adminPass, _ := crypto.GeneratePassword(32)  // "xK9!mP2@qR5#..."
dbPass, _    := crypto.GeneratePassword(32)  // "pL7$nQ3@wT6#..."

// Render the YAML template with tenant-specific values
values, _ := helm.GenerateValues(helm.ValuesInput{
    TenantID:   tenantID,
    AdminPass:  adminPass,
    DBPass:     dbPass,
    BaseDomain: "lkdc.wso2.com",
    Plan:       plan,
})
// Result: 200 lines of Harbor Helm values YAML
```

### Step 3: Helm Install

```go
w.store.UpdateDeploymentStatus(ctx, tenantID, db.StatusDeploying, "")
w.setProgress(ctx, tenantID, "helm_install", "STARTING")

// Tell Helm to install Harbor in the tenant's namespace
w.helm.Install(ctx, tenantID, namespace, values)
// Creates: ~25 K8s objects (Deployments, Services, PVCs, etc.)

w.setProgress(ctx, tenantID, "helm_install", "READY")
```

```
Kubernetes after this step:
  namespace/tenant-a-management:
    deployment/harbor-harbor-core        0/1 Running (starting)
    deployment/harbor-harbor-registry    0/1 Running
    deployment/harbor-harbor-portal      0/1 Running
    deployment/harbor-harbor-jobservice  0/1 Running
    deployment/harbor-harbor-database    0/1 Running
    deployment/harbor-harbor-redis       0/1 Running
    pvc/harbor-harbor-registry           Bound
    pvc/harbor-harbor-database           Bound
    ingress/harbor-harbor                registry.tenant-a.lkdc.wso2.com
```

### Step 4: Wait for Pods

```go
podCtx, cancel := context.WithTimeout(ctx, 8*time.Minute)
defer cancel()

// Poll every 10 seconds until all pods are Ready
// Updates the progress column in DB as each component becomes ready
err = w.k8s.WaitForAllReady(podCtx, namespace, func(status map[string]bool) {
    // This callback runs every 10s with current pod readiness
    progress := map[string]string{}
    for comp, ready := range status {
        if ready { progress[comp] = "READY" } else { progress[comp] = "STARTING" }
    }
    w.store.UpdateProgress(ctx, tenantID, progress)
    // → UI polls GET /registry and shows live component status!
})
```

```
Progress column over time:
  t=0:   {"namespace":"READY","helm_install":"READY","postgresql":"STARTING",...}
  t=30s: {"postgresql":"READY","redis":"STARTING","core":"STARTING",...}
  t=2m:  {"postgresql":"READY","redis":"READY","core":"READY","registry":"STARTING",...}
  t=5m:  {"postgresql":"READY","redis":"READY","core":"READY","registry":"READY",...}
  t=7m:  all components READY
```

### Step 5: Harbor Bootstrap

```go
registryURL := fmt.Sprintf("https://registry.%s.%s", tenantID, baseDomain)
harborClient := harbor.NewClient(registryURL, adminPass)

// Wait up to 3 more minutes for Harbor's API to respond
bootstrapCtx, _ := context.WithTimeout(ctx, 3*time.Minute)
harborClient.WaitReady(bootstrapCtx)  // polls /api/v2.0/ping

// 3-step bootstrap:
robot, _ := harborClient.Bootstrap(ctx)
//  1. Configure: set auth_mode, disable self-registration, set scan schedule
//  2. CreateProject: create "library" project (private, auto-scan)
//  3. CreateRobotAccount: create "ci-default" robot → returns name + secret
```

### Step 6: Encrypt and Store Credentials

```go
// Encrypt both credentials before storing
encToken, tokenNonce, _ := w.cipher.EncryptString(robot.Secret)
encAdmin, adminNonce, _ := w.cipher.EncryptString(adminPass)

// Save to DB — plaintext NEVER written
w.store.SaveCredentials(ctx, &db.RegistryCredentials{
    TenantID:         tenantID,
    RobotUsername:    robot.Name,       // "robot$ci-default"
    EncryptedToken:   encToken,         // AES-256-GCM ciphertext
    TokenNonce:       tokenNonce,       // nonce for robot token
    AdminUsername:    "admin",
    EncryptedAdminPW: encAdmin,         // AES-256-GCM ciphertext
    AdminPWNonce:     adminNonce,       // nonce for admin password
})
```

### Step 7: Mark READY

```go
// Final step — set registry URL and flip status to READY
w.store.SetDeploymentReady(ctx, tenantID, registryURL)

w.metrics.ProvisionResult(tenantID, "success")  // emit Prometheus metric
w.logger.Info("harbor deployment complete",
    zap.String("tenant", tenantID),
    zap.String("url", registryURL),
)
```

---

## 6. Error Handling — The `fail` Helper

If any step fails, the deployment is marked FAILED and the error is stored:

```go
fail := func(step string, err error) {
    msg := fmt.Sprintf("step %s failed: %v", step, err)
    w.logger.Error("provision failed",
        zap.String("tenant", tenantID),
        zap.String("step", step),
        zap.Error(err),
    )
    w.store.UpdateDeploymentStatus(ctx, tenantID, db.StatusFailed, msg)
    w.metrics.ProvisionResult(tenantID, "failure")
}

// Example usage:
if err := w.k8s.EnsureNamespace(ctx, namespace, tenantID); err != nil {
    fail("namespace", err)
    return   // ← stop the function, don't continue
}
```

The user sees: `status: "FAILED"`, `errorMessage: "step helm_install failed: dial tcp: ..."`.

They can then retry via the API (creates a new PENDING row).

---

## 7. Progress Updates — Live UI Feedback

Every state change is written to the `progress` JSONB column. The UI polls `GET /registry` every 5 seconds and renders the current state:

```
User sees (animated progress indicator):
  ✅ Namespace created
  ✅ Helm install submitted
  🔄 PostgreSQL starting...
  🔄 Redis starting...
  🔄 Harbor Core starting...
  ⏳ Registry starting...
  ⏳ JobService starting...
  ⏳ Portal starting...
  ⏳ Bootstrap...
```

This is possible because `WaitForAllReady` has a **progress callback** — called every 10 seconds with fresh status — which writes to the DB, which the UI reads.

---

## 8. Timeout Hierarchy

```
Total deploy budget: ~15 minutes max

ctx (from main goroutine — cancelled only on SIGTERM)
  └── deploy goroutine
        └── podCtx (8 minute timeout)
              └── bootstrapCtx (3 minute timeout)
```

If pods don't come up in 8 minutes → `podCtx.Done()` fires → `WaitForAllReady` returns error → `fail("pods_ready", ...)` → status FAILED.

---

## 🏋️ Exercises

### Exercise 1 — Insert a PENDING Row and Watch the Worker
```bash
# Insert a fake PENDING row
docker exec -it implementation-of-auto-harbor-deployment-postgres-1 \
  psql -U provisioner -d registry_provisioner -c "
    INSERT INTO registry_deployments (tenant_id, namespace, status, plan)
    VALUES ('exercise-tenant', 'exercise-tenant-management', 'PENDING', 'starter');
  "

# Watch the provisioner logs
docker logs implementation-of-auto-harbor-deployment-provisioner-1 -f

# What happens? (It will fail at the K8s step since no cluster — that's fine)
# After 5 seconds, the worker should try to pick it up

# Check the status after:
docker exec -it implementation-of-auto-harbor-deployment-postgres-1 \
  psql -U provisioner -d registry_provisioner -c "
    SELECT tenant_id, status, error_message FROM registry_deployments;
  "
```

### Exercise 2 — Trace the Full deploy() Function
Open [backend/internal/worker/deploy_worker.go](../backend/internal/worker/deploy_worker.go).

Draw the call graph:
```
deploy()
  ├── k8s.EnsureNamespace()
  ├── k8s.ApplyNetworkPolicy()
  ├── helm.PlanFor()
  ├── crypto.GeneratePassword() × 2
  ├── helm.GenerateValues()
  ├── helm.Install()
  ├── k8s.WaitForAllReady()
  ├── harbor.NewClient()
  ├── harborClient.WaitReady()
  ├── harborClient.Bootstrap()
  │     ├── Configure()
  │     ├── CreateProject()
  │     └── CreateRobotAccount()
  ├── cipher.EncryptString() × 2
  ├── store.SaveCredentials()
  └── store.SetDeploymentReady()
```

### Exercise 3 — What Makes the Worker "Safe" to Run in 2 Replicas?
Answer these questions:
1. Two worker goroutines both call `processPending()` at the same time. Why won't they both process the same tenant?
2. The deploy worker crashes halfway through deploying tenant-a. What happens when it restarts? Does it pick up where it left off?
3. Why is the `fail` helper a function (closure) instead of just inline code?

### Exercise 4 — State Machine Violations
Which of these state transitions should be REJECTED?

| From | To | Allowed? |
|------|----|---------|
| PENDING → DEPLOYING | | ? |
| DEPLOYING → READY | | ? |
| READY → PENDING | | ? |
| FAILED → READY | | ? |
| READY → DELETING | | ? |
| DELETED → DEPLOYING | | ? |

### Exercise 5 — Design a Retry Mechanism
Currently, if deployment FAILS, the user must manually retry (POST /registry again). Design an automatic retry:
- How many retries before giving up?
- How long to wait between retries (exponential backoff)?
- What extra column would you add to the DB?
- Would you add the retry logic to the worker or a separate goroutine?

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Async deployment** | Return 202 immediately; worker deploys in background |
| **Job queue** | PostgreSQL table as queue; PENDING rows = jobs to process |
| **Worker loop** | Goroutine polling DB every 5s for PENDING rows |
| **State machine** | Finite states with allowed transitions (PENDING→DEPLOYING→READY) |
| **progress callback** | WaitForAllReady calls this every 10s → live UI updates |
| **fail helper** | Closure that marks FAILED + logs + emits metric on any error |
| **FOR UPDATE SKIP LOCKED** | Two workers can't claim the same job |
| **Timeout hierarchy** | nested contexts limit each phase of deployment |

**Next:** [11 — Prometheus & Observability →](./11-prometheus-and-observability.md)
