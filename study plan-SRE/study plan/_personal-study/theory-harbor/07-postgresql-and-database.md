# 07 — PostgreSQL & Database Patterns

> **Goal:** Understand relational databases, SQL, how this system's three tables work, and the clever `FOR UPDATE SKIP LOCKED` pattern that prevents two workers from processing the same job.

---

## 1. Why a Database?

The deploy worker could just run in memory — but what happens if the server restarts?

```
WITHOUT a database:
  User clicks "Create Registry" → state in memory: { tenant-a: DEPLOYING }
  Server crashes / restarts
  State lost — nobody knows it was deploying
  User sees nothing — confused
  Harbor half-deployed in K8s — orphaned resources

WITH PostgreSQL:
  User clicks "Create Registry" → row inserted: { tenant_id: "tenant-a", status: "PENDING" }
  Server crashes / restarts
  Worker starts → reads "PENDING" row → continues deployment
  State is durable — survives any number of restarts
```

The database is the **source of truth** for every deployment's state.

---

## 2. Relational Database Basics

A relational database stores data in **tables** (rows and columns), like spreadsheets — but with relationships between them and powerful query language (SQL).

```
Table: registry_deployments
┌───────────┬──────────────────────┬───────────┬──────────────────────────────┐
│ tenant_id │ namespace            │ status    │ created_at                   │
├───────────┼──────────────────────┼───────────┼──────────────────────────────┤
│ tenant-a  │ tenant-a-management  │ READY     │ 2024-01-15 10:00:00          │
│ tenant-b  │ tenant-b-management  │ DEPLOYING │ 2024-01-15 10:05:00          │
│ tenant-c  │ tenant-c-management  │ PENDING   │ 2024-01-15 10:10:00          │
└───────────┴──────────────────────┴───────────┴──────────────────────────────┘

Primary Key: tenant_id (each row is uniquely identified by tenant_id)
```

### Basic SQL Operations

```sql
-- INSERT: add a new row
INSERT INTO registry_deployments (tenant_id, namespace, status, plan)
VALUES ('tenant-d', 'tenant-d-management', 'PENDING', 'professional');

-- SELECT: read rows
SELECT tenant_id, status FROM registry_deployments
WHERE status = 'PENDING';

-- UPDATE: modify a row
UPDATE registry_deployments
SET status = 'READY', ready_at = now()
WHERE tenant_id = 'tenant-d';

-- DELETE: remove a row
DELETE FROM registry_deployments
WHERE tenant_id = 'tenant-d';
```

---

## 3. This System's Three Tables

### Table 1: `registry_deployments`

Tracks the full lifecycle of each Harbor instance:

```sql
CREATE TABLE registry_deployments (
    tenant_id     TEXT PRIMARY KEY,     -- "tenant-a" (unique key)
    namespace     TEXT NOT NULL,        -- "tenant-a-management"
    status        TEXT NOT NULL CHECK (status IN (
                    'PENDING','DEPLOYING','READY','FAILED','DELETING','DELETED'
                  )),
    registry_url  TEXT,                 -- "https://registry.tenant-a.lkdc.wso2.com"
    helm_release  TEXT,                 -- "harbor-tenant-a"
    plan          TEXT,                 -- "starter" / "professional" / "enterprise"
    progress      JSONB DEFAULT '{}',   -- {"namespace":"READY","helm_install":"STARTING"}
    error_message TEXT,                 -- "step helm_install failed: ..."
    worker_lock   TEXT,                 -- hostname of worker processing this row
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    ready_at      TIMESTAMPTZ          -- when it became READY (nullable)
);
```

**State machine enforced by the CHECK constraint:**

```
PENDING ──► DEPLOYING ──► READY
                │
                └──────► FAILED
READY ──► DELETING ──► DELETED
```

The `CHECK (status IN (...))` means PostgreSQL will **reject** any UPDATE that sets status to an invalid value — the DB enforces the state machine.

### Table 2: `registry_credentials`

Stores **encrypted** Harbor credentials. Plaintext passwords never touch this table:

```sql
CREATE TABLE registry_credentials (
    tenant_id          TEXT PRIMARY KEY
                       REFERENCES registry_deployments(tenant_id) ON DELETE CASCADE,
    robot_username     TEXT NOT NULL,        -- "robot$ci-default"
    encrypted_token    BYTEA NOT NULL,       -- AES-256-GCM encrypted robot token
    token_nonce        BYTEA NOT NULL,       -- 12-byte nonce used for encryption
    admin_username     TEXT NOT NULL DEFAULT 'admin',
    encrypted_admin_pw BYTEA NOT NULL,       -- encrypted Harbor admin password
    admin_pw_nonce     BYTEA NOT NULL,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    rotated_at         TIMESTAMPTZ           -- last rotation time
);
```

- `REFERENCES ... ON DELETE CASCADE` — if the deployment row is deleted, credentials are automatically deleted too
- `BYTEA` — binary data type (stores raw bytes, not text)

### Table 3: `audit_log`

An **append-only** record of every action taken. Nothing is ever updated or deleted here:

```sql
CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,      -- auto-incrementing number
    tenant_id   TEXT NOT NULL,
    action      TEXT NOT NULL,             -- "registry.create", "credentials.rotate"
    actor_id    TEXT,                      -- user's ID from JWT
    actor_email TEXT,                      -- user's email from JWT
    source_ip   TEXT,                      -- where the request came from
    result      TEXT NOT NULL,             -- "success" or "failure"
    details     JSONB DEFAULT '{}',        -- additional context
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant_id ON audit_log(tenant_id);
```

---

## 4. JSONB — Storing Flexible Data

`JSONB` is PostgreSQL's binary JSON type. It lets you store flexible structured data inside a column:

```sql
-- The progress column stores a JSON object
UPDATE registry_deployments
SET progress = '{"namespace":"READY","helm_install":"STARTING","postgresql":"PENDING"}'
WHERE tenant_id = 'tenant-a';

-- You can query inside the JSON
SELECT * FROM registry_deployments
WHERE progress->>'namespace' = 'READY';

-- In Go: stored as []byte, marshalled/unmarshalled with encoding/json
```

Why JSONB instead of separate columns? The progress object has a variable number of components that change during deployment — JSONB avoids needing to ALTER TABLE every time we add a new progress step.

---

## 5. The `FOR UPDATE SKIP LOCKED` Pattern ⭐

This is the most important database technique in the project. The provisioner runs **2 replicas** for high availability. Both replicas have a worker goroutine polling for PENDING jobs every 5 seconds.

**Problem without locking:**

```
Time 00: Both workers query: SELECT * FROM ... WHERE status='PENDING'
          Both see: tenant-c is PENDING
Time 01: Worker-1 starts deploying tenant-c
Time 01: Worker-2 ALSO starts deploying tenant-c  ← DISASTER! Two Helm installs!
```

**Solution: `FOR UPDATE SKIP LOCKED`**

```sql
-- This is GetOldestPending() in db/postgres.go
UPDATE registry_deployments
SET worker_lock = $1, status = 'DEPLOYING', updated_at = now()
WHERE tenant_id = (
    SELECT tenant_id FROM registry_deployments
    WHERE status = 'PENDING' AND worker_lock IS NULL
    ORDER BY created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED    -- ← the magic
)
RETURNING tenant_id, namespace, ...
```

How it works:

```
Worker-1 reaches FOR UPDATE SKIP LOCKED:
  → PostgreSQL locks row for tenant-c
  → Returns tenant-c to Worker-1
  → Worker-1 updates: status=DEPLOYING, worker_lock="replica-1"

Worker-2 reaches FOR UPDATE SKIP LOCKED (same millisecond):
  → PostgreSQL sees tenant-c is already locked
  → SKIPS it (doesn't block — moves on)
  → No other PENDING rows → returns nothing
  → Worker-2 does nothing this tick
```

```
Timeline:
  5s: W1 picks tenant-c   W2 gets nothing (locked)
 10s: W1 still deploying  W2 gets nothing
 15s: W1 still deploying  W2 picks tenant-d (new job!)
```

`SKIP LOCKED` is specifically designed for **job queues** — it allows multiple workers to process different jobs simultaneously without blocking each other.

---

## 6. Transactions — All or Nothing

A **transaction** is a group of SQL statements that either ALL succeed or ALL fail together:

```sql
BEGIN;
    UPDATE registry_deployments SET status = 'READY', ready_at = now() WHERE tenant_id = 'tenant-a';
    INSERT INTO audit_log (tenant_id, action, result) VALUES ('tenant-a', 'deploy.complete', 'success');
COMMIT;
-- If COMMIT fails, both changes are rolled back — no partial state
```

This system uses implicit transactions for most operations (each SQL statement is its own transaction). For the SKIP LOCKED query, the entire `UPDATE...WHERE...SELECT...FOR UPDATE SKIP LOCKED` runs atomically.

---

## 7. Connection Pooling

Opening a new database connection is expensive (~50ms). Instead, keep a **pool** of open connections and reuse them:

```go
// From db/postgres.go
db.SetMaxOpenConns(cfg.MaxOpenConns)    // max 25 connections
db.SetMaxIdleConns(cfg.MaxIdleConns)    // keep 5 idle connections ready
db.SetConnMaxLifetime(cfg.ConnMaxLifetime) // recycle after 30 minutes
```

```
Goroutine wants to query DB:
  ├── pool has idle connection → borrow it → run query → return to pool
  └── pool is full (all 25 in use) → WAIT for one to be returned
                                      (bounded queue — no memory explosion)
```

---

## 8. The Go DB Layer

```go
// database/sql is Go's standard DB interface — works with any DB driver
import (
    "database/sql"
    _ "github.com/lib/pq"  // ← PostgreSQL driver (blank import = register driver)
)

// Connect
db, err := sql.Open("postgres", "postgres://user:pass@host:5432/dbname?sslmode=disable")

// Execute (INSERT/UPDATE/DELETE)
_, err = db.ExecContext(ctx,
    `UPDATE registry_deployments SET status = $2 WHERE tenant_id = $1`,
    tenantID, status,
)

// Query single row
row := db.QueryRowContext(ctx,
    `SELECT tenant_id, status FROM registry_deployments WHERE tenant_id = $1`,
    tenantID,
)
var id, status string
row.Scan(&id, &status)   // ← populate variables from the row

// Query multiple rows
rows, _ := db.QueryContext(ctx, `SELECT tenant_id, status FROM registry_deployments`)
defer rows.Close()
for rows.Next() {
    var id, status string
    rows.Scan(&id, &status)
    fmt.Println(id, status)
}
```

**Notice `$1`, `$2`** — these are **parameterised queries**, not string concatenation. This **prevents SQL injection**:

```go
// SAFE (parameterised):
db.QueryRowContext(ctx, `SELECT * FROM users WHERE id = $1`, userID)

// DANGEROUS (never do this!):
db.QueryRowContext(ctx, `SELECT * FROM users WHERE id = ` + userID)
// If userID = "1 OR 1=1" → returns ALL users!
```

---

## 9. Schema Migration

When the app starts, it runs `store.Migrate()` which executes the schema SQL:

```go
func (s *Store) Migrate() error {
    _, err := s.db.Exec(schemaSQL)
    return err
}

const schemaSQL = `
CREATE TABLE IF NOT EXISTS registry_deployments (
    ...
);
CREATE TABLE IF NOT EXISTS registry_credentials (
    ...
);
CREATE TABLE IF NOT EXISTS audit_log (
    ...
);
CREATE INDEX IF NOT EXISTS idx_audit_log_tenant_id ON audit_log(tenant_id);
`
```

`IF NOT EXISTS` makes this **idempotent** — safe to run on every startup. If tables already exist, it does nothing.

---

## 🏋️ Exercises

### Exercise 1 — Connect to the Dev Database
```bash
# Connect to the running PostgreSQL container
docker exec -it implementation-of-auto-harbor-deployment-postgres-1 \
  psql -U provisioner -d registry_provisioner

# In psql:
\dt                           -- list tables
\d registry_deployments       -- describe the table
SELECT * FROM registry_deployments;   -- show all rows (empty initially)
\q                            -- quit
```

### Exercise 2 — Create a Registry Row Manually
```bash
docker exec -it implementation-of-auto-harbor-deployment-postgres-1 \
  psql -U provisioner -d registry_provisioner -c "
    INSERT INTO registry_deployments
      (tenant_id, namespace, status, plan)
    VALUES
      ('test-tenant', 'test-tenant-management', 'PENDING', 'starter');
  "

# Now query the API — does it see the new row?
curl -s http://localhost:8080/api/v1/tenants/test-tenant/registry | python3 -m json.tool
```

### Exercise 3 — Understand SKIP LOCKED
Run this SQL manually and think through what happens:
```sql
-- Open two psql connections simultaneously (two terminals)
-- In connection 1:
BEGIN;
SELECT tenant_id FROM registry_deployments
WHERE status = 'PENDING'
FOR UPDATE SKIP LOCKED;
-- DON'T COMMIT YET

-- In connection 2 (while connection 1 is still open):
SELECT tenant_id FROM registry_deployments
WHERE status = 'PENDING'
FOR UPDATE SKIP LOCKED;
-- What does connection 2 get?

-- Now commit connection 1:
COMMIT;

-- Run connection 2's query again — what happens now?
```

### Exercise 4 — Explore JSONB
```sql
-- After creating a row (exercise 2), update its progress:
UPDATE registry_deployments
SET progress = '{"namespace":"READY","helm_install":"STARTING"}'
WHERE tenant_id = 'test-tenant';

-- Query inside the JSON:
SELECT progress->>'namespace'
FROM registry_deployments
WHERE tenant_id = 'test-tenant';

-- What operator do you use to get a JSON field as text?
```

### Exercise 5 — Understand Cascading Delete
```sql
-- Insert into credentials (will fail if no deployment row)
INSERT INTO registry_credentials
  (tenant_id, robot_username, encrypted_token, token_nonce,
   encrypted_admin_pw, admin_pw_nonce)
VALUES
  ('test-tenant', 'robot$ci-default', 'fake', 'nonce',
   'fake', 'nonce');

-- Now delete the deployment row:
DELETE FROM registry_deployments WHERE tenant_id = 'test-tenant';

-- What happened to the credentials row?
SELECT * FROM registry_credentials WHERE tenant_id = 'test-tenant';
```

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Table** | Rows and columns — structured data storage |
| **Primary Key** | Unique identifier for each row (`tenant_id`) |
| **CHECK constraint** | DB enforces valid values (the status state machine) |
| **REFERENCES + CASCADE** | Foreign key — credentials are deleted with deployments |
| **JSONB** | Binary JSON column — flexible nested data (`progress`) |
| **FOR UPDATE SKIP LOCKED** | Job queue locking — two workers can't take the same job |
| **Transaction** | Group of statements that succeed or fail together |
| **Parameterised query** | `$1`, `$2` placeholders — prevents SQL injection |
| **Connection pool** | Reuse open DB connections — faster queries |
| **Idempotent migration** | `IF NOT EXISTS` — safe to run on every startup |

**Next:** [08 — JWT & Authentication →](./08-jwt-and-authentication.md)
