# 04 — Slices and Maps

Two data structures you'll use constantly: **slices** (ordered lists) and **maps** (key-value pairs).

---

## Slices — Ordered Lists

A slice is a resizable list of values of the same type:

```go
// Declare an empty slice
var names []string

// Declare with values
plans := []string{"starter", "professional", "enterprise"}

// Declare with make (when you know the size upfront)
progress := make([]string, 0, 10)   // length 0, capacity 10
```

### Reading Elements

```go
plans := []string{"starter", "professional", "enterprise"}
fmt.Println(plans[0])    // "starter"    — index starts at 0
fmt.Println(plans[2])    // "enterprise"
fmt.Println(len(plans))  // 3
```

### Adding Elements

```go
names := []string{"acme", "beta"}
names = append(names, "gamma")      // append returns the new slice
names = append(names, "delta", "epsilon")   // multiple at once
```

### Iterating with for range

```go
plans := []string{"starter", "professional", "enterprise"}

for i, plan := range plans {
    fmt.Printf("%d: %s\n", i, plan)
}
// 0: starter
// 1: professional
// 2: enterprise

// If you don't need the index:
for _, plan := range plans {
    fmt.Println(plan)
}
```

### Slicing a Slice

```go
s := []int{1, 2, 3, 4, 5}
fmt.Println(s[1:3])   // [2 3]  — from index 1 up to (not including) 3
fmt.Println(s[:2])    // [1 2]  — from start to index 2
fmt.Println(s[2:])    // [3 4 5] — from index 2 to end
```

---

## []byte — Byte Slices

`[]byte` is a slice of bytes. It's used everywhere for raw binary data, file contents, encrypted values, and JSON:

```go
// String to bytes
b := []byte("hello")     // [104 101 108 108 111]

// Bytes to string
s := string(b)           // "hello"

// Read a file into []byte
key, err := os.ReadFile("/etc/secrets/master-key")
// key is []byte

// JSON marshal into []byte
data, err := json.Marshal(myStruct)
// data is []byte containing {"field":"value",...}
```

From `crypto/aes.go`:
```go
func (c *Cipher) Encrypt(plaintext []byte) (ciphertext, nonce []byte, err error) {
    nonce = make([]byte, c.aesGCM.NonceSize())   // allocate nonce bytes
    io.ReadFull(rand.Reader, nonce)               // fill with random bytes
    ciphertext = c.aesGCM.Seal(nil, nonce, plaintext, nil)
    return ciphertext, nonce, nil
}
```

---

## Maps — Key-Value Pairs

A map stores key-value pairs. Keys must all be the same type; values must all be the same type:

```go
// Declare an empty map
progress := map[string]string{}

// Declare with values
progress := map[string]string{
    "namespace":    "READY",
    "helm_install": "STARTING",
    "bootstrap":    "PENDING",
}

// Add or update
progress["helm_install"] = "READY"

// Read
status := progress["namespace"]   // "READY"

// Read with existence check
val, exists := progress["bootstrap"]
if !exists {
    fmt.Println("bootstrap not started yet")
}

// Delete
delete(progress, "old_key")

// Length
fmt.Println(len(progress))   // 3
```

### Iterating a Map

```go
for key, value := range progress {
    fmt.Printf("%s = %s\n", key, value)
}
// order is NOT guaranteed — maps are unordered in Go
```

### Maps with Complex Values

```go
// map[string]interface{} — value can be anything (string, int, bool, struct)
details := map[string]interface{}{
    "plan":      "starter",
    "namespace": "acme-management",
    "retries":   3,
    "success":   true,
}
```

From `handlers/registry.go`:
```go
h.auditLog.Log(c.Request.Context(), audit.Event{
    Details: map[string]interface{}{
        "plan":      req.Plan,
        "namespace": namespace,
    },
})
```

---

## Progress Tracking in the Worker

The deploy worker uses `map[string]string` to track each step:

```go
// From worker/deploy_worker.go
func (w *Worker) setProgress(ctx context.Context, tenantID, component, status string) {
    deployment, err := w.store.GetDeployment(ctx, tenantID)
    if err != nil || deployment == nil {
        return
    }
    progress := deployment.Progress
    if progress == nil {
        progress = map[string]string{}   // initialize if nil
    }
    progress[component] = status         // set one step's status
    w.store.UpdateProgress(ctx, tenantID, progress)
}
```

And in `WaitForAllReady`:

```go
err = w.k8s.WaitForAllReady(podCtx, namespace, func(status map[string]bool) {
    progress := make(map[string]string)
    for comp, ready := range status {   // iterate the status map
        if ready {
            progress[comp] = "READY"
        } else {
            progress[comp] = "STARTING"
        }
    }
    w.store.UpdateProgress(ctx, tenantID, progress)
})
```

---

## make vs literal

```go
// Literal — preferred when you have initial values
m := map[string]string{"a": "1"}
s := []string{"x", "y"}

// make — preferred when starting empty
m := make(map[string]string)    // empty map, ready to use
s := make([]string, 0, 10)      // empty slice, capacity hint

// var — zero value (nil map/slice) — careful: you can't write to a nil map
var m map[string]string   // nil — m["key"] = "val" will PANIC
m = make(map[string]string)   // now it's safe
```

---

## Exercises

### Exercise 1 — Progress Tracker

Write a program that simulates the 7-step deploy progress:

```go
package main

import "fmt"

func main() {
    steps := []string{
        "namespace", "helm_install", "harbor-core", "harbor-db",
        "harbor-redis", "bootstrap", "credentials",
    }
    progress := make(map[string]string)

    // Mark each step READY one by one, printing the whole map each time
    for _, step := range steps {
        progress[step] = "READY"
        printProgress(progress)
    }
}

func printProgress(p map[string]string) {
    fmt.Println("--- Progress ---")
    for k, v := range p {
        fmt.Printf("  %s: %s\n", k, v)
    }
}
```

---

### Exercise 2 — Filter Slice

Write a function `readyTenants(statuses map[string]string) []string` that:
- Takes a map of `tenantID → status`
- Returns a slice of tenant IDs where status is `"READY"`

```go
func readyTenants(statuses map[string]string) []string {
    // your code
}

func main() {
    statuses := map[string]string{
        "acme":  "READY",
        "beta":  "DEPLOYING",
        "gamma": "READY",
        "delta": "FAILED",
    }
    fmt.Println(readyTenants(statuses))   // ["acme" "gamma"] (order may vary)
}
```

---

### Exercise 3 — Word Count

Write a function that counts how many times each word appears:

```go
func wordCount(sentence string) map[string]int {
    // hint: strings.Fields(sentence) splits by spaces
}

func main() {
    counts := wordCount("acme acme beta gamma acme beta")
    for word, count := range counts {
        fmt.Printf("%s: %d\n", word, count)
    }
    // acme: 3
    // beta: 2
    // gamma: 1
}
```

---

### Exercise 4 — Read Real Code

Open [worker/deploy_worker.go](../../backend/internal/worker/deploy_worker.go).

1. What type is `progress` in `setProgress`? What happens if `deployment.Progress` is nil?
2. In the `WaitForAllReady` callback, `status map[string]bool` — what are the keys? What are the values?
3. How does the callback convert `map[string]bool` to `map[string]string`?
4. Why does the progress use `string` values ("READY", "STARTING") instead of `bool`? What's the advantage?

---

## Key Takeaways

| Concept | Syntax |
|---------|--------|
| Slice literal | `[]string{"a", "b"}` |
| Append | `s = append(s, "c")` |
| Map literal | `map[string]string{"k": "v"}` |
| Map write | `m["key"] = "value"` |
| Map read | `val := m["key"]` |
| Map check | `val, ok := m["key"]` |
| Delete from map | `delete(m, "key")` |
| Iterate | `for k, v := range collection {}` |
| Bytes | `[]byte("text")` ↔ `string(bytes)` |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md).

### Recall questions (no peeking)
1. How do you add to a slice, and why must you reassign (`s = append(...)`)?
2. What two values does `val, ok := m["key"]` give you?
3. What happens if you write to a `nil` map?
4. Is map iteration order guaranteed?
5. When do you reach for `[]byte` instead of `string`?

### Make these Anki cards (front → back)
- Append → `s = append(s, "c")` — must reassign (append may return a new backing array)
- Map existence check → `val, ok := m["key"]`
- Write to a `nil` map → **PANIC**; must `make(map[...]...)` first
- Map iteration order → **not guaranteed** (unordered)
- Delete from map → `delete(m, "key")`; `[]byte("hi")` ↔ `string(b)`

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 3 (word count)** from memory.
- [ ] **Day 7 (Friday redo):** answer **Exercise 4** on `setProgress` in `deploy_worker.go`; explain the nil-map panic aloud.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [05 — Interfaces →](./05-interfaces.md)
