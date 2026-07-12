# Theory Hub — Harbor Auto-Deployment System

> Zero to understanding. Read these in order — each file builds on the one before.

---

## What This System Does (In One Paragraph)

A user clicks **"Create Registry"** in a web portal. Behind the scenes, the system:
1. Saves a row to a database saying "tenant X wants a registry"
2. A background worker picks it up and spins up a **full, isolated Harbor instance** inside Kubernetes (using Helm)
3. Harbor gets bootstrapped with a project and a robot account
4. Credentials are **encrypted** and stored
5. The UI polls every few seconds and shows "READY" with the registry URL

That's the whole system. The files below teach you every technology involved.

---

## Reading Order

| # | File | What You'll Understand After Reading |
|---|------|---------------------------------------|
| 01 | [Containers & Docker](./01-containers-and-docker.md) | What a container is, how Docker works, why we package apps as images |
| 02 | [Container Registries](./02-container-registry.md) | What a registry is, push/pull, tags, why Harbor instead of Docker Hub |
| 03 | [Harbor Deep Dive](./03-harbor-deep-dive.md) | All Harbor components, robot accounts, projects, vulnerability scanning |
| 04 | [Kubernetes Fundamentals](./04-kubernetes-fundamentals.md) | Pods, Deployments, Services, Namespaces, Secrets, RBAC, PVCs |
| 05 | [Helm — Kubernetes Package Manager](./05-helm-package-manager.md) | Charts, releases, values templates, install/upgrade/uninstall |
| 06 | [Go Language & REST APIs](./06-go-language-and-rest-api.md) | Go fundamentals, goroutines, Gin framework, how the API server works |
| 07 | [PostgreSQL & Database Patterns](./07-postgresql-and-database.md) | SQL, transactions, SKIP LOCKED pattern, JSONB, why we use a DB here |
| 08 | [JWT & Authentication](./08-jwt-and-authentication.md) | JWT structure, JWKS, Bearer tokens, tenant guard, dev bypass |
| 09 | [Cryptography — AES-256-GCM](./09-cryptography.md) | Symmetric encryption, nonces, why credentials are never stored in plain text |
| 10 | [Async Workers & State Machines](./10-async-workers-state-machines.md) | Why async, polling loops, the 7-step deploy flow, error handling |
| 11 | [Prometheus & Observability](./11-prometheus-and-observability.md) | Metrics, counters/histograms, ServiceMonitor, alerts, audit logs |
| 12 | [Network Policies & Tenant Isolation](./12-network-policies-and-security.md) | K8s NetworkPolicy, why tenants need isolation, ingress/egress rules |
| 13 | [Everything Together — Full Flow](./13-full-flow-end-to-end.md) | Button click → READY: the complete journey through every layer |

---

## Technology Map

```
┌─────────────────────────────────────────────────────────────┐
│                    User's Browser                           │
│              React + TypeScript + Tailwind                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP (REST)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Registry Provisioner (Go / Gin)                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │   Auth   │  │  Rate    │  │ Handlers │  │  Metrics   │  │
│  │  (JWKS)  │  │ Limiter  │  │ (CRUD)   │  │ (Prom)     │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Deploy Worker (goroutine, polls every 5s)          │    │
│  │  Step 1: Namespace  Step 2: Helm values             │    │
│  │  Step 3: Helm install  Step 4: Wait pods ready      │    │
│  │  Step 5: Harbor bootstrap  Step 6: Encrypt creds    │    │
│  │  Step 7: Mark READY                                 │    │
│  └─────────────────────────────────────────────────────┘    │
└───────┬─────────────────┬──────────────────┬────────────────┘
        │                 │                  │
        ▼                 ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
│  PostgreSQL  │  │  Kubernetes  │  │  Harbor (per tenant) │
│  (state,     │  │  API Server  │  │  registry.X.wso2.com │
│   creds,     │  │  (namespaces,│  │  ┌──────────────────┐│
│   audit)     │  │   network    │  │  │ Core  Registry   ││
└──────────────┘  │   policies)  │  │  │ Trivy JobService ││
                  └──────────────┘  │  │ Portal PostgreSQL││
                        │           │  └──────────────────┘│
                        ▼           └──────────────────────┘
                ┌──────────────┐
                │  Helm SDK    │
                │ (installs    │
                │  Harbor      │
                │  chart)      │
                └──────────────┘
```

---

## Key Vocabulary (Quick Reference)

| Term | Plain English |
|------|---------------|
| **Container** | A lightweight isolated process — like a mini-VM but sharing the host OS kernel |
| **Image** | A snapshot/blueprint that containers are created from |
| **Registry** | A server that stores and serves container images (like GitHub, but for images) |
| **Harbor** | A self-hosted enterprise container registry with scanning, RBAC, and projects |
| **Kubernetes (K8s)** | A system that runs containers at scale — schedules, restarts, networks them |
| **Helm** | A package manager for Kubernetes — like apt/npm but for K8s apps |
| **Namespace** | A logical partition inside Kubernetes — each tenant gets their own |
| **Deployment** | A K8s object that says "run N copies of this container, restart if it crashes" |
| **JWT** | A signed token that proves who you are and what permissions you have |
| **JWKS** | A public key set used to verify JWTs without calling an auth server |
| **AES-256-GCM** | Military-grade symmetric encryption — used to encrypt stored passwords |
| **Goroutine** | A lightweight thread in Go — the worker runs in one of these |
| **SKIP LOCKED** | A PostgreSQL trick so two workers don't process the same job |
| **Robot Account** | A Harbor service account for CI/CD pipelines to push/pull images |

---

Start with [01 — Containers & Docker →](./01-containers-and-docker.md)
