# BayWork — System Architecture
### Recommended Architecture for 100% Availability, Low Cost, and High Performance
**v1.0 · May 2026 · Tech-stack section updated June 2026**

---

## 0. Full Tech Stack (source of truth)

> **Read this first.** Sections 1–13 below describe the *aspirational target architecture* (React, hexagonal,
> full AWS). This section records the **actual tech stack as built and as currently planned**, and flags where
> it diverges from that target. Where they conflict, **this section wins.**

### 0.1 Current stack — what is actually built and running

| Layer | Technology | Notes |
|-------|-----------|-------|
| **Frontend** | **Vanilla HTML5 + CSS3 + JavaScript** (ES modules) | No framework, **no build step**; served as static files (`console/`) |
| Frontend styling | CSS custom properties (design tokens in `brand.css`), responsive | No Tailwind/Sass |
| Frontend API client | Hand-rolled `fetch` wrapper (`console/js/api.js`) with auto token-refresh | |
| **Runtime** | **Node.js 20** | |
| **Language** | **TypeScript 5.4** | `ts-node` in dev, `tsc` build |
| **Web framework** | **Express 5** | |
| **ORM** | **Drizzle ORM** + `drizzle-kit` | migrations, studio, push |
| **DB driver** | **node-postgres (`pg`)** | connection pool |
| **Database** | **PostgreSQL 15** | |
| **Multi-tenancy** | **Schema-per-tenant** (`garage_xxxxxxxx` schemas + shared `public` registry) | isolation via `SET search_path` |
| **Auth** | **JWT** access token + **HttpOnly refresh cookie**, `bcryptjs` hashing | refresh-token rotation, RBAC |
| Security middleware | `helmet`, `cors`, `cookie-parser`, `express-rate-limit` | |
| Config | `dotenv` | |
| **Containerization** | **Docker + Docker Compose** (`db` + `api` services) | |
| **App architecture** | **Layered monolith** (routes → Drizzle → Postgres) | *not* hexagonal (see §0.3) |
| **Version control** | **Git** — two repos: code (root) + docs (`docs/`) | no Claude attribution on commits |

### 0.2 Planned / target stack — adopt as growth justifies cost

| Layer | Technology | When |
|-------|-----------|------|
| Hosting (lean, now) | Supabase / Railway / Cloudflare (~$45–80/mo) | Year 1–2 |
| Hosting (scale) | **AWS**: ECS Fargate, Aurora Serverless v2, RDS Proxy, ALB, CloudFront, S3, SES | Year 3+ |
| Real-time | **SSE** (Server-Sent Events); Redis (ElastiCache) Pub/Sub for multi-instance | when live dashboard is needed |
| CI/CD | **GitHub Actions** (build → test → deploy) | at first real customers |
| Infrastructure as Code | Terraform or AWS CDK | at AWS migration |
| Monitoring | CloudWatch + **Sentry** | at launch |
| PDF generation | client `window.print()` now; server-side (Puppeteer) later | |
| Email / SMS | AWS SES / transactional email; local SMS gateway | as features ship |

### 0.3 Divergences from the original v1 target (deliberate)

| Item | v1 target (§4, §11 below) | Actual decision |
|------|---------------------------|-----------------|
| **Frontend framework** | React 18 + Tailwind + Zustand + TanStack Query | **Vanilla HTML/CSS/JS — and staying vanilla** (CRUD app doesn't need a SPA framework) |
| **App architecture** | Modular Hexagonal (ports & adapters) | **Layered Express monolith** (simpler; refactor later only if needed) |
| **Hosting** | Full AWS from day one | **Lean stack first**, migrate to AWS pieces from Year 3 as revenue covers cost |
| **Multi-tenancy** | (some sections imply `garageId` rows) | **Schema-per-tenant** — the build is correct here |

*Rationale: keep the stack boring, inspectable, and cheap while pre-revenue; add framework/cloud complexity only
when scale or team size demands it.*

---

## 1. Decision Summary

| Goal | Requirement | Architecture Answer |
|------|------------|-------------------|
| Availability | 100% (zero downtime) | Active-active multi-AZ, auto-failover |
| Cost | Low cloud hosting spend | Serverless containers + serverless DB (scale to zero) |
| Speed | Fast UI + API | Edge CDN + DB connection pool + indexed PostgreSQL |
| Maintainability | Solo developer → small team | Modular Hexagonal Monolith (not microservices) |
| Future-proof | Phase 1 → 3, potential extraction | Domain boundaries ready to extract if needed |

**Chosen pattern: Modular Hexagonal Architecture (Ports and Adapters) deployed as a single containerized service on AWS.**

---

## 2. Why This Pattern, Not Others

### Why NOT Microservices?

Microservices would cost **5–10× more** and require a team to operate. For ~100–500 garages in Phase 1:

| | Microservices | Chosen (Modular Monolith) |
|--|--------------|--------------------------|
| AWS cost/month | $200–400+ (10+ services, each needs ECS task + ALB rule + RDS connection) | $50–80 |
| Deployment complexity | Service mesh, distributed tracing, eventual consistency | Single container + one DB |
| Developer count needed | 3–5 engineers | 1 developer |
| Network latency | Added on every inter-service call | Zero (in-process) |
| Cold start / boot time | Each service boots independently | Single process |

Microservices make sense when: you have multiple teams, you need to scale individual domains independently at very different rates, or when the monolith is already too large to work in. None of those apply here in Phase 1.

### Why NOT a Simple Layered Monolith?

A flat `routes → controllers → services → db` layered monolith works but makes testing hard (business logic is coupled to Express and to SQL), and domain boundaries are invisible — leading to spaghetti as the codebase grows. Hexagonal architecture costs almost nothing extra to implement but prevents that rot.

### Why NOT Service-Oriented Architecture (SOA)?

SOA is essentially microservices with a shared message bus — even more complexity, similar cost. It is designed for large enterprises with legacy systems. Overkill here.

### Why Hexagonal (Ports and Adapters)?

- Business logic is **framework-agnostic and DB-agnostic** — tested without hitting real databases
- Each domain (Customer, Vehicle, Service, Invoice, Inventory, Auth) has a clear boundary
- Adding a new "delivery mechanism" (REST API, CLI, background job) does not change business logic
- When the time comes to extract a domain to its own service, the boundary is already drawn

---

## 3. Full Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        INTERNET                             │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              AWS CloudFront (Global Edge CDN)               │
│                                                             │
│   /api/* ──────────────────────────────────► ALB            │
│   /*  (React SPA) ──────────────────────► S3 Bucket         │
└──────────────────────────┬──────────────────────────────────┘
                           │ /api/*
                           ▼
┌─────────────────────────────────────────────────────────────┐
│         AWS Application Load Balancer (ALB)                 │
│         Health checks every 30s                             │
└───────────────┬───────────────────────────┬─────────────────┘
                │                           │
     ┌──────────▼──────────┐   ┌────────────▼────────────┐
     │  ECS Fargate Task 1 │   │  ECS Fargate Task 2     │
     │  AZ: ap-south-1a   │   │  AZ: ap-south-1b        │
     │  Node.js API        │   │  Node.js API            │
     │  0.5 vCPU / 1GB    │   │  0.5 vCPU / 1GB         │
     └──────────┬──────────┘   └────────────┬────────────┘
                └──────────┬────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
  ┌───────────────┐  ┌──────────┐  ┌───────────────┐
  │ Aurora        │  │   S3     │  │  AWS SES      │
  │ Serverless v2 │  │  Files   │  │  Emails       │
  │ PostgreSQL    │  │  PDFs    │  └───────────────┘
  │               │  │  Logos   │
  │ Primary  (1a) │  └──────────┘
  │ Replica  (1b) │
  │ Auto-failover │
  └───────────────┘
          │
  ┌───────────────┐
  │ Secrets Mgr   │
  │ CloudWatch    │
  │ ECR (images)  │
  └───────────────┘
```

### Availability Guarantee

| Component | AWS SLA | Failure Response |
|-----------|---------|-----------------|
| CloudFront | 99.99% | Edge serves cached assets; API via backup path |
| ALB | 99.99% | Routes only to healthy ECS tasks |
| ECS Fargate (2 tasks, 2 AZs) | 99.95% | If AZ-1 dies, AZ-2 continues; ALB routes all traffic there |
| Aurora Serverless v2 (Multi-AZ) | 99.99% | Automatic failover to replica in ~30s |
| S3 | 99.999999999% durability | N/A |
| **Composite** | **~99.9%** | **< 9 hours downtime per year** |

> True 100% availability (zero downtime ever) is mathematically impossible on any cloud or on-premise system. "100% availability" in practice means 99.9%–99.99% with zero-downtime deployments and automatic failover — which this architecture achieves.

---

## 4. Application Architecture — Hexagonal (Clean) Design

```
┌─────────────────────────────────────────────────────────────┐
│                   DELIVERY LAYER                            │
│  Express HTTP routes · SSE endpoint · Cron jobs             │
│  (thin — no business logic here)                            │
└───────────────────────────┬─────────────────────────────────┘
                            │ calls
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                APPLICATION LAYER (Use Cases)                │
│                                                             │
│  CreateService · UpdateServiceStatus · GenerateInvoicePDF   │
│  AddInventoryItem · RestockItem · AuthenticateUser          │
│                                                             │
│  • Orchestrates domain objects                              │
│  • Calls ports (interfaces), never adapters directly        │
│  • Contains transaction boundaries                          │
└──────────────┬───────────────────────────┬──────────────────┘
               │                           │
               ▼                           ▼
┌──────────────────────────┐  ┌────────────────────────────┐
│    DOMAIN LAYER          │  │  PORTS (Interfaces)        │
│                          │  │                            │
│  ● Customer              │  │  ICustomerRepository       │
│  ● Vehicle               │  │  IVehicleRepository        │
│  ● Service + JobCard     │  │  IServiceRepository        │
│  ● Invoice               │  │  IInvoiceRepository        │
│  ● InventoryItem         │  │  IInventoryRepository      │
│  ● StockTransaction      │  │  IEmailPort                │
│  ● Staff / User          │  │  IFileStoragePort          │
│  ● Notification          │  │  IEventBusPort (SSE)       │
│                          │  │                            │
│  Pure business logic     │  │  (implemented by adapters) │
│  No framework imports    │  └────────────────────────────┘
└──────────────────────────┘               │
                                           ▼
                        ┌──────────────────────────────────┐
                        │    INFRASTRUCTURE ADAPTERS       │
                        │                                  │
                        │  DrizzleCustomerRepository       │
                        │  DrizzleVehicleRepository        │
                        │  S3FileStorageAdapter            │
                        │  SESEmailAdapter                 │
                        │  SSEEventBusAdapter              │
                        │  PgConnectionPool                │
                        └──────────────────────────────────┘
```

### Domain Modules (each is a self-contained folder)

```
src/
├── domains/
│   ├── auth/
│   │   ├── auth.entity.ts         ← User, Session, RefreshToken
│   │   ├── auth.use-cases.ts      ← Login, Logout, RefreshToken, ResetPassword
│   │   ├── auth.port.ts           ← IUserRepository interface
│   │   └── auth.adapter.ts        ← Drizzle implementation
│   ├── customers/
│   │   ├── customer.entity.ts
│   │   ├── customer.use-cases.ts
│   │   ├── customer.port.ts
│   │   └── customer.adapter.ts
│   ├── vehicles/
│   ├── services/                  ← "service" = job card
│   ├── invoices/
│   ├── inventory/
│   ├── staff/
│   └── notifications/
│
├── delivery/
│   ├── http/
│   │   ├── routes/                ← Express route handlers (thin)
│   │   └── middleware/            ← JWT guard, role guard, rate limit
│   └── sse/
│       └── sse.handler.ts         ← Live dashboard event stream
│
├── infrastructure/
│   ├── database/
│   │   ├── schema.ts              ← Drizzle schema definitions
│   │   └── pool.ts                ← PgBouncer/pg connection pool
│   ├── storage/
│   │   └── s3.adapter.ts
│   ├── email/
│   │   └── ses.adapter.ts
│   └── events/
│       └── sse-bus.ts
│
└── shared/
    ├── errors/                    ← DomainError, NotFoundError, etc.
    ├── types/                     ← Shared TypeScript types
    └── utils/                     ← ID generation, pagination, etc.
```

---

## 5. Data Architecture

### Multi-Tenancy: Schema-per-Tenant

```
PostgreSQL (Aurora Serverless v2)
│
├── public schema                        ← managed by drizzle-kit push
│   ├── garages              (id, name, owner_name, email, phone, plan, schema_name, active)
│   ├── user_index           (id, email, schema_name, garage_id)   ← login routing table
│   └── pending_registrations (id, garage_name, owner_name, email, password_hash, ref_number, approved)
│
├── garage_k8x2m9pq schema   ← Tenant A  (created by provisioning.ts on approval)
│   ├── users
│   ├── refresh_tokens
│   ├── customers
│   ├── vehicles
│   ├── services
│   ├── service_items
│   ├── invoices
│   ├── invoice_parts
│   ├── invoice_labor
│   ├── invoice_service_lines
│   ├── inventory_items
│   └── activity_log
│
└── garage_r3nw7cfx schema   ← Tenant B  (same structure, fully isolated)
    └── (identical tables, zero shared rows)
```

**Login routing via `user_index`:**
Because user emails live inside tenant schemas, a login request can't know which schema to query without a global lookup. The `user_index` table in `public` maps `email → schema_name`. On login the API reads `user_index` first, then routes to the correct tenant schema for password verification.

**Why schema-per-tenant over row-level isolation?**

| | Schema-per-tenant | Row-level (tenant_id column) |
|--|------------------|-----------------------------|
| Data isolation | Complete — schemas are separate namespaces | Requires `WHERE tenant_id = ?` on every query |
| Accidental cross-tenant data leak | Impossible | Possible if a query forgets the WHERE clause |
| Backup per tenant | `pg_dump -n garage_abc123` | Complex — must filter rows |
| Migration | Run migration per schema | Single migration, simpler |
| Performance at scale | Each schema has its own indexes, no row competition | Shared indexes grow large |

### Key Database Indexes

```sql
-- Per-tenant schema (created for each garage on activation)

CREATE INDEX idx_customers_phone      ON customers(phone_primary);
CREATE INDEX idx_customers_nic        ON customers(nic);
CREATE INDEX idx_customers_name_fts   ON customers USING GIN (to_tsvector('english', name));

CREATE INDEX idx_vehicles_reg_no      ON vehicles(reg_no);
CREATE INDEX idx_vehicles_owner       ON vehicles(owner_customer_id);

CREATE INDEX idx_services_status      ON services(status);
CREATE INDEX idx_services_date        ON services(dropoff_at DESC);
CREATE INDEX idx_services_technician  ON services(assigned_to_user_id);

CREATE INDEX idx_invoices_status      ON invoices(status);
CREATE INDEX idx_invoices_customer    ON invoices(customer_id);
CREATE INDEX idx_invoices_due         ON invoices(due_date) WHERE status != 'paid';

CREATE INDEX idx_inventory_low_stock  ON inventory_items(qty_in_stock)
  WHERE qty_in_stock <= alert_threshold;

CREATE INDEX idx_activity_log_entity  ON activity_log(entity_type, entity_id);
```

### Connection Pooling

Aurora Serverless v2 has a max connection limit (approximately 90 for the minimum ACU). With 2 ECS tasks, each running multiple concurrent requests, we need connection pooling:

```
ECS Task 1 (Node.js) ──►  PgBouncer sidecar  ──►┐
                           (pool: 10 conns)        │
                                                    ├──► Aurora Serverless v2
ECS Task 2 (Node.js) ──►  PgBouncer sidecar  ──►┘
                           (pool: 10 conns)
```

Or use **RDS Proxy** (AWS managed, $0.015/vCPU-hr, ~$10/month):

```
ECS Tasks ──► RDS Proxy ──► Aurora Serverless v2
(multiplexes connections, handles failover transparently)
```

RDS Proxy is recommended because it also handles Aurora failover transparently — tasks never need to handle reconnection logic.

---

## 6. Security Architecture

```
Request Flow with Security Layers:

Browser
  │
  ▼
CloudFront (HTTPS enforced, HTTP redirects to HTTPS)
  │         (WAF rules: rate limiting, SQL injection protection)
  ▼
ALB (TLS termination)
  │
  ▼
Express Middleware Stack:
  ├── helmet()              ← Security headers (CSP, HSTS, etc.)
  ├── cors()                ← Strict origin whitelist
  ├── express-rate-limit()  ← 100 req/15min (general), 5 req/15min (auth)
  ├── verifyJWT()           ← Access token validation (in-memory only)
  ├── resolveGarageTenant() ← Sets DB schema from JWT claim
  ├── requireRole([...])    ← Role-based access check
  └── Route Handler
```

### Token Architecture

```
Login Response:
  ├── access_token (JWT)
  │   ├── Expiry: 15 minutes
  │   ├── Stored: in-memory (JS variable, never localStorage)
  │   └── Payload: { userId, garageId, role, schemaName }
  │
  └── refresh_token (HttpOnly cookie)
      ├── Expiry: 7 days (30 days if "remember me")
      ├── Flags: HttpOnly, Secure, SameSite=Strict
      └── Rotation: each /auth/refresh call issues a new refresh token
                    old one is immediately invalidated
                    reuse detection: if stolen token is used → entire family revoked
```

### Data Security

| Layer | Mechanism |
|-------|-----------|
| Transport | TLS 1.2+ enforced by CloudFront + ALB |
| At rest (DB) | Aurora encryption with AWS KMS key |
| At rest (S3) | SSE-S3 (server-side encryption) |
| Passwords | bcrypt, cost factor 12 |
| Secrets | AWS Secrets Manager (DB creds, SES creds) — never in `.env` files |
| Audit | Every mutation logged to `activity_log` with user + diff |
| Soft delete | Records flagged `deleted_at`, never hard-deleted |

---

## 7. Real-Time Architecture (SSE)

Server-Sent Events (SSE) is used instead of WebSockets for the live dashboard and service status updates. SSE is simpler, uses standard HTTP, and is sufficient for one-directional server → client push.

```
ECS Task 1                          ECS Task 2
  │                                    │
  │  Technician updates service        │
  │  status via REST API               │
  ▼                                    │
POST /services/:id/status              │
  │                                    │
  ├── DB write (Aurora)                │
  │                                    │
  └── In-process event bus ──► SSE clients connected to THIS task
                                       │
                               Problem: clients connected to Task 2
                               don't receive the event
```

**Solution: Redis Pub/Sub as event relay**

```
ECS Task 1                          ECS Task 2
  │                                    │
  │  PUBLISH to Redis channel          │  SUBSCRIBE to Redis channel
  │  "garage:abc123:events"            │  "garage:abc123:events"
  ▼                                    ▼
Redis (ElastiCache Serverless)  ──► Task 2 receives event ──► SSE push to clients
```

**Cost of Redis ElastiCache Serverless:** ~$0.0065/GB-hr data + $0.0034/ECU-hr compute. For very low traffic: approximately **$3–8/month.**

Alternatively, for Phase 1 with minimal concurrent users, use **sticky sessions on ALB** (all connections from one garage always go to the same task) — free, but less elegant. Switch to Redis when you have enough users for multiple tasks to matter.

---

## 8. CI/CD Architecture

```
Developer pushes to GitHub
      │
      ▼
GitHub Actions Workflow
      │
      ├── 1. Build & Test
      │   ├── npm run build
      │   ├── npm run test (Jest, >80% coverage required)
      │   └── npm run lint
      │
      ├── 2. Docker Build
      │   └── docker build → push to AWS ECR
      │
      ├── 3. Deploy (Blue/Green via ECS)
      │   ├── Register new ECS Task Definition with new image
      │   ├── Update ECS Service (rolling deployment, 0 downtime)
      │   └── ALB health checks confirm tasks are healthy before routing
      │
      └── 4. Post-deploy
          ├── Run DB migrations (Drizzle migrate)
          └── Notify Slack/email on success or failure
```

**Zero-downtime deployments:** ECS rolling updates keep old tasks running until new ones pass health checks. At no point is the service unavailable.

---

## 9. Cost Analysis

### Phase 1 (0–100 garages, low traffic)

| Service | Config | $/month (est.) |
|---------|--------|---------------|
| ECS Fargate | 2 tasks × 0.5 vCPU × 1GB RAM | $12–18 |
| Aurora Serverless v2 | 0.5–2 ACU, Multi-AZ | $20–35 |
| RDS Proxy | 0.5 vCPU equivalent | $8–12 |
| ALB | 1 load balancer, ~5 LCU/hr | $18–22 |
| S3 | 10GB storage + 50GB transfer | $2–4 |
| CloudFront | 50GB out, 1M requests | $4–8 |
| ECR | 2GB image storage | $0.20 |
| SES | 10,000 emails/month | $1 |
| Secrets Manager | 5 secrets | $0.25 |
| CloudWatch + Sentry | Logs + errors | $3–5 |
| **Total** | | **~$68–105/month** |

At LKR 2,999/month/garage × 23 garages = LKR 68,977 (~$230 USD) — this covers infrastructure with profit margin.

### Phase 2 (100–500 garages)

Add auto-scaling: ECS tasks scale 2→8 based on CPU. Aurora scales 0.5→16 ACU automatically. Estimated cost: **$150–300/month**, covered by revenue from ~50+ garages.

### Phase 3+ (500+ garages, Multi-Location Enterprise)

At this point, consider extracting the most load-bearing domains (Services, Invoices) into separate microservices. The hexagonal architecture makes this straightforward — the domain boundary is already defined.

---

## 10. Folder Structure (Full Project)

```
baywork/
├── docs/
│   ├── architecture.md          ← this file
│   ├── design-system.md         ← brand + design tokens
│   └── project-overview.md      ← master index
│
├── Phase 1/
│   ├── requirement-analysis.md
│   ├── phase-1-detailed.md
│   └── landing-page.md
│
├── website/                     ← Phase 1 static HTML prototype
│   ├── index.html
│   ├── login.html
│   ├── register.html
│   ├── dashboard.html
│   ├── styles/brand.css
│   └── scripts/
│
├── apps/
│   ├── web/                     ← React + TypeScript frontend
│   │   ├── src/
│   │   │   ├── pages/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   └── store/
│   │   ├── public/
│   │   └── package.json
│   │
│   └── api/                     ← Node.js + Express backend
│       ├── src/
│       │   ├── domains/         ← hex architecture domains
│       │   ├── delivery/        ← HTTP routes, SSE
│       │   ├── infrastructure/  ← DB, S3, SES adapters
│       │   └── shared/
│       ├── drizzle/
│       │   ├── schema.ts
│       │   └── migrations/
│       └── package.json
│
├── infra/                       ← Infrastructure as Code
│   ├── terraform/               ← or AWS CDK
│   │   ├── ecs.tf
│   │   ├── aurora.tf
│   │   ├── cloudfront.tf
│   │   └── iam.tf
│   └── docker/
│       └── Dockerfile
│
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## 11. Technology Decisions Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | React 18 + TypeScript | Component reuse, strong typing, rich ecosystem |
| State | Zustand + TanStack Query | Light global state; server state managed by RQ (caching, refetch) |
| Forms | React Hook Form + Zod | Schema validation shared with backend |
| Styling | Tailwind CSS v3 | Rapid UI with design token integration |
| PDF | @react-pdf/renderer | Job cards and invoices rendered in-browser |
| Backend | Node.js 20 + Express 5 | Single JS ecosystem, async by default |
| ORM | Drizzle ORM | Type-safe, close to SQL, fast, no runtime magic |
| Database | PostgreSQL 15 (Aurora Serverless v2) | ACID compliance, schema-per-tenant, full-text search |
| Auth | JWT (access) + HttpOnly cookie (refresh) | Industry standard, stateless, secure |
| Real-time | SSE + Redis Pub/Sub | Simpler than WebSockets for one-directional push |
| File storage | AWS S3 | Cheap, durable, integrates with CloudFront |
| Email | AWS SES | $0.10 per 1,000 emails — cheapest reliable option |
| Container | Docker → ECS Fargate | Serverless containers, no EC2 management |
| CDN | CloudFront | Global edge, integrated with S3 and ALB |
| IaC | Terraform (or AWS CDK) | Reproducible infrastructure |
| CI/CD | GitHub Actions | Free for public repos, tight GitHub integration |
| Monitoring | CloudWatch + Sentry | AWS-native logs + application error tracking |

---

## 12. Architecture Evolution Path

```
Phase 1 (now)                Phase 2–3                  Future (1000+ garages)
─────────────────────────    ────────────────────────    ─────────────────────────
Modular Hexagonal Monolith   Same monolith +             Extract hot domains:
                             + Redis Pub/Sub SSE          ┌─ Invoice Service
ECS: 2 tasks                 + ECS auto-scaling (2–8)     ├─ Notification Service
Aurora: 0.5 ACU              + Aurora: 0.5–16 ACU         └─ Reporting Service
Cost: ~$80/mo                + S3 lifecycle rules          Monolith keeps: Auth,
                             Cost: ~$200/mo                Customers, Vehicles,
                                                           Services, Inventory
                                                           Cost: ~$400–600/mo
                                                           (covered by revenue)
```

The key insight: **hexagonal architecture means each domain can be extracted without rewriting the business logic.** You just add a new HTTP delivery adapter + deploy that domain as a separate service. The rest of the monolith calls it via HTTP instead of in-process.

---

## 13. Alternative Cost-Optimized Option (If AWS Budget is Tight)

If AWS costs are prohibitive early on (before first paying customers), this alternative stack achieves similar availability for less:

| Service | Provider | Cost |
|---------|----------|------|
| Backend hosting | Railway.app (2 replicas, 2 regions) | $20/mo |
| PostgreSQL | Supabase Pro | $25/mo |
| Frontend | Cloudflare Pages | Free |
| File storage | Cloudflare R2 (S3-compatible) | $0.015/GB |
| Email | Resend.com | Free to 3,000/mo then $20/mo |
| **Total** | | **~$45–65/mo** |

**Tradeoff:** Railway/Supabase have fewer enterprise features and less geographic redundancy than AWS. Acceptable for Phase 1 beta, not ideal for Phase 2+ with paying customers who expect uptime.

---

*BayWork Architecture v1.0 · May 2026*
*"Modular, hexagonal, serverless — built to scale without rebuilding."*
