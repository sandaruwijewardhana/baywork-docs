# 10 — Gin Framework

Gin is the HTTP web framework used by the provisioner. It handles routing, middleware, request parsing, and JSON responses. It wraps Go's standard `net/http` package with a much nicer API.

---

## Why Gin

Go's built-in `net/http` is low-level. You'd have to manually parse URL parameters, bind JSON, write middleware chains, etc. Gin does all that with minimal overhead.

```go
// Without Gin (raw net/http) — verbose
func handler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    var req struct { Plan string }
    json.Unmarshal(body, &req)
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(202)
    json.NewEncoder(w).Encode(map[string]string{"status": "PENDING"})
}

// With Gin — clean
func handler(c *gin.Context) {
    var req struct { Plan string `json:"plan"` }
    c.ShouldBindJSON(&req)
    c.JSON(http.StatusAccepted, gin.H{"status": "PENDING"})
}
```

---

## Setting Up a Gin Router

From `api/server.go`:

```go
func (s *Server) buildRouter() http.Handler {
    r := gin.New()   // empty router (no default middleware)

    // Global middleware — runs for EVERY request
    r.Use(middleware.Recovery(s.logger))
    r.Use(middleware.RequestLogger(s.logger))
    r.Use(cors.New(corsConfig))

    // Health check endpoints (no auth required)
    r.GET("/healthz", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    r.GET("/readyz", func(c *gin.Context) {
        if err := s.store.Ping(); err != nil {
            c.JSON(http.StatusServiceUnavailable, gin.H{"status": "db not ready"})
            return
        }
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    // Versioned API group — prefix: /api/v1
    api := r.Group("/api/v1")
    api.Use(middleware.RateLimiter())
    api.Use(middleware.DevAuthBypass())   // auth middleware

    // Tenant-scoped group — prefix: /api/v1/tenants/:tenantId
    tenants := api.Group("/tenants/:tenantId")
    tenants.Use(middleware.TenantGuard(s.logger))

    // Registry routes — prefix: /api/v1/tenants/:tenantId/registry
    registry := tenants.Group("/registry")
    registry.POST("",                      h.Create)
    registry.GET("",                       h.Get)
    registry.DELETE("",                    h.Delete)
    registry.GET("/credentials",           h.GetCredentials)
    registry.POST("/credentials/rotate",   h.RotateCredentials)
    registry.GET("/pull-secret",           h.GetPullSecret)

    return r
}
```

---

## `gin.Context` — The Request/Response Object

Every handler receives `*gin.Context`. It gives you access to:

```go
func (h *Handler) Create(c *gin.Context) {

    // --- READ INPUT ---

    // URL parameter: /tenants/:tenantId → c.Param("tenantId")
    tenantID := c.Param("tenantId")      // "acme"

    // Query string: /pull-secret?namespace=prod → c.Query("namespace")
    ns := c.Query("namespace")           // "prod"
    ns = c.DefaultQuery("namespace", "default")   // with fallback

    // Request body as JSON (binds and validates using struct tags)
    var req struct {
        Plan string `json:"plan" binding:"required,oneof=starter professional enterprise"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return   // ← important: return after writing a response
    }

    // Client IP address
    ip := c.ClientIP()

    // Request context (for DB calls, K8s calls)
    ctx := c.Request.Context()

    // --- WRITE OUTPUT ---

    // JSON response
    c.JSON(http.StatusOK, gin.H{
        "tenantId": tenantID,
        "status":   "READY",
    })

    // JSON response with a struct
    c.JSON(http.StatusOK, MyResponse{TenantID: tenantID})

    // Non-JSON response (YAML pull secret)
    c.Header("Content-Type", "application/yaml")
    c.String(http.StatusOK, yamlString)

    // Set a response header
    c.Header("X-Request-ID", requestID)
}
```

---

## `gin.H` — Quick Map for JSON

`gin.H` is just `map[string]interface{}`. It's a shortcut for building JSON responses:

```go
// These are identical
c.JSON(200, gin.H{"status": "ok"})
c.JSON(200, map[string]interface{}{"status": "ok"})

// Nested
c.JSON(200, gin.H{
    "tenantId":    "acme",
    "status":      "READY",
    "registryUrl": "https://...",
    "quickstart": gin.H{
        "login": "docker login ...",
        "push":  "docker push ...",
    },
})
```

---

## Middleware

Middleware is a function that wraps every request. It can:
- Run before the handler (validate auth, log request, check rate limit)
- Call `c.Next()` to pass control to the next middleware/handler
- Run after the handler (log response time, add headers)
- Short-circuit with `c.AbortWithStatus(401)` — skip all following middleware/handlers

```go
func SomeMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // BEFORE handler
        start := time.Now()

        c.Next()   // ← call next middleware or handler

        // AFTER handler (handler has already written the response)
        duration := time.Since(start)
        log.Printf("request took %s", duration)
    }
}
```

### Auth Middleware Pattern

From `middleware/auth.go` (conceptual):

```go
func DevAuthBypass() gin.HandlerFunc {
    return func(c *gin.Context) {
        if os.Getenv("DEV_SKIP_AUTH") == "true" {
            // Inject fake claims so handlers can read them
            c.Set("claims", &Claims{
                UserID: "dev-user",
                Email:  "dev@test.com",
            })
            c.Next()
            return
        }

        // Production: validate JWT
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
            return   // AbortWithStatus + return = short-circuit, handler never runs
        }

        claims, err := validateJWT(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
            return
        }

        c.Set("claims", claims)
        c.Next()
    }
}

// Reading from context in a handler:
func GetClaims(c *gin.Context) *Claims {
    claims, _ := c.Get("claims")
    return claims.(*Claims)
}
```

### TenantGuard Middleware

```go
func TenantGuard(logger *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := GetClaims(c)
        urlTenantID := c.Param("tenantId")

        // Reject if JWT tenantId doesn't match the URL :tenantId
        if claims.TenantID != urlTenantID {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "TENANT_MISMATCH",
            })
            return
        }
        c.Next()
    }
}
```

### Rate Limiter Middleware (conceptual)

```go
func RateLimiter() gin.HandlerFunc {
    // Store last request time per tenant (in memory, or Redis in production)
    lastRequest := map[string]time.Time{}
    var mu sync.Mutex

    return func(c *gin.Context) {
        tenantID := c.Param("tenantId")
        mu.Lock()
        last := lastRequest[tenantID]
        if time.Since(last) < 10*time.Minute {
            mu.Unlock()
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "RATE_LIMITED",
            })
            return
        }
        lastRequest[tenantID] = time.Now()
        mu.Unlock()
        c.Next()
    }
}
```

---

## Request Binding and Validation

Gin uses struct tags to validate JSON bodies automatically:

```go
var req struct {
    Plan     string `json:"plan"     binding:"required,oneof=starter professional enterprise"`
    Tenant   string `json:"tenant"   binding:"required,min=3,max=50"`
    Replicas int    `json:"replicas" binding:"min=1,max=10"`
}

if err := c.ShouldBindJSON(&req); err != nil {
    // err describes what failed: "plan" is required, etc.
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}
// req.Plan is guaranteed to be "starter", "professional", or "enterprise"
```

Common binding tags:
- `required` — field must be present and non-zero
- `oneof=a b c` — must be one of these values
- `min=N` / `max=N` — minimum/maximum length (string) or value (number)
- `email` — must be valid email
- `url` — must be valid URL

---

## Returning Different Content Types

```go
// JSON (most common)
c.JSON(200, gin.H{"key": "value"})

// Plain string
c.String(200, "hello world")

// YAML or any text
c.Header("Content-Type", "application/yaml")
c.Header("Content-Disposition", `attachment; filename="secret.yaml"`)
c.String(200, yamlContent)

// File download
c.File("/path/to/file.pdf")

// Stream (for large files)
c.Stream(func(w io.Writer) bool {
    // write to w in chunks
    return false   // false = done
})
```

---

## Exercises

### Exercise 1 — Hello Gin

Create a simple Gin server:

```bash
mkdir -p ~/go-exercises/gin-hello
cd ~/go-exercises/gin-hello
go mod init gin-hello
go get github.com/gin-gonic/gin
```

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "pong"})
    })

    r.GET("/tenant/:id", func(c *gin.Context) {
        id := c.Param("id")
        c.JSON(http.StatusOK, gin.H{
            "tenantId": id,
            "status":   "READY",
        })
    })

    r.Run(":8080")
}
```

```bash
go run main.go &
curl localhost:8080/ping
curl localhost:8080/tenant/acme
```

---

### Exercise 2 — Add a POST Handler

Add `POST /registry` that:
1. Binds JSON body: `{"plan": "starter"}`
2. Validates `plan` is one of: starter, professional, enterprise
3. Returns `{"tenantId": "acme", "status": "PENDING", "plan": "starter"}`

Use the `ShouldBindJSON` pattern with struct tags.

---

### Exercise 3 — Write a Middleware

Write a middleware `RequestLogger()` that:
1. Records `time.Now()` before the request
2. Calls `c.Next()`
3. After the handler, prints: `"[METHOD] /path - STATUS - Xms"`

Add it to your Gin router with `r.Use(RequestLogger())`.

---

### Exercise 4 — Read the Real Handlers

Open [api/handlers/registry.go](../../backend/internal/api/handlers/registry.go).

1. In `Create()`, what happens if the tenant already has a FAILED deployment? How is the retry handled?
2. In `GetCredentials()`, the robot token is decrypted before sending. Why is it stored encrypted in the DB?
3. In `GetPullSecret()`, what is a "docker pull secret"? What K8s object does it create?
4. In `Delete()`, the handler returns `202 Accepted` immediately. Who actually uninstalls Helm? (hint: there's a TODO comment)

---

## Key Takeaways

| Concept | Code |
|---------|------|
| New router | `r := gin.New()` |
| Route group | `api := r.Group("/api/v1")` |
| Add middleware | `r.Use(MyMiddleware())` |
| URL param | `c.Param("tenantId")` |
| Query string | `c.Query("namespace")` |
| Bind JSON | `c.ShouldBindJSON(&req)` |
| JSON response | `c.JSON(200, gin.H{"key": "val"})` |
| String response | `c.String(200, "text")` |
| Set header | `c.Header("Content-Type", "application/yaml")` |
| Short-circuit | `c.AbortWithStatusJSON(401, gin.H{...})` |
| Store in context | `c.Set("key", value)` |
| Read from context | `val, _ := c.Get("key")` |
| Pass to next | `c.Next()` |

## 🧠 Retention — lock this chapter in

> Tied to the **Retention System** in [`../sre-mastery/00-curriculum.md`](../sre-mastery/00-curriculum.md).

### Recall questions (no peeking)
1. What single argument does every Gin handler receive?
2. `c.Param("x")` vs `c.Query("x")` — what's the difference?
3. What does `c.ShouldBindJSON(&req)` do with the struct's `binding:` tags?
4. What is `c.Next()` for in middleware, and what runs after it?
5. Why do you write `c.AbortWithStatusJSON(...)` **and** `return` together?

### Make these Anki cards (front → back)
- Handler signature → `func(c *gin.Context)`
- `c.Param("tenantId")` → URL path segment; `c.Query("ns")` → query string `?ns=`
- `binding:"required,oneof=..."` → Gin validates the JSON body and rejects invalid input
- Middleware `c.Next()` → pass control onward; code **after** `Next()` runs post-handler
- Short-circuit → `c.AbortWithStatusJSON(...)` + `return` (handler never runs); `gin.H` = `map[string]interface{}`

### Spaced-repetition schedule for this chapter
- [ ] **Day 1:** exercises + Anki cards.
- [ ] **Day 3:** redo **Exercise 3 (write a `RequestLogger` middleware)** from memory.
- [ ] **Day 7 (Friday redo):** answer **Exercise 4** on the real `handlers/registry.go`; explain the middleware chain order aloud.
- [ ] **Day 14 / 30:** Anki reviews; reread on any miss.

---

**Next:** [11 — Dependency Injection →](./11-dependency-injection.md)
