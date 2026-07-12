# 09 — Standard Library Essentials

Go's standard library is large and high quality. These are the packages used directly in the provisioner — learn these and you can read 80% of Go code.

---

## `fmt` — Formatting and Printing

```go
// Print to stdout
fmt.Println("Hello")                        // + newline
fmt.Printf("tenant: %s port: %d\n", t, p)  // formatted + newline
fmt.Print("no newline")                     // no newline

// Build a string (don't print)
s := fmt.Sprintf("harbor-%s", tenantID)     // returns string

// Build an error
err := fmt.Errorf("helm install: %w", err)  // returns error, %w wraps original

// Format verbs:
// %s  = string
// %d  = integer
// %f  = float (%.2f = 2 decimal places)
// %v  = any value (default format)
// %+v = struct with field names
// %T  = type name
// %w  = wrap error (only in fmt.Errorf)
```

---

## `os` — Operating System Interface

```go
// Read environment variable
val := os.Getenv("DATABASE_URL")     // "" if not set
os.Setenv("GIN_MODE", "release")     // set env var

// Read file into []byte
key, err := os.ReadFile("/etc/secrets/master-key")

// Write file
err = os.WriteFile("/tmp/values.yaml", data, 0600)

// Create directory (and parents)
err = os.MkdirAll("/tmp/helm-charts", 0700)

// Delete file
err = os.Remove("/tmp/values.yaml")

// Get hostname (used in worker_lock column)
hostname, _ := os.Hostname()

// Exit program
os.Exit(1)   // exit with error code

// Open a file
f, err := os.Open("/path/to/file")     // read-only
f, err := os.Create("/path/to/file")   // create or truncate
defer f.Close()
```

---

## `strings` — String Manipulation

```go
strings.HasPrefix("https://example.com", "https://")  // true
strings.HasSuffix("harbor-acme", "acme")               // true
strings.Contains("hello world", "world")               // true
strings.TrimSpace("  hello  ")                         // "hello"
strings.TrimPrefix("https://example.com", "https://")  // "example.com"
strings.ToLower("READY")                               // "ready"
strings.ToUpper("ready")                               // "READY"
strings.Split("a,b,c", ",")                            // ["a" "b" "c"]
strings.Join([]string{"a","b"}, "-")                   // "a-b"
strings.Replace("harbor-harbor", "harbor-", "", 1)     // "harbor"
strings.ReplaceAll("a.b.c", ".", "-")                  // "a-b-c"
strings.Fields("  hello   world  ")                    // ["hello" "world"]
strings.Count("acme-acme", "acme")                     // 2
```

From `handlers/registry.go`:
```go
loginServer := strings.TrimPrefix(deployment.RegistryURL, "https://")
// "https://registry.acme.lkdc.com" → "registry.acme.lkdc.com"

message := fmt.Sprintf("A registry is already %s for %s",
    strings.ToLower(string(existing.Status)), tenantID)
// "A registry is already deploying for acme"
```

---

## `strconv` — String Conversions

```go
// String to int
n, err := strconv.Atoi("8080")     // n = 8080, or error if not a number
n, err := strconv.ParseInt("8080", 10, 64)   // base 10, 64-bit

// Int to string
s := strconv.Itoa(8080)            // "8080"

// String to bool
b, err := strconv.ParseBool("true")   // true

// Float
f, err := strconv.ParseFloat("3.14", 64)
```

---

## `time` — Time and Duration

```go
// Current time
now := time.Now()           // time.Time
now.UTC()                   // UTC timezone

// Format (Go uses a reference time: Mon Jan 2 15:04:05 MST 2006)
now.Format(time.RFC3339)    // "2026-06-03T10:30:00Z"
now.Format("2006-01-02")    // "2026-06-03"

// Parse
t, err := time.Parse(time.RFC3339, "2026-06-03T10:30:00Z")

// Duration
5 * time.Second      // 5s
8 * time.Minute      // 8m
30 * time.Second     // 30s
time.Duration(n) * time.Second   // dynamic duration

// Arithmetic
later := now.Add(5 * time.Minute)
elapsed := time.Since(created)   // duration since created
elapsed.Seconds()                 // float64
elapsed.Minutes()

// Comparison
t1.Before(t2)
t1.After(t2)
t1.Equal(t2)
```

From `handlers/registry.go`:
```go
resp["createdAt"] = deployment.CreatedAt.Format(time.RFC3339)

elapsed := int(time.Since(deployment.CreatedAt).Seconds())
resp["elapsedSeconds"] = elapsed

elapsedSec := int(deployment.ReadyAt.Sub(deployment.CreatedAt).Seconds())
resp["provisionedInSeconds"] = elapsedSec
```

---

## `encoding/json` — JSON Encoding/Decoding

```go
// Struct to JSON (marshal)
type Response struct {
    Status string `json:"status"`
    URL    string `json:"url,omitempty"`   // omitempty = skip if empty
}

r := Response{Status: "READY", URL: "https://..."}
data, err := json.Marshal(r)   // data = []byte of JSON
s := string(data)              // {"status":"READY","url":"https://..."}

// JSON to struct (unmarshal)
var r Response
err = json.Unmarshal(data, &r)

// Map to JSON
m := map[string]interface{}{"key": "value", "num": 42}
data, _ = json.Marshal(m)

// JSON to map
var m map[string]interface{}
json.Unmarshal(data, &m)
val := m["key"].(string)   // type assertion needed
```

From `db/postgres.go`:
```go
// Store progress map as JSON in DB
func progressJSON(m map[string]string) []byte {
    if m == nil {
        m = map[string]string{}
    }
    b, _ := json.Marshal(m)   // ["namespace":"READY","bootstrap":"STARTING"]
    return b
}

// Read it back
if len(progressRaw) > 0 {
    json.Unmarshal(progressRaw, &d.Progress)
}
```

---

## `encoding/base64` — Base64 Encoding

Base64 converts binary data to safe ASCII text. Used in the pull-secret endpoint:

```go
// Encode
encoded := base64.StdEncoding.EncodeToString([]byte("user:password"))
// "dXNlcjpwYXNzd29yZA=="

// Decode
decoded, err := base64.StdEncoding.DecodeString(encoded)
// []byte("user:password")
```

From `handlers/registry.go`:
```go
// Build docker login auth string: base64("username:password")
auth := base64.StdEncoding.EncodeToString(
    []byte(creds.RobotUsername + ":" + robotToken),
)

// Encode the full dockerconfig JSON for the K8s Secret
dockerConfigJSON, _ := json.Marshal(dockerConfig)
encoded := base64.StdEncoding.EncodeToString(dockerConfigJSON)
```

---

## `net/http` — HTTP Client and Server

### HTTP Client (making outbound requests)

```go
// Simple GET
resp, err := http.Get("https://api.example.com/health")
defer resp.Body.Close()
body, _ := io.ReadAll(resp.Body)

// Custom client (with timeout)
client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Get("https://...")

// POST with JSON body
data, _ := json.Marshal(payload)
resp, err := http.Post("https://api.example.com/create",
    "application/json",
    bytes.NewReader(data))

// Custom request
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
req.Header.Set("Authorization", "Bearer "+token)
resp, err := client.Do(req)
```

### HTTP Server

```go
// From cmd/main.go
httpServer := &http.Server{
    Addr:         ":8080",
    Handler:      srv.Router(),   // Gin router
    ReadTimeout:  30 * time.Second,
    WriteTimeout: 60 * time.Second,
}

httpServer.ListenAndServe()

// Graceful shutdown — waits for in-flight requests to complete
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
httpServer.Shutdown(ctx)
```

### Status Codes

```go
http.StatusOK           // 200
http.StatusAccepted     // 202
http.StatusBadRequest   // 400
http.StatusUnauthorized // 401
http.StatusForbidden    // 403
http.StatusNotFound     // 404
http.StatusConflict     // 409
http.StatusTooManyRequests // 429
http.StatusInternalServerError // 500
```

---

## `database/sql` — SQL Database Interface

```go
// Open connection (doesn't actually connect yet)
db, err := sql.Open("postgres", dsn)

// Configure pool
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(5)
db.SetConnMaxLifetime(5 * time.Minute)

// Test connectivity
err = db.PingContext(ctx)

// Execute (INSERT, UPDATE, DELETE)
result, err := db.ExecContext(ctx,
    "INSERT INTO users (id, name) VALUES ($1, $2)",
    userID, name)
rowsAffected, _ := result.RowsAffected()

// Query one row
row := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id=$1", id)
var name string
err = row.Scan(&name)   // Scan reads the value into the variable
if err == sql.ErrNoRows {
    // row not found
}

// Query multiple rows
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users WHERE active=$1", true)
defer rows.Close()
for rows.Next() {
    var id, name string
    rows.Scan(&id, &name)
    // process row
}

// sql.NullString — for nullable columns
var url sql.NullString
row.Scan(&url)
if url.Valid {
    fmt.Println(url.String)   // has a value
}
```

---

## `crypto/rand` and `io` — Secure Random Data

```go
// Generate N random bytes (cryptographically secure)
b := make([]byte, 32)
_, err := io.ReadFull(rand.Reader, b)

// From crypto/aes.go
nonce = make([]byte, c.aesGCM.NonceSize())
io.ReadFull(rand.Reader, nonce)
```

---

## Exercises

### Exercise 1 — JSON Round Trip

Write a program that:
1. Defines a `DeployStatus` struct with `TenantID`, `Status`, `Progress map[string]string`
2. Creates one with some progress data
3. Marshals it to JSON and prints it
4. Unmarshals it back and prints specific fields

```go
type DeployStatus struct {
    TenantID string            `json:"tenantId"`
    Status   string            `json:"status"`
    Progress map[string]string `json:"progress"`
}
```

---

### Exercise 2 — HTTP Client

Write a Go program that:
1. Makes a GET request to `https://192.168.10.6/api/v2.0/ping` (from the debug pod)
2. Prints the status code and response body
3. Uses a 5-second timeout
4. Skips TLS verification (hint: `&tls.Config{InsecureSkipVerify: true}`)

```go
import (
    "crypto/tls"
    "net/http"
    "io"
    "fmt"
)

func main() {
    tr := &http.Transport{
        TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
    }
    client := &http.Client{Transport: tr, Timeout: 5 * time.Second}
    // ... make request ...
}
```

---

### Exercise 3 — Time Calculation

Write a function `deployDuration(createdAt, readyAt time.Time) string` that returns:
- `"provisioned in 5m 32s"` format
- Use `time.Duration`'s `.Minutes()` and `.Seconds()` methods

---

### Exercise 4 — String Builder

Reproduce this logic from `GetPullSecret`:

1. Take `username = "robot$acme+readonly"` and `token = "sometoken123"`
2. Combine as `"username:token"`
3. Base64 encode it
4. Print the result

Then decode it back and verify you get the original.

---

## Key Takeaways

| Package | Key Functions |
|---------|--------------|
| `fmt` | `Println`, `Printf`, `Sprintf`, `Errorf` |
| `os` | `Getenv`, `ReadFile`, `MkdirAll`, `Hostname` |
| `strings` | `TrimPrefix`, `Split`, `ToLower`, `Contains` |
| `strconv` | `Atoi`, `Itoa`, `ParseBool` |
| `time` | `Now()`, `Since()`, `Format(RFC3339)`, durations |
| `encoding/json` | `Marshal`, `Unmarshal` |
| `encoding/base64` | `EncodeToString`, `DecodeString` |
| `net/http` | `http.Client`, `http.Server`, status codes |
| `database/sql` | `ExecContext`, `QueryRowContext`, `Scan` |
| `crypto/rand` | `rand.Reader` with `io.ReadFull` |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md). This is a reference chapter — don't memorize every function; memorize *which package* does what, and look up the rest.

### Recall questions (no peeking)
1. `os.Getenv` of an unset variable returns what?
2. `json.Marshal` returns what type? How do you unmarshal into a struct?
3. What does `omitempty` do in a JSON struct tag?
4. What does `row.Scan(&x)` do, and what is `sql.ErrNoRows`?
5. Why does Go's time format use the exact string `2006-01-02 15:04:05`?

### Make these Anki cards (front → back)
- `json.Marshal` → returns `([]byte, error)`; `json.Unmarshal(data, &struct)` reads back
- `omitempty` → omit the field from JSON output when it's the zero value
- `base64.StdEncoding` → `EncodeToString` / `DecodeString` (binary → ASCII), used in the pull-secret
- Go time reference date → `Mon Jan 2 15:04:05 MST 2006` (i.e. `01/02 03:04:05 PM '06`)
- `db.QueryRowContext` + `row.Scan(&v)` → read one row into vars; `sql.ErrNoRows` = not found

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 1 (JSON round trip)** from memory.
- [ ] **Day 7 (Friday redo):** redo **Exercise 4 (base64 encode/decode the robot login)** from memory.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [10 — Gin Framework →](./10-gin-framework.md)
