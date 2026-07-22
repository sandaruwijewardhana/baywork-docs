# BayWork — Feature Roadmap
**v1.0 · 2026-07-22**

Your original feature list, organized into modules, each followed by suggested
additions and a status note against the current codebase. Use the checkboxes
to track build progress feature-by-feature. `✅ scaffolded` means the DB
table/route already exists in some form; `⛔ not started` means nothing exists yet.

---

## 1. Customers

### Planned
- [ ] Full registered customer record: name, NIC, WhatsApp, mobile, join date
- [ ] Retrieve/search customer records

**Status:** ✅ scaffolded — `customers` table has name, nic, phone_primary,
whatsapp_number, created_at, plus a `draft` state for incomplete registrations.
List/search/get endpoints exist (`api/src/routes/customers.ts`).

### Suggested additions
- [ ] **Loyalty / repeat-customer tier** — auto-tag customers by visit count or
  lifetime spend (Bronze/Silver/Gold). Cheap to compute from existing invoice
  data; gives the owner a reason to treat regulars differently and gives you a
  future upsell (targeted promos to "Gold" customers).
- [ ] **Document vault per customer** — store scanned NIC/license copies, signed
  estimate approvals. Common paper-trail requirement that garages currently
  keep in a physical folder.
- [ ] **Post-service satisfaction rating** — one-tap 1–5 rating sent after a
  service is marked completed. Feeds the garage's public reputation (see §9)
  and gives the owner an early warning on bad experiences before they become
  public complaints.
- [ ] **Referral tracking** — "referred by" field on registration; referrer gets
  a small credit/discount code. Turns word-of-mouth (how most SL garages
  already grow) into something trackable and incentivized.

---

## 2. Vehicles

### Planned
- [ ] Full vehicle record: reg no, owner, starting mileage, fuel, brand, model, color
- [ ] Retrieve records
- [ ] Full service history exportable as PDF

**Status:** ✅ scaffolded (record + retrieval) — `vehicles` table covers all
listed fields. ⛔ PDF export — no PDF generation exists anywhere in the codebase
yet; needs a library choice (`pdfkit`/`puppeteer`) and a new endpoint.

### Suggested additions
- [ ] **Mileage/date-based service reminders** — "next service due at 45,000 km
  or 3 months, whichever first." This is the single most requested feature in
  every garage-software category worldwide, and almost no SL garage does it
  today. Pairs naturally with §10 messaging.
- [ ] **Insurance & revenue license expiry tracking** — two date fields +
  reminder. Sri Lanka-specific pain point (annual renewal, everyone forgets).
  A garage that reminds you before your license expires builds real loyalty.
- [ ] **Before/after photo documentation per visit** — attach photos to a
  service record (dent, existing damage, parts replaced). Protects the garage
  from "you scratched my car" disputes — a real, frequent complaint category.
- [ ] **QR sticker per vehicle** — printable QR linking straight to that
  vehicle's record. Scan at drop-off instead of typing the reg number;
  useful even on the phone-only setup (§13).
- [ ] **Resale-ready maintenance certificate** — a shareable PDF summary of
  full service history, styled for a buyer. Unique to your market: full paper
  trail from BayWork becomes a selling point when the owner resells the car,
  and it's a reason for owners to *insist* their garage uses BayWork.

---

## 3. Services

### Planned
- [ ] Service recorded per vehicle
- [ ] Customer-reported issues
- [ ] Checklist
- [ ] Assign technician + bay
- [ ] Estimated time and cost
- [ ] Status flow: pending → in progress → ready → completed

**Status:** ✅ mostly scaffolded — `services` + `service_items` tables, job
card creation, checklist items, `assignedToUserId`, `estimatedMins` per
checklist item, and the full status enum already exist
(`api/src/routes/services.ts`). ⛔ **Bay** doesn't exist anywhere in the schema
yet (no `bays` table, no bay column on services) — needs a `bays` table plus
`bayId` on `services`. ⛔ **Overall estimated cost** for the job doesn't exist
either — only per-checklist-item time estimates.

### Suggested additions
- [ ] **Customer approval step for extra work** — the single biggest
  operational gap in every workshop: mechanic finds more issues mid-job,
  garage does the work anyway, customer is shocked at pickup. Add a
  "pending customer approval" sub-state with an approve/decline link sent by
  SMS/WhatsApp (§10) before extra items are added to the invoice. This alone
  is a strong differentiator against paper-based garages.
- [ ] **Bay/lift calendar view** — once bays exist, a day-view calendar of
  which bay is occupied by which job, so reception can promise a realistic
  drop-off slot instead of guessing.
- [ ] **Warranty tracking on parts/labor** — "this part is warrantied 6 months
  / 5,000 km." Flags automatically if the same vehicle comes back with a
  related complaint inside the window, so the garage does free/discounted
  rework instead of double-billing a customer who's about to leave a bad review.
- [ ] **Job templates** — save a checklist as a reusable template ("Full
  Service — Petrol", "Brake Pad Change") so receptionists don't retype the
  same 8 checklist items every time.

---

## 4. Invoices

### Planned
- [ ] All invoices recorded
- [ ] Add parts from inventory
- [ ] Add labor cost, discounts
- [ ] Create invoice as PDF / printed

**Status:** ✅ scaffolded — `invoices`, `invoice_parts`, `invoice_labor`,
`invoice_service_lines` tables exist with discount and grand total fields
(`api/src/routes/invoices.ts`). ⛔ PDF/print output — not implemented yet.

### Suggested additions
- [ ] **VAT/tax line handling** — Sri Lanka VAT-registered garages need a tax
  line on invoices; right now there's no tax field on `invoices` at all.
  Needed before any garage above hobbyist size can use this for real
  accounting.
- [ ] **Partial payments / installments** — record multiple payments against
  one invoice with running balance. Common for bigger jobs (engine work,
  accident repair) where the customer pays in two parts.
- [ ] **Credit notes / refunds** — a linked negative invoice for returns or
  billing corrections, instead of editing/deleting the original (keeps the
  audit trail the `activity_log` table already exists to support).
- [ ] **Recurring/AMC invoices** — annual maintenance contract customers get
  auto-generated invoices on schedule. Turns one-off jobs into recurring
  revenue for the garage, which is a good retention hook for BayWork itself.

---

## 5. Inventory

### Planned
- [ ] Add items with quantity, buying price, selling price
- [ ] Auto-reduce stock from invoices
- [ ] Net profit calculation

**Status:** ✅ scaffolded — `inventory_items` with cost/selling price exists.
⛔ **Auto-deduction on invoice** — need to confirm/wire the trigger from
`invoice_parts` insert to `inventory_items.qty_in_stock` decrement (not found
in current `invoices.ts`/`inventory.ts`). ⛔ **Net profit** — dashboard
currently sums `grand_total` (revenue) only; no cost-of-goods-sold calculation
against `cost_price` yet.

### Suggested additions
- [ ] **Low-stock reorder suggestions** — `alert_threshold` already exists on
  the table; surface it as a "reorder list" the owner can screenshot and send
  to a supplier over WhatsApp. Cheap win, immediate daily value.
- [ ] **Supplier price comparison** — track cost price by supplier over time so
  the owner sees "Supplier A is 8% cheaper than Supplier B this quarter."
- [ ] **Barcode scan for stock in/out** — phone camera as a barcode scanner
  (no extra hardware) speeds up stock counts for garages with real inventory volume.
- [ ] **Shrinkage/anomaly flag** — if stock drops faster than invoiced usage
  accounts for, flag it. Protects owners from staff pilferage, a real concern
  raised in your own market-viability notes about trust in hired staff.

---

## 6. Accounts & Roles

### Planned
- [ ] Owner creates sub-accounts with different access levels
  (technician, receptionist, owner, cashier)

**Status:** ✅ scaffolded and already documented in detail — see
`roles-permissions.md`. Four roles exist in `roleEnum`. That doc already lists
`branch_manager`, `accountant`, `parts_manager`, `service_advisor` as future
roles, and field-level visibility / time-based access / two-person approval as
future permission features — treat those as part of this roadmap too, don't
duplicate the planning.

### Suggested additions
- [ ] **Per-user PIN for shared devices** — small garages often share one
  tablet/phone. A quick 4-digit PIN switch between logged-in staff (without
  full re-login) matches how they actually work.
- [ ] **Shift-based accountability** — tag each job/invoice with which shift
  (morning/evening) it was created in, useful once payroll (§14) is in.

---

## 7. Branding & White-labeling

### Planned
- [ ] Owner customizes logo, colors
- [ ] Invoice format, colors, headers customizable

**Status:** ⛔ not started — `settings.ts` currently only exposes garage
profile fields (name, phone, address, plan) and usage counts. No logo/color/
invoice-template fields exist on the `garages` table yet.

### Suggested additions
- [ ] **Logo + primary color** stored on the `garages` row, applied to the
  customer-facing invoice PDF and the garage's public site (§9) — one field
  set, two surfaces reused.
- [ ] **Invoice footer customization** — terms & conditions text, bank details
  for transfers, warranty statement. Garages want their own legal boilerplate
  on every invoice.

---

## 8. AI Assistant

### Planned
- [ ] Ask natural-language questions ("what's today's profit?")
- [ ] Add a customer or service by voice or typing, no button navigation

**Status:** ⛔ not started — no AI/LLM integration exists in the codebase yet.

### Suggested additions
- [ ] **Start read-only, then add writes** — "what's today's profit" (query)
  is far safer to ship first than "add a customer" (mutation). Ship the
  query assistant, prove it's reliable, then allow AI-triggered writes behind
  a confirmation step ("I'll create customer Kamal Perera, 077xxxxxxx — confirm?").
  Skipping the confirm step on a multi-tenant financial system is a real risk
  (misheard voice input creating wrong invoices/discounts).
- [ ] **Daily WhatsApp digest** — instead of the owner having to ask, push an
  unprompted daily summary ("Today: 12 jobs, LKR 45,000 revenue, 2 low-stock
  items") to the owner's WhatsApp every evening. Feels more "AI assistant"
  than a chat box, and needs no UI at all.
- [ ] **Anomaly narration** — "Discount on invoice #482 was 35%, higher than
  your usual 10%" — the AI's most useful job in a financial system is often
  flagging outliers, not just answering questions.

---

## 9. Garage Public Site & Customer Portal

### Planned
- [ ] Each garage gets a customizable landing page (services, offers)
- [ ] Registered customers log in to schedule bay/date/time
- [ ] Customers view their vehicle records and live service status

**Status:** ⛔ not started — no public-facing site or customer-auth flow
exists; current auth is staff-only (owner/technician/cashier/receptionist).

### Suggested additions
- [ ] **Public review display** — surface the satisfaction ratings from §1 on
  the garage's public page (with owner's ability to hide, not edit, individual
  ones). This is the strongest local differentiator: almost no SL garage has
  any online reputation today — being first gives early-adopter garages a
  visible edge over competitors in their area.
- [ ] **Live "your car is ready" status page** — a link (no login needed, just
  a token) the customer can bookmark to watch their specific vehicle's status
  without navigating a whole portal. Lower friction than requiring account
  creation for a one-off visit.
- [ ] **Slot-based booking, not just date** — real-time bay/technician
  availability so double-booking can't happen, rather than a date field the
  garage has to manually reconcile.

---

## 10. Messaging

### Planned
- [ ] Owner sends promotional messages to customers
- [ ] Automated "service started / completed" and cost messages

**Status:** ⛔ not started — `whatsapp_number` field exists on `customers`,
but no messaging provider integration exists yet.

### Suggested additions
- [ ] **WhatsApp Business API, not SMS, as the primary channel** — WhatsApp
  penetration in Sri Lanka is very high and it's free for the customer to
  receive; SMS costs the garage per message and gets ignored more. Build the
  message templates (service started/ready/completed, invoice link,
  reminder) once, send via WhatsApp first with SMS as a fallback only where
  WhatsApp delivery fails.
- [ ] **Opt-out / consent tracking** — a `marketing_opt_in` flag per customer,
  respected by the promotional-send feature specifically (transactional
  messages like "car is ready" always send). Keeps you clean if WhatsApp/SMS
  regulation tightens later — cheap to add now, painful to retrofit.
- [ ] **Template approval workflow** — WhatsApp Business API requires
  pre-approved message templates; build the send feature around fixed
  templates with variables from day one rather than free-text, or the
  integration won't work at all.

---

## 11. IoT / Diagnostics

### Planned
- [ ] Store and analyze vehicle scanner (OBD) reports

**Status:** ⛔ not started.

### Suggested additions
- [ ] **Start with manual entry, not live device integration** — let a
  technician type/paste OBD fault codes into the service record and show the
  plain-language meaning from a static code lookup table. This gets 80% of
  the customer-facing value (a fault code that means something instead of
  "P0300") with none of the hardware integration cost or vendor dependency —
  save real scanner/Bluetooth integration for once there's demand proven.
- [ ] **Fault-code history per vehicle** — recurring codes across visits are a
  strong "this needs deeper diagnosis" signal for the mechanic and a strong
  trust-building data point to show the customer.

---

## 12. Workers / Staff Operations

### Planned
- [ ] Fingerprint attendance machine integration
- [ ] Monthly performance assessment
- [ ] Salary handling
- [ ] Workers log in from phone/tablet, see assigned jobs, use checklist

**Status:** ⛔ not started, except the "see assigned jobs + checklist" half —
that's already covered by the `technician` role and `service_items` (§3/§6).

### Suggested additions
- [ ] **Phone-camera clock-in as fallback** — fingerprint hardware is a real
  cost and setup barrier for a small garage; a selfie + geofence/QR clock-in
  from the worker's own phone gets most of the value with zero hardware,
  and matches the "phone-only garages" goal in §13.
- [ ] **Commission-based pay** — % of labor value on jobs a technician
  completed, calculated automatically from completed service records.
  Common pay structure in SL garages that a flat "salary" field alone
  won't capture.
- [ ] **Leave/absence tracking** tied to the same attendance data used for
  payroll, so you're not building two separate systems.

---

## 13. Phone-Only / Low-End Device Support

### Planned
- [ ] Full system usable from a phone only, no feature loss, for small garages

**Status:** Architecturally supported already (React web app, responsive by
default) but **not verified** — no low-end-device or offline testing has been
done yet.

### Suggested additions
- [ ] **PWA with offline queueing** — service worker caches the app shell;
  writes made with no signal (job card created, checklist ticked in a bay
  with poor coverage) queue locally and sync when back online. This is the
  actual hard problem behind "phone-only, no feature loss" — worth calling
  out as its own build item rather than assuming responsive CSS covers it.
- [ ] **Low-data mode** — a setting that skips loading photos/heavy assets on
  slow connections, relevant outside Colombo where 4G is inconsistent.

---

## 14. New categories not in your original list

### Multi-branch / franchise support
If a garage owner opens a second location, they'll want one login seeing
both branches' numbers separately and combined. Needs a `branches` table and
`branchId` scoping — cheaper to design the schema for this now (even if
unused) than to retrofit multi-tenancy-within-a-tenant later. `branch_manager`
role is already noted as a future role in `roles-permissions.md`.

### Accounting export
CSV/Excel export of invoices and expenses in a format an external accountant
can import (or direct integration with whatever accounting software SL
accountants commonly use). Garages doing real money will need this for tax
filing regardless of how good your own reports are.

### Customer self-service estimate requests
A simple public form ("describe your issue, upload a photo") that creates a
draft customer + draft service request for the garage to triage — a lighter
weight entry point than full booking (§9), useful for garages that want
leads without needing customers to create accounts first.

### Data export / "no lock-in" guarantee
A one-click full data export (customers, vehicles, service history) in a
plain format. Counterintuitive but effective trust-builder for skeptical
first-time SaaS adopters in a market that's never used software before —
"you can leave anytime with your data" lowers the barrier to trying you at all.

### Multi-language UI (Sinhala / Tamil)
Given the target market, this may matter more than several items above it.
Worth deciding early since it affects how UI strings are structured in code
now, not just a translation pass added later.

---

## Suggested build order

Roughly matching what's already scaffolded vs. greenfield, cheapest-high-impact first:

1. Finish what's partially built: invoice PDF export, inventory auto-deduction on invoice, net profit calc, service estimated-cost field, bay table.
2. Service reminders + WhatsApp integration (§2, §10) — these two together are the strongest paper-to-digital pitch for a first-time customer.
3. Customer approval step for extra work (§3) — biggest daily pain point removed.
4. Branding/logo on invoices + public garage page basics (§7, §9) — needed before any garage will show BayWork to their own customers.
5. AI assistant, read-only queries first (§8).
6. Workers/payroll, IoT, multi-branch — larger builds, tackle once the core loop above is solid and in real garages' hands.
