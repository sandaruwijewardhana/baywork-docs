# Phase 1 — Technical Explanation

---

## Docker Compose Explanation

### Container & Port Mapping Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Your Machine (Host)                       │
│                                                                  │
│   Browser          localhost:3001          localhost:5432        │
│      │                   │                       │              │
└──────┼───────────────────┼───────────────────────┼──────────────┘
       │                   │                       │
       │         ┌─────────▼───────────────────────▼──────────┐
       │         │          Docker Internal Network             │
       │         │                                              │
       │         │  ┌─────────────────────┐                    │
       │         │  │     api container   │                     │
       └─────────►  │   Node.js / Express │                     │
                 │  │      PORT 3001      │                     │
                 │  │                     │                     │
                 │  │  ./console mounted  │                     │
                 │  │  → /app/public      │                     │
                 │  └──────────┬──────────┘                     │
                 │             │ postgres://db:5432              │
                 │             │ (internal hostname "db")        │
                 │  ┌──────────▼──────────┐                     │
                 │  │     db container    │                     │
                 │  │   PostgreSQL 15     │                     │
                 │  │      PORT 5432      │                     │
                 │  │                     │                     │
                 │  │  pgdata volume      │                     │
                 │  │  → /var/lib/        │                     │
                 │  │    postgresql/data  │                     │
                 │  └─────────────────────┘                     │
                 │                                              │
                 └──────────────────────────────────────────────┘

  Named Volume:  pgdata  →  persists DB data across restarts
  Bind Mount:    ./console  →  /app/public  (live, no rebuild)
```

### Startup Order

```
docker compose up
        │
        ▼
  Start db container
        │
        ▼
  Healthcheck: pg_isready -U baywork
  (every 5s, up to 5 retries)
        │
        ▼ (passes)
  Start api container
        │
        ▼
  Express listens on :3001
  Drizzle connects to postgres://db:5432
```

---

### Line-by-Line Breakdown

#### Top Level

| Line | What it does |
|------|-------------|
| `version: '3.9'` | Compose file format version. 3.9 enables `condition: service_healthy` in `depends_on`. |

---

#### `db` service — PostgreSQL

| Line | What it does |
|------|-------------|
| `image: postgres:15-alpine` | Official Postgres 15 on Alpine Linux (~10 MB vs ~300 MB full Debian). Same database, smaller image. |
| `restart: unless-stopped` | Auto-restarts on crash. Does NOT restart if you explicitly run `docker stop`. |
| `POSTGRES_USER: baywork` | Creates a DB user named `baywork` on first boot. |
| `POSTGRES_PASSWORD: baywork123` | Sets that user's password. Change before production. |
| `POSTGRES_DB: baywork` | Creates a database named `baywork` owned by that user. |
| `pgdata:/var/lib/postgresql/data` | Named volume mount. All table data written here survives container restarts and rebuilds. Without this, every `docker compose down` wipes the database. |
| `ports: "5432:5432"` | `host:container` port mapping. Lets you connect from your laptop via TablePlus, DBeaver, or `psql` at `localhost:5432`. |
| `test: pg_isready -U baywork` | Postgres built-in CLI tool. Returns exit 0 only when the DB is fully initialized and accepting connections. |
| `interval: 5s` | Run the healthcheck every 5 seconds. |
| `timeout: 5s` | If the check takes longer than 5s, count it as a failure. |
| `retries: 5` | After 5 consecutive failures, mark the container as `unhealthy`. |

---

#### `api` service — Node.js / Express

| Line | What it does |
|------|-------------|
| `build: ./api` | Build a Docker image from `./api/Dockerfile` instead of pulling one. Runs `npm install` and TypeScript compile. |
| `restart: unless-stopped` | Same auto-restart policy as db. |
| `DATABASE_URL: postgres://baywork:baywork123@db:5432/baywork` | `@db` resolves to the db container's IP on Docker's internal network. No hardcoded IPs needed. |
| `JWT_SECRET` | Signs 15-minute access tokens. Must be a strong random string in production. |
| `JWT_REFRESH_SECRET` | Signs 7-day refresh tokens. Separate from access secret so a leaked access token cannot forge refresh tokens. |
| `PORT: 3001` | Express binds to this port inside the container. |
| `NODE_ENV: development` | Enables verbose error messages, disables some production optimizations. |
| `ports: "3001:3001"` | Exposes the API to your browser at `http://localhost:3001`. |
| `depends_on: db: condition: service_healthy` | API container does not start until Postgres passes its healthcheck. Eliminates the entire class of "DB not ready" startup crashes. |
| `./console:/app/public` | Bind mount — your local `console/` folder is live inside the container. Edit any HTML/CSS/JS file and the change is immediately served. No rebuild required. |

---

#### Named Volumes

| Line | What it does |
|------|-------------|
| `volumes: pgdata:` | Declares `pgdata` as a Docker-managed named volume. Stored in `/var/lib/docker/volumes/` on the host. Survives `docker compose down`. Only deleted with `docker compose down -v` or `docker volume rm`. |

---

### What Happens on `docker compose up`

1. Docker reads the file and creates an internal network for the two services.
2. Pulls `postgres:15-alpine` if not cached locally.
3. Starts the `db` container and begins running healthchecks every 5 seconds.
4. Once `pg_isready` returns success, marks `db` as healthy.
5. Builds the `api` image from `./api/Dockerfile`.
6. Starts the `api` container. Express connects to Postgres using hostname `db`.
7. `./console` is mounted into the container — all HTML pages are immediately served as static files.
8. Browser can now reach the app at `http://localhost:3001`.

---

### Architecture Note

This is a **layered monolith**, not hexagonal architecture:

```
Browser (console/)
    ↓ HTTP
Express API (api/)
    ↓ Drizzle ORM
PostgreSQL (db)
```

Hexagonal (Ports & Adapters) separates business logic from infrastructure via abstract interfaces. We are not doing that in sub-phase 1 — route handlers talk directly to Drizzle. This is intentional: hexagonal adds 2–3x the file count and pays off only when swapping infrastructure or writing unit tests with fake DB adapters. What matters for scale at this stage is already in place:

- **Row-level tenant isolation** — `garageId` on every business table
- **Stateless API** — JWTs mean any number of API containers can run behind a load balancer
- **DB as a separate service** — one env var change points it at AWS RDS

---

## Database Schema — Schema-per-Tenant Architecture

### Overview

The database is split into two layers. Every garage gets its own private PostgreSQL schema — a fully isolated namespace with its own tables, indexes, and enum types. No `garageId` columns anywhere.

```
PostgreSQL
│
├── public schema          ← one set of tables, shared registry
│   ├── garages            ← approved tenants (id, name, schema_name, plan)
│   ├── user_index         ← email → schema_name (login routing)
│   └── pending_registrations
│
├── garage_k8x2m9pq        ← Garage A — completely private
│   ├── users, refresh_tokens
│   ├── customers, vehicles
│   ├── services, service_items
│   ├── invoices, invoice_parts, invoice_labor, invoice_service_lines
│   ├── inventory_items
│   └── activity_log
│
└── garage_r3nw7cfx        ← Garage B — zero shared rows with Garage A
    └── (identical structure)
```

**Three source files manage this:**

| File | Purpose |
|------|---------|
| `publicSchema.ts` | Drizzle definitions for the public schema tables |
| `tenantSchema.ts` | Drizzle definitions for per-garage tables (no garageId) |
| `provisioning.ts` | Raw SQL that creates a new garage schema + all tables + indexes |

`drizzle-kit push` only touches the public schema. Tenant schemas are created by `provisionGarageSchema()` when a registration is approved.

---

### How a request is routed to the right schema

```
POST /api/invoices
Bearer token: { userId: 5, garageId: 3, schemaName: "garage_k8x2m9pq", role: "cashier" }
        │
        ▼
withTenantDb("garage_k8x2m9pq", async (db) => {
  SET search_path TO "garage_k8x2m9pq", public   ← Postgres routes table names here
  return db.select().from(invoices)...            ← queries garage_k8x2m9pq.invoices
})
```

The `withTenantDb()` function in `client.ts` borrows a connection from the pool, runs `SET search_path`, executes the callback, and releases. The same Drizzle query `db.select().from(invoices)` hits a different physical table depending on the schema name in the JWT.

---

### Why `user_index` exists

User emails live inside tenant schemas. When someone tries to log in with just an email and password, the API doesn't know which schema to look in. The `user_index` table in `public` is a global email directory:

```
Login: POST /api/auth/login { email: "ali@speedauto.lk", password: "..." }
  │
  ▼
SELECT schema_name FROM public.user_index WHERE email = 'ali@speedauto.lk'
  → "garage_k8x2m9pq"
  │
  ▼
withTenantDb("garage_k8x2m9pq") → find user → bcrypt.compare → issue JWT
```

`user_index` is kept in sync: when a user is created in a tenant schema, a row is inserted into `user_index`. When deleted, removed from both.

---

---

### Enums

```typescript
export const roleEnum = pgEnum('role', ['root_admin','technician','cashier','receptionist']);
export const serviceStatusEnum = pgEnum('service_status', ['pending','in_progress','ready','completed','cancelled']);
export const invoiceStatusEnum = pgEnum('invoice_status', ['draft','sent','paid','overdue']);
export const paymentMethodEnum = pgEnum('payment_method', ['cash','card','bank_transfer','pending']);
```

| Line | What it does |
|------|-------------|
| `pgEnum('role', [...])` | Creates a Postgres ENUM type — a column that can only hold one of the listed values. Postgres enforces this at the DB level, not just in code. |
| `roleEnum` values | `root_admin` = super admin / garage owner. `technician` = workshop floor. `cashier` = billing. `receptionist` = front desk. |
| `serviceStatusEnum` values | The lifecycle of a job card: `pending` → `in_progress` → `ready` → `completed`. `cancelled` is a terminal state. |
| `invoiceStatusEnum` values | The lifecycle of an invoice: `draft` → `sent` → `paid`. `overdue` is set automatically when due date passes. |
| `paymentMethodEnum` values | How the customer paid. `pending` means not yet collected. |

---

### `users` Table

```typescript
export const users = pgTable('users', {
  id:           serial('id').primaryKey(),
  name:         text('name').notNull(),
  email:        text('email').notNull().unique(),
  passwordHash: text('password_hash').notNull(),
  role:         roleEnum('role').notNull().default('technician'),
  phone:        text('phone'),
  active:       boolean('active').notNull().default(true),
  createdAt:    timestamp('created_at').defaultNow(),
});
```

| Column | What it does |
|--------|-------------|
| `serial('id').primaryKey()` | Auto-incrementing integer ID. Postgres increments it automatically on every insert. |
| `text('email').notNull().unique()` | Email must exist and must be unique across all users — enforced by a DB index. |
| `passwordHash` | Never store plain passwords. `bcrypt.hash(password, 10)` is stored here. The real password is never written to DB. |
| `roleEnum('role').default('technician')` | New staff members default to technician (lowest privilege). Must be manually elevated by admin. |
| `active` | Soft disable: set to `false` instead of deleting the user. Preserves history. |

---

### `refreshTokens` Table

```typescript
export const refreshTokens = pgTable('refresh_tokens', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id),
  token:     text('token').notNull().unique(),
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
});
```

| Column | What it does |
|--------|-------------|
| `references(() => users.id)` | Foreign key — every refresh token is linked to a user. If the user row is deleted, this row can cascade-delete too. |
| `token` | The actual random refresh token string stored here. When a user logs in, a new row is inserted. On logout, this row is deleted. |
| `expiresAt` | 7-day expiry. The API checks this on every refresh request and rejects expired tokens even if the signature is valid. |

Why this table exists: refresh tokens must be revocable. JWTs are stateless — once issued, you can't cancel them. By storing refresh tokens in DB, you can log a user out from all devices by deleting their rows here.

---

### `pendingRegistrations` Table

```typescript
export const pendingRegistrations = pgTable('pending_registrations', {
  ...
  refNumber:    text('ref_number').notNull().unique(),
  approved:     boolean('approved').notNull().default(false),
  ...
});
```

| Column | What it does |
|--------|-------------|
| `refNumber` | Generated on registration (e.g. `BW-2026-4821`). Shown to the garage owner as their tracking number. |
| `approved` | `false` by default. Root admin reviews and sets to `true`, which triggers actual garage + user creation. |
| `passwordHash` | Password is hashed at registration time and stored here, so when approved, the user account is created with the password they chose. |

This table is a holding area. New garages are not live until approved. Prevents anyone from self-registering and accessing the system.

---

### `customers` Table

```typescript
export const customers = pgTable('customers', {
  displayId:      text('display_id').notNull().unique(), // CUS-0001
  phonePrimary:   text('phone_primary').notNull(),
  nic:            text('nic'),
  deletedAt:      timestamp('deleted_at'),
  ...
});
```

| Column | What it does |
|--------|-------------|
| `displayId` | Human-readable ID shown on screen (`CUS-0001`). The `id` (integer) is used internally for joins. Display IDs are shown to users because integer IDs leak record counts. |
| `phonePrimary` | Required. Most Sri Lankan customers are found by phone number, not name. |
| `nic` | National ID card number. Optional but useful for identity verification. |
| `deletedAt` | Soft delete — when a customer is "deleted" from UI, `deletedAt` is set to now. Row stays in DB. All queries filter `WHERE deleted_at IS NULL`. History and invoices are preserved. |

---

### `vehicles` Table

```typescript
export const vehicles = pgTable('vehicles', {
  regNo:           text('reg_no').notNull().unique(),
  ownerCustomerId: integer('owner_customer_id').references(() => customers.id),
  currentMileage:  integer('current_mileage'),
  chassisNo:       text('chassis_no'),
  ...
});
```

| Column | What it does |
|--------|-------------|
| `regNo` | Vehicle registration plate number. Unique — one row per plate. The most common search key in a garage. |
| `ownerCustomerId` | Foreign key to `customers`. A vehicle belongs to a customer. One customer can own many vehicles. |
| `currentMileage` | Updated each time the vehicle comes in. Used to track service intervals. |
| `chassisNo` | VIN / chassis number for identity verification when plate is unclear. |

---

### `services` Table (Job Cards)

```typescript
export const services = pgTable('services', {
  jobNo:            text('job_no').notNull().unique(), // JC-2026-0001
  customerId:       integer('customer_id').references(() => customers.id),
  vehicleId:        integer('vehicle_id').notNull().references(() => vehicles.id),
  assignedToUserId: integer('assigned_to_user_id').references(() => users.id),
  status:           serviceStatusEnum('status').notNull().default('pending'),
  mileageAtDropoff: integer('mileage_at_dropoff'),
  dropoffAt:        timestamp('dropoff_at').defaultNow(),
  estPickupAt:      timestamp('est_pickup_at'),
  priority:         text('priority').default('normal'),
  ...
});
```

| Column | What it does |
|--------|-------------|
| `jobNo` | Unique job card number shown to customers and staff (`JC-2026-0001`). |
| `vehicleId` | Required (notNull). Every service is done on a vehicle. Customer is optional — walk-ins without registered customer. |
| `assignedToUserId` | Which technician is doing this job. Can be null (unassigned). |
| `status` | Current stage of the job. Moves through the kanban board. |
| `mileageAtDropoff` | Recorded when vehicle arrives. Combined with `currentMileage` on vehicles table to track distance since last service. |
| `dropoffAt` | When the vehicle arrived. Defaults to now on creation. |
| `estPickupAt` | Estimated ready time. Shown to customer on the status SMS/screen. |
| `priority` | `normal`, `high`, `urgent`. Affects sort order on kanban board. |

---

### `serviceItems` Table

```typescript
export const serviceItems = pgTable('service_items', {
  serviceId:     integer('service_id').notNull().references(() => services.id),
  description:   text('description').notNull(),
  status:        text('status').notNull().default('not_started'),
  estimatedMins: integer('estimated_mins'),
  orderIndex:    integer('order_index').default(0),
});
```

A service (job card) has multiple line items — e.g. "Oil Change", "Brake Pad Replacement", "Tyre Rotation". Each item has its own status and estimated time. `orderIndex` controls the display order on the job card.

---

### `invoices` Table

```typescript
export const invoices = pgTable('invoices', {
  invoiceNo:      text('invoice_no').notNull().unique(), // INV-2026-0001
  discountAmount: decimal('discount_amount', { precision: 10, scale: 2 }).default('0'),
  grandTotal:     decimal('grand_total', { precision: 10, scale: 2 }).default('0'),
  paymentMethod:  paymentMethodEnum('payment_method').default('pending'),
  dueDate:        timestamp('due_date'),
  deletedAt:      timestamp('deleted_at'),
  ...
});
```

| Column | What it does |
|--------|-------------|
| `decimal(..., precision: 10, scale: 2)` | Stores money with exactly 2 decimal places. `precision: 10` = up to 10 digits total. Never use `float` for money — floating point arithmetic causes rounding errors (e.g. `0.1 + 0.2 = 0.30000000000000004`). |
| `discountAmount` | Total discount in rupees. The cashier discount limit rule (≤10% without approval) is enforced in application logic, not DB. |
| `grandTotal` | Stored as a computed field — sum of all parts + labor + service lines minus discount. Stored here so historical invoices don't change if prices change. |
| `dueDate` | Payment deadline. A background job will flip status from `sent` to `overdue` when this passes. |

---

### Invoice Line Tables (`invoiceParts`, `invoiceLabor`, `invoiceServiceLines`)

An invoice is broken into three types of charges:

```
Invoice
  ├── invoiceParts        ← spare parts used (linked to inventory)
  ├── invoiceLabor        ← mechanic time (hours × rate)
  └── invoiceServiceLines ← flat charges (inspection fee, service charge)
```

| Table | Example row |
|-------|-------------|
| `invoiceParts` | "Engine Oil 5W30 × 4L @ LKR 850 = LKR 3,400" |
| `invoiceLabor` | "Engine overhaul — 6 hrs @ LKR 1,500/hr = LKR 9,000" |
| `invoiceServiceLines` | "AC Service — LKR 2,500" |

Splitting into three tables makes it easy to subtotal by category and to link parts back to inventory for stock deduction.

---

### `inventoryItems` Table

```typescript
export const inventoryItems = pgTable('inventory_items', {
  displayId:      text('display_id').notNull().unique(), // ITM-0001
  qtyInStock:     integer('qty_in_stock').notNull().default(0),
  alertThreshold: integer('alert_threshold').notNull().default(5),
  costPrice:      decimal('cost_price', ...),
  sellingPrice:   decimal('selling_price', ...),
  ...
});
```

| Column | What it does |
|--------|-------------|
| `qtyInStock` | Current stock count. Decremented automatically when a part is added to an invoice. |
| `alertThreshold` | When `qtyInStock` drops to or below this number, a low-stock alert fires. Default is 5 units. |
| `costPrice` | What the garage paid. |
| `sellingPrice` | What the customer is charged. The margin is `sellingPrice - costPrice`. |

---

### `activityLog` Table

```typescript
export const activityLog = pgTable('activity_log', {
  userId:     integer('user_id'),
  action:     text('action').notNull(),
  entityType: text('entity_type'),
  entityId:   integer('entity_id'),
  metadata:   jsonb('metadata'),
  createdAt:  timestamp('created_at').defaultNow(),
});
```

| Column | What it does |
|--------|-------------|
| `action` | What happened: `'invoice.created'`, `'service.status_changed'`, `'user.login'`. |
| `entityType` + `entityId` | Which record was affected: `entityType='invoice'`, `entityId=42`. |
| `metadata` | `jsonb` (JSON blob) stores the before/after diff or extra context. Flexible — no fixed columns needed. |

Every write operation in the API inserts a row here. This is the audit trail — who did what, when, to which record.

---

## Database Client (`api/src/db/client.ts`)

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { schema });
```

| Line | What it does |
|------|-------------|
| `import { drizzle }` | Drizzle ORM — the type-safe query builder that sits on top of raw PostgreSQL. |
| `import { Pool } from 'pg'` | The `pg` library is the raw PostgreSQL driver. `Pool` manages a group of DB connections — up to 10 by default — so the API doesn't open a new connection on every request. |
| `new Pool({ connectionString: process.env.DATABASE_URL })` | Reads the DB URL from the environment variable set in docker-compose (`postgres://baywork:baywork123@db:5432/baywork`). Never hardcode credentials in source code. |
| `drizzle(pool, { schema })` | Wraps the pool with Drizzle. Passing `{ schema }` enables relational queries (`.with()`, `.findMany()`). |
| `export const db` | Single exported instance. Every route file imports this same `db` object and uses it. One pool, shared across all requests. |

---

## Express Entry Point (`api/src/index.ts`)

```typescript
import express from 'express';
import cookieParser from 'cookie-parser';
import cors from 'cors';
import helmet from 'helmet';
import { authRouter } from './routes/auth';
```

All imports. Each is a middleware package:

| Import | What it does |
|--------|-------------|
| `express` | The web framework. Handles HTTP routing, request parsing, response sending. |
| `cookie-parser` | Parses cookies from incoming requests into `req.cookies`. Needed to read the `refreshToken` HttpOnly cookie. |
| `cors` | Cross-Origin Resource Sharing — controls which domains can call this API. Without it, a browser on `app.baywork.lk` would be blocked from calling `api.baywork.lk`. |
| `helmet` | Sets security HTTP headers automatically: `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`, etc. One line replaces ~15 manual header settings. |
| `authRouter` | The auth routes (`/api/auth/login`, `/api/auth/register`, etc.) defined in a separate file and plugged in here. |

---

```typescript
const app = express();
app.use(helmet());
app.use(cors({ origin: true, credentials: true }));
app.use(express.json());
app.use(cookieParser());
app.use(express.static('public'));
```

| Line | What it does |
|------|-------------|
| `const app = express()` | Creates the Express application instance. Everything is attached to this object. |
| `app.use(helmet())` | Applies all security headers to every response. Must come first in middleware chain. |
| `app.use(cors({ origin: true, credentials: true }))` | Allows any origin (`origin: true` = mirror the request's Origin header). `credentials: true` is required to allow cookies (the refresh token) to be sent cross-origin. In production, `origin` should be a whitelist of allowed domains. |
| `app.use(express.json())` | Parses incoming request bodies as JSON and populates `req.body`. Without this, `req.body` is `undefined`. |
| `app.use(cookieParser())` | Parses cookies and populates `req.cookies`. Required for reading `req.cookies.refreshToken`. |
| `app.use(express.static('public'))` | Serves all files in the `public/` folder (which is the `console/` mount from Docker) as static files. `GET /dashboard.html` serves the file directly. |

---

```typescript
app.use('/api/auth', authRouter);
app.get('/api/health', (_, res) => res.json({ status: 'ok' }));
```

| Line | What it does |
|------|-------------|
| `app.use('/api/auth', authRouter)` | Mounts the auth router at `/api/auth`. All routes defined in `authRouter` are prefixed with `/api/auth` — so `router.post('/login')` becomes `POST /api/auth/login`. |
| `app.get('/api/health', ...)` | Health check endpoint. Docker's healthcheck and AWS ALB call this to verify the API is running. Returns `{ status: 'ok' }`. The `_` is the request object — named `_` to signal it is intentionally unused. |

---

```typescript
app.listen(process.env.PORT || 3001, () =>
  console.log(`API running on :${process.env.PORT || 3001}`)
);
```

| Part | What it does |
|------|-------------|
| `app.listen(...)` | Starts the HTTP server and binds it to a port. After this line, the API accepts incoming connections. |
| `process.env.PORT \|\| 3001` | Reads the PORT env var. Falls back to 3001 if not set. In Docker, `PORT=3001` is set via docker-compose environment. |
| `() => console.log(...)` | Callback that fires once the server is ready. Logs confirmation to the Docker container's stdout, visible in `docker compose logs`. |

---

### How These Three Files Work Together

```
Request: POST /api/auth/login
         { email: "...", password: "..." }

index.ts
  │
  ├── helmet() adds security headers
  ├── cors() checks origin
  ├── express.json() parses body into req.body
  ├── cookieParser() parses cookies into req.cookies
  │
  └── app.use('/api/auth', authRouter)
            │
            └── routes/auth.ts handles POST /login
                      │
                      └── db (from client.ts)
                            │
                            └── schema.ts defines what "users" table looks like
                                  │
                                  └── PostgreSQL
```

`schema.ts` defines the shape. `client.ts` connects to the database. `index.ts` wires the HTTP server together. Every request flows through all three.

---

## Sub-Phase 1 — File Review Checklist

Goal of sub-phase 1: **Docker running, API answering, database connected.**
Tick each file after you have read and understood it.

### Root
```
[ ]  docker-compose.yml          Orchestrates db + api containers
[ ]  .gitignore                  Ignore node_modules, .env, _archive
```

### API config (`api/`)
```
[ ]  api/package.json            Dependency list + npm scripts
[ ]  api/tsconfig.json           TypeScript compiler settings
[ ]  api/Dockerfile              Recipe to build the api container image
[ ]  api/.env                    DATABASE_URL, JWT secrets, PORT
[ ]  api/drizzle.config.ts       Tells drizzle-kit where the schema is
```

### Database layer (`api/src/db/`)
```
[ ]  publicSchema.ts             Shared tables: garages, user_index, pending_registrations
[ ]  tenantSchema.ts             Per-garage tables: users, customers, vehicles, services, invoices...
[ ]  provisioning.ts             Creates a new garage schema (CREATE SCHEMA + tables) on approval
[ ]  client.ts                   publicDb + withTenantDb() — how the app talks to Postgres
[ ]  schema.ts                   Re-export of public + tenant schema (convenience)
```

### App entry (`api/src/`)
```
[ ]  index.ts                    Express app boot — middleware, routes, health check, listen
```

### Middleware (`api/src/middleware/`)
```
[ ]  auth.ts                     requireAuth — verifies JWT, attaches req.auth
[ ]  roles.ts                    requireRole — checks user role
```

### Suggested observation order
Each file builds on the previous one:

1. `docker-compose.yml`  → how the two containers connect
2. `Dockerfile`          → how the api image is built
3. `package.json`        → what libraries are installed
4. `publicSchema.ts`     → the shared tables
5. `tenantSchema.ts`     → the per-garage tables
6. `provisioning.ts`     → how a garage schema gets created
7. `client.ts`           → how queries reach the right schema
8. `index.ts`            → how a request flows in
9. `auth.ts` + `roles.ts` → how requests are guarded

### Not part of Sub-Phase 1 (these belong to Sub-Phase 2)
```
     api/src/routes/auth.ts      login / register / refresh / logout / me
     api/src/routes/admin.ts     approve / reject registrations
     api/src/db/seed.ts          demo garage + owner login
     console/login.html          wired to API.login()
     console/register.html       wired to register endpoint
     console/js/api.js           frontend API client
```

> Note: The original beta-launch plan listed sub-phase 1 as just `schema.ts`, `client.ts`,
> `index.ts`, and `docker-compose.yml`. The extra db files (`publicSchema.ts`, `tenantSchema.ts`,
> `provisioning.ts`) came from the **schema-per-tenant refactor** decided later — they replaced
> the single `schema.ts`. They are foundation files, so they belong in this phase 1 review.
