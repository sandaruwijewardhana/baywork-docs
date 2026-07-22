# Chat Handoff — BayWork (updated 2026-07-18, evening)

> Paste/read this to resume with zero context loss. Replaces the older handoff.

## Current state (all VERIFIED working)

- **Backend** (`api/`): Express 5 + TS + Drizzle, schema-per-tenant Postgres. ALL routes built & E2E-tested:
  auth (jti-fixed refresh), admin approval, customers (registered/draft/deleted lifecycle,
  phone/NIC partial-unique dup-blocking, restore, vehiclesCount, whatsappNumber/dateOfBirth/draft columns
  — migrated into existing schemas), vehicles, services (state machine), invoices (transactional create,
  server totals, stock deduction, removed/not-registered flags, summary stats), inventory (+stockValue),
  users (owner adds staff → tenant users + public.user_index in one tx), settings (GET/PATCH garage
  profile + usage), dashboard (rich GET /api/dashboard payload matching old dashboard.html shape).
- **React console** (`web/`): Vite + React 18 + TS strict + Tailwind v4 (@theme inline mapping the
  4-palette CSS vars) + React Router + TanStack Query + react-hook-form + zod. ALL pages migrated
  (Dashboard, Customers+Edit, Vehicles, Services kanban, ServiceNew, Invoices, InvoiceEdit
  builder/viewer, Inventory, Settings, Login, Register). Build passes; **Docker now serves web/dist**
  (compose volume `./web/dist:/app/public`), SPA fallback works. Old `console/` kept on disk, unserved.
- **Deploy**: `start-beta.ps1` builds web/ then `docker compose up -d` then quick tunnel.
  Secrets in root `.env` (gitignored), DB port bound 127.0.0.1. Tunnel URL changes each run.
- Zod v4 lesson applied: no `z.coerce` with RHF — form fields are strings, convert at submit boundary.
- Docs: `docs/beta-test-runbook.md` (incl. §10 DB cleanup), `docs/user-manual.md`.
- Demo login: owner@speedauto.lk / baywork123 (garage_qpnb3idu). Also tech@speedauto.lk / techpass123.

## ACTIVE TASK — design port status (updated)

User wants the React app to look **PIXEL-EXACT** like `D:\Car Service Management Project\BayWork-design\`
(the design source of truth: index.html landing 64K, dashboard.html 58K, login.html split-panel,
all other pages + styles/brand.css + styles/console.css + partials/sidebar.html + branding.md).
Also: **`/` must be the LANDING PAGE (index), not login**; console after login.

**DONE (verified live):** design CSS shared via web/public/styles + imported globally; landing.html at `/`
(links → /login /register /dashboard, dev scripts stripped); Login.tsx exact (login.css extracted);
Layout.tsx exact sidebar/topbar (+ live strip wired to /dashboard/stats + /users, mobile drawer);
Dashboard.tsx EXACT design markup w/ real data (welcome hero, capacity bays, KPI sparkline cards,
mini board, pickups timeline, revenue bars w/ 7D/30D toggle, service-mix donut computed from data,
derived smart insights, activity, low stock, top customers, leaderboard, FAB); Customers.tsx +
CustomerEdit.tsx design classes with ALL features kept (drafts/deleted/restore/WhatsApp/DOB/cancel-draft).
Per-page design CSS extracted to web/src/pages/*-design.css (import per page).
Old console archived at _archive/console-vanilla.

**REMAINING pages to reskin with design markup (logic already done, swap JSX to console.css/design classes,
import the matching *-design.css):** Vehicles, Services (kanban: services-design.css has kb-* styles),
ServiceNew, Invoices, InvoiceEdit, Inventory, Settings, Register (register-design.css = multi-step wizard).
Design references: BayWork-design/<page>.html (read markup, mirror exactly). Then rebuild web + verify + tunnel.

**Agreed approach (do NOT restyle by approximating in Tailwind):**
1. Copy `BayWork-design/styles/` + `assets/` into `web/public/` so both landing + app share the exact CSS.
2. Landing: copy `BayWork-design/index.html` → `web/public/` (e.g. landing.html), rewrite links
   (login.html→/login, register.html→/register, dashboard.html→/dashboard) and STRIP the
   unpkg react/babel “tweaks” dev scripts at the bottom. Express serves it at exact `/` BEFORE the SPA
   fallback (add `app.get('/', …)` in api/src/index.ts). React route `/` → Navigate to /dashboard (dev).
3. Import `styles/brand.css` + `styles/console.css` globally in the React app (index.css or main.tsx)
   so the design's classes (.sidebar, .sb-item, .stat-*, .kb-*, .plate, .btn…) work verbatim in JSX.
4. Rewrite `web/src/pages/Login.tsx` using the EXACT login.html markup/classes (auth-wrap, auth-brand
   pitch + plate-cluster, auth-form-panel, toggle-pw, remember row, divider, signup-cta) — already read,
   structure known. Wire to existing useAuth/RHF logic. “Back to home” → `/`.
5. Rewrite `web/src/components/Layout.tsx` with the design sidebar markup (partials/sidebar.html:
   .sb-brand/.sb-section/.sb-nav/.sb-foot with role label “Admin · <garage>”) + the design topbar from
   dashboard.html (crumbs, live strip “N staff online · N jobs active”, search box, bell).
6. Rewrite `web/src/pages/Dashboard.tsx` to match dashboard.html exactly: dark welcome hero card
   (greeting + summary sentence), workshop capacity card (B1..B10 bay grid from capacity),
   KPI strip (jobs today/revenue/avg completion/pending invoices w/ sparkline styles), workshop floor
   3-col mini board, today's-pickups timeline, revenue trend bars + service mix donut, smart insights,
   recent activity, low stock, top customers, technician leaderboard, FAB (+) menu.
   Data: existing GET /api/dashboard (has summary/capacity/kpi/liveJobs/pickups/revenue/serviceMix/
   activity/lowStock/topCustomers/leaderboard/staffOnline; avgHours currently 0 — render gracefully).
   READ `BayWork-design/dashboard.html` in chunks for exact markup before writing.
7. Then port remaining pages' markup similarly (customers/vehicles/services/invoices/inventory/settings/
   register) — same pattern: design classes, React logic already exists, swap the JSX skin.
8. Rebuild web → `docker compose up -d api` (or just rebuild web; volume picks up dist) → verify
   localhost:3001 + tunnel. Watch for Tailwind preflight vs design-CSS conflicts (preflight resets
   button/h1 styles — design CSS mostly sets them explicitly; spot-check).

Tunnel currently running in background (bq259fjg5) at
https://thoughts-documentary-mailto-gives.trycloudflare.com — dies with session; restart via start-beta.ps1.

## Standing rules (memory files exist for these)
- BayWork = learning + REAL production. No shortcuts; principled solutions; real frameworks; teach root causes.
- "explain X" → teach-to-rewrite style; "explain the logic" → bottom-up hardware→OS→mechanism style.
- Git commits: author Sandaru only, NO AI attribution.
- Keep tenantSchema.ts ↔ provisioning.ts in sync + write migrations for existing garage_% schemas.
