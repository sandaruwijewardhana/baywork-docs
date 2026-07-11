# BayWork — Investment Prospectus & Financial Model
### Garage Management SaaS · Sri Lanka · 5-Year Plan (v3 — lean Sri-Lankan cost base)
**v3.0 · June 2026 · Prepared for investor discussions**

> 📊 **Companion spreadsheet:** [`baywork-financial-model.xlsx`](baywork-financial-model.xlsx) — every number below
> is driven by **live, connected formulas**. Edit any yellow input (customers, salaries, prices, pre-money, raise)
> and the whole model — revenue, costs, profit, valuation, ownership — recalculates automatically.

---

> ## ⚠️ Important notice — read first
> This is a **planning and negotiation model**, not a certified valuation, audited statement, or legal advice.
> The author is the founder, not a licensed financial advisor, registered valuer, or attorney-at-law.
> **Before signing anything, engage a Chartered Accountant (CA Sri Lanka) and an Attorney-at-Law / Company
> Secretary.** All figures are estimates (±20%). FX assumption: **LKR 300 = USD 1.00**.

---

## 0. What changed in v3

- **Valuation brought down to Sri-Lankan reality.** A pre-revenue SL software startup is worth far less than
  a global one — lower exit multiples (1.5–2.5× ARR, not 3–4×) and higher discount rates (65–70% IRR). Base
  pre-money is now **LKR 16M** (post-money LKR 20M ≈ USD 67K), down from the earlier LKR 40M.
- **Real per-employee salaries, all under LKR 100k/month**, rising as the company grows (see §5C). Founder takes
  **LKR 75k/month in Year 1**, increasing yearly.
- **Leaner Year 1–2 costs:** no paid business email or monitoring tools until Year 3 (free tiers used), cheaper
  cost-optimized cloud early, and a lower 2% blended payment-gateway fee. **Nothing functional was cut.**
- The revenue plan (120 → 2,000 customers) is unchanged.

---

## 1. Executive Summary — the deal in one box

| | |
|---|---|
| **Company** | BayWork — garage/workshop management SaaS for Sri Lanka |
| **Stage** | Pre-revenue, MVP near-complete |
| **Market (TAM)** | ~10,000 car/light-vehicle garages (excl. bikes/three-wheelers) |
| **Year-5 plan** | 2,000 customers (~20% of TAM) · **LKR 133M ARR (≈ USD 444K)** |
| **Recommended pre-money valuation** | **LKR 16M** (≈ USD 53K) — base case |
| **Recommended raise** | **LKR 4M** (≈ USD 13K) — growth capital + ~18-month buffer |
| **Investor stake** | **20%** |
| **Founder retains** | **80%** — full legal control preserved |
| **Share structure** | LKR 16.00 / share · 1,000,000 founder + 250,000 new shares |
| **Investor flexibility** | Amount is adjustable — see §9 and the Excel calculator |

---

## 2. Product & business model

Subscription (SaaS), three tiers:

| Tier | Target | Price/mo | ≈ USD | Key features |
|---|---|---|---|---|
| **Basic** | 1–3 mechanics · small garages | **LKR 2,999** | $10 | Customer & vehicle DB, job cards, basic invoicing, 50 SMS, 1 user |
| **Professional** ⭐ | 4–8 mechanics · medium | **LKR 5,999** | $20 | + inventory, multi-mechanic, analytics, 200 SMS, 5 users |
| **Enterprise** | 8+ mechanics · multi-location | **LKR 12,999** | $43 | + multi-location, API, custom reports, 500 SMS, unlimited users |

**Revenue levers:** annual plans (2 months free, cuts churn) · SMS top-ups · Basic→Pro upsell (expansion
revenue) · parts-supplier referral fees. **Moat:** switching cost rises with each month of customer history.

---

## 3. Market sizing (sourced)

> The earlier "1,445" was an **online-directory scrape** (Google-listed garages only — 73% Western Province,
> impossible province splits). It is the *beachhead*, not the market.

| Layer | Definition | Number | Basis |
|---|---|---|---|
| **TAM** | Car/light-vehicle service & repair garages (excl. 2-/3-wheeler) | **~10,000** | Triangulated (fleet ÷ shop · census % · intl. ratios) |
| **Beachhead** | Garages already online (easiest first wins) | 1,445 | Directory scrape |
| **SOM (Year 5)** | Our target | 2,000 | ≈ **20% of TAM** |

First-mover (<5% of market is digitised, no serious competitor); win early adopters first (Rogers).

---

## 4. Revenue / income plan

| Year | Customers | TAM pen. | ARPU/mo | MRR (LKR) | **Run-rate ARR (LKR)** | ARR (USD) | Recognized rev. (LKR) |
|---|---|---|---|---|---|---|---|
| **Y1** | 120 | ~1% | 4,600 | 552,000 | **6.6 M** | $22,080 | ~3.31 M |
| **Y2** | 400 | ~4% | 5,000 | 2,000,000 | **24.0 M** | $80,000 | ~15.31 M |
| **Y3** | 850 | ~9% | 5,250 | 4,462,500 | **53.6 M** | $178,500 | ~38.78 M |
| **Y4** | 1,400 | ~14% | 5,450 | 7,630,000 | **91.6 M** | $305,200 | ~72.56 M |
| **Y5** | 2,000 | ~20% | 5,556 | 11,112,000 | **133.3 M** | $444,480 | ~112.45 M |

*Run-rate ARR = MRR×12 (valuation basis). Recognized revenue = collected during the year (P&L basis).*

---

## 5. Cost plan — leaner Sri-Lankan base

### 5A. Technology & tools + admin (annual LKR)

> **Year 1–2 deliberately strip paid tooling** — business email, monitoring, and transactional email run on
> **free tiers** (cost = 0) until Year 3. Cloud uses the **cost-optimized stack** (Supabase/Railway/Cloudflare,
> ~$45–80/mo) early, migrating to AWS from Year 3 as the customer base grows. Payment gateway is a **blended 2%**
> (mix of card + bank transfer), not 3%. **Functional needs (hosting, DB, SMS, payments, domain) are all kept.**

| Line item | Y1 | Y2 | Y3 | Y4 | Y5 |
|---|--:|--:|--:|--:|--:|
| Cloud / hosting | 162,000 | 288,000 | 900,000 | 1,800,000 | 2,700,000 |
| ↳ *(USD/mo)* | *$45* | *$80* | *$250* | *$500* | *$750* |
| Domain + DNS | 10,000 | 10,000 | 10,000 | 10,000 | 10,000 |
| Business email | **0** | **0** | 108,000 | 172,800 | 259,200 |
| Monitoring + dev tools | **0** | **0** | 180,000 | 288,000 | 432,000 |
| Transactional email | **0** | **0** | 36,000 | 64,800 | 90,000 |
| SMS gateway (COGS)¹ | 108,000 | 360,000 | 765,000 | 1,260,000 | 1,800,000 |
| Payment gateway (2% of rev.) | 66,000 | 306,000 | 776,000 | 1,451,000 | 2,249,000 |
| Admin (secretary + accounting) | 120,000 | 180,000 | 280,000 | 380,000 | 480,000 |
| **TECH + ADMIN — ANNUAL** | **~0.47M** | **~1.14M** | **~3.05M** | **~5.43M** | **~8.02M** |
| **TECH + ADMIN — MONTHLY** | **~39K** | **~95K** | **~255K** | **~452K** | **~668K** |

¹ Largely offset by SMS top-up revenue — kept in full to stay conservative.
**One-time setup (Year 0):** company registration + secretary ≈ **LKR 25,000–40,000**.

### 5B. Full operating cost (adds salaries + marketing)

| Cost block | Y1 | Y2 | Y3 | Y4 | Y5 |
|---|--:|--:|--:|--:|--:|
| Tech + Admin (5A) | 0.47M | 1.14M | 3.05M | 5.43M | 8.02M |
| Salaries (5C) | 1.56M | 2.98M | 5.77M | 7.90M | 12.46M |
| Sales & marketing (non-salary) | 0.40M | 0.80M | 1.40M | 2.20M | 3.00M |
| **TOTAL OPERATING — ANNUAL** | **~2.43M** | **~4.92M** | **~10.23M** | **~15.52M** | **~23.48M** |
| **TOTAL OPERATING — MONTHLY** | **~202K** | **~410K** | **~852K** | **~1.29M** | **~1.96M** |

### 5C. Salary schedule — per employee, per year (all under LKR 100k/month)

**Monthly salary per person (LKR):**

| Role | Y1 | Y2 | Y3 | Y4 | Y5 |
|---|--:|--:|--:|--:|--:|
| **Founder** | 75,000 | 90,000 | 120,000 | 160,000 | 200,000 |
| Sales rep *(each)* | 55,000 | 58,000 | 62,000 | 66,000 | 70,000 |
| Support *(each)* | — | 42,000 | 45,000 | 48,000 | 52,000 |
| Developer *(each)* | — | — | 85,000 | 90,000 | 95,000 |
| Ops / manager | — | — | — | — | 90,000 |

**Headcount:**

| Role | Y1 | Y2 | Y3 | Y4 | Y5 |
|---|--:|--:|--:|--:|--:|
| Founder | 1 | 1 | 1 | 1 | 1 |
| Sales | 1 | 2 | 3 | 4 | 5 |
| Support | 0 | 1 | 2 | 3 | 4 |
| Developer | 0 | 0 | 1 | 1 | 2 |
| Ops | 0 | 0 | 0 | 0 | 1 |
| **Total people** | **2** | **4** | **7** | **9** | **13** |

**Annual salary cost per role (LKR) = headcount × monthly × 12:**

| Role | Y1 | Y2 | Y3 | Y4 | Y5 |
|---|--:|--:|--:|--:|--:|
| Founder | 900,000 | 1,080,000 | 1,440,000 | 1,920,000 | 2,400,000 |
| Sales | 660,000 | 1,392,000 | 2,232,000 | 3,168,000 | 4,200,000 |
| Support | 0 | 504,000 | 1,080,000 | 1,728,000 | 2,496,000 |
| Developer | 0 | 0 | 1,020,000 | 1,080,000 | 2,280,000 |
| Ops | 0 | 0 | 0 | 0 | 1,080,000 |
| **TOTAL SALARIES** | **1,560,000** | **2,976,000** | **5,772,000** | **7,896,000** | **12,456,000** |

> Founder pay starts **below LKR 100k** (LKR 75k) in Year 1 and rises as the company develops. Sales and
> developer pay stay **under LKR 100k/month** throughout. Sales commissions/incentives sit in the marketing
> line, not salary, so these figures are clean base pay.

---

## 6. Profitability

| | Y1 | Y2 | Y3 | Y4 | Y5 |
|---|--:|--:|--:|--:|--:|
| Recognized revenue (LKR) | 3.31M | 15.31M | 38.78M | 72.56M | 112.45M |
| Total operating cost (LKR) | 2.43M | 4.92M | 10.23M | 15.52M | 23.48M |
| **Operating profit (LKR)** | **+0.89M** | **+10.39M** | **+28.55M** | **+57.03M** | **+88.98M** |
| Operating margin | +27% | +68% | +74% | +79% | +79% |
| Cumulative profit | +0.89M | +11.28M | +39.83M | +96.86M | +185.84M |

> **Honest read of those margins.** 74–79% operating margins look high because the cost base is deliberately
> lean (sub-LKR-100k salaries, free-tier tooling, low SL overheads). **In practice you reinvest most of this
> surplus into faster growth** (more sales/support staff, marketing) — so realised *cash* margins are lower
> while you scale. The surplus is your **growth fuel**, not idle profit. The model stays profitable from Year 1,
> which means the raise is **growth capital + buffer**, not survival money.

### Unit economics
ARPU LKR 4,600→5,556 · gross margin ~85% · churn target ≤3%/mo · **LTV ≈ LKR 155K** · CAC ~LKR 8–15K ·
**LTV:CAC ≈ 10×** · CAC payback < 3 months · Rule of 40 cleared from Year 2.

---

## 7. Valuation — Sri-Lankan reality

> **Why smaller than global SaaS.** Sri Lanka has a thin acquirer market and high country/illiquidity risk, so
> exit multiples are lower (**1.5–2.5× ARR**) and investors demand higher returns (**65–70% IRR**). Both pull the
> present value down hard.

### VC Method (value the exit, discount back)
- Year-5 exit ARR **LKR 133.3M** × **2×** = **LKR 266.7M** terminal.
- Discounted at **65% IRR over 5 yrs** (÷12.23) → **LKR 21.8M** today.
- Range: 1.5× / 70% IRR → ~LKR 13M ··· 2.5× / 65% IRR → ~LKR 27M.

### Berkus (SL-scaled, pre-revenue) → ~LKR 12–15M today.

### Triangulated pre-money — today

| Scenario | Pre-money (LKR) | ≈ USD |
|---|--:|--:|
| Conservative | 8–12M | $27–40K |
| **Base (anchor)** | **~16M** | **~$53K** |
| Optimistic (after 5–10 paying garages) | 22–28M | $73–93K |

> 🔑 Land **5–10 paying garages before raising** — real revenue can lift pre-money ~50% (cheapest capital you'll
> ever get).

### Future value milestones (investor return)

| End of | ARR | Company value (2× ARR) | Investor's 20% worth |
|---|--:|--:|--:|
| Year 3 | 53.6M | ~107M | ~21M |
| Year 5 | 133.3M | ~267M | ~53M |

On a **LKR 4M** investment, a ~**LKR 53M** stake in Year 5 ≈ **~13× gross** — the upside that compensates for a
low, risky entry valuation.

---

## 8. The raise & cap table

This is a **primary** round — the **LKR 4M goes into the company** to fund growth (founder does not pocket it).

| | Value |
|---|---|
| Pre-money valuation | **LKR 16,000,000** |
| Raise | **LKR 4,000,000** |
| Post-money | **LKR 20,000,000** |
| Share price | **LKR 16.00** |
| Founder shares | 1,000,000 |
| New shares to investor | 250,000 |
| **Total shares** | **1,250,000** |
| **Investor** | **20.00%** |
| **Founder** | **80.00%** ✅ |

> Founder keeps **80%** — full board control (>50%) and special-resolution power (≥75%). Investor at 20% is below
> the 25% blocking-minority threshold.

---

## 9. ⭐ Investor Quick-Adjust Ready-Reckoner

Use the **Excel** ([`baywork-financial-model.xlsx`](baywork-financial-model.xlsx)) for live changes — type a raise
amount and the ownership updates instantly. The math (also in the sheet):

```
Post-money = Pre + Raise   ·   Investor % = Raise / (Pre + Raise)   ·   Founder % = Pre / (Pre + Raise)
Share price = Pre / Founder shares   ·   New shares = Raise / Share price
```

**Investor % for any raise, at three valuations:**

| Raise (LKR) | @ Pre **10M** | @ Pre **16M** (base) | @ Pre **25M** |
|---|--:|--:|--:|
| 2,000,000 | 16.67% | 11.11% | 7.41% |
| 3,000,000 | 23.08% | 15.79% | 10.71% |
| **4,000,000** | 28.57% | **20.00%** | 13.79% |
| 5,000,000 | 33.33% | 23.81% | 16.67% |
| 6,000,000 | 37.50% | 27.27% | 19.35% |
| 8,000,000 | 44.44% | 33.33% | 24.24% |

**Detail at base pre-money LKR 16M (LKR 16.00 / share):**

| Raise | New shares | Total shares | Investor % | Founder % | Control |
|---|--:|--:|--:|--:|:--:|
| 2,000,000 | 125,000 | 1,125,000 | 11.11% | 88.89% | ✅ |
| 3,000,000 | 187,500 | 1,187,500 | 15.79% | 84.21% | ✅ |
| **4,000,000** | **250,000** | **1,250,000** | **20.00%** | **80.00%** | ✅ |
| 5,000,000 | 312,500 | 1,312,500 | 23.81% | 76.19% | ✅ |
| 6,000,000 | 375,000 | 1,375,000 | 27.27% | 72.73% | ⚠️ investor >25% (blocking minority) |
| 8,000,000 | 500,000 | 1,500,000 | 33.33% | 66.67% | ⚠️ investor can block special resolutions |

> **Guard-rails:** keep founder **>75%** to retain special-resolution power; **never** let one early investor
> exceed **25%**. To take a bigger cheque, **raise the pre-money** (move toward the 25M column) rather than give
> up more equity.

---

## 10. Control (Companies Act No. 7 of 2007)

| Threshold | Controls | Founder at 80% |
|---|---|---|
| **>50%** ordinary resolution | Directors, accounts, dividends, day-to-day | ✅ |
| **≥75%** special resolution | Amend Articles, name, capital, restructuring | ✅ |
| **>25%** blocking minority | Block a special resolution | Investor 20% **cannot** |

Negotiate a **short** "reserved matters" veto list in the Shareholders' Agreement (selling the company, new
shares, large debt — yes; product/hiring/operations — no). Keep board majority, pre-emptive rights (s.51), and
assign all project IP to the company.

---

## 11. Sri Lankan legal checklist

- **Pvt Ltd** via Registrar of Companies (eROC); **Company Secretary mandatory**; 1–50 shareholders.
- **No par value shares** (s.49) — uses "stated capital." Resolutions: ordinary >50%, **special 75%**.
- Allotment & **pre-emptive rights** (s.51) drafted into round docs.
- **Foreign investor:** IT open to 100% foreign ownership; funds via an **Inward Investment Account (IIA)**;
  consider BOI. *(Local investor → ignore.)*
- **Tax:** corporate income tax on profit; check IT/export concessions with your CA; stamp duty on shares & SHA.
- Engage a **Chartered Accountant** + **Attorney-at-Law / Company Secretary**.

---

## 12. Founder-protection term sheet

1× **non-participating** liquidation preference · **broad-based weighted-average** anti-dilution (no full
ratchet) · reverse-vesting only with credit for work done · **short** reserved-matters list · keep pre-emptive
rights · drag-along with a price floor · ESOP ~10% (negotiate pre/post-money) · founder board majority.

---

## 13. Use of funds (LKR 4M seed)

| Use | Approx. | Why |
|---|--:|---|
| Founder runway (Year 1) | 0.90M | Full-time through Year 1 |
| First sales hire | 0.66M | The acquisition engine |
| Sales & marketing | 0.50M | Field demos, fuel, materials |
| Cloud + tools + SMS/gateway | 0.40M | ~18 months infra |
| Legal, incorporation, accounting | 0.50M | Pvt Ltd, SHA, CA |
| Contingency buffer | 1.04M | Lumpy revenue protection |
| **Total** | **4.00M** | ~18-month buffer; model is profitable from Year 1 |

---

## Appendix — assumptions (all editable in the Excel)

FX 300 · TAM ~10,000 · customers 120/400/850/1,400/2,000 · ARPU 4,600→5,556 · gross margin ~85% · churn ≤3%/mo ·
gateway 2% · SMS 150/cust/mo @ LKR 0.50 · exit multiple 2× · IRR 65% · pre-money LKR 16M · raise LKR 4M ·
founder shares 1,000,000.

**Theories used:** time value of money · risk–return · VC method (Sahlman) · Berkus · TAM/SAM/SOM · unit
economics (LTV/CAC) · Rule of 40 · operating leverage · switching-cost moat (Porter) · first-mover · diffusion
of innovation (Rogers) · pre/post-money dilution · liquidation preference · anti-dilution · pre-emptive rights ·
blocking minority.

---

*BayWork Investment Prospectus v3.0 · June 2026 · Lean SL cost base · Companion: baywork-financial-model.xlsx*
*Planning estimates only. Engage a Chartered Accountant and an Attorney-at-Law before signing.*
