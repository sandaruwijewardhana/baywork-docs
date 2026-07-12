# 08 — JWT & Authentication

> **Goal:** Understand how JWT tokens work, what JWKS is, and how this system validates who you are and what you're allowed to do.

---

## 1. The Authentication Problem

The API must only allow legitimate users to create/delete registries. How does the server know who's calling?

```
Option 1: Username + Password on every request
  Problem: password travels on every API call — risky, slow

Option 2: Session cookie
  Problem: doesn't work for mobile apps, microservices, CLIs

Option 3: JWT Bearer token
  → Client logs in ONCE to an auth server → gets a signed token
  → Token is sent with every API request
  → API server verifies the token's signature (no DB lookup!)
  → Claims inside the token say who you are and what role you have
```

---

## 2. What Is a JWT?

A **JSON Web Token (JWT)** is a signed, base64-encoded JSON string. It has three parts separated by dots:

```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6IlRFTkFOVF9BRE1JTiJ9.SIG
│─────────────────────│ │─────────────────────────────────────────────────────│ │───│
         Header                              Payload                          Signature
    (algorithm)                          (the claims)                    (verifies it)
```

### Header (decoded)
```json
{
  "alg": "RS256",    ← signing algorithm (RSA + SHA256)
  "typ": "JWT",
  "kid": "key-id-1" ← which key was used to sign this
}
```

### Payload (decoded) — the Claims
```json
{
  "sub":       "user-abc-123",          ← subject (user ID)
  "email":     "sandaruw@wso2.com",
  "tenant_id": "acme-corp",            ← custom claim
  "role":      "TENANT_ADMIN",         ← custom claim
  "iss":       "https://accounts.lkdc.wso2.com",  ← issuer
  "aud":       "registry-provisioner", ← audience (who this token is for)
  "iat":       1705312800,             ← issued at (unix timestamp)
  "exp":       1705316400              ← expires at (1 hour later)
}
```

### Signature
```
RSA_SHA256(base64(header) + "." + base64(payload), privateKey)
```

The auth server **signs** the token with its **private key**. The API server **verifies** the signature with the auth server's **public key**. Only the private key holder can create valid tokens — anyone with the public key can verify them.

---

## 3. JWKS — Public Key Distribution

**JWKS** (JSON Web Key Set) is a standard way to publish the public keys used to verify JWTs.

```
Auth Server                               API Server
───────────                               ──────────
Signs tokens with:                        Verifies tokens with:
  private key (NEVER shared)                public key (fetched from JWKS endpoint)

Publishes at:
  GET /.well-known/jwks.json
  {
    "keys": [
      {
        "kty": "RSA",
        "kid": "key-id-1",
        "n":   "<public key modulus>",
        "e":   "AQAB"
      }
    ]
  }
```

The API server **never needs to call the auth server** for each request — it fetches the public keys once (and refreshes every 15 minutes) and verifies signatures locally.

```
Without JWKS (stateful sessions):
  Request → API server → "is this token valid?" → Auth server
                                                   (every single request!)
  Problem: auth server becomes a bottleneck

With JWKS (stateless JWTs):
  Request → API server → check signature locally (public key in cache)
  No call to auth server — fast, scalable
```

---

## 4. JWT Validation in This System

```go
// From backend/internal/api/middleware/auth.go

type JWTValidator struct {
    jwksEndpoint string  // "https://accounts.lkdc.wso2.com/.well-known/jwks.json"
    issuer       string  // "https://accounts.lkdc.wso2.com"
    audience     string  // "registry-provisioner"
    cache        *jwk.Cache  // refreshes keys every 15 minutes
}

func (v *JWTValidator) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {

        // 1. Extract the Bearer token from the Authorization header
        tokenStr := extractBearerToken(c.GetHeader("Authorization"))
        // Authorization: Bearer eyJhbGci...
        if tokenStr == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "MISSING_TOKEN"})
            return
        }

        // 2. Get the current public key set (from cache, refreshed every 15 min)
        keySet, _ := v.cache.Get(context.Background(), v.jwksEndpoint)

        // 3. Verify the token: signature + issuer + audience + expiry
        token, err := jwt.Parse([]byte(tokenStr),
            jwt.WithKeySet(keySet),    // verify signature
            jwt.WithValidate(true),    // check exp, iat
            jwt.WithIssuer(v.issuer),  // must match
            jwt.WithAudience(v.audience), // must match
        )
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"error": "INVALID_TOKEN"})
            return
        }

        // 4. Extract custom claims
        claims := extractClaims(token)

        // 5. Role check — must be TENANT_ADMIN or PLATFORM_ADMIN
        if claims.Role != "TENANT_ADMIN" && claims.Role != "PLATFORM_ADMIN" {
            c.AbortWithStatusJSON(403, gin.H{"error": "INSUFFICIENT_ROLE"})
            return
        }

        // 6. Store claims in context for handlers to use
        c.Set("claims", claims)
        c.Next()
    }
}
```

### Extracting Claims

```go
func extractClaims(token jwt.Token) *Claims {
    tenantID, _ := token.Get("tenant_id")  // custom claim
    role, _      := token.Get("role")
    email, _     := token.Get("email")

    return &Claims{
        TenantID: tenantID.(string),
        UserID:   token.Subject(),          // standard "sub" claim
        Email:    email.(string),
        Role:     role.(string),
    }
}
```

---

## 5. The TenantGuard — Per-Resource Authorization

Authentication answers: "Who are you?"
Authorization answers: "Are you allowed to do this?"

The `TenantGuard` middleware ensures a `TENANT_ADMIN` can only touch **their own tenant**:

```go
// From backend/internal/api/middleware/auth.go

func TenantGuard() gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := GetClaims(c)       // from JWTValidator middleware

        urlTenant := c.Param("tenantId")  // from URL: /tenants/{tenantId}/registry

        // PLATFORM_ADMIN can access ANY tenant (admin of the whole platform)
        // TENANT_ADMIN can only access THEIR OWN tenant
        if claims.Role != "PLATFORM_ADMIN" && claims.TenantID != urlTenant {
            c.AbortWithStatusJSON(403, gin.H{
                "error":   "TENANT_MISMATCH",
                "message": fmt.Sprintf("You do not have access to tenant %s", urlTenant),
            })
            return
        }
        c.Next()
    }
}
```

```
Example: sandaruw has JWT with tenant_id="acme-corp", role="TENANT_ADMIN"

GET /api/v1/tenants/acme-corp/registry   → ALLOWED  (tenantId matches)
GET /api/v1/tenants/other-corp/registry  → 403 FORBIDDEN (tenantId mismatch)

If role were PLATFORM_ADMIN:
GET /api/v1/tenants/any-tenant/registry  → ALLOWED
```

---

## 6. JWKS Key Cache

The library `lestrrat-go/jwx/v2` manages the key cache:

```go
func NewJWTValidator(jwksEndpoint, issuer, audience string, logger *zap.Logger) (*JWTValidator, error) {
    cache := jwk.NewCache(context.Background())

    // Register the endpoint — will be fetched periodically
    cache.Register(jwksEndpoint, jwk.WithMinRefreshInterval(15*time.Minute))

    // Pre-fetch immediately (fail fast if endpoint is wrong)
    cache.Refresh(context.Background(), jwksEndpoint)

    return &JWTValidator{..., cache: cache}, nil
}
```

```
App startup:
  → fetch keys from JWKS endpoint (fail if unreachable)
  → cache keys in memory

Every API request:
  → read keys from in-memory cache (0ms, no network call)
  → verify JWT signature

Every 15 minutes (background):
  → fetch fresh keys from JWKS endpoint
  → update cache
  (supports key rotation — auth server can rotate keys without downtime)
```

---

## 7. The Dev Bypass

In local development, there's no real auth server. `DEV_SKIP_AUTH=true` bypasses all JWT validation:

```go
// From backend/internal/api/middleware/auth.go

// DevAuthBypass injects a PLATFORM_ADMIN claim without validating any token.
// Only used when DEV_SKIP_AUTH=true. Never enable in production.
func DevAuthBypass() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Set("claims", &Claims{
            TenantID: "dev-tenant",
            UserID:   "dev-user",
            Email:    "dev@localhost",
            Role:     "PLATFORM_ADMIN",   // ← full access
        })
        c.Next()
    }
}
```

```go
// In server.go — chosen at startup based on env var
if os.Getenv("DEV_SKIP_AUTH") == "true" {
    s.logger.Warn("DEV_SKIP_AUTH is enabled — JWT validation is disabled")
    api.Use(middleware.DevAuthBypass())    // no auth check
} else {
    jwtValidator, _ := middleware.NewJWTValidator(...)
    api.Use(jwtValidator.Middleware())    // real auth check
}
```

> ⚠️ This is why the server logs `"DEV_SKIP_AUTH is enabled"` at startup — a visible reminder that security is off.

---

## 8. Rate Limiting — Protecting the API

The `ByTenant` middleware prevents abuse — one tenant can't hammer the API:

```go
// From backend/internal/api/middleware/ratelimit.go
func ByTenant(createRate, readRate float64) gin.HandlerFunc {
    // Token bucket algorithm:
    //   - bucket refills at `createRate` tokens per second
    //   - each request consumes 1 token
    //   - if bucket is empty: 429 Too Many Requests
}
```

```
Parameters from server.go:
  middleware.ByTenant(1.0/600, 1.0)
  //               ^create rate  ^read rate
  //               1 per 600s    1 per second

CREATE registry: max 1 per 10 minutes  (expensive operation)
GET/DELETE:      max 1 per second      (cheap operations)
```

The **token bucket algorithm**:

```
Bucket for tenant-a (create operations):
  Capacity: 1 token
  Refill rate: 1 token / 600 seconds

  Request 1 (t=0s):     token available → serve request, bucket now empty
  Request 2 (t=5s):     no token → 429 Too Many Requests
  Request 3 (t=601s):   bucket refilled → serve request
```

---

## 9. The Full Auth Flow

```
Browser / CLI
     │
     │  1. User logs in to Identity Provider (LKDC Auth)
     │     POST /oauth2/token → receives JWT
     │
     │  2. Every API call includes the JWT
     │  Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
     │
     ▼
Registry Provisioner API
     │
     │  3. RateLimiter: tenant under rate limit?
     │                 NO → 429
     │
     │  4. JWTValidator Middleware:
     │     a. Extract Bearer token from header
     │     b. Fetch public keys from JWKS cache
     │     c. Verify signature (RSA-SHA256)
     │     d. Verify: not expired, correct issuer, correct audience
     │     e. Extract claims: tenantId, role, email, userId
     │     f. Role check: TENANT_ADMIN or PLATFORM_ADMIN?
     │                 NO → 403
     │
     │  5. TenantGuard Middleware:
     │     a. Get tenantId from URL parameter
     │     b. If role=TENANT_ADMIN: JWT tenantId must match URL tenantId
     │                 NO → 403
     │
     │  6. Handler runs with claims available
     │     claims.TenantID → "acme-corp"
     │     claims.Role     → "TENANT_ADMIN"
     │
     ▼
     Response
```

---

## 🏋️ Exercises

### Exercise 1 — Decode a Real JWT
```bash
# Generate a fake JWT for inspection (online tool: jwt.io)
# Or decode a JWT manually:
echo "eyJhbGciOiJSUzI1NiJ9" | base64 -d  # decode header

# Take any JWT (the dots separate the three parts):
TOKEN="eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIn0.sig"
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null  # decode payload
```

### Exercise 2 — See the Dev Auth in Action
```bash
# With DEV_SKIP_AUTH=true, any request works without a token:
curl -s http://localhost:8080/api/v1/tenants/dev-tenant/registry

# The server logs show which tenant/user was "authenticated"
docker logs implementation-of-auto-harbor-deployment-provisioner-1 --tail 5
```

### Exercise 3 — Understand Token Bucket Rate Limiting
The code uses `ByTenant(1.0/600, 1.0)`:
- How many CREATE requests can one tenant make in 10 minutes?
- How many GET requests can one tenant make in 1 second?
- What HTTP status code does it return when rate limited?

Open [backend/internal/api/middleware/ratelimit.go](../backend/internal/api/middleware/ratelimit.go) and find where `429` is returned.

### Exercise 4 — TenantGuard Logic
Given a JWT with `tenant_id: "acme-corp"` and `role: "TENANT_ADMIN"`:

Which of these requests are allowed or blocked by TenantGuard?
- `GET /api/v1/tenants/acme-corp/registry` → ?
- `GET /api/v1/tenants/other-corp/registry` → ?
- `DELETE /api/v1/tenants/acme-corp/registry` → ?

Now change the role to `PLATFORM_ADMIN` — what changes?

### Exercise 5 — Why Validate the Audience?
The JWT contains `"aud": "registry-provisioner"`. The validator checks this with `jwt.WithAudience("registry-provisioner")`.

Why is this check important? What attack does it prevent?

Hint: Imagine you have two services — a billing API and this registry API — both accepting JWTs from the same auth server.

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **JWT** | A signed token containing claims (who you are, what role) |
| **Header** | Algorithm + key ID used to sign |
| **Payload** | The claims: sub, role, tenant_id, exp, iss, aud |
| **Signature** | Proves the token wasn't tampered with |
| **RS256** | RSA with SHA-256 — asymmetric signing |
| **JWKS** | Endpoint that publishes the public keys to verify JWTs |
| **Key cache** | Refreshed every 15 min — avoid network call per request |
| **TenantGuard** | Ensures TENANT_ADMIN can only access their own tenant |
| **PLATFORM_ADMIN** | Cross-tenant admin role — can access any tenant |
| **DEV_SKIP_AUTH** | Dev-only bypass — injects fake PLATFORM_ADMIN claims |
| **Rate limiting** | Token bucket — 1 create/10min, 1 read/sec per tenant |

**Next:** [09 — Cryptography: AES-256-GCM →](./09-cryptography.md)
