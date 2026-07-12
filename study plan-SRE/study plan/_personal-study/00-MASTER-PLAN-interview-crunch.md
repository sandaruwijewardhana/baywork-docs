# Master Plan — Interview Crunch (Jun 22 → Sep 1, 2026)

**Owner:** Sandaruw · **Created:** Mon 2026-06-22 · **Hard deadline:** interviews **Sept week 1 (~Sep 1–7)**
**Constraint:** onsite internship **7:30am–7:30pm**, focused on the Registry project. Free study time = early morning + evening + weekends only.
**Decisions locked:** target roles = **both SWE + SRE/Platform** · **DSA language = Java** · NeetCode-150 roadmap.

> **The mission until Sep 1:** be able to (a) solve any DSA problem cold, (b) hold a system-design conversation, (c) tell a compelling story about the Registry project, and (d) answer SRE/Linux/K8s depth questions. After Sep 1, resume the full SRE code-mastery curriculum (`sre-mastery/00-curriculum.md`).

---

## The 5 things, and how they actually relate (stop treating them as 5 separate jobs)

| You said | Reality in this plan |
|---|---|
| **SRE mastery** | Throttled. Only Module 0 + the 2 PRs + system-design fuel run before Sep 1. Full curriculum resumes after. |
| **Registry project (internship)** | This is your **day job (12h)** AND your #1 interview asset. Don't "study" it after hours — *mine it for stories while you work.* |
| **curriculum.md** | The `sre-mastery/00-curriculum.md` plan. Paused-but-warm; PR milestones kept because they're interview proof. |
| **LeetCode / DSA** | **THE priority.** Hardest gate, the thing you must not be weak on. Gets your freshest hour every day. |
| **SE jobs** | The umbrella. = DSA + system design + behavioral/STAR + resume + the project story + referrals. |

**Priority order of your free hours:** DSA  >  interview prep (system design + behavioral + project story)  >  the 2 PRs  >  everything else in the SRE curriculum.

---

## TIMELINE — 10 weeks to Sep 1

| Week | Dates | DSA topic (NeetCode-150, Java) | Interview / systems / project |
|---|---|---|---|
| **0 setup** | this week | Big-O refresher · **Arrays & Hashing** · **Two Pointers** (~15 problems) | Resume v1 draft · pick target companies · request referrals · set up LeetCode + Anki |
| **1** | Jun 22–28 | finish Arrays/Two-Pointers · **Sliding Window** | Resume v1 done · System Design fundamentals (scaling, LB, caching) |
| **2** | Jun 29–Jul 5 | **Stack** · **Binary Search** | Resume v2 polished + LinkedIn/GitHub · 6 STAR stories drafted from project |
| **3** | Jul 6–12 | **Linked List** · start **Trees** | 🎯 **PR #1 merged** (docs/example fix — workflow practice) · SysDesign: DB sharding/replication, CAP |
| **4** | Jul 13–19 | **Trees** (BFS/DFS, BST) | SysDesign: queues, consistency · practice design: URL shortener, rate limiter |
| **5** | Jul 20–26 | **Tries** · **Heap / Priority Queue** | SysDesign: **design a container registry** (you'll crush this) · STAR stories refined |
| **6** | Jul 27–Aug 2 | **Backtracking** · **Graphs** (DFS/BFS) | Module 0 controller pattern (helps PR #2 + SRE Qs) · 1st **mock interview** |
| **7** | Aug 3–9 | **Graphs** (topo, union-find) · **1-D DP** | 🎯 **PR #2 attempt** (kube-vip / longhorn good-first-issue = real proof) · mock #2 |
| **8** | Aug 10–16 | **2-D DP** · **Greedy** | SysDesign: design monitoring/SLO system (SRE flavor) · mock #3 |
| **9** | Aug 17–23 | **Intervals** · **Advanced Graphs** · **Bit/Math** | Behavioral polish · company-specific research · mock #4 |
| **10** | Aug 24–31 | **Full timed mocks** + redo weakest topics + pattern review | Final story rehearsal · logistics · **sleep & taper** |

**Why this order:** DSA topics build on each other (arrays→pointers→windows→stack→search→lists→trees→graphs→DP). Mocks start Week 6 — solo solving ≠ interview performance; you need reps under pressure. PRs land before September so they're live on your GitHub when interviewers look.

---

## DAILY TIMETABLE — weekdays (onsite 7:30–7:30)

| Time | Block | Notes |
|---|---|---|
| 05:45–06:45 | **DSA — 1 problem, full effort** | Freshest brain → hardest task first. New problem or yesterday's redo. |
| 06:45–07:20 | Commute / prep | — |
| 07:30–19:30 | **Onsite — Registry project** | *While working:* note 1 thing worth a STAR story or a "why does it work this way" question. Use lunch (~15 min) for **Anki reviews**. |
| 20:00–20:30 | Dinner / reset | Protect this. |
| 20:30–22:00 | **Main block (1.5h)** — rotates daily | Mon/Wed: System Design · Tue/Thu: 2nd DSA problem · Fri: PR work or SRE Module-0 lab |
| 22:00–22:20 | **Anki cards + journal `## Recall`** | 3–5 cards on today's DSA *pattern* (not the specific problem) + 1 recall question. |
| 22:20–23:00 | Wind down → **sleep by 23:00** | Non-negotiable. 7h sleep beats a 6th study hour. |

**Weekday total ≈ 2.5–3h focused study + Anki.** That's enough *because weekends carry the volume.*

## DAILY TIMETABLE — weekends (the engine)

**Saturday (~7h)**
- 08:30–11:30 **DSA deep block** (3h): 2–3 new problems on the week's topic, write clean Java, then read the optimal solution.
- 12:30–14:00 **System design** (1.5h): one full design end-to-end, drawn out.
- 15:00–17:00 **PR work / SRE Module-0 lab** (2h) on your guest cluster.
- 17:00–17:30 Anki + log the week.

**Sunday (~6h)**
- 08:30–11:00 **DSA** (2.5h): mock-style timed set OR redo the week's hardest problems *from memory* (Friday "redo" ritual lives here).
- 12:00–13:00 **Behavioral / STAR + resume** (1h).
- 14:00–16:00 **SRE curriculum lab or PR** (2h) — only after DSA is done.
- 16:00–16:30 **Weekly review**: tick this plan, pick next week's targets, export Anki deck.
- **Take Sunday evening OFF.** Recovery is part of the plan.

**Weekly total ≈ 28h outside work.** Sustainable *only* if you protect sleep and the Sunday-evening break.

---

## TODO LIST — by track

### A. DSA (the priority)
- [ ] Create a LeetCode account; follow the **NeetCode 150** list. Solve in **Java**.
- [ ] Week 0: re-derive Big-O for every data structure; memorize Java's `ArrayList`/`HashMap`/`ArrayDeque`/`PriorityQueue`/`TreeMap` APIs cold (these are your interview toolkit).
- [ ] One Anki card per *pattern* you learn (e.g. "two-pointer when array is sorted", "BFS for shortest unweighted path"), not per problem.
- [ ] From Week 6: 1 mock interview/week (Pramp or interviewing.io, free) — solve out loud, timed.
- [ ] Keep an "errors log": every problem you couldn't solve → why → redo it 3 days later.

### B. System Design
- [ ] Read **Grokking the System Design Interview** OR the free **github.com/donnemartin/system-design-primer** (one source, cover-to-cover).
- [ ] Fundamentals first: load balancing, caching, DB replication/sharding, CAP, message queues, consistency models, rate limiting.
- [ ] Practice designs: URL shortener · rate limiter · news feed · chat · **object storage / container registry** (your home-field advantage — practice explaining Harbor's architecture as a design answer).
- [ ] SRE-flavored: design a metrics/monitoring system; design for an SLO; capacity planning.

### C. Registry project → interview asset
- [ ] Write **6–8 STAR stories** (Situation-Task-Action-Result). Your memory files already hold the raw material: the KVI operator pattern, the **stale-Harbor recovery procedure**, the **schema migration fix for `registry_name`**, **AES-256-GCM** secret handling, **JWT auth flow**, **DB worker locking**, multi-tenant isolation. Each = one story.
- [ ] Be able to whiteboard the system architecture in 3 minutes (CR → controller → Harbor + Longhorn on Harvester).
- [ ] Prepare answers for: "hardest bug you debugged", "a design tradeoff you made", "something you'd do differently".

### D. The 2 PRs (proof of skill — keep on schedule)
- [ ] **PR #1 (by ~Jul 12):** low-risk docs/example fix in Harvester / Longhorn / kube-vip docs. Learn DCO `git commit -s` + CLA + good-first-issue workflow.
- [ ] **PR #2 (by ~Aug 9):** real code change via `label:good-first-issue` in **kube-vip** (top pick) or **longhorn-manager**. This is the line on your résumé that beats a certificate.

### E. Resume / applications / referrals
- [ ] Resume v2 by end of Week 2: quantify the registry project impact; list the stack (Go, K8s operators, Harbor, Longhorn, Harvester/RKE2).
- [ ] LinkedIn + GitHub profile aligned with resume; pin the registry work and the PRs.
- [ ] **Ask for referrals now** — referrals multiply callback rates and need lead time. List 10 target companies; reach out to any contacts this week.
- [ ] Research each company that's interviewing you (product, stack, recent news) the weekend before.

### F. Other things worth focusing on (you asked)
- [ ] **Mock interviews with real humans** — the single highest-ROI add beyond solo grinding.
- [ ] **CS fundamentals refresh** for the verbal rounds: OS (processes/threads/scheduling/memory), concurrency, databases (indexes, transactions, isolation), networking (you're already strong). 1 topic/week, light.
- [ ] **One public writeup** (blog/GitHub gist) of a code trace or the registry architecture — signals communication, which interviewers weight heavily.
- [ ] **Offer negotiation** basics — read up before any offer call; don't accept on the spot.
- [ ] **Health = part of the plan.** 7h sleep, daily movement, Sunday evening off. Burnout in week 6 loses you the interviews; pace it.

---

## What I am deliberately NOT doing before Sep 1 (and why)
- The deep **Phase 3–7** SRE code dives (KubeVirt, Longhorn engine, Rancher provisioning internals). They're multi-week and don't move the interview needle faster than DSA does. **Resume them Sep 8+** with the original `sre-mastery/00-curriculum.md`.
- Hardware/BMC/immutable-OS track — still blocked on access, and lowest interview relevance. Keep the access request filed; act when granted.

## Definition of "ready" (the Sep 1 bar)
- [ ] Can solve a random NeetCode-150 medium in <30 min, talking through it, in Java.
- [ ] Can drive a system-design discussion for 40 min without freezing.
- [ ] Can tell 6+ crisp STAR stories from the registry project.
- [ ] PR #1 merged, PR #2 open or merged.
- [ ] Resume + LinkedIn + referrals out for 10 target companies.
