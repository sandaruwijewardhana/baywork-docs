# 02 — Functions and Error Handling

Go has **no exceptions**. There is no `try/catch`. Errors are just values returned from functions — you check them explicitly every time.

This is the single most important pattern to understand. It appears hundreds of times in the codebase.

---

## Functions with Multiple Return Values

Go functions can return multiple values:

```go
// Returns TWO values: a result and an error
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil   // nil = "no error"
}
```

Calling it:
```go
result, err := divide(10, 2)
if err != nil {
    fmt.Println("error:", err)
    return   // stop here, don't use result
}
fmt.Println(result)   // 5.0
```

**The pattern is always:**
```
value, err := someFunction()
if err != nil {
    // handle it — log, return, panic
}
// use value safely here
```

---

## nil — The Zero Value for Pointers and Interfaces

`nil` in Go means "nothing / empty / not set". It's used for:
- No error: `return nil`
- No result: `return nil, nil`
- Not found: `return nil, nil` (no user, no deployment)

```go
func getDeployment(tenantID string) (*Deployment, error) {
    if tenantID == "" {
        return nil, fmt.Errorf("tenantID is required")   // error
    }
    if notFound {
        return nil, nil   // no error, but also no result
    }
    return &deployment, nil   // success
}

// Caller checks both:
d, err := getDeployment("acme")
if err != nil {
    // actual error
}
if d == nil {
    // not found
}
// d is safe to use here
```

---

## Error Wrapping — Adding Context

When you catch an error and return it, always add context so you know WHERE it came from:

```go
func (s *Store) New(dsn string) (*Store, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("open db: %w", err)
        //                     ^^^^^^^^^^  ^^ %w wraps the original
    }
    return &Store{db: db}, nil
}
```

`%w` wraps the error. The caller sees:
```
"open db: dial tcp 127.0.0.1:5432: connection refused"
```
instead of just:
```
"dial tcp 127.0.0.1:5432: connection refused"
```

Each layer adds context. By the time the error reaches the log, you know the full call chain.

From `db/postgres.go`:
```go
func New(cfg config.DBConfig, logger *zap.Logger) (*Store, error) {
    db, err := sql.Open("postgres", cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("open db: %w", err)
    }
    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("ping db: %w", err)
    }
    return &Store{db: db, logger: logger}, nil
}
```

---

## The Blank Identifier `_`

If a function returns two values but you don't need one, use `_`:

```go
// You need the error but not the number of rows affected
_, err := s.db.ExecContext(ctx, "INSERT ...")
if err != nil {
    return err
}

// You need the result but are 100% sure it won't error (rare — be careful)
b, _ := json.Marshal(progress)
```

In the codebase, `_` appears when:
- The number of rows affected is irrelevant
- JSON marshal of a known-safe value (e.g. `map[string]string`)

---

## defer — Run on Function Exit

`defer` schedules a function call to run when the surrounding function returns — no matter how it returns (normal, error, panic):

```go
func (s *Store) New(cfg config.DBConfig) (*Store, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()   // ← cancel() runs when New() returns, always

    // ... do work ...
}
```

Another common use — close a file or DB connection:

```go
func main() {
    store, _ := db.New(cfg.DB, logger)
    defer store.Close()   // ← Close() runs when main() exits

    // ... rest of program
}   // store.Close() runs here
```

Think of `defer` as "do this cleanup no matter what happens next."
Always runs functoin infront of that after something returend from that functon.NO matter where 'defer' located always runs after END of a function
---

## panic and recover

`panic` stops the program immediately (like an unhandled exception). You almost never use it yourself. The Gin framework uses `recover` internally to catch panics from handlers and return a 500 instead of crashing the server.

```go
// Don't do this unless something is truly unrecoverable
panic("this should never happen")

// This is fine in config loading — if DATABASE_URL is missing, we WANT to crash at startup
func mustEnv(key string) string {
    v := os.Getenv(key)
    if v == "" {
        panic(fmt.Sprintf("required env var %s not set", key))
    }
    return v
}
```

---

## Named Return Values (you'll see these occasionally)

```go
// The return values have names — useful for clarity in complex functions
func encrypt(plaintext []byte) (ciphertext, nonce []byte, err error) {
    // ... work ...
    return   // "naked return" — returns the named values as-is
}
```

From `crypto/aes.go`:
```go
func (c *Cipher) Encrypt(plaintext []byte) (ciphertext, nonce []byte, err error) {
    nonce = make([]byte, c.aesGCM.NonceSize())
    if _, err = io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, nil, fmt.Errorf("generate nonce: %w", err)
    }
    ciphertext = c.aesGCM.Seal(nil, nonce, plaintext, nil)
    return ciphertext, nonce, nil
}
```

---

## Exercises

### Exercise 1 — Write a Safe Divider

Write a function `safeDivide(a, b int) (int, error)` that:
- Returns an error if `b` is 0
- Otherwise returns `a / b`

Call it 3 times: with b=0, b=2, b=5. Handle the error each time.

```go
package main

import (
    "fmt"
)

func safeDivide(a, b int) (int, error) {
    // your code here
}

func main() {
    // call safeDivide 3 times with different inputs
}
```

---

### Exercise 2 — Error Wrapping Chain

Write 3 functions that call each other. Each one should wrap the error from the one below:

```
readConfig() calls openFile() calls os.Open()
```

When `os.Open` fails (use a fake path like `/nonexistent`), the final error message should look like:
```
read config: open file: open /nonexistent: no such file or directory
```

```go
package main

import (
    "fmt"
    "os"
)

func openFile(path string) error {
    _, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("open file: %w", err)
    }
    return nil
}

func readConfig() error {
    // call openFile, wrap its error
}

func main() {
    err := readConfig()
    fmt.Println(err)   // should show the full chain
}
```

---

### Exercise 3 — defer Cleanup

Write a function `processFile(path string)` that:
1. Opens a file
2. Defers closing it (so it always closes)
3. Reads and prints its content

Test it by creating a small text file first:
```bash
echo "tenant=acme" > /tmp/test.txt
```

```go
package main

import (
    "fmt"
    "os"
)

func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("open: %w", err)
    }
    defer f.Close()   // ← runs when processFile returns

    // Read content (hint: use os.ReadFile for simplicity, or io.ReadAll(f))
    content, err := os.ReadFile(path)
    if err != nil {
        return fmt.Errorf("read: %w", err)
    }
    fmt.Println(string(content))
    return nil
}

func main() {
    if err := processFile("/tmp/test.txt"); err != nil {
        fmt.Println("error:", err)
    }
}
```

---

### Exercise 4 — Count Error Checks in the Real Code

Open [worker/deploy_worker.go](../../backend/internal/worker/deploy_worker.go).

1. Count how many `if err != nil` checks appear
2. What does the `fail` helper function do? Why is it defined as a local function?
3. What happens to the deployment status when any step fails?
4. Why does the worker call `go w.deploy(...)` instead of just `w.deploy(...)`?

---

## Key Takeaways

| Concept | Pattern |
|---------|---------|
| Return error | `func f() (Result, error)` |
| Check error | `if err != nil { return ..., err }` |
| No error | `return result, nil` |
| Add context | `fmt.Errorf("doing X: %w", err)` |
| Ignore value | `_, err := f()` |
| Always cleanup | `defer resource.Close()` |
| Crash at startup | `panic("required config missing")` |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). This is the most-used pattern in the whole codebase — over-learn it.

### Recall questions (no peeking)
1. Go has no `try/catch` — how does a function signal an error?
2. What does `return nil, nil` (two nils) mean to the caller?
3. What does `%w` do that `%v` does not?
4. *Exactly* when does a `defer`'d call run — and in what order if there are several?
5. When is `panic` the correct choice? (give the `mustEnv` reasoning)

### Make these Anki cards (front → back)
- Error pattern → `value, err := f(); if err != nil { return ..., err }`
- `%w` → wraps the original error so it can be unwrapped; builds the `"ping db: dial tcp...: connection refused"` chain
- `defer` → runs when the surrounding function returns, **no matter how** (return/error/panic); multiple defers run **LIFO** (last-in-first-out)
- `_` (blank identifier) → discard a return value you don't need
- `panic` in `mustEnv` → fail-fast at startup for missing required config

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 2 (error wrapping chain)** from memory.
- [ ] **Day 7 (Friday redo):** answer **Exercise 4** on the real `deploy_worker.go`; explain the `%w` error chain aloud.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [03 — Structs and Methods →](./03-structs-and-methods.md)
