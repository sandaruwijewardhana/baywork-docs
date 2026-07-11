# Full-Stack Developer — Complete Learning Syllabus
### Zero → interview-ready, whole modern stack (fundamentals + frameworks)
**v2.0 · June 2026**

> **This is a syllabus, not a textbook.** It lists *what to learn and in what order* — topic names to look up,
> study, and practice. No explanations here; go learn each line. (Deep teaching notes + exercises + self-tests
> for individual topics live in [`learning/`](learning/).)
>
> **Scope change in v2.** v1 was fundamentals-only and BayWork-shaped (vanilla stack). v2 covers the **whole
> modern full-stack surface an interview tests** — React, TypeScript, Tailwind, Next.js, state management,
> GraphQL, testing, cloud, system design — while keeping the **fundamentals-first ordering** (interviewers
> punish people who know React but can't explain closures or the event loop).
>
> **Honest timeline.** Full mastery of *everything* below is realistically **~5–6 months** at 6–8 hrs/day.
> Your interview is ~3 months out, so see the **⏱️ 3-Month Interview-Priority Track** (right after "How to use").
> "Expert who can answer anything" keeps compounding after — this builds the foundation that makes it inevitable.

---

## How to use this syllabus (the method)

- **70% building, 30% reading.** Every module has a **build milestone** — do it from a blank file, no AI, no copy-paste.
- **No AI while learning.** Docs + your brain. AI only *after* you already understand a concept.
- **Fundamentals before frameworks.** Learn JS before React, SQL before ORMs, CSS before Tailwind. Non-negotiable.
- **Teach-back test.** After each topic, explain it aloud. If you can't, you don't know it yet.
- **Daily rhythm:** ~2 hrs new topic → ~4 hrs building → ~1 hr review/notes. Keep a written notebook.
- **Weekly:** ship the build to GitHub. **Per phase:** one larger integrated project.

---

## ⏱️ 3-Month Interview-Priority Track (do these first if time-boxed)

If the interview comes before you finish everything, hit these in order — they're what most full-stack interviews actually test:

1. **JS fundamentals cold** (Modules 1–4): closures, `this`, event loop, async/await, prototypes — *most-asked*.
2. **TypeScript** (Module 9) — expected on almost every modern role.
3. **React core + hooks** (Modules 10–11) — the #1 framework interviewers probe.
4. **HTML/CSS + Tailwind + responsive** (Modules 2, 12) — live-coding a UI is common.
5. **REST APIs + Node/Express** (Modules 5–6) — backend round.
6. **SQL + a database** (Module 7) — joins, indexes, N+1 come up constantly.
7. **Auth (JWT/OAuth) + web security/OWASP** (Module 18).
8. **DSA basics + Big-O** (Module 22) — coding-round gatekeeper.
9. **System design basics** (Module 23) — even junior roles ask "how would you build X".
10. **Git, testing basics, one deployment** (Modules 1, 19, 20).

Everything else deepens you; the above gets you through the door.

---

## Core reference sources (bookmark)

- **Roadmaps:** roadmap.sh (Frontend, Backend, Full-Stack, DevOps, React, System Design)
- **Curricula:** The Odin Project · freeCodeCamp · Fullstackopen (Helsinki) · CS50
- **Primary docs:** MDN · TypeScript Handbook · **React docs (react.dev)** · **Next.js docs** · **Tailwind docs** ·
  Node.js · Express · NestJS · PostgreSQL · **Prisma** · Drizzle · Redis · MongoDB · Docker · Kubernetes · **GraphQL.org** ·
  TanStack Query · Redux Toolkit · Vite · Jest/Vitest · Playwright · Git (Pro Git)
- **Depth (later):** "You Don't Know JS" · "Designing Data-Intensive Applications" · "Refactoring UI" · OWASP Top 10 ·
  "System Design Interview" (Alex Xu) · "Cracking the Coding Interview"

---

# PHASE 0 — Foundations (Weeks 1–4)

## Module 1 — Computing, the web & tooling
- Web basics: client–server, request/response, what a server is *(→ notes in `learning/`)*
- HTTP: methods, status codes, headers, request/response anatomy
- DNS, IP, ports, URLs/URIs; TCP vs UDP (intro); browser render pipeline (high level)
- Number systems, text encoding (ASCII/UTF-8), how code runs (compile vs interpret)
- **Command line / Bash**; **Git & GitHub** (branch, merge, rebase, remote, PR, conflicts)
- **Package managers:** npm vs pnpm vs yarn; `package.json`, semver, lockfiles
- Editor mastery (VS Code), **ESLint + Prettier**, EditorConfig
- **Build:** environment set up; repo on GitHub with clean history

## Module 2 — HTML & CSS (+ preprocessors)
- HTML5 semantics, forms, inputs, accessibility (a11y/ARIA), SEO basics
- CSS: selectors, specificity, cascade, box model, units, colors, typography
- **Layout: Flexbox + Grid**, positioning, responsive/mobile-first, media queries
- Custom properties, transitions, transforms, animations, pseudo-classes/elements
- CSS methodology: BEM; **Sass/SCSS** (variables, nesting, mixins); CSS Modules (awareness)
- **Build:** a multi-page responsive site from scratch

## Module 3 — JavaScript core
- Types, coercion, operators, control flow; functions, arrow functions
- **Scope, hoisting, closures, `this`, execution context** *(most-asked interview area)*
- Arrays/objects, `Map`/`Set`, destructuring, spread/rest, template literals
- Array methods (`map`/`filter`/`reduce`/`find`/`sort`…)
- DOM manipulation, events (bubbling/capturing/delegation), `localStorage`, `fetch`
- **Build:** interactive vanilla-JS CRUD app with persistence

## Module 4 — JavaScript advanced & async
- **Event loop**, call stack, task vs microtask queue (deep)
- Callbacks → **Promises → async/await**, error handling, `Promise.all/race/allSettled`
- Prototypes & prototypal inheritance; ES6 classes; `this` binding, `call`/`apply`/`bind`
- Higher-order & pure functions, immutability, currying (awareness)
- Modules (ESM vs CommonJS), generators/iterators (awareness)
- **Build:** app consuming a public API with real async data flows

---

# PHASE 1 — Core Full-Stack, no framework (Weeks 5–8)

## Module 5 — Node.js & backend fundamentals
- Node runtime, event loop in Node, non-blocking I/O
- Core modules: `http`, `fs`, `path`, `events`, streams, `process`, env vars
- Modules, npm scripts; build an HTTP server from the **raw `http` module**
- REST principles: resources, verbs/semantics, statelessness, idempotency, status codes
- **Build:** REST API with no framework

## Module 6 — Express & API design
- Routing, middleware chain, `next()`, routers, error middleware
- Body/cookie parsing, **CORS**, helmet, rate limiting, logging
- API design: versioning, pagination, filtering, sorting, search
- **Validation** with Zod/Joi; layered structure (routes → controllers → services)
- **Build:** full CRUD REST API with validation + error handling

## Module 7 — Databases & SQL (+ NoSQL intro)
- Relational model, keys, relationships (1:1, 1:N, N:M)
- **SQL:** SELECT/INSERT/UPDATE/DELETE, **JOINs**, GROUP BY, aggregates, subqueries, CTEs
- **Data modeling & normalization** (1NF–3NF), denormalization, **indexes**, constraints
- **Transactions & ACID**, isolation levels (intro); PostgreSQL specifics, `psql`, `EXPLAIN`
- **NoSQL intro:** document vs key-value vs wide-column vs graph; when relational vs NoSQL
- **Build:** design + query a Postgres DB by hand

## Module 8 — ORMs & full-stack integration
- ORM vs query builder vs raw SQL; the **N+1 problem**
- **Drizzle** and **Prisma** (schema, queries, relations, **migrations**) — know both
- Connection pooling, transactions in code
- Multi-tenancy: shared-schema vs schema-per-tenant
- **Build:** full-stack CRUD (vanilla frontend + Express + Postgres + ORM), end to end

---

# PHASE 2 — Modern Frontend Framework Stack (Weeks 9–13)

## Module 9 — TypeScript (deep)
- Types, interfaces vs type aliases, unions/intersections, literals, enums
- **Generics**, type inference, narrowing, type guards, discriminated unions
- Utility types (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`…), `keyof`, mapped/conditional types
- `tsconfig`, strict mode, declaration files, typing 3rd-party libs
- TS with Node and with React (typing props, hooks, events)
- **Build:** convert a prior JS project fully to strict TypeScript

## Module 10 — React fundamentals
- Why frameworks exist; SPA concept; **JSX**, the virtual DOM, reconciliation
- Components, props, composition, conditional & list rendering, keys
- **State: `useState`**; events; controlled inputs/forms
- **`useEffect`**, effect dependencies, cleanup, the render lifecycle
- Lifting state up; component design; **Vite** project setup
- **Build:** a multi-component React app (e.g. the BayWork console UI, in React)

## Module 11 — React advanced
- Hooks deep: `useContext`, `useRef`, `useMemo`, `useCallback`, `useReducer`, **custom hooks**
- **Performance:** re-render causes, memoization, `React.memo`, keys, code-splitting/lazy
- Patterns: compound components, render props, HOCs (awareness), error boundaries, portals
- **Forms:** React Hook Form + **Zod** validation; **routing:** React Router
- **Build:** a data-driven dashboard with routing, forms, and custom hooks

## Module 12 — Styling & UI systems
- **Tailwind CSS** (utility-first, config, responsive, dark mode, `@apply`)
- Component libraries: **shadcn/ui**, Radix, MUI, Chakra (pick one, know the landscape)
- CSS-in-JS (styled-components/Emotion — awareness), CSS Modules
- Design systems, tokens, theming; **Refactoring UI** principles; accessibility in components
- **Build:** rebuild your dashboard's UI with Tailwind + a component library

## Module 13 — State management & data fetching
- Local vs global state; **Context** limits; when you need a store
- **Redux Toolkit** (store, slices, reducers, thunks) and **Zustand** — know both
- **TanStack Query (React Query):** server state, caching, invalidation, mutations, optimistic updates
- Forms state, URL state; build tools (**Vite** deep, esbuild/Webpack awareness)
- **Build:** wire the React app to your Express API using TanStack Query + a store

---

# PHASE 3 — Meta-Framework & Advanced Backend (Weeks 14–17)

## Module 14 — Next.js (the full-stack React framework)
- App Router, file-based routing, layouts, loading/error UI
- **Rendering models:** CSR, **SSR, SSG, ISR**, streaming; **React Server Components**
- Server Actions, Route Handlers (API routes), data fetching & caching
- Middleware, metadata/SEO, image/font optimization, env config
- Deployment (Vercel); Remix (awareness)
- **Build:** rebuild the app as a Next.js full-stack app (SSR + API routes)

## Module 15 — Modern backend frameworks & architecture
- **NestJS** (modules, controllers, providers, DI, guards, pipes) and/or **Fastify**; Express recap
- Layered vs **Hexagonal (ports & adapters)** vs MVC; dependency injection
- **Prisma** deep (relations, migrations, transactions); repository pattern
- Background jobs & schedulers (BullMQ / cron)
- **Build:** re-architect the API in NestJS (or clean layered Express) with DI

## Module 16 — API paradigms & real-time
- REST recap; **GraphQL** (schema, resolvers, queries/mutations, Apollo Server/Client, N+1/DataLoader)
- **tRPC** (end-to-end type safety); gRPC & protobuf (awareness)
- **Real-time:** WebSockets, **Socket.io**, Server-Sent Events, polling — trade-offs
- Webhooks (send + receive/verify), idempotency keys
- **Build:** add a GraphQL endpoint and a real-time feature (live updates) to the app

## Module 17 — NoSQL, caching & messaging
- **MongoDB** (documents, collections, Mongoose, aggregation) — when to use vs SQL
- **Redis:** caching patterns, sessions, pub/sub, rate limiting; cache invalidation
- Message queues: BullMQ, RabbitMQ, **Kafka** (concepts, when each)
- Search: Postgres full-text, Elasticsearch (awareness)
- **Build:** add Redis caching + a queue-backed background job to the app

---

# PHASE 4 — Auth, Testing, DevOps & Cloud (Weeks 18–21)

## Module 18 — Authentication, authorization & security
- Password hashing (**bcrypt/argon2**), salting
- Sessions vs tokens; **JWT** (structure, signing, expiry, refresh, rotation); cookies (HttpOnly/SameSite/Secure)
- **OAuth 2.0 & OpenID Connect** (flows); providers: **NextAuth/Auth.js, Auth0, Clerk**
- RBAC/ABAC, middleware guards; **OWASP Top 10** (XSS, CSRF, SQLi, SSRF…), HTTPS/TLS, CORS, secrets
- **Build:** full auth (email/password + one OAuth provider + roles + protected routes)

## Module 19 — Testing
- Test types: **unit / integration / e2e**; the testing pyramid; **TDD**
- **Jest / Vitest** (assertions, mocks, spies, coverage)
- **React Testing Library** (component tests, user-event); API/integration testing (supertest)
- **e2e: Playwright / Cypress**; mocking (MSW); CI test runs
- **Build:** add a real test suite (unit + component + one e2e flow) to the app

## Module 20 — DevOps & deployment
- **Docker** deep: images, layers, multi-stage builds, `docker-compose`, volumes, networks
- Linux basics, **Nginx** reverse proxy, HTTPS (Let's Encrypt), domains/DNS
- **CI/CD: GitHub Actions** (build → test → deploy), environments, secrets
- Hosting: Vercel/Netlify (frontend), Railway/Render/Fly (backend), managed DBs
- Monorepos: Turborepo/Nx (awareness); env management, 12-factor app
- **Build:** dockerize + deploy the full app with a CI/CD pipeline and real domain + HTTPS

## Module 21 — Cloud & scaling
- **AWS core:** EC2, S3, RDS/Aurora, Lambda, CloudFront, ALB, IAM, ECS/Fargate (concepts + one hands-on)
- **Serverless** model; **Kubernetes** (pods, services, deployments — intro/awareness)
- **Infrastructure as Code:** Terraform basics
- **Observability:** structured logging, metrics, tracing, alerting; **Sentry**
- CDN, caching layers, load balancing, autoscaling, blue-green/rolling deploys
- **Build:** deploy a piece to AWS (e.g. S3+CloudFront static, or a Lambda) + add Sentry

---

# PHASE 5 — CS Foundations, System Design & Interview Prep (Weeks 22–24)

## Module 22 — Computer science foundations
- **Data structures:** arrays, linked lists, stacks, queues, hash tables, trees, BSTs, heaps, graphs, tries
- **Algorithms:** **Big-O**, recursion, sorting, searching, patterns (two-pointer, sliding window, BFS/DFS, DP intro)
- **Networking deep:** TCP/IP, TCP handshake, HTTP/1.1 vs 2 vs 3, **TLS handshake**, DNS, sockets, the 4-tuple
- **OS:** processes vs threads, concurrency vs parallelism, memory (stack/heap), scheduling, I/O
- **DB internals:** B-tree indexes, query planning, **MVCC**, WAL, isolation levels

## Module 23 — System design & architecture
- Architecture: monolith, **layered, hexagonal, microservices, event-driven, CQRS**, serverless — when each
- **Domain-Driven Design** basics; API gateway; service communication (sync vs async)
- **Scalability:** statelessness, horizontal/vertical scaling, load balancing, replication, **sharding/partitioning**
- Caching strategies, **CAP theorem**, consistency models, message queues, idempotency
- Designing a SaaS end to end: requirements → components → data model → trade-offs
- Real-time at scale, rate limiting, multi-tenancy, observability
- **Practice:** design 5–8 classic systems (URL shortener, chat, feed, rate limiter, e-commerce, the BayWork SaaS)

## Module 24 — Interview preparation
- **DSA grind:** LeetCode (easy → medium) daily; patterns over memorization; timed practice
- **System design mocks:** whiteboard the designs from Module 23 out loud
- **Behavioral:** STAR method, project deep-dives, "tell me about a hard bug"
- **Framework trivia:** React internals, TS gotchas, HTTP/DB Q&A drills
- **Mock interviews** (peer, Pramp/interviewing.io); portfolio + README polish
- **CAPSTONE:** ship a full production-grade SaaS (React/Next + typed API + DB + auth + tests + deployed), solo, no AI

---

## Cross-cutting threads (practice *every* week, not once)
- **Git** — commit daily, feature branches, conventional commits, clean PRs
- **TypeScript** — once learned (Module 9), use it everywhere after
- **Debugging** — DevTools, Node debugger, stack traces, breakpoints, bisection
- **Testing** — write tests alongside features once you learn them
- **Reading docs & others' code** — primary sources over tutorials/AI
- **DSA** — 3–5 problems/week from Module 4 onward
- **Writing** — notes/explanations in your own words (cements knowledge)

---

# APPENDIX — Blind spots (the things you'd otherwise miss)

> The modules above are the spine. These are topics beginners **don't know to look for** — they never appear in
> "learn X" courses but every real high-end SaaS needs them. Fold in *as you build*. Syllabus-only.

## A. Real-world technical topics
- **Dates, times & timezones** (UTC storage, DST) — the #1 silent bug source
- **Money & decimals** — never floats; integer cents / decimal types
- **Regular expressions (regex)**
- **Unicode & encoding**, i18n & l10n (Sinhala/Tamil, RTL)
- **File uploads** — images/PDFs, validation, storage, resizing
- **Email** — SMTP, transactional, templates, deliverability (SPF/DKIM), bounces
- **SMS & push notifications**
- **Payments** — gateway integration (Stripe / PayHere / local), webhooks, subscriptions, idempotency, refunds
- **Background jobs & queues** — retries, dead-letter queues, cron
- **Caching** — HTTP caching, Redis, invalidation, TTLs
- **Full-text search**; **pagination** (offset vs cursor); **rate limiting / throttling / idempotency**
- **Webhooks** (send + receive/verify)
- **Real-time** — WebSockets vs SSE vs polling
- **Logging & observability** — structured logs, metrics, tracing, alerting, error tracking
- **Config & secrets** — 12-factor, env separation, secret rotation
- **API docs** — OpenAPI/Swagger, Postman/Insomnia
- **Validation everywhere** (client AND server); **error handling philosophy**
- **DB ops** — backups & restore drills, seed data, soft deletes, audit logs, **migrations discipline**
- **Dependency hygiene** — lockfiles, `npm audit`, supply-chain risk
- **Browser DevTools mastery**

## B. Engineering craft & professional skills
- Clean code; **SOLID, DRY, KISS, YAGNI**; separation of concerns
- **Design patterns** (factory, strategy, observer, repository, singleton…)
- Refactoring safely (with tests); **code review**; Git workflows; semantic versioning
- Debugging methodology; reading stack traces & unfamiliar codebases
- Searching & asking well; **technical writing** (READMEs, ADRs)
- Task breakdown & estimation; editor/terminal efficiency; mental models over memorization

## C. High-end SaaS product concerns
- **UI/UX fundamentals** (layout, hierarchy, spacing, type, color; design systems)
- Product thinking; MVP scoping; **domain modeling / DDD**
- **Subscription & billing** (plans, proration, dunning, trials); admin panels
- Analytics & product metrics; UX states (onboarding, empty, loading, error, success)
- **Data privacy & compliance** (GDPR, Sri Lanka PDPA), consent, retention
- **Accessibility (WCAG)**; audit trails; reliability (backups, DR, incident response, status pages, SLAs)
- Security posture (secure defaults, threat modeling, pen-test mindset)

## D. Learning traps to avoid
- **Tutorial hell** (watching ≠ knowing) — always build from a blank file
- Copy-paste without understanding; **framework-before-fundamentals**
- Shiny-object syndrome; not reading official docs; **not shipping**
- Learning in isolation (join communities); skipping DSA; imposter syndrome & burnout — sustainable pace wins

---

## "Can I answer any question?" — self-assessment (must explain each, out loud, with examples)
- [ ] URL → Enter → page: the full journey · client–server · request/response
- [ ] Event loop; sync vs async; promises/async-await; microtasks
- [ ] `let`/`const`/`var`, closures, `this`, prototypes
- [ ] Box model, Flexbox, Grid; responsive; **Tailwind utility model**
- [ ] **React:** JSX, virtual DOM, reconciliation, hooks, re-render/perf, when effects run
- [ ] **TypeScript:** generics, narrowing, utility types
- [ ] **Next.js:** CSR vs SSR vs SSG vs ISR; RSC; server actions
- [ ] REST vs GraphQL vs tRPC; status codes/verbs
- [ ] SQL joins, normalization, indexes, transactions/isolation; N+1
- [ ] SQL vs NoSQL trade-offs; Redis caching
- [ ] JWT vs sessions; OAuth 2.0/OIDC; OWASP Top 10
- [ ] Testing pyramid; unit vs integration vs e2e
- [ ] Docker vs VM; CI/CD; one cloud deploy; Kubernetes basics
- [ ] Big-O; core data structures & their operations
- [ ] TCP vs UDP, TLS handshake, HTTP/2 vs 1.1
- [ ] Processes vs threads; MVCC/transactions
- [ ] Monolith vs microservices vs hexagonal; CAP; scaling; sharding
- [ ] Design a scalable multi-tenant SaaS from scratch

---

## After the plan (the real path to "expert")
- Build 3–5 more real projects of rising complexity
- Read **"Designing Data-Intensive Applications"** and **"System Design Interview"** cover to cover
- Contribute to open source; read production codebases
- Go deep on one specialty (frontend perf, databases, or distributed systems)
- Repetition over time — expertise is accumulated reps, not a sprint

---

*Full-Stack Learning Syllabus v2.0 · A map, not the territory — the learning is in the doing.*
