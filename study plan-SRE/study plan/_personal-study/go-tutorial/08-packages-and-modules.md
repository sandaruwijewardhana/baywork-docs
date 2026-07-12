# 08 — Packages and Modules

Go organises code into **packages** (directories) and **modules** (repositories). Understanding this is essential for reading import paths and knowing where code lives.

---

## Package — One Directory = One Package

Every `.go` file starts with `package <name>`. All files in the same directory must have the same package name:

```
backend/internal/db/
├── postgres.go     → package db
└── helpers.go      → package db   ← same package, different file
```

You use a package by importing its path and calling its exported names:

```go
// In worker/deploy_worker.go
import "github.com/wso2/lkdc/registry-provisioner/internal/db"

// Then use exported names from that package
store, err := db.New(cfg.DB, logger)   // db.New — from db package
deployment := &db.RegistryDeployment{} // db.RegistryDeployment — exported type
```

---

## Exported vs Unexported

In Go, **capitalisation** controls visibility:

```go
// Exported — accessible from other packages (starts with uppercase)
type Store struct { ... }
func New(...) (*Store, error) { ... }
const StatusReady RegistryStatus = "READY"

// Unexported — private to this package (starts with lowercase)
type progressJSON []byte        // only usable inside package db
func scanDeployment(...) { }    // only callable inside package db
```

This replaces `public`/`private` keywords from other languages:
- Uppercase = public
- Lowercase = private

From `db/postgres.go`:
```go
type Store struct {
    db     *sql.DB      // lowercase — unexported field, can't be accessed from outside
    logger *zap.Logger  // lowercase — same
}

func (s *Store) GetDeployment(...) { }   // uppercase — exported method
func scanDeployment(...) { }             // lowercase — unexported helper
```

---

## Import Paths

Import paths look like URLs but are just directory paths relative to the module root:

```go
import (
    // Standard library (no URL prefix)
    "context"
    "fmt"
    "os"
    "time"

    // Third-party packages (from go.mod)
    "github.com/gin-gonic/gin"
    "go.uber.org/zap"

    // Your own packages (module name + path)
    "github.com/wso2/lkdc/registry-provisioner/internal/db"
    "github.com/wso2/lkdc/registry-provisioner/internal/worker"
)
```

The last segment of the path is the package name used in code:
- `internal/db` → use as `db.Something`
- `internal/worker` → use as `worker.Something`
- `gin-gonic/gin` → use as `gin.Something`

**Alias an import** if the name conflicts:
```go
import (
    helmaction "helm.sh/helm/v3/pkg/action"   // alias to avoid conflict
)

helmaction.NewInstall(...)
```

---

## The `internal/` Directory

Any package inside `internal/` can ONLY be imported by code within the same module. Code outside your module gets a compile error if it tries to import your internals.

```
backend/
├── cmd/main.go                         ← CAN import internal/*
├── internal/
│   ├── api/server.go                   ← CAN import internal/db, internal/worker
│   ├── db/postgres.go                  ← CAN import internal/config
│   └── worker/deploy_worker.go         ← CAN import internal/helm, internal/k8s
└── go.mod
```

If someone tries to import `github.com/wso2/lkdc/registry-provisioner/internal/db` from outside this module → compile error.

This is the Go way to hide implementation details.

---

## go.mod — The Module File

`go.mod` declares:
1. The module name (import path prefix)
2. The Go version
3. All external dependencies

```
module github.com/wso2/lkdc/registry-provisioner

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    go.uber.org/zap v1.27.0
    helm.sh/helm/v3 v3.14.0
    k8s.io/client-go v0.29.0
    github.com/lib/pq v1.10.9
    ...
)
```

The module name (`github.com/wso2/lkdc/registry-provisioner`) is the prefix for ALL your internal import paths.

---

## go.sum — The Lockfile

`go.sum` records the exact checksum of every dependency. Never edit this manually. Go manages it automatically.

---

## Common `go` Commands

```bash
# Download all dependencies listed in go.mod
go mod download

# Clean up go.mod (remove unused, add missing)
go mod tidy

# Run a file directly
go run cmd/main.go

# Build a binary
go build -o registry-provisioner ./cmd/main.go

# Run all tests
go test ./...

# Run tests in one package
go test ./internal/crypto/...

# Check for data races
go test -race ./...

# Show all packages in this module
go list ./...
```

---

## The `package main` Special Case

`package main` is the entry point package. The `main()` function in a `main` package is where the program starts.

```
cmd/
└── main.go    ← package main, func main() — this is the binary entry point
```

You can only have one `main()` function per binary. All other packages are libraries.

---

## init() — Package Initialisation

If a package has an `init()` function, Go calls it automatically before `main()`:

```go
// From cmd/main.go
func init() {
    os.MkdirAll("/tmp/helm-values", 0700)
    os.MkdirAll("/tmp/helm-charts", 0700)

    if os.Getenv("ENV") == "production" {
        os.Setenv("GIN_MODE", "release")
    }
}
```

This runs once, before `main()`, automatically. Use it for one-time setup.

---

## Blank Import `_ "package/path"`

Sometimes you import a package only for its `init()` side effect, not to use its exported names:

```go
import _ "github.com/lib/pq"   // from db/postgres.go
```

The `pq` package's `init()` registers the PostgreSQL driver with Go's `database/sql`. You never call anything from `pq` directly — just importing it is enough.

---

## Package Naming Conventions

| Rule | Example |
|------|---------|
| Short, lowercase | `db`, `api`, `helm`, `k8s` |
| No underscores | `deployworker` not `deploy_worker` |
| Match directory name | directory `db/` → `package db` |
| Don't repeat module | `db.Store` not `db.DBStore` |
| `_test.go` suffix | test files, excluded from normal build |

---

## Exercises

### Exercise 1 — Create a Package

Create this structure:

```
myapp/
├── go.mod
├── main.go
└── tenant/
    └── tenant.go
```

In `tenant/tenant.go`:
```go
package tenant

type Tenant struct {
    ID   string
    Plan string
}

func New(id, plan string) *Tenant {
    return &Tenant{ID: id, Plan: plan}
}

func (t *Tenant) Summary() string {
    return t.ID + " (" + t.Plan + ")"
}
```

In `main.go`:
```go
package main

import (
    "fmt"
    "myapp/tenant"   // use your module name from go.mod
)

func main() {
    t := tenant.New("acme", "starter")
    fmt.Println(t.Summary())
}
```

Create `go.mod`:
```bash
cd myapp
go mod init myapp
go run main.go
```

---

### Exercise 2 — Exported vs Unexported

Add these to `tenant/tenant.go`:

```go
// Add one exported and one unexported helper

func (t *Tenant) IsReady() bool {      // exported
    return t.validate()                 // calls unexported
}

func (t *Tenant) validate() bool {     // unexported
    return t.ID != "" && t.Plan != ""
}
```

Try calling `t.validate()` from `main.go`. What error do you get?

---

### Exercise 3 — Read go.mod

Open [backend/go.mod](../../backend/go.mod).

1. What is the full module name?
2. List 5 external dependencies and what each one is for
3. What Go version is required?
4. What does the blank import `_ "github.com/lib/pq"` in `db/postgres.go` do?

---

### Exercise 4 — Package Map

Draw a dependency diagram for the provisioner packages:
- Which packages does `worker` import?
- Which packages does `api/handlers` import?
- Which package has no imports of other internal packages (the "leaf")?
- Why can `internal/db` NOT import `internal/worker`?

Hint: Read the import blocks at the top of each file.

---

## Key Takeaways

| Concept | Rule |
|---------|------|
| Package = directory | One directory = one package |
| Exported | Starts with uppercase |
| Unexported | Starts with lowercase |
| Module name | In `go.mod`, prefix for all import paths |
| internal/ | Can only be imported within the same module |
| `go mod tidy` | Cleans up dependencies |
| `go run` | Compile + run in one step |
| `package main` | Entry point — must have `func main()` |
| Blank import | `_ "pkg"` — imports for side effect only |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). This chapter is your map for navigating *any* Go repo — directly feeds the Source-Code Track.

### Recall questions (no peeking)
1. What single thing controls whether a name is exported?
2. What is special about the `internal/` directory?
3. What three things does `go.mod` declare?
4. What does a blank import `_ "github.com/lib/pq"` actually do?
5. When does an `init()` function run?

### Make these Anki cards (front → back)
- Exported → **Uppercase** first letter; unexported → lowercase
- `internal/` → importable **only within the same module**
- Blank import `_ "pkg"` → runs that package's `init()` side-effect only (e.g. registers the `pq` SQL driver)
- `go.mod` → module name + Go version + dependency list
- `init()` → runs once, automatically, **before** `main()`

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 1 (create a multi-package app with `go mod init`)** from memory.
- [ ] **Day 7 (Friday redo):** do **Exercise 4 (draw the provisioner package dependency map)** from memory; explain why `internal/db` can't import `internal/worker`.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [09 — Standard Library Essentials →](./09-stdlib-essentials.md)
