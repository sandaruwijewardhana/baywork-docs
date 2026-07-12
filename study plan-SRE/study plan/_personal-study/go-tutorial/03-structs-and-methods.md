# 03 — Structs and Methods

Go has no classes. Instead it has **structs** (data) and **methods** (behaviour attached to structs). Together they replace everything you'd use a class for.

---

## Structs — Grouping Related Data

A struct groups fields together under one name:

```go
type RegistryDeployment struct {
    TenantID     string
    Namespace    string
    Status       RegistryStatus     // a custom type (string underneath)
    RegistryURL  string
    HelmRelease  string
    Plan         string
    Progress     map[string]string  // key-value map
    ErrorMessage string
    CreatedAt    time.Time
    UpdatedAt    time.Time
    ReadyAt      *time.Time         // pointer — can be nil (not ready yet)
}
```

This is from `db/postgres.go`. Every row in the `registry_deployments` table maps to one `RegistryDeployment`.

### Creating a Struct

```go
// Method 1: name each field (preferred — clear and order-independent)
d := RegistryDeployment{
    TenantID:    "acme",
    Namespace:   "acme-management",
    Status:      StatusPending,
    HelmRelease: "harbor-acme",
    Plan:        "starter",
}

// Method 2: zero value (all fields = their zero value: "", 0, false, nil)
var d RegistryDeployment
d.TenantID = "acme"

// Method 3: with &  — creates a pointer to the struct (most common in this codebase)
d := &RegistryDeployment{
    TenantID: "acme",
    ...
}
```

### Accessing Fields

```go
fmt.Println(d.TenantID)       // "acme"
fmt.Println(d.Status)         // "PENDING"
d.Status = StatusDeploying    // update a field
```

---

## Pointers — `*` and `&`

This is the most confusing part for beginners. Take it slow.

A **pointer** stores the memory address of a value, not the value itself.

```go
// Normal variable — stores the value
x := 42

// Pointer — stores the ADDRESS of x
p := &x       // & means "address of x"

// Dereference — follow the pointer to get the value
fmt.Println(*p)   // 42

// Modify through pointer
*p = 100
fmt.Println(x)    // 100 — x changed because p points to x
```

**Why pointers matter for structs:**

```go
// This passes a COPY of d — changes inside don't affect the original
func badUpdate(d RegistryDeployment) {
    d.Status = StatusReady   // only changes the copy
}

// This passes a POINTER to d — changes affect the original
func goodUpdate(d *RegistryDeployment) {
    d.Status = StatusReady   // changes the original
}

d := RegistryDeployment{Status: StatusPending}
goodUpdate(&d)               // & = "send the address"
fmt.Println(d.Status)        // "READY"
```

In Go, almost all structs are passed as pointers (`*StructName`) because:
1. Large structs are cheap to pass (just an 8-byte address)
2. The function can modify the original

---

## Methods — Behaviour Attached to Structs

A method is a function with a **receiver** — it belongs to a struct type:

```go
// Value receiver — gets a COPY. Can't modify the original.
func (d RegistryDeployment) IsReady() bool {
    return d.Status == StatusReady
}

// Pointer receiver — gets a POINTER. CAN modify the original.
func (s *Store) Close() error {
    return s.db.Close()
}
```

**Which receiver to use:**
- Use pointer receiver `*T` when the method needs to modify the struct, OR when the struct is large
- Use value receiver `T` only for small read-only checks
- In this codebase: **all methods use pointer receivers** (`*Store`, `*Worker`, `*Cipher`)

Calling a method:

```go
s := &Store{...}
err := s.Close()    // Go automatically handles the pointer
```

---

## The `New()` Constructor Pattern

Go has no constructors. Instead, packages define a `New()` function that creates and returns the struct:

```go
// From db/postgres.go
func New(cfg config.DBConfig, logger *zap.Logger) (*Store, error) {
    db, err := sql.Open("postgres", cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("open db: %w", err)
    }
    // configure the connection pool
    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    // verify connectivity
    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("ping db: %w", err)
    }
    return &Store{db: db, logger: logger}, nil
}
```

Called as:
```go
store, err := db.New(cfg.DB, logger)
```

This pattern:
1. Does validation and setup
2. Returns `nil, error` if something fails
3. Returns a ready-to-use `*Store` on success

Every major component follows this: `db.New`, `crypto.NewCipher`, `k8s.NewHarvesterClient`, `helm.NewDeployer`, `worker.New`.

---

## Struct Tags — JSON and DB Metadata

Struct fields can have **tags** that tell libraries how to handle them:

```go
var req struct {
    Plan string `json:"plan" binding:"required,oneof=starter professional enterprise"`
}
```

From `handlers/registry.go`. The tags mean:
- `json:"plan"` — when converting to/from JSON, use the key `"plan"` (not `"Plan"`)
- `binding:"required"` — Gin will reject the request if this field is missing
- `binding:"oneof=..."` — Gin validates the value is one of these options

Tags are just strings — the libraries use reflection to read them at runtime.

---

## Embedding — Struct Composition

Go doesn't have inheritance, but structs can embed other structs:

```go
type Base struct {
    CreatedAt time.Time
    UpdatedAt time.Time
}

type Deployment struct {
    Base               // ← embedded — all Base fields and methods are promoted
    TenantID string
}

d := Deployment{}
d.CreatedAt = time.Now()   // works directly — promoted from Base
```

This codebase doesn't use embedding much, but you'll see it in Gin's `gin.Context` and in standard library types.

---

## Exercises

### Exercise 1 — Define and Use a Struct

Define a struct `Tenant` with fields:
- `ID string`
- `Plan string`
- `IsActive bool`

Write a method `Describe() string` on it that returns:
```
"Tenant acme (starter) — active"
```
or
```
"Tenant acme (starter) — inactive"
```

```go
package main

import "fmt"

type Tenant struct {
    // your fields
}

func (t *Tenant) Describe() string {
    // your code
}

func main() {
    t := &Tenant{ID: "acme", Plan: "starter", IsActive: true}
    fmt.Println(t.Describe())
}
```

---

### Exercise 2 — New() Constructor

Write a `NewTenant(id, plan string) (*Tenant, error)` function that:
- Returns an error if `id` is empty
- Returns an error if `plan` is not one of: `"starter"`, `"professional"`, `"enterprise"`
- Returns a valid `*Tenant` otherwise with `IsActive: true`

```go
func NewTenant(id, plan string) (*Tenant, error) {
    // your code
}
```

---

### Exercise 3 — Pointer vs Value

Run this code and predict the output BEFORE running:

```go
package main

import "fmt"

type Counter struct {
    Value int
}

func increment(c Counter) {
    c.Value++
}

func incrementPtr(c *Counter) {
    c.Value++
}

func main() {
    c := Counter{Value: 0}
    increment(c)
    fmt.Println("after increment:", c.Value)   // what is this?

    incrementPtr(&c)
    fmt.Println("after incrementPtr:", c.Value)   // and this?
}
```

---

### Exercise 4 — Read the Real Structs

Open [db/postgres.go](../../backend/internal/db/postgres.go).

1. What is the type of `Store.db`? What does it represent?
2. Why is `ReadyAt` a `*time.Time` (pointer) instead of `time.Time`?
3. Look at `RegistryCredentials` — why are `EncryptedToken` and `TokenNonce` stored as `[]byte` and not `string`?
4. What does `AuditEntry.Details map[string]interface{}` mean? (hint: `interface{}` = "any type")

---

## Key Takeaways

| Concept | Syntax |
|---------|--------|
| Define struct | `type Name struct { Field Type }` |
| Create (value) | `Name{Field: value}` |
| Create (pointer) | `&Name{Field: value}` |
| Value receiver | `func (n Name) Method() {}` |
| Pointer receiver | `func (n *Name) Method() {}` |
| Constructor | `func New(...) (*Name, error)` |
| Address of | `&variable` |
| Dereference | `*pointer` |
| Struct tag | `json:"fieldName" binding:"required"` |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). Pointers are the part people forget first — drill them.

### Recall questions (no peeking)
1. Go has no classes — what two things replace them?
2. What does `&x` give you, and what does `*p` give you?
3. Value receiver vs pointer receiver — which one can modify the struct?
4. Why is `ReadyAt` a `*time.Time` instead of a `time.Time`?
5. What three things does the `New()` constructor pattern do?

### Make these Anki cards (front → back)
- Create a struct pointer → `&RegistryDeployment{...}`
- `&` → "address of"; `*` → "dereference" (follow the pointer to the value)
- Pointer receiver `func (s *Store)` → can modify the original + cheap to pass for big structs
- `New()` → validate + set up + return `(*T, error)` (Go has no real constructors)
- Struct tag `json:"plan" binding:"required"` → libraries read these via reflection

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 3 (pointer vs value — predict the output before running)**.
- [ ] **Day 7 (Friday redo):** answer **Exercise 4** on the real `db/postgres.go` structs; explain `&`/`*` aloud.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [04 — Slices and Maps →](./04-slices-and-maps.md)
