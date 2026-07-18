# BayWork — User Manual (Beta)

*For garage owners and staff. No technical knowledge needed.*

---

## 1. Getting in

1. Open the BayWork link you were given in any browser (phone or computer).
2. Enter your **email** and **password**, press **Sign in**.
3. You land on the **Dashboard**. Your name and role appear at the bottom of the left menu.
4. To leave, click **Sign out** at the bottom of the menu. Always sign out on shared computers.

Forgot your password? Ask your garage owner — password reset is coming after beta.

**New garage?** Use **Register** on the login page. You'll get a reference number
(like `BW-XYZ123`). Your garage goes live after BayWork approves it — then log in
with the email and password you chose.

---

## 2. The screens at a glance

| Menu item | What it's for |
|---|---|
| **Dashboard** | Today's numbers: jobs, money, what's ready for pickup |
| **Customers** | Everyone who brings you vehicles |
| **Vehicles** | Every vehicle you've serviced, linked to its owner |
| **Services** | The job board — every vehicle currently in the workshop |
| **Invoices** | Billing: create, mark paid, print |
| **Inventory** | Parts and consumables in stock, with low-stock warnings |

Every list has a **search box** — just start typing a name, phone number, or plate.

---

## 3. Daily flow — a vehicle from gate to gate

This is the core loop you'll do many times a day:

> **Customer arrives → Job card → Work → Ready → Invoice → Paid → Vehicle leaves**

### Step 1 — First-time customer? Add them
**Customers → + Add customer.** Only **name** and **phone** are required.
BayWork gives them an ID like `CUS-0001` automatically.

### Step 2 — First-time vehicle? Register it
**Vehicles → + Register vehicle.** Enter the plate (BayWork uppercases it),
brand, model, and pick the owner. Each plate can only exist once.

### Step 3 — Open a job card
**Services → + New job card.**
- Type the **plate number** — the vehicle and owner appear automatically.
- Choose the service type and priority (**Urgent** jobs show highlighted on the board).
- Write what the customer reported.
- Add **checklist items** — what the technician must do. Use the Quick-add chips for common tasks.
- Assign a technician (you can see how busy each one is) or leave unassigned.
- **Create job card** → gets a number like `JC-2026-0001`.

### Step 4 — Track the job on the board
The Services board has four columns: **Pending → In Progress → Ready → Completed**.
Click a job card and choose:
- **Start work** — technician has begun
- **Mark ready** — done, waiting for the customer
- **Complete — picked up** — the vehicle has left
- **Cancel job** — if the work is off
- **Back to workshop** — if a "ready" vehicle needs more work

Jobs must go through the steps in order — you can't jump from Pending straight
to Completed. That's intentional: the board always tells the truth.

### Step 5 — Bill it
**Invoices → New invoice** (or from the customer's page).
- Pick the customer and vehicle.
- Add lines. Click the little type pill to switch a line between **Part / Labour / Service**.
- Parts + qty + unit price → totals calculate as you type. Add a discount if agreed.
- If a part came from your **Inventory**, stock is reduced automatically —
  and BayWork will refuse to sell more than you have.
- **Getting paid now?** Pick Cash / Card / Bank transfer and press **Mark as paid**.
- **Paying later?** Save it — it stays a **Draft**; mark it **Sent** when handed over.

Invoice numbers (`INV-2026-0001`) are automatic. **Print** uses your browser's print dialog.

---

## 4. Inventory

- **+ Add item** — name, category, cost price, selling price, current stock,
  and an **alert level**.
- When stock falls to the alert level, the item turns red and appears in the
  low-stock banner and on the Dashboard.
- **Restock** — click the item's Restock button, enter how many arrived
  (and the new cost if it changed).
- Don't edit stock numbers directly — always use Restock, so there's a record
  of every movement.

---

## 5. Dashboard — your morning glance

- **Jobs today / in motion / ready** — what needs attention right now
- **Revenue today** — money from invoices paid today
- **Pending invoices** — money still owed to you
- **Low stock** — order these before they run out
- **Activity** — who did what, most recent first

Numbers update automatically — leave it open on a screen in the office.

---

## 6. Tips & rules worth knowing

- **Nothing is ever truly lost.** "Deleting" a customer, vehicle, or job hides it
  from lists but keeps the history for your records.
- **Only draft invoices can be deleted** — a paid invoice is permanent
  (that's your accounting trail). Deleting a draft returns its parts to stock.
- **Every action is logged** — the activity feed shows who created, changed,
  or paid what, and when.
- Sessions refresh themselves — you stay signed in for up to 7 days of use;
  signing out ends it immediately.
- Your garage's data is completely separate from every other garage on BayWork —
  isolated at the database level, not just hidden.

---

## 7. If something looks wrong

| Symptom | Try |
|---|---|
| "Session expired" / bounced to login | Sign in again — your data is safe |
| A list looks stale | Refresh the page (F5) |
| "Too many login attempts" | Wait 15 minutes — this protects you from password guessing |
| Red error toast when saving | Read it — it names the field that needs fixing |
| The site won't open at all | Tell the owner — the server or tunnel may need a restart (`start-beta.ps1`) |

---

*BayWork Beta · this manual matches the July 2026 build*
