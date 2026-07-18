# BayWork Beta — Test Runbook

**Purpose:** verify every feature works, in the order a real garage would use them.
Each step lists the exact action and the **expected result**. Tick as you go.

**Test accounts**
| Role | Email | Password |
|---|---|---|
| Garage owner (demo) | `owner@speedauto.lk` | `baywork123` |

**URLs**
- Local: `http://localhost:3001`
- Public: the `https://….trycloudflare.com` URL printed by `start-beta.ps1` (changes each restart)

---

## 0. Stack up

| # | Action | Expected |
|---|--------|----------|
| 0.1 | Run `start-beta.ps1` (or `docker compose up -d`) | "API healthy on http://localhost:3001", tunnel URL printed |
| 0.2 | Open `http://localhost:3001/api/health` | `{"status":"ok", ...}` |
| 0.3 | Open the public tunnel URL on your **phone using mobile data** | Login page loads over HTTPS |

## 1. Authentication

| # | Action | Expected |
|---|--------|----------|
| 1.1 | Open `/login.html`, log in with demo owner | Redirects to dashboard; your name + role show bottom-left in the sidebar |
| 1.2 | Log in with a wrong password | "Invalid credentials" error, no redirect |
| 1.3 | Try wrong password **21 times within 15 min** | "Too many login attempts" (rate limit kicks in at 20) |
| 1.4 | While logged in, open `/customers.html` | Page loads with data |
| 1.5 | Open a new **incognito** window → `/dashboard.html` | Bounced to `/login.html` (no token) |
| 1.6 | Click **Sign out** in the sidebar | Back at login; pressing Back does not restore the session |
| 1.7 | Stay idle 16+ minutes, then click something | Works seamlessly (access token silently refreshed via cookie) |

## 2. Garage registration & approval (multi-tenant)

| # | Action | Expected |
|---|--------|----------|
| 2.1 | Open `/register.html`, submit a new garage (use a fresh email) | Confirmation screen with a reference number `BW-…` |
| 2.2 | Register again with the **same email** | "Email already registered" (409) |
| 2.3 | Approve it (no UI yet — run in PowerShell, logged-in owner token required): see snippet below | Response contains `garageId`, `schemaName` (`garage_` + 8 chars), `ownerUserId` |
| 2.4 | Log out, log in with the **new garage's** email + password | Lands on an **empty** dashboard — zero customers/jobs (proof of tenant isolation) |
| 2.5 | Log back in as demo owner | Demo data still intact — the two garages never mix |

```powershell
# 2.3 — approve the newest pending registration
$U = "http://localhost:3001"
$t = (Invoke-RestMethod "$U/api/auth/login" -Method Post -ContentType 'application/json' `
      -Body '{"email":"owner@speedauto.lk","password":"baywork123"}').accessToken
$h = @{ Authorization = "Bearer $t" }
$regs = Invoke-RestMethod "$U/api/admin/registrations" -Headers $h
$regs                                          # find the id of your registration
Invoke-RestMethod "$U/api/admin/registrations/<ID>/approve" -Method Post -Headers $h
```

## 3. Customers

| # | Action | Expected |
|---|--------|----------|
| 3.1 | Customers page → **+ Add customer** → fill only Name | Blocked: "Primary phone is required" |
| 3.2 | Add name + phone (e.g. *Nimal Perera / 0771234567*), Save | Toast "Customer CUS-0001 created", back to list, row visible |
| 3.3 | Add 11 customers total | Pager appears; page 2 works |
| 3.4 | Type part of a name / phone / NIC in search | List filters as you type (debounced) |
| 3.5 | Click a row | Edit page opens with all fields + meta card populated |
| 3.6 | Change the phone, Save | Toast "Customer saved"; reload shows new value |
| 3.7 | Enter email as `TEST@Email.LK ` (caps + space) | Stored as `test@email.lk` (normalized) |
| 3.8 | **Delete customer** button → confirm | Gone from the list (soft delete — row kept in DB with `deleted_at`) |
| 3.9 | **Export CSV** | CSV of the current page downloads |

## 4. Vehicles

| # | Action | Expected |
|---|--------|----------|
| 4.1 | Vehicles → **+ Register vehicle**, leave reg empty, Save | Blocked: registration/brand/model required |
| 4.2 | Register `ka-1234` / Toyota / Prius, owner = Nimal | Saved; reg shows as **KA-1234** (auto-uppercase); owner name in the row |
| 4.3 | Register another vehicle with reg `KA-1234` | Blocked: "already exists" (409) |
| 4.4 | Open Nimal in Customers | Vehicle appears under **Linked vehicles** |
| 4.5 | Click the pencil on a vehicle row → change colour → Save | Updated in list |
| 4.6 | Click the wrench icon on a vehicle row | Jumps to New job card with the reg pre-filled |

## 5. Services (job cards)

| # | Action | Expected |
|---|--------|----------|
| 5.1 | Services → **+ New job card** → type `KA-12` in registration | Vehicle found: Prius + owner chip shown; mileage pre-filled |
| 5.2 | Type a reg that doesn't exist | "No vehicle found … Register it first →" |
| 5.3 | Add 2–3 checklist items (typed or Quick-add chips), pick priority **Urgent**, Create | Toast "Job card JC-2026-0001 created"; lands on the kanban; card in **Pending** with red urgent styling |
| 5.4 | Click the card → **Start work** | Card moves to **In Progress** |
| 5.5 | Click it → try to imagine skipping — there is no "Complete" offered from In Progress | Only *Mark ready* / *Cancel* are offered (state machine enforced) |
| 5.6 | **Mark ready** → then **Complete — picked up** | Card ends in **Completed**, faded |
| 5.7 | Chips **All / Today / My jobs / Overdue** | Counts are real; clicking filters the board |
| 5.8 | Leave the page open 1+ min | "Updated HH:MM:SS" refreshes (60 s auto-refresh) |

## 6. Inventory

| # | Action | Expected |
|---|--------|----------|
| 6.1 | Inventory → **+ Add item**: *Oil Filter*, category *Filters*, cost 650, sell 850, stock 10, alert 5 | Toast "Item added"; SKU `ITM-0001`; stats update |
| 6.2 | Add an item with opening stock **below** its alert level | Red low-stock banner appears with count; item qty shows red |
| 6.3 | Click **View low-stock items →** in the banner | List filters to low-stock only; chip shows "Low stock ✕" |
| 6.4 | **Restock** on an item → qty 5 | Stock increases by 5; toast confirms |
| 6.5 | Click a row → edit selling price → Save | Updated |
| 6.6 | Search by name/brand/SKU; click a category chip | Filters work |

## 7. Invoices — the money path

| # | Action | Expected |
|---|--------|----------|
| 7.1 | Invoices → open `invoice-edit.html` (New invoice) | Header "New invoice / Draft"; customer + vehicle dropdowns live |
| 7.2 | Pick Nimal → vehicle list narrows to his vehicles | Confirmed |
| 7.3 | Add line: *Oil Filter*, qty 2, price 850 (leave type **Part**) | Row total 1,700 appears live |
| 7.4 | Add line, click the type pill twice → **Labour**, qty 2, price 500 | Labour subtotal 1,000; grand total updates live |
| 7.5 | Set Discount 100 | TOTAL = 2,600.00 exactly |
| 7.6 | Pick payment **Cash**, click **Mark as paid →** | Toast "Invoice INV-2026-0001 created & paid"; back on the list, status **Paid** |
| 7.7 | Check the oil-filter stock in Inventory | **Reduced by 2** (only if the part came from inventory via API; manual description lines don't deduct) |
| 7.8 | Create another invoice, leave payment as **Pending**, click Mark as paid | Blocked: "Pick a payment method…" |
| 7.9 | Save one invoice as **draft**, open it → **Delete draft** | Allowed, and any deducted stock is returned |
| 7.10 | Open a **paid** invoice → look for Delete | Not offered (only drafts can be deleted) |
| 7.11 | Status tabs All/Draft/Sent/Paid/Overdue | Real counts; clicking filters |
| 7.12 | Open an invoice → **Print** | Browser print dialog with the invoice |

## 8. Dashboard

| # | Action | Expected |
|---|--------|----------|
| 8.1 | Open Dashboard after the tests above | Greeting summary shows real jobs-in-motion + today's revenue (2,600 from 7.6) |
| 8.2 | Live board mini-columns | Match the kanban counts |
| 8.3 | Revenue chart | Today's bar reflects the paid invoice |
| 8.4 | Low stock panel | Shows the low item from 6.2 |
| 8.5 | Activity feed | Recent actions listed ("… invoice created", "… service status changed") |

## 9. Stability & security spot-checks

| # | Action | Expected |
|---|--------|----------|
| 9.1 | `curl http://localhost:3001/api/customers` (no token) | `401 {"error":"Missing token"}` |
| 9.2 | `curl http://localhost:3001/api/nonsense` | JSON `404 {"error":"Not found"}` (not the HTML page) |
| 9.3 | Create a customer named `<script>alert(1)</script>` | Renders as literal text everywhere — no popup (XSS-escaped) |
| 9.4 | `docker compose restart api` mid-use | Console recovers on next click; refresh keeps you logged in |
| 9.5 | Reboot the laptop, run `start-beta.ps1` | Everything returns; data intact (Postgres volume). New tunnel URL printed |

---

## 10. Cleaning up the database after testing

Three levels — pick the smallest one that does the job.

### A. Wipe the test data inside the demo garage (keep logins)

Empties every business table in the demo tenant schema but keeps the garage and
its users, so you can log straight back in to a clean slate.
ID counters restart, so the next customer is `CUS-0001` again.

```powershell
docker exec carservicemanagementproject-db-1 psql -U baywork -d baywork -c "
SET search_path TO garage_qpnb3idu;
TRUNCATE customers, vehicles, services, service_items,
         invoices, invoice_parts, invoice_labor, invoice_service_lines,
         inventory_items, activity_log
RESTART IDENTITY CASCADE;"
```

> `garage_qpnb3idu` is the demo garage's schema. To find any garage's schema:
> `docker exec carservicemanagementproject-db-1 psql -U baywork -d baywork -c "SELECT name, schema_name FROM public.garages;"`

### B. Remove a test garage entirely (e.g. one created in §2)

Deletes the tenant schema **and** its registry rows. Irreversible.

```powershell
# 1. Find the schema + id of the garage to remove
docker exec carservicemanagementproject-db-1 psql -U baywork -d baywork -c "SELECT id, name, schema_name FROM public.garages;"

# 2. Replace <SCHEMA> and <ID> below, then run
docker exec carservicemanagementproject-db-1 psql -U baywork -d baywork -c "
DROP SCHEMA \"<SCHEMA>\" CASCADE;
DELETE FROM public.user_index WHERE schema_name = '<SCHEMA>';
DELETE FROM public.garages WHERE id = <ID>;"

# 3. Optionally clear its leftover registration request
docker exec carservicemanagementproject-db-1 psql -U baywork -d baywork -c "DELETE FROM public.pending_registrations WHERE email = '<the test email>';"
```

### C. Full factory reset (nuke everything, start fresh)

Destroys the Postgres volume — **all garages, all data, all logins**.

```powershell
docker compose down -v          # stop stack + delete the pgdata volume
docker compose up -d            # fresh empty database, public tables recreated on demand
cd api
npm run db:push                 # recreate public schema tables (drizzle-kit)
$env:DATABASE_URL='postgres://baywork:baywork123@localhost:5432/baywork'; npm run db:seed
# → recreates the demo garage: owner@speedauto.lk / baywork123
```

**Rule of thumb:** after each full runbook pass use **A**; use **B** for the
throwaway garage from §2; keep **C** for when you want a guaranteed-pristine
demo before showing an investor or onboarding the first real garage.

---

## Known limitations (beta, by design)

- Quick-tunnel URL changes on each restart (fix: named Cloudflare tunnel — needs one interactive `cloudflared tunnel login`).
- Registration approval has **no UI yet** — use the PowerShell snippet in §2.3.
- Customer *Date of birth* and *Tags* fields are disabled (no DB columns yet).
- Checklist items can be set at job creation; ticking them off later has API support but no UI yet.
- Invoice "internal notes" box is not persisted (only the customer-visible notes are).
- Printing uses the browser's print dialog — no PDF generation yet (post-beta item).
