# 11 — Dependency Injection

Dependency Injection (DI) means: instead of creating your dependencies inside a function, you receive them as parameters. This is the backbone of how the provisioner is structured.

---

## The Problem

Without DI, components create their own dependencies:

```go
// BAD — Worker creates its own Store
type Worker struct{}

func (w *Worker) deploy(tenantID string) {
    // Worker hardcodes how to get a DB connection
    db, _ := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    db.ExecContext(...)
}
```

Problems:
- Can't test Worker without a real database
- Can't swap the database for a different one
- Can't configure it from outside
- Hidden dependency — not obvious what Worker needs

---

## DI: Receive Dependencies, Don't Create Them

```go
// GOOD — Worker receives a Store
type Worker struct {
    store *db.Store   // ← injected from outside
    helm  *helm.Deployer
    k8s   *k8s.Client
    // ...
}

func New(store *db.Store, helm *helm.Deployer, k8s *k8s.Client, ...) *Worker {
    return &Worker{store: store, helm: helm, k8s: k8s}
}

// Now you can test with a fake store:
fakeStore := &FakeStore{...}
worker := worker.New(fakeStore, fakeHelm, fakeK8s, ...)
```

---

## The Constructor Pattern in the Provisioner

`cmd/main.go` is entirely about wiring dependencies together. Read it top to bottom:

```go
func main() {
    // 1. Foundation — things with no dependencies
    logger, _ := zap.NewProduction()
    cfg, _ := config.Load()

    // 2. Storage layer — needs: config + logger
    store, _ := db.New(cfg.DB, logger)
    defer store.Close()

    // 3. Security — needs: master key file
    masterKey, _ := os.ReadFile(cfg.Encryption.MasterKeyPath)
    cipher, _ := crypto.NewCipher(masterKey)

    // 4. Infrastructure clients — need: config
    harvesterClient, _ := k8s.NewHarvesterClient(cfg.Kubernetes.HarvesterKubeconfigPath)
    helmDeployer, _ := helm.NewDeployer(cfg.Helm, cfg.Kubernetes.HarvesterKubeconfigPath, logger)

    // 5. Cross-cutting concerns — need: store + logger
    auditLog := audit.NewLogger(store, logger)
    reg := metrics.NewRegistry()

    // 6. Application services — need: everything above
    deployWorker := worker.New(store, helmDeployer, harvesterClient, cipher,
                               cfg.Helm, auditLog, reg, logger)

    // 7. HTTP layer — needs: store + cipher + auditLog + reg + logger
    srv := api.NewServer(cfg, store, cipher, auditLog, reg, logger)

    // 8. Runtime
    go deployWorker.Run(context.Background())
    httpServer := &http.Server{Handler: srv.Router()}
    httpServer.ListenAndServe()
}
```

Each layer only knows about the layer below it. Nothing creates its own dependencies.

---

## Constructor Signatures Show Dependencies

Looking at a constructor tells you exactly what a component needs:

```go
// worker.New — needs 8 things
func New(
    store    *db.Store,          // ← database
    helm     *helm.Deployer,     // ← Helm on Harvester
    k8s      *k8s.Client,        // ← Kubernetes on Harvester
    cipher   *crypto.Cipher,     // ← encryption
    helmCfg  config.HelmConfig,  // ← configuration values
    auditLog *audit.Logger,      // ← audit trail
    metrics  *metrics.Registry,  // ← prometheus
    logger   *zap.Logger,        // ← logging
) *Worker {
    return &Worker{...}
}
```

If you want to understand what `Worker` can do, this constructor is your map.

---

## Passing Config as Struct

Rather than passing 10 individual env-var strings, config is read once and passed as a typed struct:

```go
// config.go reads all env vars once
cfg, err := config.Load()

// Now pass it to whoever needs it
helmDeployer, _ := helm.NewDeployer(cfg.Helm, ...)
//                                   ^^^^^^^^
//                               HelmConfig struct — not individual strings

// Inside helm.NewDeployer:
func NewDeployer(cfg config.HelmConfig, ...) (*Deployer, error) {
    // cfg.HarborChartVer, cfg.BaseDomain, cfg.StorageClass — all typed
}
```

Benefits:
- Config validated once at startup (not scattered across every component)
- Typed fields — IDE autocomplete, compile-time checking
- Easy to pass subsets: `cfg.Helm` just for Helm-related config

---

## Functional Options Pattern (advanced, optional)

Sometimes constructors need many optional parameters. The Options pattern is common in Go libraries:

```go
type Server struct {
    port    int
    timeout time.Duration
}

// Option is a function that modifies Server
type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}

// Constructor accepts zero or more options
func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080, timeout: 30 * time.Second}  // defaults
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Caller specifies only what they want to change
s := NewServer(
    WithPort(9090),
    WithTimeout(60 * time.Second),
)
```

The provisioner doesn't use this pattern but you'll see it in `zap.NewProduction()`, `http.Client{}`, and third-party libraries.

---

## Testing with DI

DI makes testing easy — you inject fake dependencies:

```go
// Define what you need as an interface (not a concrete type)
type Deployer interface {
    Install(ctx context.Context, tenantID, ns string, values []byte) error
}

// Fake for tests
type MockDeployer struct {
    called   bool
    failWith error
}

func (m *MockDeployer) Install(ctx context.Context, tenantID, ns string, values []byte) error {
    m.called = true
    return m.failWith   // nil = success, or return an error to simulate failure
}

// Test the worker without real Helm/K8s
func TestWorkerDeployHelmFailure(t *testing.T) {
    fakeStore := &FakeStore{}
    fakeHelm := &MockDeployer{failWith: fmt.Errorf("chart not found")}
    fakeK8s := &MockK8s{}

    w := worker.New(fakeStore, fakeHelm, fakeK8s, ...)
    w.deploy(ctx, &db.RegistryDeployment{TenantID: "acme"})

    // Assert store was updated to FAILED
    if fakeStore.status != db.StatusFailed {
        t.Errorf("expected FAILED, got %s", fakeStore.status)
    }
}
```

Without DI (if Worker created its own Helm client), you couldn't test the failure path without a real Kubernetes cluster.

---

## Closures as Dependencies

Sometimes a dependency is just a function, not a full struct. The deploy worker uses this for the progress callback:

```go
// WaitForAllReady accepts a callback function as dependency
func (c *Client) WaitForAllReady(
    ctx context.Context,
    namespace string,
    onProgress func(status map[string]bool),   // ← function dependency
) error {
    // ... poll pods ...
    onProgress(currentStatus)   // call the injected callback
}

// Caller injects the behaviour they want
w.k8s.WaitForAllReady(podCtx, namespace, func(status map[string]bool) {
    // This function is injected — WaitForAllReady doesn't know what it does
    progress := make(map[string]string)
    for comp, ready := range status {
        if ready { progress[comp] = "READY" } else { progress[comp] = "STARTING" }
    }
    w.store.UpdateProgress(ctx, tenantID, progress)
})
```

---

## Exercises

### Exercise 1 — Wire a Mini App

Build a small app with DI:

```
Database (fake, in-memory)
    └── UserStore (needs Database)
            └── UserAPI (needs UserStore)
```

```go
package main

import "fmt"

// "Database" — fake in-memory
type Database struct {
    data map[string]string
}

func NewDatabase() *Database {
    return &Database{data: map[string]string{}}
}

func (d *Database) Set(key, value string) { d.data[key] = value }
func (d *Database) Get(key string) (string, bool) {
    v, ok := d.data[key]
    return v, ok
}

// UserStore — depends on Database
type UserStore struct {
    db *Database
}

func NewUserStore(db *Database) *UserStore {
    return &UserStore{db: db}
}

func (s *UserStore) Save(id, name string) {
    s.db.Set(id, name)
}

func (s *UserStore) Find(id string) (string, bool) {
    return s.db.Get(id)
}

// UserAPI — depends on UserStore
type UserAPI struct {
    store *UserStore
}

func NewUserAPI(store *UserStore) *UserAPI {
    return &UserAPI{store: store}
}

func (a *UserAPI) CreateUser(id, name string) {
    a.store.Save(id, name)
    fmt.Printf("created user %s\n", id)
}

func (a *UserAPI) GetUser(id string) {
    name, ok := a.store.Find(id)
    if !ok {
        fmt.Printf("user %s not found\n", id)
        return
    }
    fmt.Printf("found user %s: %s\n", id, name)
}

// Wire everything together
func main() {
    db := NewDatabase()
    store := NewUserStore(db)
    api := NewUserAPI(store)

    api.CreateUser("u-001", "Alice")
    api.GetUser("u-001")
    api.GetUser("u-999")
}
```

---

### Exercise 2 — Swap a Dependency

Modify the app above:
1. Define a `Storage` interface with `Set(key, value string)` and `Get(key string) (string, bool)`
2. Make `UserStore` accept `Storage` instead of `*Database`
3. Write `RedisStorage` (fake — just print "would call Redis: Set key=X val=Y")
4. Wire with `RedisStorage` instead of `Database` — the `UserAPI` should not need to change

---

### Exercise 3 — Map main.go

Read [cmd/main.go](../../backend/cmd/main.go) and draw a dependency tree:

```
main()
 ├── logger (no deps)
 ├── cfg (no deps)
 ├── store (needs: cfg.DB, logger)
 ├── cipher (needs: masterKey file)
 ├── harvesterClient (needs: cfg.Kubernetes.HarvesterKubeconfigPath)
 ├── helmDeployer (needs: ...)
 ├── reg (needs: ...)
 ├── auditLog (needs: ...)
 ├── deployWorker (needs: ...)
 └── srv (needs: ...)
```

Fill in the `...` for each component.

---

## Key Takeaways

| Principle | In Code |
|-----------|---------|
| Receive don't create | Accept `*db.Store` as param, don't `sql.Open` inside |
| Constructor = dependency list | `func New(a, b, c) *Component` tells you what it needs |
| Config struct | Read once in `config.Load()`, pass as typed struct |
| Closures as deps | Pass `func(...)` when you need injectable behaviour |
| Testing | Inject fakes/mocks instead of real dependencies |
| `internal/` enforces layers | Prevents circular dependencies |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md).

### Recall questions (no peeking)
1. DI in one sentence — receive vs. create?
2. Why does DI make a component testable?
3. What does a constructor's parameter list tell you about a component?
4. Why pass `cfg.Helm` (a struct) instead of 10 individual strings?
5. What is a "closure as a dependency" — give the worker's progress-callback example.

### Make these Anki cards (front → back)
- DI → **receive** dependencies as parameters; don't create them inside
- Constructor `func New(a, b, c) *T` → its parameter list **is** the dependency list
- DI + interfaces → swap a real dependency for a mock/fake in tests
- `cmd/main.go` → wires everything **bottom-up** (logger → config → store → … → server)
- Closure dependency → pass a `func(...)` to inject behavior (e.g. `WaitForAllReady`'s progress callback)

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 2 (swap a dependency behind an interface)** from memory.
- [ ] **Day 7 (Friday redo):** redo **Exercise 3 (draw the `main.go` dependency tree)** from memory.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [12 — Full Codebase Tour →](./12-codebase-tour.md)
