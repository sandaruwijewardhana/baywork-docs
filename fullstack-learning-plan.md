# BayWork Founder — 3-Month Full-Stack Learning Syllabus
### From zero to building a high-end SaaS by hand (no AI)
**v1.0 · June 2026**

> **This is a syllabus, not a textbook.** It lists *what to learn and in what order* — topic names to look up,
> study, and practice. It deliberately contains **no explanations**; your job is to go learn each line.
>
> **Honest expectation.** In 3 months of focused study (target ~6–8 hrs/day) you will not become a 10-year
> "expert." You *will* become someone who can **build a full SaaS end-to-end unaided and understand every layer**.
> True mastery keeps compounding after this. This plan builds the foundation that makes that inevitable.

---

## How to use this syllabus (the method)

- **70% building, 30% reading.** You cannot learn to code by watching. Every week has a **build milestone** —
  do it without copying, without AI.
- **No AI while learning.** Use docs and your brain. AI after you already understand the concept, never before.
- **One pass, then depth.** Learn a topic just enough to build with it, then deepen when it breaks.
- **Teach-back test.** After each topic, explain it out loud as if teaching. If you can't, you don't know it yet.
- **Daily rhythm:** ~2 hrs new topic reading → ~4 hrs building → ~1 hr review/notes. Keep a written notebook.
- **Weekly:** ship the build milestone to GitHub. **Monthly:** a bigger integrated project.

---

## Core reference sources (bookmark these)

- **Roadmaps:** roadmap.sh (Frontend, Backend, Full-Stack, DevOps, SQL)
- **Curricula:** The Odin Project · freeCodeCamp · CS50 (Harvard) · Fullstackopen (University of Helsinki)
- **Docs (primary sources):** MDN Web Docs · Node.js docs · Express docs · PostgreSQL docs · Drizzle ORM docs ·
  Docker docs · TypeScript Handbook · Git (Pro Git book)
- **Depth (later):** "You Don't Know JS" (Kyle Simpson) · "Designing Data-Intensive Applications" (Kleppmann) ·
  "Refactoring UI" · OWASP Top 10 · "The Rust of the Web" — HTTP/networking specs

---

# MONTH 1 — Foundations & Frontend

## Week 1 — Computing, the web & tooling
**Outcome: understand how code and the web physically work; be fluent in the terminal and Git.**
- How the web works: client–server model, request/response, what a "server" is
- HTTP basics: methods (GET/POST/PUT/PATCH/DELETE), status codes, headers, request/response anatomy
- DNS, IP addresses, ports, URLs/URIs
- How the internet moves data: packets, TCP vs UDP (intro), the client→DNS→server→browser journey
- Browser fundamentals: what the browser does with HTML/CSS/JS (render pipeline, high level)
- Number systems (binary/hex), text encoding (ASCII, UTF-8), bits vs bytes
- Operating system & file system basics; how programs run (source → interpreter/compiler → machine)
- **Command line / Bash:** navigation, files/dirs, permissions, pipes, redirects, env vars
- **Git & GitHub:** repo, staging, commit, branch, merge, rebase, remote, push/pull, PRs, `.gitignore`, resolving conflicts
- Editor mastery: VS Code — shortcuts, extensions, integrated terminal, debugger
- **Build:** set up your dev environment; put a repo on GitHub with a proper commit history

## Week 2 — HTML & CSS
**Outcome: build any static, responsive, accessible page from a design.**
- HTML5: semantic elements, document structure, metadata, `<head>` vs `<body>`
- Forms: inputs, labels, validation attributes, `<select>`, `<textarea>`, buttons, form submission
- Accessibility (a11y): semantic markup, ARIA basics, keyboard navigation, alt text
- SEO basics: meta tags, headings hierarchy, semantic structure
- CSS fundamentals: selectors, specificity, the cascade, inheritance, box model
- Units (px/rem/em/%/vh/vw), colors, typography, spacing
- **Layout: Flexbox, CSS Grid**, positioning (static/relative/absolute/fixed/sticky), z-index
- Responsive design: media queries, mobile-first, breakpoints, fluid layouts
- CSS custom properties (variables), transitions, transforms, animations, pseudo-classes/elements
- CSS methodology: BEM naming, organizing stylesheets, design tokens
- **Build:** a multi-page responsive marketing site from scratch (no framework)

## Week 3 — JavaScript core
**Outcome: make any page interactive with vanilla JS.**
- Syntax, `var`/`let`/`const`, primitive types, type coercion, operators
- Control flow: conditionals, loops, switch, truthy/falsy
- Functions: declarations vs expressions, arrow functions, parameters/arguments, return
- Scope, hoisting, closures, the `this` keyword, execution context
- Data structures: arrays, objects, `Map`, `Set`; destructuring, spread/rest, template literals
- Array methods: `map`, `filter`, `reduce`, `forEach`, `find`, `some`, `every`, `sort`
- The DOM: selecting/creating/updating/removing elements, traversal
- Events: listeners, event object, bubbling/capturing, delegation
- Browser APIs: `localStorage`/`sessionStorage`, `fetch`, timers, `console`
- Forms in JS: reading inputs, validation, preventing default
- **Build:** an interactive CRUD app in the browser (e.g. task/customer manager) with persistence

## Week 4 — JavaScript advanced, async & TypeScript
**Outcome: handle asynchronous code and add type safety.**
- The **event loop**, call stack, task queue, microtasks (deep)
- Asynchronous JS: callbacks → Promises → `async`/`await`, error handling with try/catch
- `fetch`/AJAX in depth, working with JSON, consuming REST APIs
- Prototypes & prototypal inheritance; ES6 classes, inheritance, `static`
- Higher-order functions, pure functions, immutability, functional patterns
- Modules: ES modules (`import`/`export`) vs CommonJS
- **TypeScript:** types, interfaces vs type aliases, unions/intersections, generics, enums, `tsconfig`, type inference, narrowing, utility types
- **Build:** convert your Week-3 app to TypeScript; wire it to a real public API with async data

---

# MONTH 2 — Backend & Databases

## Week 5 — Node.js & backend fundamentals
**Outcome: understand server-side JS and build an HTTP server from scratch.**
- Node runtime & architecture, the event loop in Node, non-blocking I/O
- `npm`, `package.json`, dependencies vs devDependencies, semver, scripts
- Modules in Node (CommonJS vs ESM), `require`/`import`
- Core modules: `fs`, `path`, `http`, `os`, `events`, streams (intro), `process`, env vars
- HTTP server from the raw `http` module: routing, parsing, responses
- REST principles: resources, HTTP verbs/semantics, statelessness, idempotency, status codes
- **Build:** a REST API using **only** the raw Node `http` module (no framework)

## Week 6 — Express & API design
**Outcome: build a production-shaped REST API.**
- Express: app, routing, route params/query, `Router`, middleware chain, `next()`
- Middleware: body parsing (JSON/urlencoded), cookies, CORS, helmet, rate limiting, logging
- Error handling middleware, centralized errors, async error handling
- API design: resource modeling, versioning, pagination, filtering, sorting, search
- Request validation (Zod or Joi), DTOs, sanitization
- Project structure: routes → controllers → services → data layer (layered architecture)
- **Build:** a full CRUD REST API (customers/vehicles/services) with validation and error handling

## Week 7 — Databases & SQL
**Outcome: design a schema and write real SQL by hand.**
- Relational model: tables, rows, columns, primary keys, foreign keys
- Relationships: one-to-one, one-to-many, many-to-many (join tables)
- **SQL:** `SELECT`, `WHERE`, `INSERT`, `UPDATE`, `DELETE`, `ORDER BY`, `LIMIT`
- **JOINs** (inner/left/right/full), `GROUP BY`, aggregate functions, `HAVING`, subqueries, CTEs
- Data modeling & **normalization** (1NF/2NF/3NF), when to denormalize
- Indexes (B-tree, when/why), constraints (unique, not null, check), transactions & ACID
- PostgreSQL specifics: data types, `ENUM`, schemas, `psql` CLI, `EXPLAIN` (intro)
- **Build:** design a normalized schema on paper, create it in Postgres, and query it entirely in `psql`

## Week 8 — ORM & full-stack integration
**Outcome: connect frontend ↔ API ↔ database into one working app.**
- ORMs vs raw SQL: trade-offs; query builders
- **Drizzle ORM:** schema definitions, `select`/`insert`/`update`/`delete`, relations, joins
- **Migrations:** `drizzle-kit`, schema changes, versioning the database
- Connection pooling, transactions in code, the **N+1 query** problem
- Multi-tenancy patterns: shared-schema (tenant_id) vs **schema-per-tenant** (your architecture)
- Wiring it together: vanilla frontend → `fetch` → Express → Drizzle → Postgres
- **Build:** a complete full-stack CRUD app (frontend + API + Postgres + Drizzle), end to end

---

# MONTH 3 — Auth, Security, DevOps, Architecture & CS depth

## Week 9 — Authentication, authorization & web security
**Outcome: build a secure auth system and know how apps get attacked.**
- Password security: hashing vs encryption, **bcrypt**, salting, why never plain-text
- Sessions vs tokens; cookies (HttpOnly, Secure, SameSite), CSRF implications
- **JWT:** structure (header/payload/signature), signing/verification, expiry, access vs refresh tokens, rotation
- Authorization: role-based access control (RBAC), middleware guards, least privilege
- **Web security — OWASP Top 10:** XSS, SQL injection, CSRF, broken auth, insecure config
- HTTPS/TLS, CORS in depth, secrets management, input validation as defense
- **Build:** a complete auth system — register, login, refresh, logout, protected routes, roles

## Week 10 — DevOps & deployment
**Outcome: containerize and ship an app to the public internet.**
- **Docker:** images vs containers, `Dockerfile`, layers, `docker-compose`, volumes, networks, env config
- Linux server basics: shell, processes, logs, package managers, permissions
- Reverse proxies (nginx), HTTPS certificates (Let's Encrypt), domains & DNS records
- Hosting models: PaaS (Railway, Supabase, Render) vs IaaS (AWS EC2/ECS); managed databases
- Cloud building blocks: object storage (S3), CDN (CloudFront/Cloudflare), load balancers
- **CI/CD:** GitHub Actions — build, test, deploy pipelines; environment secrets
- Monitoring & logging basics; error tracking (Sentry)
- **Build:** dockerize your full-stack app and deploy it live with a real domain + HTTPS

## Week 11 — Computer science foundations
**Outcome: the theory that lets you answer "any question" and pass interviews.**
- **Data structures:** arrays, linked lists, stacks, queues, hash tables/maps, trees, binary search trees, heaps, graphs
- **Algorithms:** Big-O notation (time/space complexity), recursion, sorting, searching, common patterns (two-pointer, sliding window, BFS/DFS)
- **Networking deep:** TCP/IP model, TCP handshake, HTTP/1.1 vs HTTP/2 vs HTTP/3, TLS handshake, DNS resolution, sockets, the 4-tuple
- **Operating systems:** processes vs threads, concurrency vs parallelism, memory (stack/heap), scheduling, I/O, file systems
- **How databases work inside:** B-tree indexes, query planning/optimization, transactions, MVCC, WAL, isolation levels
- Caching concepts: memory hierarchy, cache invalidation, Redis basics

## Week 12 — System design, architecture & capstone
**Outcome: design a scalable SaaS and prove you can build one solo.**
- Architecture patterns: monolith, layered, **hexagonal (ports & adapters)**, microservices, event-driven — and when each is right
- Scalability: statelessness, horizontal vs vertical scaling, load balancing, caching layers, database replication/sharding (concepts)
- **System design:** how to design a SaaS from requirements → components → data model → trade-offs; CAP theorem; message queues; real-time (WebSockets vs SSE)
- API paradigms: REST vs GraphQL vs gRPC (when each)
- **Testing:** unit / integration / end-to-end, test-driven development (TDD) basics, tools (Jest, Vitest, Playwright)
- Performance: profiling, database query optimization, frontend performance (Core Web Vitals)
- **CAPSTONE:** rebuild a full SaaS (BayWork-scale: auth, multi-tenant, CRUD across 4–5 modules, invoicing, dashboard) **solo, from an empty folder, with zero AI.** This is the graduation exam.

---

## Cross-cutting threads (practice *every* week, not once)
- **Git** — commit daily, branch per feature, write clean messages
- **Debugging** — browser devtools, Node debugger, reading stack traces, `console`/breakpoints
- **Reading official docs** — build the habit of primary sources over tutorials
- **Writing** — keep notes/explanations in your own words (this cements knowledge)
- **Data structures & algorithms** — 2–3 practice problems/week (LeetCode easy→medium) from Week 4 on

---

## "Can I answer any question?" — self-assessment checklist
By the end you should be able to **explain out loud, with examples**, every item below. If you can't, revisit it.

- [ ] What happens, end to end, when you type a URL and press Enter
- [ ] How the JS event loop schedules sync vs async work
- [ ] Difference between `let`/`const`/`var`, closures, and `this`
- [ ] How the box model, flexbox, and grid lay out a page
- [ ] What a REST API is and why status codes/verbs matter
- [ ] How JWT auth works and why refresh tokens exist
- [ ] SQL joins, normalization, and when to add an index
- [ ] What an ORM does and the N+1 problem
- [ ] How schema-per-tenant isolation works (your own architecture)
- [ ] What Docker containers are and how they differ from VMs
- [ ] Big-O of common operations on arrays/maps/trees
- [ ] TCP vs UDP, the TLS handshake, HTTP/2 vs HTTP/1.1
- [ ] Processes vs threads; how transactions/MVCC keep data consistent
- [ ] Monolith vs microservices vs hexagonal — and the trade-offs
- [ ] How you'd design a scalable multi-tenant SaaS from scratch

---

## After the 3 months (the real path to "expert")
This syllabus makes you **independent**. "Expert who answers *anything*" comes from what follows:
- Build 3–5 more real projects of increasing complexity
- Read **"Designing Data-Intensive Applications"** cover to cover
- Contribute to open source; read other people's production code
- Go deep on one specialty (databases, distributed systems, or frontend performance)
- Repetition over time — expertise is accumulated reps, not a 90-day sprint

---

*BayWork Founder Learning Plan v1.0 · A map, not the territory — the learning is in the doing.*
