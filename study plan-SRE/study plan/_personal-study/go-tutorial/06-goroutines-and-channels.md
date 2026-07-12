# 06 — Goroutines and Channels

Goroutines let you do multiple things at the same time. Channels let goroutines communicate safely. Together they power the provisioner's concurrent architecture.

---

## The Problem: Serving Requests While Deploying

When a tenant requests a registry, Harbor takes 3-8 minutes to deploy. You can't block the HTTP server waiting — other tenants won't get a response.

Go's answer: run the deploy in a separate **goroutine** while the HTTP server keeps responding.

```
HTTP request arrives
       │
       ▼
Handler writes PENDING to DB
Handler returns 202 Accepted immediately   ← tenant gets response in <1ms
       │
Background goroutine (already running)
       │
       ▼
Worker polls DB every 5s
Picks up PENDING record
Runs 7-step deploy (takes minutes)
Updates DB to READY
       │
Tenant polls GET /registry until status = READY
```

---

## Goroutines — `go`

A goroutine is a function running concurrently. Start one with the `go` keyword:

```go
// Sequential — waits for each to finish
deploy("acme")     // waits ~5 min
deploy("beta")     // then waits ~5 min
// Total: ~10 min

// Concurrent — both run at the same time
go deploy("acme")  // returns immediately, deploy runs in background
go deploy("beta")  // returns immediately, deploy runs in background
// Total: ~5 min (both deploy in parallel)
```

Goroutines are **not OS threads**. Go manages thousands of goroutines on a small number of OS threads. They're cheap — you can have millions.

### Real example from `cmd/main.go`:

```go
// Goroutine 1: background deploy worker
go deployWorker.Run(context.Background())

// Goroutine 2: HTTP server
go func() {
    logger.Info("registry provisioner starting", zap.Int("port", cfg.Server.Port))
    if err := httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        logger.Fatal("server error", zap.Error(err))
    }
}()

// Main goroutine: block until shutdown signal
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit   // waits here
```

Three goroutines run simultaneously:
1. Worker loop — polls DB every 5s, deploys Harbor
2. HTTP server — handles API requests
3. Main — waits for SIGTERM

### Anonymous function goroutine

```go
// go func() { ... }() is common
// The () at the end immediately calls the anonymous function
go func() {
    // code here runs in a new goroutine
}()
```

---

## Channels — Goroutine Communication

A channel is a pipe. One goroutine sends into it; another receives from it. It's safe to use from multiple goroutines.

```go
// Create a channel
ch := make(chan string)

// Send (blocks until someone receives)
ch <- "hello"

// Receive (blocks until someone sends)
msg := <-ch

// Buffered channel (can hold N items without blocking)
ch := make(chan string, 5)
```

### The quit channel pattern (from `main.go`):

```go
quit := make(chan os.Signal, 1)         // buffered channel of size 1
signal.Notify(quit, syscall.SIGTERM)    // OS sends SIGTERM into this channel

// ... start goroutines ...

<-quit   // BLOCKS here — main goroutine waits until a signal arrives

// When Kubernetes sends SIGTERM (pod shutdown):
// signal.Notify puts the signal into the quit channel
// <-quit unblocks, main continues to graceful shutdown
```

### Simple channel example:

```go
func longTask(result chan string) {
    // simulate work
    time.Sleep(2 * time.Second)
    result <- "task done"
}

func main() {
    ch := make(chan string, 1)
    go longTask(ch)         // runs in background
    fmt.Println("waiting...")
    msg := <-ch             // blocks until longTask sends
    fmt.Println(msg)        // "task done"
}
```

---

## select — Wait on Multiple Channels

`select` is like `switch` but for channels. It waits for whichever channel is ready first:

```go
// From worker/deploy_worker.go
func (w *Worker) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():        // context cancelled → stop
            w.logger.Info("deploy worker stopped")
            return
        case <-ticker.C:          // ticker fires every 5s → do work
            w.processPending(ctx)
        }
    }
}
```

Every 5 seconds `ticker.C` sends a value → `processPending` runs.
If the context is cancelled → `ctx.Done()` fires → worker exits.

`select` picks whichever case fires first. If both fire at once, it picks one randomly.

---

## time.Ticker and time.Timer

```go
// Ticker — fires repeatedly
ticker := time.NewTicker(5 * time.Second)
defer ticker.Stop()   // always stop it to avoid goroutine leak

for {
    <-ticker.C   // blocks until next tick
    doWork()
}

// Timer — fires once after a delay
timer := time.NewTimer(30 * time.Second)
<-timer.C   // blocks for 30 seconds
doSomething()

// Simpler one-shot wait
time.Sleep(5 * time.Second)   // blocks the goroutine for 5s
```

---

## sync.WaitGroup — Wait for Multiple Goroutines

When you need to launch several goroutines and wait for ALL of them:

```go
import "sync"

var wg sync.WaitGroup

for _, tenant := range tenants {
    wg.Add(1)              // tell WaitGroup we're adding one goroutine
    go func(t string) {
        defer wg.Done()    // tell WaitGroup this goroutine finished
        deploy(t)
    }(tenant)
}

wg.Wait()   // blocks until all goroutines call Done()
fmt.Println("all tenants deployed")
```

---

## sync.Mutex — Protect Shared Data

If two goroutines read/write the same data without coordination, you get a **race condition** (unpredictable, often corrupted data).

```go
import "sync"

type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()         // acquire lock — only one goroutine can proceed
    defer c.mu.Unlock() // release lock when done
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

In the provisioner, this problem is solved differently: the deploy state lives in **PostgreSQL** with `FOR UPDATE SKIP LOCKED` — the DB is the single source of truth, so goroutines don't need a mutex for coordination.

---

## Goroutine Leak — The Danger

A goroutine that never exits is a **leak**. Memory grows forever. Common causes:

```go
// LEAK: channel nobody reads from
ch := make(chan string)
go func() {
    ch <- "result"   // blocks forever if nobody receives
}()
// goroutine stuck forever

// FIX: use buffered channel or always have a reader
ch := make(chan string, 1)   // can send without a receiver

// LEAK: goroutine ignoring context
go func() {
    for {
        doWork()          // never checks ctx.Done()
        time.Sleep(5*time.Second)
    }
}()

// FIX: always check context
go func() {
    for {
        select {
        case <-ctx.Done():
            return   // clean exit
        case <-time.After(5 * time.Second):
            doWork()
        }
    }
}()
```

The worker in `deploy_worker.go` correctly uses `ctx.Done()` to stop cleanly.

---

## Exercises

### Exercise 1 — Simple Goroutine

Run two functions concurrently and wait for both:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func deploy(tenant string, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("deploying %s...\n", tenant)
    time.Sleep(1 * time.Second)   // simulate work
    fmt.Printf("%s deployed!\n", tenant)
}

func main() {
    var wg sync.WaitGroup
    tenants := []string{"acme", "beta", "gamma"}

    for _, t := range tenants {
        wg.Add(1)
        go deploy(t, &wg)
    }

    wg.Wait()
    fmt.Println("all done")
}
```

Run it. What order do the deploys complete? Run it 3 times — does the order change?

---

### Exercise 2 — Worker Loop

Write a worker that:
1. Polls every 2 seconds
2. Stops when it has done 5 iterations
3. Uses a channel to signal completion

```go
package main

import (
    "fmt"
    "time"
)

func worker(done chan struct{}) {
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()

    count := 0
    for {
        select {
        case <-done:
            fmt.Println("worker stopped")
            return
        case <-ticker.C:
            count++
            fmt.Printf("poll #%d\n", count)
            if count >= 5 {
                fmt.Println("worker finished")
                return
            }
        }
    }
}

func main() {
    done := make(chan struct{})
    go worker(done)
    time.Sleep(15 * time.Second)
    close(done)
    time.Sleep(1 * time.Second)
}
```

---

### Exercise 3 — Race Condition Demo

Run this program with `go run -race exercise3.go`:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    count := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            count++   // ← RACE: multiple goroutines writing count simultaneously
        }()
    }

    wg.Wait()
    fmt.Println("count:", count)   // will NOT be 1000 — data race!
}
```

Then fix it by adding a `sync.Mutex`. Run it again — now it should reliably print 1000.

---

### Exercise 4 — Read the Real Worker

Open [worker/deploy_worker.go](../../backend/internal/worker/deploy_worker.go).

1. In `Run()`, what two channels does `select` wait on?
2. In `processPending()`, why does it call `go w.deploy(...)` instead of `w.deploy(...)`?
3. What happens if `w.deploy` panics? (hint: nothing protects it — what would you add?)
4. The ticker fires every 5 seconds. Could two deploys for the same tenant overlap? How does the DB prevent this?

---

## Key Takeaways

| Concept | Syntax / Pattern |
|---------|-----------------|
| Start goroutine | `go funcName(args)` |
| Anonymous goroutine | `go func() { ... }()` |
| Create channel | `ch := make(chan Type)` |
| Buffered channel | `ch := make(chan Type, N)` |
| Send | `ch <- value` |
| Receive | `value := <-ch` |
| Select | `select { case <-ch1: ... case <-ch2: ... }` |
| Ticker | `ticker := time.NewTicker(5*time.Second)` |
| Wait for group | `var wg sync.WaitGroup; wg.Add(1); go...; wg.Wait()` |
| Mutex | `mu.Lock(); defer mu.Unlock()` |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md).

### Recall questions (no peeking)
1. What does `go f()` do, and why is it used for Harbor deploys?
2. Difference between `make(chan T)` and `make(chan T, N)`?
3. What does `<-quit` do in `main.go`, and when does it unblock?
4. What is `select` for, and what happens if two cases are ready at once?
5. What is a goroutine leak, and how does the worker avoid one?

### Make these Anki cards (front → back)
- `go f()` → run `f` concurrently; returns immediately
- Buffered channel `make(chan T, N)` → can hold N items without a receiver
- Send / receive → `ch <- v` / `v := <-ch` (both **block** on an unbuffered channel)
- `select` → switch for channels; takes whichever is ready first (random if tie)
- Goroutine-leak fix → always check `ctx.Done()` in loops; worker coordination is **PostgreSQL `FOR UPDATE SKIP LOCKED`**, not a mutex

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 3 (race condition: run with `-race`, then fix with a mutex)**.
- [ ] **Day 7 (Friday redo):** answer **Exercise 4** on the real worker `Run()`; explain the leak-fix pattern aloud.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [07 — Context →](./07-context.md)
