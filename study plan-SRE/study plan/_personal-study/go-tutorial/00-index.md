# Go Tutorial — Registry Provisioner

This folder teaches Go from zero, using the real code inside `backend/` as the learning target.
By the end you will be able to read, understand, and modify every line of the provisioner.

---

## Reading Order

| File | What You Learn | Real Code You'll Understand After |
|------|---------------|----------------------------------|
| [01-first-steps.md](./01-first-steps.md) | Variables, types, functions, hello world | `config/config.go` — `envStr`, `envInt` |
| [02-functions-and-errors.md](./02-functions-and-errors.md) | Multiple returns, `error`, `if err != nil` | Every file — this pattern is everywhere |
| [03-structs-and-methods.md](./03-structs-and-methods.md) | Structs, pointer receivers, `New()` pattern | `db/postgres.go` — `Store`, `New()` |
| [04-slices-and-maps.md](./04-slices-and-maps.md) | Slices, maps, `for range` | `worker/deploy_worker.go` — progress map |
| [05-interfaces.md](./05-interfaces.md) | Interfaces, implicit satisfaction | `harbor/bootstrap.go` — `Client` |
| [06-goroutines-and-channels.md](./06-goroutines-and-channels.md) | `go`, channels, `select` | `cmd/main.go` — `go deployWorker.Run(...)` |
| [07-context.md](./07-context.md) | `context.Context`, timeouts, cancellation | `worker/deploy_worker.go` — `podCtx` |
| [08-packages-and-modules.md](./08-packages-and-modules.md) | `package`, `import`, `go.mod`, `internal/` | Entire `backend/` folder structure |
| [09-stdlib-essentials.md](./09-stdlib-essentials.md) | `os`, `fmt`, `strings`, `json`, `time`, `sql` | `db/postgres.go`, `crypto/aes.go` |
| [10-gin-framework.md](./10-gin-framework.md) | Routing, handlers, middleware, JSON binding | `api/server.go`, `api/handlers/registry.go` |
| [11-dependency-injection.md](./11-dependency-injection.md) | Constructor pattern, wiring, testing | `cmd/main.go` — all the `New(...)` calls |
| [12-codebase-tour.md](./12-codebase-tour.md) | Full walkthrough of every file | The whole provisioner |

---

## The Big Picture — What You Are Building

```
HTTP Request (POST /api/v1/tenants/acme/registry)
        │
        ▼
   cmd/main.go              ← Entry point. Wires everything together.
        │
        ├── api/server.go   ← Gin HTTP server. Routes requests to handlers.
        │       │
        │       └── api/handlers/registry.go  ← Reads request, writes DB, replies JSON.
        │
        ├── worker/deploy_worker.go  ← Goroutine. Polls DB every 5s. Runs 7-step deploy.
        │       │
        │       ├── helm/deployer.go          ← Installs Harbor chart on Harvester via Helm SDK.
        │       ├── k8s/client.go             ← Creates namespaces, waits for pods (client-go).
        │       └── harbor/bootstrap.go       ← Calls Harbor REST API to configure it.
        │
        ├── db/postgres.go           ← All SQL queries. Reads/writes registry_deployments table.
        ├── crypto/aes.go            ← AES-256-GCM encrypt/decrypt for robot tokens.
        ├── config/config.go         ← Reads env vars into typed Config struct.
        ├── audit/logger.go          ← Writes audit_log rows (non-blocking).
        └── metrics/prometheus.go    ← Prometheus counters and histograms.
```

---

## How to Run Go Exercises

Install Go (if not already):
```bash
sudo apt install golang-go   # Ubuntu/Debian
go version                   # should print go1.22+
```

Run any exercise:
```bash
# Create a file
mkdir -p ~/go-exercises && cd ~/go-exercises
nano 01-hello.go

# Run it
go run 01-hello.go
```

You do NOT need to compile first. `go run` compiles and runs in one step.

---

## 🧠 Retention & Weekly Review (read this — it's how you don't forget)

Every chapter now ends with a **🧠 Retention** block: recall questions, Anki cards to create, and a per-chapter spaced-repetition schedule. This is wired to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). The rule: **passive re-reading doesn't work — you forget it in ~a week.** Active recall + spaced repetition is the fix.

### The loop for every chapter
1. Read the chapter + do its 4 exercises.
2. Create that chapter's **Anki cards** (from the Retention block).
3. Do your Anki reviews **daily** (~10 min) — the app resurfaces each card right before you'd forget it.
4. **Every Friday**, redo one earlier chapter's "read the real code" exercise **with the doc closed** (active recall on the real `backend/`).

### Master weekly-review schedule (spaced so older chapters resurface)
| Week | New chapters | Friday "redo from memory" |
|------|-------------|---------------------------|
| 1 | 01 → 04 | Ch 01 Exercise 3 (env reader) |
| 2 | 05 → 08 | Ch 02 Ex 2 (error chain) + Ch 05 Ex 2 (implement `error`) |
| 3 | 09 → 11 | Ch 06 Ex 3 (race + mutex) + Ch 07 Ex 1 (timeout) |
| 4 | 12 (the tour) | Ch 10 Ex 3 (middleware) + Ch 11 Ex 3 (DI tree) |
| 5+ | — (maintenance) | **Ch 12 Final Exercise from memory**, weekly, until all 12 Qs answerable cold |

> When you can do the **Ch 12 Final Exercise** (trace a full request, file+line for all 12 questions) from memory, the Go tutorial has stuck — and you're ready for the **Source-Code Track** (reading Harbor/Rancher controllers is the same skill scaled up).
