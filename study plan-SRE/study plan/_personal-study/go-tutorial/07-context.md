# 07 — Context

`context.Context` is Go's standard way to pass **cancellation signals** and **deadlines** through a chain of function calls.

Almost every function in the provisioner accepts `ctx context.Context` as its first parameter. If you see `ctx`, this is what it is.

---

## The Problem Context Solves

When a deploy takes 8 minutes, what happens if:
- The pod gets a shutdown signal halfway through?
- The tenant sends a DELETE request while deploy is running?
- The network call to Harbor hangs forever?

Without context, nothing — the goroutine runs forever or hangs. With context, you can cancel work cleanly from the outside.

---

## context.Background() and context.TODO()

These are the starting points — empty contexts with no deadline and no cancellation:

```go
ctx := context.Background()   // use in main(), tests, and when starting goroutines
ctx := context.TODO()         // placeholder when you haven't decided what to do yet
```

From `cmd/main.go`:
```go
go deployWorker.Run(context.Background())   // worker runs with no deadline
```

---

## context.WithTimeout — Auto-Cancellation After Duration

Create a context that automatically cancels after a time limit:

```go
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()   // always call cancel to free resources — even if timeout fires first

// Now do work that must complete within 5 seconds
err := db.PingContext(ctx)
// If 5 seconds pass: ctx is cancelled, PingContext returns an error
```

From `db/postgres.go`:
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
if err := db.PingContext(ctx); err != nil {
    return nil, fmt.Errorf("ping db: %w", err)
}
```

From `worker/deploy_worker.go`:
```go
// Give pods up to 8 minutes to become ready
podCtx, cancel := context.WithTimeout(ctx, 8*time.Minute)
defer cancel()
err = w.k8s.WaitForAllReady(podCtx, namespace, progressCb)

// Give Harbor API up to 3 minutes to respond after pods are ready
bootstrapCtx, cancel2 := context.WithTimeout(ctx, 3*time.Minute)
defer cancel2()
if err := harborClient.WaitReady(bootstrapCtx); err != nil { ... }
```

---

## context.WithCancel — Manual Cancellation

Create a context you can cancel yourself:

```go
ctx, cancel := context.WithCancel(parentCtx)

go func() {
    doWork(ctx)   // will stop when cancel() is called
}()

// Later, from another goroutine or signal handler:
cancel()   // sends cancellation signal to all goroutines using this ctx
```

From `cmd/main.go` — graceful shutdown:
```go
ctx, cancel := context.WithTimeout(context.Background(), cfg.Server.ShutdownTimeout)
defer cancel()
if err := httpServer.Shutdown(ctx); err != nil {
    logger.Error("server forced shutdown", zap.Error(err))
}
```

---

## ctx.Done() — The Cancellation Signal

`ctx.Done()` returns a channel that gets closed when the context is cancelled:

```go
func (w *Worker) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():      // ← this fires when context is cancelled
            w.logger.Info("deploy worker stopped")
            return              // clean exit
        case <-ticker.C:
            w.processPending(ctx)
        }
    }
}
```

When `deployWorker.Run(context.Background())` is called with `context.Background()`, the worker runs forever (Background context never cancels). In a production system, you'd pass a cancellable context and cancel it on shutdown.

---

## ctx.Err() — Why Was It Cancelled?

```go
if ctx.Err() == context.DeadlineExceeded {
    // timeout fired
}
if ctx.Err() == context.Canceled {
    // someone called cancel()
}
```

---

## Context Hierarchy — Parent and Child

Contexts form a tree. Cancelling a parent automatically cancels all children:

```
context.Background()                  ← root (never cancels)
  │
  └── deployCtx (no deadline)
        │
        ├── podCtx (8 min deadline)   ← WaitForAllReady
        │
        └── bootstrapCtx (3 min)      ← harborClient.WaitReady
```

If the pod crashes and the deploy goroutine's context is cancelled, both `podCtx` and `bootstrapCtx` are cancelled automatically.

```go
// Parent
deployCtx, deployCancel := context.WithCancel(context.Background())

// Children inherit from parent
podCtx, podCancel := context.WithTimeout(deployCtx, 8*time.Minute)
defer podCancel()

// If deployCancel() is called, podCtx is also cancelled
// If podCtx times out, only podCtx is cancelled (parent is not affected)
```

---

## Passing Context Through Functions

Every function that does I/O should accept and pass context:

```go
// HTTP handler — context comes from the request
func (h *Handler) Create(c *gin.Context) {
    ctx := c.Request.Context()   // already set up by Gin

    deployment, err := h.store.GetDeployment(ctx, tenantID)
    //                                        ^^^
}

// Store method — passes ctx to the DB driver
func (s *Store) GetDeployment(ctx context.Context, tenantID string) (*RegistryDeployment, error) {
    row := s.db.QueryRowContext(ctx, "SELECT ...")
    //                          ^^^
}
```

If the HTTP client disconnects, `c.Request.Context()` is cancelled → `QueryRowContext` returns early → no wasted DB work.

---

## context.WithValue — Passing Data Through Context

You can attach values to a context and read them later:

```go
type contextKey string

const claimsKey contextKey = "claims"

// Set value
ctx = context.WithValue(ctx, claimsKey, claims)

// Read value (in middleware or handler)
claims, ok := ctx.Value(claimsKey).(*Claims)
```

From `middleware/auth.go` — JWT claims are stored in context and read in handlers:
```go
// middleware sets claims
c.Set("claims", parsedClaims)   // Gin's version of context.WithValue

// handler reads claims
claims := middleware.GetClaims(c)   // retrieves from Gin context
```

---

## Hands-On — Runnable Examples

The snippets above are pulled from the provisioner so you can see context in real code. The examples below are complete `main()` programs you can paste into the Go Playground or a scratch file and run end-to-end to *watch* the behaviour.

As a reminder: you cannot modify an existing context. You must pass a parent context into a function to derive a new, controlled child context.

### 1. `context.WithTimeout` (the automatic egg timer)

This is the most heavily used context function in production. You pass it an existing parent context and a duration. It gives you back two things:

1. The new **child context** (which automatically kills itself when time runs out).
2. A **cancel function** (to clean up resources if your code finishes *before* the time runs out).

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 1. Start with our empty root background context
	parentCtx := context.Background()

	// 2. Create a child context that expires in exactly 2 seconds
	ctx, cancel := context.WithTimeout(parentCtx, 2*time.Second)

	// Crucial rule: Always call cancel() at the end to release memory timers!
	defer cancel()

	// 3. Launch a background task passing our timeout context down
	go fetchData(ctx)

	// Keep main alive long enough to watch the results
	time.Sleep(3 * time.Second)
}

func fetchData(ctx context.Context) {
	select {
	case <-time.After(5 * time.Second):
		// This simulates a slow server that takes 5 seconds to reply
		fmt.Println("Fetch completed successfully!")
	case <-ctx.Done():
		// Because the context timeout is 2s, this case wins at the 2-second mark!
		fmt.Println("🚨 Fetch aborted: Context timed out after 2 seconds.")
	}
}
```

### 2. `context.WithCancel` (the manual stop button)

Use this when you don't know *when* an operation should end, but you want the ability to manually kill it from the outside via a function call. Instead of an automatic timer, it depends entirely on you executing the returned `cancel()` function.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 1. Create a manually cancelable child context
	ctx, cancel := context.WithCancel(context.Background())

	// 2. Start a continuous background worker loop
	go backgroundJob(ctx)

	// Simulate the application running for 1.5 seconds
	time.Sleep(1500 * time.Millisecond)

	fmt.Println("Main: User triggered a manual cancel!")
	cancel() // 💥 This instantly closes the ctx.Done() channel inside the worker!

	// Small pause to see the worker's shutdown print log
	time.Sleep(100 * time.Millisecond)
}

func backgroundJob(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("Worker: Cleaning up and shutting down safely.")
			return
		default:
			fmt.Println("Worker: Processing an item...")
			time.Sleep(500 * time.Millisecond)
		}
	}
}
```

### 3. What is `defer cancel()` and why is it mandatory?

Whenever you use `WithTimeout` or `WithCancel`, the Go runtime allocates resources behind the scenes (internal timers and trackers in the parent tree) to watch that context.

If your code completes its work successfully *before* the 2-second timeout hits, those background tracking structures continue to sit in your RAM until the full timer expires.

Calling `defer cancel()` ensures that the moment your function returns, the child context is cleanly dismantled, immediately freeing your server's RAM from any leftover timer configuration. This is why the codebase snippets above always pair `WithTimeout`/`WithCancel` with `defer cancel()`.

### Summary checklist of derived contexts

| Function | How it stops | Common real-world use case |
| --- | --- | --- |
| **`context.WithTimeout`** | Automatically after a relative duration (e.g., `5s`). | Cutting off database queries or slow HTTP API calls. |
| **`context.WithDeadline`** | Automatically at an absolute time (e.g., `17:00`). | Batch jobs that must end before a specific hour. |
| **`context.WithCancel`** | Manually when you explicitly call `cancel()`. | Graceful shutdowns triggered by OS signals (`SIGTERM`). |

Now that you know how contexts carry **stop triggers** down a chain of functions, the next section ([context.WithValue](#contextwithvalue--passing-data-through-context), above) shows how contexts can also pass read-only metadata — like user accounts or tracking IDs — through your system.

---

## Exercises

### Exercise 1 — Timeout Demo

Write a function that does work that takes 3 seconds. Call it with a 1-second timeout and see what happens:

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func doSlowWork(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()   // context cancelled or timed out
    case <-time.After(3 * time.Second):
        return nil   // work completed
    }
}

func main() {
    // 1-second timeout — should fail
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    err := doSlowWork(ctx)
    if err != nil {
        fmt.Println("slow work failed:", err)   // context deadline exceeded
    } else {
        fmt.Println("slow work done")
    }

    // 5-second timeout — should succeed
    ctx2, cancel2 := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel2()

    err = doSlowWork(ctx2)
    if err != nil {
        fmt.Println("slow work failed:", err)
    } else {
        fmt.Println("slow work done")   // should print this
    }
}
```

---

### Exercise 2 — Cancellable Worker

Write a worker that:
1. Polls every 1 second
2. Prints the tick number
3. Stops when context is cancelled

Call it and cancel after 4 seconds:

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    tick := 0
    for {
        select {
        case <-ctx.Done():
            fmt.Println("worker stopped:", ctx.Err())
            return
        case <-ticker.C:
            tick++
            fmt.Println("tick:", tick)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
    defer cancel()

    worker(ctx)   // runs until ctx times out
    fmt.Println("main done")
}
```

---

### Exercise 3 — Context Chain

Write a function that creates a chain of three contexts, each with a shorter timeout:
- Parent: 10 seconds
- Child: 5 seconds
- Grandchild: 2 seconds

Use `grandchild.Done()` in a goroutine and print which context caused the cancellation.

---

### Exercise 4 — Read Real Context Usage

Open [worker/deploy_worker.go](../../backend/internal/worker/deploy_worker.go).

1. `Run(ctx context.Context)` receives a `context.Background()`. What would happen if you changed it to pass a cancellable context and called cancel on shutdown?
2. `podCtx, cancel := context.WithTimeout(ctx, 8*time.Minute)` — why is 8 minutes chosen? What happens if Harbor pods take longer?
3. `defer cancel()` is called twice (podCtx and bootstrapCtx). What happens if you forget `defer cancel()`?
4. `go w.deploy(context.Background(), deployment)` — why does `deploy` use a NEW `context.Background()` instead of the ctx passed to `processPending`?

---

## Key Takeaways

| Concept | Usage |
|---------|-------|
| Root context | `context.Background()` — no deadline, no cancel |
| Timeout | `ctx, cancel := context.WithTimeout(parent, duration)` |
| Manual cancel | `ctx, cancel := context.WithCancel(parent)` |
| Always do | `defer cancel()` after WithTimeout/WithCancel |
| Wait for cancel | `<-ctx.Done()` |
| Check why | `ctx.Err()` → `context.DeadlineExceeded` or `context.Canceled` |
| Pass through | Accept `ctx context.Context` as first param in every I/O function |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). `ctx` is the first parameter of nearly every function in the codebase — know it cold.

### Recall questions (no peeking)
1. What two things does a `context.Context` carry through a call chain?
2. When do you use `context.Background()`?
3. Why must you `defer cancel()` after `WithTimeout`?
4. What does `ctx.Done()` return, and when does it fire?
5. If you cancel a parent context, what happens to its children?

### Make these Anki cards (front → back)
- Context purpose → carries **cancellation + deadlines** through a call chain
- Timeout → `ctx, cancel := context.WithTimeout(parent, d); defer cancel()`
- `ctx.Done()` → a channel that **closes** on cancel/timeout (used in `select`)
- `ctx.Err()` → `context.DeadlineExceeded` or `context.Canceled`
- Cancel a parent → all child contexts are cancelled too (it's a tree)

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 1 (timeout demo)** from memory.
- [ ] **Day 7 (Friday redo):** answer **Exercise 4** on the real `deploy_worker.go` contexts; explain parent/child cancellation aloud.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [08 — Packages and Modules →](./08-packages-and-modules.md)
