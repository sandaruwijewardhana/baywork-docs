# BayWork — Database ER Diagram & Design Theory

*Matches the live schema (July 2026). Every garage gets its own PostgreSQL schema; the
`public` schema is the shared registry. PK = primary key, FK = foreign key, UQ = unique.*

## 1. Public schema (shared registry — one per installation)

```mermaid
erDiagram
    GARAGES ||--o{ USER_INDEX : "has login entries"
    GARAGES {
        int id PK "serial"
        text name
        text owner_name
        text email UK
        text phone
        text address
        text city
        enum plan "starter|pro|enterprise"
        text schema_name UK "garage_xxxxxxxx"
        bool active
        timestamp created_at
        timestamp updated_at
    }
    USER_INDEX {
        int id PK
        text email UK "login key -> which garage"
        text schema_name "denormalized (see 3.5)"
        int garage_id FK "-> garages.id CASCADE"
        timestamp created_at
    }
    PENDING_REGISTRATIONS {
        int id PK
        text garage_name
        text owner_name
        text email
        text phone
        text plan
        text password_hash
        text ref_number UK "BW-..."
        bool approved
        timestamp approved_at
        timestamp created_at
    }
```

## 2. Tenant schema (one copy per garage — NO garage_id anywhere; isolation = schema)

```mermaid
erDiagram
    USERS ||--o{ REFRESH_TOKENS : "owns"
    USERS ||--o{ SERVICES : "assigned to"
    USERS ||--o{ ACTIVITY_LOG : "performed"
    CUSTOMERS ||--o{ VEHICLES : "owns"
    CUSTOMERS ||--o{ SERVICES : "requested"
    CUSTOMERS ||--o{ INVOICES : "billed"
    VEHICLES ||--o{ SERVICES : "serviced in"
    VEHICLES ||--o{ INVOICES : "referenced by"
    SERVICES ||--o{ SERVICE_ITEMS : "checklist"
    SERVICES ||--o{ INVOICES : "billed by"
    INVOICES ||--o{ INVOICE_PARTS : "lines"
    INVOICES ||--o{ INVOICE_LABOR : "lines"
    INVOICES ||--o{ INVOICE_SERVICE_LINES : "lines"
    INVENTORY_ITEMS ||--o{ INVOICE_PARTS : "sold as"

    USERS {
        int id PK
        text name
        text email UK
        text password_hash
        enum role "owner|technician|cashier|receptionist"
        bool active
    }
    REFRESH_TOKENS {
        int id PK
        int user_id FK "-> users.id CASCADE"
        text token UK "jti-unique JWT"
        timestamp expires_at
    }
    CUSTOMERS {
        int id PK
        text display_id UK "CUS-0001"
        text name
        text nic "partial-UQ registered"
        text phone_primary "partial-UQ registered"
        text whatsapp_number
        date date_of_birth
        bool draft "incomplete registration"
        timestamp deleted_at "soft delete"
        int created_by FK "-> users.id"
    }
    VEHICLES {
        int id PK
        text reg_no "normalized; partial-UQ registered"
        text brand "null while draft"
        text model "null while draft"
        int owner_customer_id FK "-> customers.id"
        int current_mileage "at registration"
        bool draft
        timestamp deleted_at
    }
    SERVICES {
        int id PK
        text job_no UK "JC-2026-0001"
        int customer_id FK "-> customers.id"
        int vehicle_id FK "-> vehicles.id NOT NULL"
        int assigned_to_user_id FK "-> users.id"
        enum status "pending|in_progress|ready|completed|cancelled"
        int mileage_at_dropoff "per-visit mileage"
        timestamp est_pickup_at
        timestamp deleted_at
        int created_by FK "-> users.id"
    }
    SERVICE_ITEMS {
        int id PK
        int service_id FK "-> services.id CASCADE"
        text description
        text status
        int order_index
    }
    INVOICES {
        int id PK
        text invoice_no UK "INV-2026-0001"
        int customer_id FK "-> customers.id"
        int vehicle_id FK "-> vehicles.id"
        int service_id FK "-> services.id"
        enum status "draft|sent|paid|overdue"
        enum payment_method "cash|card|bank_transfer|pending"
        decimal discount_amount "10,2"
        decimal grand_total "10,2 server-computed"
        timestamp deleted_at
        int created_by FK "-> users.id"
    }
    INVOICE_PARTS {
        int id PK
        int invoice_id FK "-> invoices.id CASCADE"
        int inventory_item_id FK "-> inventory_items.id SET NULL"
        text description "snapshot, survives item deletion"
        decimal qty
        decimal unit_price "snapshot price"
        decimal total
    }
    INVOICE_LABOR {
        int id PK
        int invoice_id FK "-> invoices.id CASCADE"
        text description
        decimal hours
        decimal rate
        decimal total
    }
    INVOICE_SERVICE_LINES {
        int id PK
        int invoice_id FK "-> invoices.id CASCADE"
        text description
        decimal amount
    }
    INVENTORY_ITEMS {
        int id PK
        text display_id UK "ITM-0001"
        text name
        text category
        int qty_in_stock "atomic +/- only"
        int alert_threshold
        decimal cost_price
        decimal selling_price
        timestamp deleted_at
    }
    ACTIVITY_LOG {
        int id PK
        int user_id FK "-> users.id SET NULL"
        text action "customer.created ..."
        text entity_type "polymorphic: no FK possible"
        int entity_id
        jsonb metadata
    }
```

## 3. The theory behind the decisions

### 3.1 Why customers have NO "vehicles" column (your question)
A customer's vehicles are found through **`vehicles.owner_customer_id → customers.id`** —
the FK to the primary key IS the connection. Storing a `vehicle_count` (or a list) on the
customer row would **duplicate derivable data**, breaking normalization: every vehicle
insert/delete/re-assignment would have to update two tables, and any missed update makes
the count silently wrong (an *update anomaly*). Rule: **store facts once; derive the rest**
— the UI's count is a `COUNT(*)` subquery over the FK, always correct by construction.

### 3.2 Normal forms in practice
- **1NF** — every column atomic: no comma-separated lists (that's *why* no "vehicles" text column).
- **2NF/3NF** — non-key columns depend on the key only: vehicle facts live on vehicles,
  not repeated per service; customer contact lives once, referenced by id everywhere.
- Line items are **separate tables** (invoice_parts/labor/service_lines), one row per line —
  the relational answer to "an invoice has many lines".

### 3.3 Keys
- **Surrogate PKs** (`serial id`) everywhere: stable, small, never carry meaning.
- **Natural identifiers** (`reg_no`, `phone`, `email`, `display_id`) are UNIQUE columns,
  *not* PKs — they can change or be re-used; a PK must never change.
- **Partial unique indexes** (`WHERE deleted_at IS NULL AND draft = false`) enforce
  "unique among the living, registered rows" — plain UNIQUE can't express a lifecycle.

### 3.4 ON DELETE strategy (three deliberate tiers)
| Behavior | Used for | Reasoning |
|---|---|---|
| `CASCADE` | tokens, checklist items, invoice lines | Meaningless without their parent |
| `SET NULL` | invoice_parts→inventory, activity_log→users | History outlives the referenced row; the snapshot columns (description/price) keep the record readable |
| `RESTRICT` (default) + **soft delete** | customers, vehicles, services, invoices | Business records are never hard-deleted; `deleted_at` hides them while every FK stays valid |

### 3.5 Deliberate denormalization (documented exceptions)
- `user_index.schema_name` duplicates `garages.schema_name` — a **login hot-path** optimization:
  one lookup resolves email → schema without a join, on every request. Safe because a schema
  name never changes after provisioning. *Denormalize only immutable data, and write it down.*
- `invoice_parts.description/unit_price` snapshot the inventory item at sale time — an invoice
  is a legal record and must not rewrite itself when prices change later.

### 3.6 Polymorphic reference (the one FK that can't exist)
`activity_log.entity_type + entity_id` points at *any* table, so SQL can't declare one FK for it.
This is a standard trade-off for audit logs: integrity is enforced by application code, and the
log is append-only so dangling ids are acceptable history.

### 3.7 FK columns are indexed
Postgres indexes PKs automatically but **not** FK columns. Every FK used in joins or
cascades gets an explicit index (owner_customer_id, customer_id, invoice_id on each line
table, service_id, user_id…) — without them each cascade/join is a sequential scan.

### 3.8 Multi-tenancy by schema
Tenant tables carry **no `garage_id`** — a `garage_id` column on every row (row-level
tenancy) is the common alternative, but schema-per-tenant gives hard isolation
(`SET search_path`), per-tenant backup/restore, and makes cross-tenant leaks structurally
impossible rather than merely forbidden by WHERE clauses.
```
