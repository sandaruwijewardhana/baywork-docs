# 01 — First Steps in Go

---

## What Go Looks Like

Every Go file starts with `package`. Code runs from `func main()`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Registry!")
}
```

Run it:
```bash
go run hello.go
# Output: Hello, Registry!
```

---

## Variables

Go has two ways to declare a variable:

```go
// Way 1: short declaration (most common inside functions)
name := "acme"       // Go figures out the type: string
port := 8080         // int
ready := false       // bool

// Way 2: explicit type (used at package level, or when you want to be clear)
var status string = "PENDING"
var count int = 0
```

**Rule:** `:=` only works inside a function. Outside a function use `var`.

### Basic Types

| Type | Example | What it stores |
|------|---------|---------------|
| `string` | `"acme"` | Text |
| `int` | `8080` | Whole number |
| `float64` | `3.14` | Decimal number |
| `bool` | `true` | True/false |
| `[]byte` | `[]byte("hi")` | Raw bytes (used for file contents, encrypted data) |

---

## Constants

Constants never change. Use them for fixed values:

```go
const StatusPending = "PENDING"
const StatusReady   = "READY"
const MaxConnections = 25
```

In your codebase, `db/postgres.go` uses this pattern:

```go
type RegistryStatus string     // ← "string underneath" custom type

const (
    StatusPending   RegistryStatus = "PENDING"
    StatusDeploying RegistryStatus = "DEPLOYING"
    StatusReady     RegistryStatus = "READY"
    StatusFailed    RegistryStatus = "FAILED"
)
```

The `( ... )` groups multiple `const` lines together — just saves typing `const` on every line.

---

## Functions

```go
// A function with no return value
func greet(name string) {
    fmt.Println("Hello,", name)
}

// A function that returns a value
func add(a int, b int) int {
    return a + b
}

// Calling them
greet("acme")       // prints: Hello, acme
result := add(3, 4) // result = 7
```

### The `fmt` Package

`fmt` is Go's built-in formatting package:

```go
fmt.Println("simple text")              // prints + newline
fmt.Printf("tenant: %s\n", tenantID)   // printf-style formatting
fmt.Sprintf("harbor-%s", tenantID)      // returns a string (doesn't print)
fmt.Errorf("open db: %w", err)          // returns an error with context
```

`%s` = string, `%d` = integer, `%v` = anything, `%w` = wrap an error.

---

## Reading Environment Variables

In `config/config.go`, these helper functions read env vars:

```go
func envStr(key, def string) string {
    if v := os.Getenv(key); v != "" {  // os.Getenv reads from environment
        return v
    }
    return def   // return the default if env var is not set
}

func envInt(key string, def int) int {
    if v := os.Getenv(key); v != "" {
        if n, err := strconv.Atoi(v); err == nil {  // Atoi = "ASCII to integer"
            return n
        }
    }
    return def
}
```

`strconv.Atoi("8080")` converts the string `"8080"` to the integer `8080`.  
`strconv.ParseBool("true")` converts `"true"` to the boolean `true`.

---

## if / else

```go
status := "PENDING"

if status == "READY" {
    fmt.Println("Harbor is up!")
} else if status == "FAILED" {
    fmt.Println("Something went wrong")
} else {
    fmt.Println("Still working...")
}
```

**Special Go trick** — assign and check in one line:

```go
if v := os.Getenv("PORT"); v != "" {
    // v is only accessible inside this if block
    port, _ = strconv.Atoi(v)
}
```

---

## switch

```go
switch status {
case "READY":
    fmt.Println("serve traffic")
case "FAILED", "DELETED":      // multiple values in one case
    fmt.Println("allow retry")
default:
    fmt.Println("wait")
}
```

From `handlers/registry.go`:

```go
switch existing.Status {
case db.StatusReady, db.StatusDeploying:
    c.JSON(http.StatusConflict, gin.H{"error": "REGISTRY_EXISTS"})
    return
case db.StatusPending:
    c.JSON(http.StatusConflict, gin.H{"error": "REGISTRY_PENDING"})
    return
case db.StatusFailed, db.StatusDeleted:
    // Allow retry — remove old record
    h.store.DeleteDeployment(...)
}
```

---

## Exercises

### Exercise 1 — Hello Tenant

Write a Go program that:
1. Stores tenant ID in a variable
2. Stores port in an integer variable
3. Prints: `"Starting registry for tenant: acme on port: 8080"`

```go
package main

import "fmt"

func main() {
    // your code here
}
```

Expected output: `Starting registry for tenant: acme on port: 8080`

---

### Exercise 2 — Status Printer

Write a function `printStatus(status string)` that prints:
- `"Harbor is ready!"` if status is `"READY"`
- `"Deploying..."` if status is `"DEPLOYING"`
- `"Unknown status: X"` for anything else (replace X with the actual value)

```go
package main

import "fmt"

func printStatus(status string) {
    // your code here
}

func main() {
    printStatus("READY")
    printStatus("DEPLOYING")
    printStatus("PENDING")
}
```

---

### Exercise 3 — Env Reader

Write a function `getPort() int` that:
1. Reads the env var `PORT`
2. Returns it as an integer if set
3. Returns `8080` as default if not set

```bash
# Test it:
PORT=9090 go run exercise3.go   # should print 9090
go run exercise3.go              # should print 8080
```

Hint: `os.Getenv`, `strconv.Atoi` — import `"os"` and `"strconv"`.

---

### Exercise 4 — Read the Real Config

Open [config/config.go](../../backend/internal/config/config.go).

Answer:
1. What happens if `DATABASE_URL` is not set? (look at `mustEnv`)
2. What is the default value for `INGRESS_CLASS`?
3. What is the default shutdown timeout?
4. Why does `Load()` return `(*Config, error)` instead of just `Config`?

---

## Key Takeaways

| Concept | Syntax |
|---------|--------|
| Short variable | `x := value` |
| Typed constant | `const StatusReady RegistryStatus = "READY"` |
| Function | `func name(param type) returnType { }` |
| Format string | `fmt.Sprintf("harbor-%s", tenantID)` |
| Read env var | `os.Getenv("KEY")` |
| String to int | `strconv.Atoi("8080")` |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). Do active recall + spaced repetition, or you lose this in ~a week. Don't memorize syntax you can look up — memorize the *rules* and re-derive the rest.

### Recall questions (close the doc, answer out loud or in writing — no peeking)
1. When can you use `:=` and when must you use `var`?
2. What's the difference between `fmt.Printf` and `fmt.Sprintf`?
3. What do `%s`, `%d`, `%v`, and `%w` each mean?
4. In `envStr`, why does it return `def` at the end instead of `""`?
5. `strconv.Atoi("8080")` returns two values — what are they?

### Make these Anki cards (front → back)
- `:=` vs `var` → `:=` only **inside** a function; `var` works at package level too
- `%w` → wraps an error (only valid inside `fmt.Errorf`)
- `os.Getenv("X")` when X is unset → returns `""` (empty string), **not** an error
- Typed constant `const StatusReady RegistryStatus = "READY"` → a string-underneath custom type that blocks invalid status values

### Spaced-repetition schedule for this chapter
- [ ] **Day 1 (today):** do the 4 exercises above + create the Anki cards.
- [ ] **Day 3:** Anki reviews + redo **Exercise 3 (Env Reader)** with the doc closed.
- [ ] **Day 7 (Friday redo):** reproduce **Exercise 4 (read the real `config.go`)** from memory; explain `:=` vs `var` aloud (Feynman).
- [ ] **Day 14 / 30:** Anki resurfaces the cards automatically — if one fails, reread that section.

---

**Next:** [02 — Functions and Errors →](./02-functions-and-errors.md)
