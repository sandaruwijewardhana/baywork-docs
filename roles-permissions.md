# BayWork — Roles & Permissions
**v1.0 · May 2026**

---

## Current roles (in schema)

Four roles exist in `roleEnum`: `root_admin`, `technician`, `cashier`, `receptionist`.

---

## Role descriptions

### `root_admin` — Garage Owner / Manager
The person who owns or runs the garage. Has full access to everything.
Typically one per garage, but a trusted manager can also hold this role.

### `technician` — Mechanic / Workshop Staff
Does the actual repair work. Lives in the Services kanban all day.
Needs to see and update job cards. Does not handle money or customer data beyond what's needed to do the job.

### `cashier` — Billing / Payments
Handles invoices, takes payment, manages parts billing.
Sees financial data. Does not manage staff or change job statuses beyond viewing.

### `receptionist` — Front Desk / Service Advisor
Takes in vehicles, creates job cards, handles customer communication.
Does not handle invoices or financial reports.

---

## Permission matrix

`✅ full` — create, read, update, delete  
`📖 read` — view only  
`✏️ own` — only records assigned to / created by themselves  
`⚡ action` — can trigger specific status changes only  
`❌ none` — no access

| Module | root_admin | technician | cashier | receptionist |
|--------|-----------|------------|---------|--------------|
| **Customers** — create, edit, soft-delete | ✅ full | 📖 read | ✏️ create + edit | ✅ full |
| **Vehicles** — create, edit, soft-delete | ✅ full | 📖 read | ✏️ create + edit | ✅ full |
| **Services** — create job cards | ✅ full | ✏️ own | 📖 read | ✅ full |
| **Services** — update status | ✅ full | ⚡ pending→in_progress→ready | ❌ none | ⚡ pending only |
| **Services** — checklist items | ✅ full | ✏️ own assigned jobs | 📖 read | ✅ full |
| **Services** — assign technician | ✅ full | ❌ none | ❌ none | ✅ full |
| **Services** — delete / cancel | ✅ full | ❌ none | ❌ none | ❌ none |
| **Invoices** — create, edit | ✅ full | ❌ none | ✅ full | 📖 read |
| **Invoices** — mark paid | ✅ full | ❌ none | ✅ full | ❌ none |
| **Invoices** — apply discount | ✅ full | ❌ none | ⚡ up to 10% | ❌ none |
| **Invoices** — delete / void | ✅ full | ❌ none | ❌ none | ❌ none |
| **Inventory** — view stock levels | ✅ full | 📖 read | ✅ full | 📖 read |
| **Inventory** — create, edit items | ✅ full | ❌ none | ✅ full | ❌ none |
| **Inventory** — delete items | ✅ full | ❌ none | ❌ none | ❌ none |
| **Staff / Team** — invite, edit, deactivate | ✅ full | ❌ none | ❌ none | ❌ none |
| **Reports / Analytics** | ✅ full | ❌ none | 📖 read | ❌ none |
| **Activity log** | ✅ full | ✏️ own actions | ✏️ own actions | ✏️ own actions |
| **Garage settings** — name, logo, plan | ✅ full | ❌ none | ❌ none | ❌ none |

---

## Service status transitions (who can move what)

```
pending ──→ in_progress ──→ ready ──→ completed
   ↓              ↓            ↓
cancelled      cancelled    cancelled
```

| From → To | root_admin | technician | cashier | receptionist |
|-----------|-----------|------------|---------|--------------|
| pending → in_progress | ✅ | ✅ (own) | ❌ | ❌ |
| in_progress → ready | ✅ | ✅ (own) | ❌ | ❌ |
| ready → completed | ✅ | ❌ | ✅ | ❌ |
| any → cancelled | ✅ | ❌ | ❌ | ❌ |
| any → pending (reopen) | ✅ | ❌ | ❌ | ❌ |

`ready → completed` is triggered by the cashier when payment is received and invoice is marked paid.

---

## Discount limits

Cashiers can apply discounts up to 10% without approval.
Anything above 10% requires a `root_admin` to apply or approve it.
This is not enforced in the UI yet — implement as a backend rule on `PATCH /invoices/:id`.

---

## How permissions are enforced

**Backend (source of truth):**
Every route checks `req.auth.role` via `requireRole(...)` middleware.
The DB query always filters by `req.auth.garageId` — a user can never touch another garage's data regardless of role.

**Frontend (UX only — never rely on this for security):**
Hide buttons and nav items based on the `role` in `sessionStorage`.
Example: cashiers never see the "Staff" settings tab.

```javascript
// console/js/auth-guard.js  (to write)
const user = API.getUser();
if (!['root_admin', 'cashier'].includes(user.role)) {
  document.querySelector('#nav-invoices').style.display = 'none';
}
```

---

## Future roles to add

These are not in the schema yet. Add when needed — the `roleEnum` in Postgres will need a migration.

### `branch_manager`
For garage chains with multiple locations. Same permissions as `root_admin` but scoped to one branch only. Requires a `branches` table and `branchId` on `users`.

### `accountant` (read-only financial)
Read-only access to invoices, payments, and reports. No ability to create or modify anything. Useful for giving an external accountant access without handing over full admin credentials.

### `parts_manager`
Focused inventory role. Full access to inventory and suppliers. No access to customers, services, or invoices. For garages large enough to have a dedicated parts person.

### `service_advisor` (elevated receptionist)
Like receptionist but can also close job cards and view invoice totals. For garages where front-desk staff handle the full customer journey including billing summary.

---

## Future permission features to consider

**Field-level visibility**
Some garages may not want technicians to see the customer's phone number or NIC. A `field_visibility` config per role could hide specific columns in the UI (and strip them from API responses).

**Time-based access**
Lock accounts outside working hours. A technician's token could refuse to refresh after 20:00. Simple to implement: add `shift_start` / `shift_end` to users and check in the refresh endpoint.

**Audit trail**
Every write already goes through `activityLog`. Future: surfacing this in a UI per-record ("who changed this invoice and when") and per-user ("what did Kamal do today").

**Two-person approval for large invoices**
Invoices above a threshold (e.g. LKR 100,000) require a second `root_admin` confirmation before being marked paid. Prevents cashier errors on big jobs.

**IP / device restrictions**
Restrict login to known devices or the garage's IP range. Useful once garages have a fixed office setup. Cloudflare Access can handle this without code changes.
