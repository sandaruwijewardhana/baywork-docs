# BayWork — 7-Day Beta Launch Plan
### Local Laptop + Cloudflare Tunnel
**v1.0 · May 2026**

---

## Starting Point

| Already done | Still needed |
|-------------|-------------|
| Landing page UI (`website/index.html`) | Node.js API + database |
| Login page UI (`website/login.html`) | Auth wired to real backend |
| Register page UI (`website/register.html`) | Console pages (customers, vehicles, services, invoices, inventory) |
| Dashboard UI prototype (`website/dashboard.html`) | Docker setup |
| Brand system + design tokens | Cloudflare tunnel |

---

## Tech Stack (kept simple for beta)

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | Vanilla HTML + CSS (existing) + `fetch()` calls | No build tools — fastest to wire up |
| Backend | Node.js 20 + Express 5 | Simple, async, JS everywhere |
| ORM | Drizzle ORM | Type-safe SQL, no magic |
| Database | PostgreSQL 15 (Docker) | Production-grade, runs locally |
| Auth | JWT (access) + HttpOnly cookie (refresh) | Secure, stateless |
| Runtime | Docker Compose | One command start |
| Tunnel | Cloudflare Tunnel (`cloudflared`) | Free public URL, no port forwarding |

---

## Project Structure to Build

```
baywork/
├── docs/                    ← you are here
├── website/                 ← existing UI prototypes (reference only)
├── docker-compose.yml       ← starts everything
│
├── api/                     ← Node.js backend
│   ├── src/
│   │   ├── db/
│   │   │   ├── schema.ts    ← all table definitions
│   │   │   └── client.ts    ← DB connection
│   │   ├── middleware/
│   │   │   ├── auth.ts      ← JWT verification
│   │   │   └── roles.ts     ← role-based access
│   │   ├── routes/
│   │   │   ├── auth.ts
│   │   │   ├── customers.ts
│   │   │   ├── vehicles.ts
│   │   │   ├── services.ts
│   │   │   ├── invoices.ts
│   │   │   └── inventory.ts
│   │   └── index.ts         ← Express app entry
│   ├── Dockerfile
│   └── package.json
│
└── console/                 ← new HTML pages for the logged-in app
    ├── login.html           ← copy + wire from website/
    ├── register.html        ← copy + wire from website/
    ├── dashboard.html
    ├── customers.html
    ├── customers-new.html
    ├── vehicles.html
    ├── services.html
    ├── services-new.html
    ├── invoices.html
    ├── invoices-new.html
    ├── inventory.html
    └── styles/ + scripts/   ← shared assets from website/
```

---

## Sub-Phase 1 — Foundation
**Day 1 Morning · ~3 hours**
**Goal: Docker running, API answering, database connected**

### What you'll learn
How Docker Compose wires services together, how Express boots, how Drizzle connects to Postgres.

### Steps

**1. Create `docker-compose.yml` at project root**

```yaml
version: '3.9'
services:
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: baywork
      POSTGRES_PASSWORD: baywork123
      POSTGRES_DB: baywork
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  api:
    build: ./api
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://baywork:baywork123@db:5432/baywork
      JWT_SECRET: super-secret-key-change-in-prod
      PORT: 3001
    ports:
      - "3001:3001"
    depends_on:
      - db
    volumes:
      - ./console:/app/public   # serves console HTML files

volumes:
  pgdata:
```

**2. Set up the API**

```bash
mkdir api && cd api
npm init -y
npm install express drizzle-orm pg @types/pg dotenv bcryptjs jsonwebtoken cookie-parser cors helmet express-rate-limit
npm install -D typescript @types/express @types/node ts-node drizzle-kit
```

**3. `api/src/db/schema.ts` — all tables at once**

```typescript
import { pgTable, serial, text, timestamp, integer,
         boolean, decimal, pgEnum, jsonb } from 'drizzle-orm/pg-core';

// Enums
export const roleEnum = pgEnum('role', ['root_admin','technician','cashier','receptionist']);
export const serviceStatusEnum = pgEnum('service_status', ['pending','in_progress','ready','completed','cancelled']);
export const invoiceStatusEnum = pgEnum('invoice_status', ['draft','sent','paid','overdue']);
export const paymentMethodEnum = pgEnum('payment_method', ['cash','card','bank_transfer','pending']);

// Auth
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

export const refreshTokens = pgTable('refresh_tokens', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id),
  token:     text('token').notNull().unique(),
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
});

export const pendingRegistrations = pgTable('pending_registrations', {
  id:           serial('id').primaryKey(),
  garageName:   text('garage_name').notNull(),
  ownerName:    text('owner_name').notNull(),
  email:        text('email').notNull(),
  phone:        text('phone').notNull(),
  address:      text('address'),
  city:         text('city'),
  plan:         text('plan').notNull().default('pro'),
  passwordHash: text('password_hash').notNull(),
  refNumber:    text('ref_number').notNull().unique(),
  approved:     boolean('approved').notNull().default(false),
  createdAt:    timestamp('created_at').defaultNow(),
});

// Core modules
export const customers = pgTable('customers', {
  id:          serial('id').primaryKey(),
  displayId:   text('display_id').notNull().unique(), // CUS-0001
  name:        text('name').notNull(),
  nic:         text('nic'),
  phonePrimary: text('phone_primary').notNull(),
  phoneSecondary: text('phone_secondary'),
  email:       text('email'),
  address:     text('address'),
  city:        text('city'),
  notes:       text('notes'),
  deletedAt:   timestamp('deleted_at'),
  createdAt:   timestamp('created_at').defaultNow(),
  createdBy:   integer('created_by'),
});

export const vehicles = pgTable('vehicles', {
  id:               serial('id').primaryKey(),
  regNo:            text('reg_no').notNull().unique(),
  brand:            text('brand').notNull(),
  model:            text('model').notNull(),
  year:             integer('year'),
  color:            text('color'),
  fuelType:         text('fuel_type'),
  transmission:     text('transmission'),
  engineCc:         integer('engine_cc'),
  chassisNo:        text('chassis_no'),
  ownerCustomerId:  integer('owner_customer_id').references(() => customers.id),
  currentMileage:   integer('current_mileage'),
  notes:            text('notes'),
  deletedAt:        timestamp('deleted_at'),
  createdAt:        timestamp('created_at').defaultNow(),
});

export const services = pgTable('services', {
  id:                serial('id').primaryKey(),
  jobNo:             text('job_no').notNull().unique(), // JC-2026-0001
  customerId:        integer('customer_id').references(() => customers.id),
  vehicleId:         integer('vehicle_id').notNull().references(() => vehicles.id),
  serviceType:       text('service_type').notNull().default('general'),
  assignedToUserId:  integer('assigned_to_user_id').references(() => users.id),
  status:            serviceStatusEnum('status').notNull().default('pending'),
  mileageAtDropoff:  integer('mileage_at_dropoff'),
  dropoffAt:         timestamp('dropoff_at').defaultNow(),
  estPickupAt:       timestamp('est_pickup_at'),
  priority:          text('priority').default('normal'),
  customerNotes:     text('customer_notes'),
  internalNotes:     text('internal_notes'),
  deletedAt:         timestamp('deleted_at'),
  createdAt:         timestamp('created_at').defaultNow(),
  createdBy:         integer('created_by'),
});

export const serviceItems = pgTable('service_items', {
  id:          serial('id').primaryKey(),
  serviceId:   integer('service_id').notNull().references(() => services.id),
  description: text('description').notNull(),
  status:      text('status').notNull().default('not_started'),
  estimatedMins: integer('estimated_mins'),
  notes:       text('notes'),
  orderIndex:  integer('order_index').default(0),
});

export const invoices = pgTable('invoices', {
  id:            serial('id').primaryKey(),
  invoiceNo:     text('invoice_no').notNull().unique(), // INV-2026-0001
  customerId:    integer('customer_id').references(() => customers.id),
  vehicleId:     integer('vehicle_id').references(() => vehicles.id),
  issueDate:     timestamp('issue_date').defaultNow(),
  dueDate:       timestamp('due_date'),
  status:        invoiceStatusEnum('status').notNull().default('draft'),
  paymentMethod: paymentMethodEnum('payment_method').default('pending'),
  discountAmount: decimal('discount_amount', { precision: 10, scale: 2 }).default('0'),
  grandTotal:    decimal('grand_total', { precision: 10, scale: 2 }).default('0'),
  notes:         text('notes'),
  deletedAt:     timestamp('deleted_at'),
  createdAt:     timestamp('created_at').defaultNow(),
  createdBy:     integer('created_by'),
});

export const invoiceParts = pgTable('invoice_parts', {
  id:              serial('id').primaryKey(),
  invoiceId:       integer('invoice_id').notNull().references(() => invoices.id),
  inventoryItemId: integer('inventory_item_id'),
  description:     text('description').notNull(),
  qty:             decimal('qty', { precision: 8, scale: 2 }).notNull(),
  unitPrice:       decimal('unit_price', { precision: 10, scale: 2 }).notNull(),
  total:           decimal('total', { precision: 10, scale: 2 }).notNull(),
});

export const invoiceLabor = pgTable('invoice_labor', {
  id:          serial('id').primaryKey(),
  invoiceId:   integer('invoice_id').notNull().references(() => invoices.id),
  description: text('description').notNull(),
  hours:       decimal('hours', { precision: 6, scale: 2 }),
  rate:        decimal('rate', { precision: 10, scale: 2 }),
  total:       decimal('total', { precision: 10, scale: 2 }).notNull(),
});

export const invoiceServiceLines = pgTable('invoice_service_lines', {
  id:          serial('id').primaryKey(),
  invoiceId:   integer('invoice_id').notNull().references(() => invoices.id),
  description: text('description').notNull(),
  amount:      decimal('amount', { precision: 10, scale: 2 }).notNull(),
});

export const inventoryItems = pgTable('inventory_items', {
  id:              serial('id').primaryKey(),
  displayId:       text('display_id').notNull().unique(), // ITM-0001
  brand:           text('brand'),
  category:        text('category').notNull(),
  name:            text('name').notNull(),
  unit:            text('unit').notNull().default('pcs'),
  qtyInStock:      integer('qty_in_stock').notNull().default(0),
  alertThreshold:  integer('alert_threshold').notNull().default(5),
  costPrice:       decimal('cost_price', { precision: 10, scale: 2 }).notNull(),
  sellingPrice:    decimal('selling_price', { precision: 10, scale: 2 }).notNull(),
  supplierName:    text('supplier_name'),
  deletedAt:       timestamp('deleted_at'),
  createdAt:       timestamp('created_at').defaultNow(),
});

export const activityLog = pgTable('activity_log', {
  id:         serial('id').primaryKey(),
  userId:     integer('user_id'),
  action:     text('action').notNull(),
  entityType: text('entity_type'),
  entityId:   integer('entity_id'),
  metadata:   jsonb('metadata'),
  createdAt:  timestamp('created_at').defaultNow(),
});
```

**4. `api/src/db/client.ts`**

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { schema });
```

**5. `api/src/index.ts`**

```typescript
import express from 'express';
import cookieParser from 'cookie-parser';
import cors from 'cors';
import helmet from 'helmet';
import { authRouter } from './routes/auth';

const app = express();
app.use(helmet());
app.use(cors({ origin: true, credentials: true }));
app.use(express.json());
app.use(cookieParser());
app.use(express.static('public'));  // serves console/ HTML files

app.use('/api/auth', authRouter);
app.get('/api/health', (_, res) => res.json({ status: 'ok' }));

app.listen(process.env.PORT || 3001, () =>
  console.log(`API running on :${process.env.PORT || 3001}`)
);
```

**6. Run it**

```bash
docker compose up --build
# Open http://localhost:3001/api/health
# Should return: {"status":"ok"}
```

### ✅ Done when
`curl http://localhost:3001/api/health` returns `{"status":"ok"}` and no DB connection errors in logs.

---

## Sub-Phase 2 — Auth End-to-End
**Day 1 Afternoon + Day 2 Morning · ~5 hours**
**Goal: Login and register pages hit real API, JWT session works**

### What you'll learn
JWT token flow, HttpOnly cookies, bcrypt hashing, middleware guards.

### Steps

**1. `api/src/routes/auth.ts`**

```typescript
import { Router } from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { db } from '../db/client';
import { users, refreshTokens, pendingRegistrations } from '../db/schema';
import { eq } from 'drizzle-orm';

const router = Router();
const JWT_SECRET = process.env.JWT_SECRET!;

// POST /api/auth/login
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const [user] = await db.select().from(users).where(eq(users.email, email));
  if (!user || !await bcrypt.compare(password, user.passwordHash))
    return res.status(401).json({ error: 'Invalid credentials' });
  if (!user.active)
    return res.status(403).json({ error: 'Account deactivated' });

  const accessToken = jwt.sign(
    { userId: user.id, role: user.role, name: user.name },
    JWT_SECRET, { expiresIn: '15m' }
  );
  const refreshToken = jwt.sign({ userId: user.id }, JWT_SECRET, { expiresIn: '7d' });

  await db.insert(refreshTokens).values({
    userId: user.id, token: refreshToken,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  res.cookie('refresh_token', refreshToken, {
    httpOnly: true, secure: false, sameSite: 'strict', maxAge: 7 * 24 * 60 * 60 * 1000
  });
  res.json({ accessToken, user: { id: user.id, name: user.name, role: user.role } });
});

// POST /api/auth/logout
router.post('/logout', async (req, res) => {
  const token = req.cookies?.refresh_token;
  if (token) await db.delete(refreshTokens).where(eq(refreshTokens.token, token));
  res.clearCookie('refresh_token');
  res.json({ ok: true });
});

// POST /api/auth/refresh
router.post('/refresh', async (req, res) => {
  const token = req.cookies?.refresh_token;
  if (!token) return res.status(401).json({ error: 'No token' });
  try {
    const payload = jwt.verify(token, JWT_SECRET) as any;
    const [stored] = await db.select().from(refreshTokens).where(eq(refreshTokens.token, token));
    if (!stored) return res.status(401).json({ error: 'Token revoked' });
    const [user] = await db.select().from(users).where(eq(users.id, payload.userId));
    const accessToken = jwt.sign(
      { userId: user.id, role: user.role, name: user.name },
      JWT_SECRET, { expiresIn: '15m' }
    );
    res.json({ accessToken });
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
});

// POST /api/auth/register
router.post('/register', async (req, res) => {
  const { garageName, ownerName, email, phone, address, city, plan, password } = req.body;
  const passwordHash = await bcrypt.hash(password, 12);
  const refNumber = 'APP-' + Date.now();
  await db.insert(pendingRegistrations).values({
    garageName, ownerName, email, phone, address, city, plan, passwordHash, refNumber
  });
  res.json({ refNumber });
});

export { router as authRouter };
```

**2. `api/src/middleware/auth.ts`**

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

export function requireAuth(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  try {
    (req as any).user = jwt.verify(token, process.env.JWT_SECRET!);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes((req as any).user?.role))
      return res.status(403).json({ error: 'Forbidden' });
    next();
  };
}
```

**3. Seed a first admin user (run once)**

```typescript
// api/src/db/seed.ts
import bcrypt from 'bcryptjs';
import { db } from './client';
import { users } from './schema';

await db.insert(users).values({
  name: 'Admin',
  email: 'admin@baywork.io',
  passwordHash: await bcrypt.hash('admin123', 12),
  role: 'root_admin',
});
console.log('Admin created: admin@baywork.io / admin123');
```

**4. Wire `console/login.html` to the real API**

In login.html, replace the fake `handleLogin` with:

```javascript
async function handleLogin(e) {
  e.preventDefault();
  const res = await fetch('/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify({
      email: document.getElementById('email').value,
      password: document.getElementById('password').value
    })
  });
  const data = await res.json();
  if (!res.ok) { showError(data.error); return; }
  localStorage.setItem('access_token', data.accessToken);
  localStorage.setItem('user', JSON.stringify(data.user));
  window.location.href = '/dashboard.html';
}
```

**5. Wire `console/register.html` to the real API**

On final step submit, call `POST /api/auth/register`, show the `refNumber` in the confirmation screen.

### ✅ Done when
- Login with `admin@baywork.io` / `admin123` redirects to dashboard
- Register form creates a row in `pending_registrations` table
- Visiting `/dashboard.html` without logging in redirects to `/login.html`

---

## Sub-Phase 3 — Customers + Vehicles
**Day 3 · ~8 hours**
**Goal: Add, list, view customers and their vehicles from real database**

### What you'll learn
REST CRUD patterns, how `requireAuth` middleware protects routes, how to build table + form pages.

### API routes to build

```
GET    /api/customers?search=&page=     list with search + pagination
POST   /api/customers                   create new
GET    /api/customers/:id               single customer with vehicles
PATCH  /api/customers/:id               update
DELETE /api/customers/:id               soft delete

GET    /api/vehicles?search=&page=      list
POST   /api/vehicles                    create
GET    /api/vehicles/:id                single vehicle
PATCH  /api/vehicles/:id                update
```

### Console pages to build

- **`console/customers.html`** — table of customers, search bar, "+ Add Customer" button
- **`console/vehicles.html`** — table of vehicles, search bar, "+ Register Vehicle" button

Use the design tokens from `website/styles/brand.css` — same sidebar shell as `website/dashboard.html`.

### Shared `console/js/api.js` helper (create once, reuse everywhere)

```javascript
// Handles token refresh automatically
const API = {
  token: localStorage.getItem('access_token'),

  async call(method, path, body) {
    let res = await fetch('/api' + path, {
      method,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`
      },
      credentials: 'include',
      body: body ? JSON.stringify(body) : undefined
    });

    if (res.status === 401) {
      // Try refresh
      const r = await fetch('/api/auth/refresh', { method: 'POST', credentials: 'include' });
      if (!r.ok) { window.location.href = '/login.html'; return; }
      const { accessToken } = await r.json();
      this.token = accessToken;
      localStorage.setItem('access_token', accessToken);
      // Retry original request
      res = await fetch('/api' + path, {
        method,
        headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${this.token}` },
        credentials: 'include',
        body: body ? JSON.stringify(body) : undefined
      });
    }
    return res.json();
  },

  get: (path) => API.call('GET', path),
  post: (path, body) => API.call('POST', path, body),
  patch: (path, body) => API.call('PATCH', path, body),
  delete: (path) => API.call('DELETE', path),
};
```

### ✅ Done when
- Can add a customer, see them in the list, click through to their detail page
- Can register a vehicle linked to that customer
- Search box filters results

---

## Sub-Phase 4 — Services + Job Cards
**Day 4 · ~8 hours**
**Goal: Create a service, update status, print a job card**

### What you'll learn
Multi-step form state in vanilla JS, how status enums work, browser print API.

### API routes to build

```
GET    /api/services?status=&page=
POST   /api/services
GET    /api/services/:id
PATCH  /api/services/:id/status    { status, note }
POST   /api/services/:id/items     add checklist item
PATCH  /api/services/:id/items/:itemId
```

### Console pages to build

- **`console/services.html`** — kanban board (3 columns) + table toggle
- **`console/services-new.html`** — 3-step creation form
- **`console/services-detail.html`** — job card view, checklist, status controls

### Job Card Print

In `services-detail.html`, a "Print Job Card" button calls `window.print()`. Add a `@media print` section to show only the job card and hide nav/sidebar:

```css
@media print {
  .sidebar, .topbar, .no-print { display: none !important; }
  .job-card-print { display: block !important; }
}
```

### ✅ Done when
- Create a service for a customer + vehicle
- See it appear in the Pending column of the kanban
- Click "In Progress" → moves column
- Click "Print Job Card" → clean A4 prints

---

## Sub-Phase 5 — Invoices
**Day 5 · ~8 hours**
**Goal: Create an invoice with parts + labor, mark as paid, print**

### What you'll learn
Dynamic table rows with live totals, decimal arithmetic in JS, print formatting.

### API routes to build

```
GET    /api/invoices?status=&page=
POST   /api/invoices              creates invoice + parts + labor + service lines
GET    /api/invoices/:id
PATCH  /api/invoices/:id
PATCH  /api/invoices/:id/status   { status: 'paid' }
```

### Console pages to build

- **`console/invoices.html`** — invoice list, status filter chips, totals at top
- **`console/invoices-new.html`** — invoice builder

### Invoice Builder Key JS

```javascript
function recalculate() {
  const partsTotal = [...document.querySelectorAll('.part-row')].reduce((sum, row) => {
    const qty = parseFloat(row.querySelector('.qty').value) || 0;
    const price = parseFloat(row.querySelector('.unit-price').value) || 0;
    row.querySelector('.total').textContent = (qty * price).toLocaleString();
    return sum + (qty * price);
  }, 0);

  const laborTotal = /* same pattern */ 0;
  const discount = parseFloat(document.getElementById('discount').value) || 0;
  const grand = partsTotal + laborTotal - discount;
  document.getElementById('grand-total').textContent = grand.toLocaleString();
}
```

### ✅ Done when
- Create an invoice with 2 parts + 1 labor line
- Grand total updates live as you type
- "Mark as Paid" → status changes to Paid with a green badge
- "Print Invoice" → clean A4 professional layout

---

## Sub-Phase 6 — Inventory + Dashboard
**Day 6 · ~8 hours**
**Goal: Inventory CRUD, low-stock alerts, dashboard shows real numbers**

### What you'll learn
Aggregation queries with Drizzle, how to poll for live data.

### API routes to build

```
GET    /api/inventory?category=&low_stock=
POST   /api/inventory
PATCH  /api/inventory/:id
POST   /api/inventory/:id/restock   { qty, newCostPrice }
GET    /api/dashboard/stats         { jobsToday, completedToday, revenueToday, pendingInvoices }
GET    /api/dashboard/activity      last 10 actions
```

### Dashboard stats query (Drizzle)

```typescript
// GET /api/dashboard/stats
const today = new Date(); today.setHours(0,0,0,0);

const [{ jobsToday }] = await db.select({ jobsToday: count() })
  .from(services)
  .where(and(gte(services.createdAt, today), isNull(services.deletedAt)));

const [{ revenue }] = await db.select({ revenue: sum(invoices.grandTotal) })
  .from(invoices)
  .where(and(gte(invoices.createdAt, today), eq(invoices.status, 'paid')));
```

### Dashboard auto-refresh

```javascript
// Refresh stats every 30 seconds without full page reload
async function refreshStats() {
  const stats = await API.get('/dashboard/stats');
  document.getElementById('jobs-today').textContent = stats.jobsToday;
  document.getElementById('revenue-today').textContent = 'LKR ' + parseInt(stats.revenueToday || 0).toLocaleString();
}
refreshStats();
setInterval(refreshStats, 30000);
```

### ✅ Done when
- Add 5 inventory items
- Dashboard shows real counts (not hardcoded demo data)
- Low stock alert appears when an item goes below threshold

---

## Sub-Phase 7 — Launch
**Day 7 · ~6 hours**
**Goal: Live on Cloudflare tunnel, tested end-to-end, ready for first real garage**

### Step 1: Install cloudflared (Windows)

```powershell
winget install Cloudflare.cloudflared
```

### Step 2: Login (one-time, opens browser)

```powershell
cloudflared tunnel login
```

### Step 3: Create a named tunnel

```powershell
cloudflared tunnel create baywork-beta
# Creates a tunnel ID, saves credentials to ~/.cloudflared/
```

### Step 4: Config file `~/.cloudflared/config.yml`

```yaml
tunnel: baywork-beta
credentials-file: C:\Users\YourName\.cloudflared\<tunnel-id>.json

ingress:
  - hostname: beta.yourdomain.com    # or use the free *.trycloudflare.com URL
    service: http://localhost:3001
  - service: http_status:404
```

### Step 5: Run the tunnel

```powershell
# Terminal 1: start the app
docker compose up

# Terminal 2: start the tunnel
cloudflared tunnel run baywork-beta
```

Your app is now at `https://beta.yourdomain.com`. Anyone with the URL can access it.

### Step 6: Set the app to auto-start on Windows login

```powershell
# Register as Windows service (runs even when you close the terminal)
cloudflared service install
docker compose up -d  # -d = detached (background)
```

### End-to-end test checklist

```
□ Open the public URL on your phone (different network — e.g. mobile data)
□ Register a new garage → see pending registration in DB
□ Log in as admin → lands on dashboard
□ Add a test customer (Nimal Perera, 0771234567)
□ Register their vehicle (Toyota Prius, KA-1234)
□ Create a service for that vehicle
□ Move service to "In Progress"
□ Print the job card — check it looks right on A4
□ Create an invoice — add 2 parts, 1 labor line
□ Mark invoice as paid
□ Check dashboard stats updated
□ Add 3 inventory items
□ Confirm low stock alert shows when qty < threshold
□ Log out → can't access console
□ Try accessing /dashboard.html without login → redirects to /login.html
```

### ✅ Done when
All checklist items pass on a device outside your home network.

---

## Daily Schedule Summary

| Day | Sub-Phase | Deliverable |
|-----|-----------|-------------|
| 1 AM | Sub-phase 1 | Docker running, API health endpoint, DB connected |
| 1 PM + 2 AM | Sub-phase 2 | Real login/register + JWT sessions working |
| 3 | Sub-phase 3 | Customers + Vehicles CRUD with real pages |
| 4 | Sub-phase 4 | Services + Job Card print |
| 5 | Sub-phase 5 | Invoices + print |
| 6 | Sub-phase 6 | Inventory + real dashboard stats |
| 7 | Sub-phase 7 | Cloudflare tunnel live, full end-to-end test |

---

## After Beta Launch

Once your brother's garage is using it for 2–4 weeks:

1. **Fix pain points** — note every "this is annoying" moment
2. **Add PDF download** — replace browser print with proper PDF (use `puppeteer` server-side)
3. **Move DB to Supabase** — eliminates data-loss risk from laptop issues (~$25/month)
4. **Add 1–2 more garages** — collect LKR 2,999–5,999/month
5. **Move to AWS** — when revenue covers $70/month hosting

---

*BayWork Beta Launch Plan v1.0 · May 2026*
