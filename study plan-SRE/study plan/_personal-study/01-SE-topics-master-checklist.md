# Software Engineer — Complete Topic Checklist (Simple → Complex)

**Owner:** Sandaruw · **Created:** 2026-06-22
**How to use:** ordered roughly easiest → hardest / foundational → advanced. Tick `[ ]`→`[x]` and set **Status** as you go.
**Status legend:** 🟢 confident (can teach it) · 🟡 shaky (need review) · 🔴 not yet · ⭐ = tested in your Sept interviews.
**Companion files:** [00-MASTER-PLAN-interview-crunch.md](00-MASTER-PLAN-interview-crunch.md) · [sre-mastery/00-curriculum.md](sre-mastery/00-curriculum.md)

> Nobody knows all of this *deeply*. The goal is broad awareness everywhere + depth on the ⭐ items and your role's core. Levels are a learning order, not strict prerequisites.

---

## LEVEL 0 — Computer & Programming Basics

- [ ] How a computer works: CPU, RAM, storage, I/O, the fetch-decode-execute cycle — Status:
- [ ] Binary, hex, two's complement, how text/numbers are encoded (ASCII, Unicode/UTF-8) — Status:
- [ ] What a compiler vs interpreter does; source → machine code; bytecode/JIT — Status:
- [ ] ⭐ Variables, types, operators, expressions — Status:
- [ ] ⭐ Control flow: if/else, loops, switch — Status:
- [ ] ⭐ Functions: parameters, return values, scope — Status:
- [ ] Input/output, reading files, command-line args — Status:
- [ ] Basic debugging: print debugging, reading stack traces — Status:

## LEVEL 1 — Core Programming in One Language (Java for you)

- [ ] ⭐ Primitive vs reference types; value vs reference semantics — Status:
- [ ] ⭐ Strings and string manipulation (immutability, builders) — Status:
- [ ] ⭐ Collections API cold: `ArrayList`, `HashMap`, `HashSet`, `ArrayDeque`, `PriorityQueue`, `TreeMap` — Status:
- [ ] ⭐ Classes, objects, fields, methods, constructors — Status:
- [ ] ⭐ Encapsulation, inheritance, polymorphism, abstraction (the 4 OOP pillars) — Status:
- [ ] Interfaces vs abstract classes — Status:
- [ ] Generics / parametric types — Status:
- [ ] ⭐ Exception handling: try/catch/finally, checked vs unchecked — Status:
- [ ] Enums, records, immutability — Status:
- [ ] Lambdas, streams, functional interfaces — Status:
- [ ] Memory model: stack vs heap, garbage collection basics — Status:
- [ ] Package/module structure, build tools (Maven/Gradle) — Status:

## LEVEL 2 — Data Structures ⭐ (interview core)

- [ ] ⭐ Big-O: time & space complexity, amortized analysis — Status:
- [ ] ⭐ Arrays & dynamic arrays — Status:
- [ ] ⭐ Hash tables / maps / sets (collisions, load factor) — Status:
- [ ] ⭐ Linked lists: singly, doubly, circular — Status:
- [ ] ⭐ Stacks & queues, deques — Status:
- [ ] ⭐ Binary trees & binary search trees — Status:
- [ ] ⭐ Heaps / priority queues — Status:
- [ ] ⭐ Graphs: adjacency list vs matrix — Status:
- [ ] Tries (prefix trees) — Status:
- [ ] Union-Find / Disjoint Set Union — Status:
- [ ] Balanced trees: AVL, Red-Black (awareness of guarantees) — Status:
- [ ] Segment trees, Fenwick/Binary Indexed Tree — Status:
- [ ] LRU / LFU cache structures — Status:

## LEVEL 3 — Algorithms ⭐ (interview core)

- [ ] ⭐ Sorting: bubble/insertion/selection (understand), quicksort, mergesort, heapsort — Status:
- [ ] ⭐ Binary search & its variants (lower/upper bound, on answer space) — Status:
- [ ] ⭐ Two pointers — Status:
- [ ] ⭐ Sliding window — Status:
- [ ] ⭐ Recursion & backtracking — Status:
- [ ] ⭐ Graph traversal: BFS, DFS — Status:
- [ ] ⭐ Topological sort — Status:
- [ ] ⭐ Dynamic programming: 1-D, 2-D, knapsack, LCS, edit distance — Status:
- [ ] ⭐ Greedy algorithms — Status:
- [ ] Divide & conquer — Status:
- [ ] Shortest path: Dijkstra, Bellman-Ford, Floyd-Warshall — Status:
- [ ] Minimum spanning tree: Kruskal, Prim — Status:
- [ ] Bit manipulation tricks — Status:
- [ ] String algorithms: KMP, Rabin-Karp — Status:
- [ ] Math: GCD/LCM, modular arithmetic, sieve of primes, combinatorics — Status:

## LEVEL 4 — Software Engineering Practice

- [ ] ⭐ Git: clone, commit, branch, merge, rebase, resolve conflicts, PR workflow — Status:
- [ ] ⭐ Clean code: naming, small functions, readability — Status:
- [ ] ⭐ Unit testing & mocking; TDD basics — Status:
- [ ] Integration & end-to-end testing; test coverage — Status:
- [ ] Debugging methodology (bisect, reproduce, isolate) — Status:
- [ ] Profiling & finding bottlenecks — Status:
- [ ] Refactoring techniques — Status:
- [ ] Code review etiquette (giving & receiving) — Status:
- [ ] Documentation & READMEs — Status:
- [ ] Linting, formatting, static analysis — Status:
- [ ] Agile/Scrum/Kanban, estimation, technical debt — Status:

## LEVEL 5 — Operating Systems ⭐ (verbal + SRE rounds)

- [ ] ⭐ Processes vs threads; context switching — Status:
- [ ] ⭐ Concurrency: race conditions, mutexes, semaphores, deadlock, atomics — Status:
- [ ] CPU scheduling algorithms — Status:
- [ ] Memory management: virtual memory, paging, segmentation, TLB — Status:
- [ ] File systems, inodes, I/O, buffering — Status:
- [ ] System calls, user vs kernel mode — Status:
- [ ] IPC: pipes, sockets, shared memory, signals — Status:
- [ ] ⭐ Linux: processes, `/proc`, `/sys`, signals, permissions — Status:
- [ ] ⭐ Linux: cgroups & namespaces (the basis of containers — your SRE area) — Status:
- [ ] Shell scripting (bash) — Status:

## LEVEL 6 — Networking ⭐ (your strength)

- [ ] ⭐ OSI & TCP/IP models — Status:
- [ ] ⭐ TCP vs UDP; 3-way handshake; flow & congestion control — Status:
- [ ] ⭐ IP addressing, subnets, CIDR, routing — Status:
- [ ] ⭐ DNS, DHCP, ARP — Status:
- [ ] ⭐ HTTP/HTTPS; methods, status codes, headers; REST semantics — Status:
- [ ] HTTP/2, HTTP/3, keep-alive, caching headers — Status:
- [ ] ⭐ TLS/SSL: handshake, certificates, PKI — Status:
- [ ] NAT, firewalls, VLANs — Status:
- [ ] Load balancing (L4 vs L7), reverse proxies, CDNs — Status:
- [ ] WebSockets, gRPC, long polling — Status:
- [ ] Overlay networks, tunneling (your Calico/IPIP depth) — Status:

## LEVEL 7 — Databases ⭐ (verbal + system design)

- [ ] ⭐ Relational model; SQL: SELECT, JOINs, GROUP BY, subqueries — Status:
- [ ] ⭐ Indexes: how they work (B-tree), when they help, EXPLAIN — Status:
- [ ] ⭐ ACID; transactions; isolation levels; locking — Status:
- [ ] Normalization & denormalization — Status:
- [ ] B-trees vs LSM-trees (storage engines) — Status:
- [ ] ⭐ SQL vs NoSQL: document, key-value, column-family, graph — when to use each — Status:
- [ ] ⭐ Replication, sharding, partitioning — Status:
- [ ] ⭐ CAP theorem; consistency models (strong vs eventual) — Status:
- [ ] Caching strategies: cache-aside, write-through, write-back — Status:
- [ ] Connection pooling, query optimization, deadlocks — Status:

## LEVEL 8 — Web, APIs & Security

- [ ] ⭐ HTTP request lifecycle; cookies, sessions — Status:
- [ ] ⭐ Authentication vs authorization; JWT, OAuth2, sessions — Status:
- [ ] ⭐ RBAC and access control — Status:
- [ ] REST API design & versioning; idempotency — Status:
- [ ] gRPC, GraphQL (awareness) — Status:
- [ ] Serialization: JSON, Protobuf — Status:
- [ ] Backend: web frameworks, middleware, ORMs — Status:
- [ ] Frontend basics: HTML/CSS/JS, DOM, SPA vs SSR (awareness) — Status:
- [ ] ⭐ Security: OWASP Top 10 (SQLi, XSS, CSRF) — Status:
- [ ] ⭐ Cryptography: symmetric vs asymmetric, hashing, AES, TLS (your AES-256-GCM/JWT work) — Status:
- [ ] Secrets management, input validation, least privilege — Status:

## LEVEL 9 — System Design & Architecture ⭐ (interview core)

- [ ] ⭐ Scalability: vertical vs horizontal; stateless services — Status:
- [ ] ⭐ Load balancing & caching layers (Redis/Memcached) — Status:
- [ ] ⭐ Database scaling in practice (read replicas, sharding) — Status:
- [ ] ⭐ Message queues & streaming: Kafka, RabbitMQ, pub/sub — Status:
- [ ] Microservices vs monolith; service discovery; API gateway — Status:
- [ ] Rate limiting, retries, idempotency, circuit breakers, backpressure — Status:
- [ ] Event-driven architecture; CQRS; event sourcing — Status:
- [ ] ⭐ Design patterns: singleton, factory, builder, observer, strategy, adapter, decorator — Status:
- [ ] ⭐ Design principles: SOLID, DRY, KISS, YAGNI — Status:
- [ ] ⭐ Practice designs: URL shortener, rate limiter, chat, news feed, **object store / container registry** — Status:
- [ ] Estimation: QPS, storage, bandwidth back-of-envelope math — Status:

## LEVEL 10 — DevOps, Cloud & SRE ⭐ (your edge)

- [ ] ⭐ CI/CD pipelines — Status:
- [ ] ⭐ Containers: Docker, image layers, registries (Harbor — your project) — Status:
- [ ] ⭐ Kubernetes: pods, deployments, services, controllers, CRDs, operators — Status:
- [ ] Infrastructure as Code: Terraform, Helm — Status:
- [ ] ⭐ Observability: metrics (Prometheus), logging, tracing, golden signals — Status:
- [ ] Cloud platforms: core compute/storage/network/IAM (AWS/GCP/Azure) — Status:
- [ ] SRE discipline: SLI/SLO/error budgets, incident response, runbooks, postmortems — Status:
- [ ] Reading & navigating large codebases; tracing flows to source (your curriculum skill) — Status:

## LEVEL 11 — Theory & Advanced (depth by role)

- [ ] Discrete math: logic, sets, combinatorics, probability — Status:
- [ ] Computational complexity: P vs NP, NP-complete (awareness) — Status:
- [ ] Automata, regular expressions, parsing/compilers basics — Status:
- [ ] Distributed systems: consensus (Raft/Paxos), leader election, clocks, quorum — Status:
- [ ] Concurrency at scale: lock-free structures, memory ordering — Status:
- [ ] Floating point & numerical precision — Status:
- [ ] (Optional) Machine learning / AI fundamentals — Status:

## LEVEL 12 — Professional & Soft Skills ⭐ (every interview)

- [ ] ⭐ Communicating technical decisions clearly — Status:
- [ ] ⭐ Behavioral storytelling: STAR method (6–8 stories from the registry project) — Status:
- [ ] Tradeoff reasoning & justifying design choices out loud — Status:
- [ ] Collaboration, mentoring, feedback — Status:
- [ ] Time management & prioritization — Status:
- [ ] Negotiation basics (for offers) — Status:

---

## Coverage snapshot (update as you go)

| Level | Theme | Mostly 🟢? |
|---|---|---|
| 0 | Computer & programming basics | ☐ |
| 1 | Core language (Java) | ☐ |
| 2 | Data structures ⭐ | ☐ |
| 3 | Algorithms ⭐ | ☐ |
| 4 | SE practice | ☐ |
| 5 | Operating systems ⭐ | ☐ |
| 6 | Networking ⭐ | ☐ |
| 7 | Databases ⭐ | ☐ |
| 8 | Web/APIs/Security | ☐ |
| 9 | System design ⭐ | ☐ |
| 10 | DevOps/Cloud/SRE ⭐ | ☐ |
| 11 | Theory & advanced | ☐ |
| 12 | Soft skills ⭐ | ☐ |

**Sept-interview minimum:** get Levels **2, 3, 9** to 🟢 and **5, 6, 7, 12** to at least 🟡. You're already strong on 6 and 10.
