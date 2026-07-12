# 05 — Interfaces

An interface is a list of method signatures. Any type that has those methods **automatically** satisfies the interface — no explicit declaration needed.

This is different from Java/C# where you write `implements InterfaceName`. In Go, satisfaction is implicit.

---

## The Problem Interfaces Solve

Without interfaces, your code is tightly coupled to one specific type:

```go
// Tightly coupled — can ONLY work with *HelmDeployer
func deploy(d *HelmDeployer, tenantID string) error {
    return d.Install(...)
}
```

With an interface, your code works with ANYTHING that has the right methods:

```go
// Define what behaviour you need
type Deployer interface {
    Install(ctx context.Context, tenantID, namespace string, values []byte) error
    Uninstall(ctx context.Context, tenantID, namespace string) error
}

// Works with ANY type that has Install + Uninstall
func deploy(d Deployer, tenantID string) error {
    return d.Install(...)
}
```

Now you can pass:
- `*HelmDeployer` — real Helm deployment (production)
- `*MockDeployer` — fake deployment (tests)
- `*DockerDeployer` — Docker Compose deployment (local dev)

All satisfy the `Deployer` interface automatically.

---

## Defining and Satisfying an Interface

```go
// 1. Define the interface
type Greeter interface {
    Greet(name string) string
}

// 2. A type that satisfies it (never says "implements Greeter")
type FormalGreeter struct{}

func (f *FormalGreeter) Greet(name string) string {
    return "Good day, " + name
}

// 3. Another type that also satisfies it
type CasualGreeter struct{}

func (c *CasualGreeter) Greet(name string) string {
    return "Hey, " + name + "!"
}

// 4. Function that accepts any Greeter
func sayHello(g Greeter, name string) {
    fmt.Println(g.Greet(name))
}

// 5. Both work
sayHello(&FormalGreeter{}, "Alice")   // "Good day, Alice"
sayHello(&CasualGreeter{}, "Bob")     // "Hey, Bob!"
```

---

## The `error` Interface

The most important interface in all of Go is `error`:

```go
// Built into Go — just one method
type error interface {
    Error() string
}
```

Every time you do `fmt.Errorf("something failed")`, Go creates a struct that implements this interface. When you print an error with `%v` or `%s`, Go calls `.Error()` on it.

```go
// Any type with an Error() method IS an error
type MyError struct {
    Code    int
    Message string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("error %d: %s", e.Code, e.Message)
}

func doSomething() error {
    return &MyError{Code: 404, Message: "not found"}
}

err := doSomething()
fmt.Println(err)   // "error 404: not found"
```

---

## Common Standard Library Interfaces

These appear constantly in Go code and in your codebase:

### `io.Reader` and `io.Writer`

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Any type with a `Read` method can be used as a source of data. Any type with `Write` can receive data. This is why:
- `os.File` works with `json.Decoder`
- `bytes.Buffer` works with `json.Encoder`
- `rand.Reader` works with `io.ReadFull`

From `crypto/aes.go`:
```go
// rand.Reader satisfies io.Reader — it produces random bytes
io.ReadFull(rand.Reader, nonce)
```

### `http.Handler`

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

Gin's router implements `http.Handler`. That's why it can be passed to `http.Server`:

```go
httpServer := &http.Server{
    Addr:    ":8080",
    Handler: srv.Router(),   // Gin router satisfies http.Handler
}
```

---

## The Empty Interface `interface{}`

`interface{}` (or `any` in newer Go) means "any type at all":

```go
// Can hold anything
var x interface{} = "hello"
x = 42
x = true
x = &Store{}

// Common use: map with mixed value types
details := map[string]interface{}{
    "plan":    "starter",   // string
    "retries": 3,           // int
    "success": true,        // bool
}
```

Used in the codebase for audit log details and Prometheus labels — anywhere the type varies.

---

## Type Assertions — Getting the Real Type Back

When you have an `interface{}`, you can extract the concrete type:

```go
var x interface{} = "hello"

// Type assertion
s, ok := x.(string)   // try to get the string
if ok {
    fmt.Println(s)   // "hello"
}

// Type switch — check multiple possibilities
switch v := x.(type) {
case string:
    fmt.Println("string:", v)
case int:
    fmt.Println("int:", v)
default:
    fmt.Println("unknown type")
}
```

---

## Interfaces in Your Codebase

The provisioner uses interfaces in a few key places:

### `gin.HandlerFunc`

Gin handlers all conform to `func(*gin.Context)`. This is an implicit interface — any function with that signature can be used as a route handler or middleware.

### `context.Context`

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

Every function that does network I/O accepts `context.Context`. It's the standard way to pass cancellation and timeouts through a call chain.

### `zap.Logger` field

The logger (`*zap.Logger`) is passed around everywhere. If you needed to test logging, you'd define a `Logger` interface and swap in a mock.

---

## Exercises

### Exercise 1 — Define and Use an Interface

Define a `Registry` interface with method:
```go
GetURL() string
```

Create two types that satisfy it:
1. `HarborRegistry` with field `URL string`
2. `MockRegistry` with field `URL string`

Write a function `printRegistryURL(r Registry)` and call it with both.

```go
package main

import "fmt"

type Registry interface {
    GetURL() string
}

// your types and methods here

func printRegistryURL(r Registry) {
    fmt.Println("Registry URL:", r.GetURL())
}

func main() {
    // test both
}
```

---

### Exercise 2 — Implement error

Create a custom error type `DeployError` with fields `TenantID string` and `Step string`.
Implement the `error` interface on it.

```go
type DeployError struct {
    TenantID string
    Step     string
}

// implement Error() string

func deploy(tenantID string) error {
    // simulate failure at helm_install
    return &DeployError{TenantID: tenantID, Step: "helm_install"}
}

func main() {
    err := deploy("acme")
    if err != nil {
        fmt.Println(err)   // should print something meaningful
    }
}
```

---

### Exercise 3 — io.Writer

Write a function `writeDeployLog(w io.Writer, tenantID, step, status string)` that writes a log line to any `io.Writer`.

Test it twice:
1. Pass `os.Stdout` → prints to terminal
2. Pass `&bytes.Buffer{}` → captures in memory

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "os"
)

func writeDeployLog(w io.Writer, tenantID, step, status string) {
    fmt.Fprintf(w, "[%s] step=%s status=%s\n", tenantID, step, status)
}

func main() {
    writeDeployLog(os.Stdout, "acme", "helm_install", "READY")

    buf := &bytes.Buffer{}
    writeDeployLog(buf, "acme", "bootstrap", "STARTING")
    fmt.Println("captured:", buf.String())
}
```

---

### Exercise 4 — Read Real Interfaces

Open [cmd/main.go](../../backend/cmd/main.go).

1. `srv.Router()` is passed as `Handler` to `http.Server`. What interface must `Router()` return?
2. `go deployWorker.Run(context.Background())` — `context.Background()` returns a `context.Context`. What does "Background" mean as a context?
3. `signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)` — `quit` is `chan os.Signal`. What is `os.Signal`? (hint: it's an interface in the standard library)

---

## Key Takeaways

| Concept | Example |
|---------|---------|
| Define interface | `type Doer interface { Do() error }` |
| Satisfy implicitly | Just have the methods — no `implements` keyword |
| `error` interface | `Error() string` — built into Go |
| Empty interface | `interface{}` or `any` — accepts any type |
| Type assertion | `s, ok := x.(string)` |
| Type switch | `switch v := x.(type) { case string: ... }` |
| `io.Reader` | "anything I can read bytes from" |
| `http.Handler` | "anything that handles HTTP requests" |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). Your hardest chapter (per the index) — slow down and over-drill it.

### Recall questions (no peeking)
1. How does a type "implement" an interface in Go — what do you write?
2. What single method defines the built-in `error` interface?
3. What does `interface{}` (a.k.a. `any`) mean?
4. What's the difference between a type assertion and a type switch?
5. Why is `io.Writer` powerful — name two different types that satisfy it.

### Make these Anki cards (front → back)
- Interface satisfaction → **implicit** — no `implements` keyword; just have the methods
- `error` interface → one method: `Error() string`
- Type assertion → `s, ok := x.(string)`
- Empty interface `interface{}` / `any` → can hold any type at all
- `io.Writer` → "anything I can `Write` bytes to" (e.g. `os.Stdout`, `bytes.Buffer`)

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 2 (implement the `error` interface)** from memory.
- [ ] **Day 7 (Friday redo):** answer **Exercise 4** on the real `cmd/main.go`; explain implicit satisfaction aloud (Feynman).
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss. (This concept underpins the whole Source-Code Track — Wrangler controllers are interfaces everywhere.)

---

**Next:** [06 — Goroutines and Channels →](./06-goroutines-and-channels.md)
