# Phase 1 — The Foundation: the machinery that makes every repo readable

> Part of the **Code-Mastery program** ([`../00-curriculum.md`](../00-curriculum.md)). Mission: read the actual source of Kubernetes / Rancher / Harvester until you can answer anything from the code. **This phase is the literacy layer** — without it, every other repo looks like noise; with it, they all read the same way.

---

## Why this is Phase 1 (and not bare metal)

Kubernetes, Rancher, Harvester, KubeVirt, and Longhorn are **all the same shape**: a **custom resource (CRD)** + a **controller** whose reconcile function runs every time that resource changes, driving *actual state* toward *desired state*. That shape is built from five things:

1. **apimachinery** — what a Kubernetes "object" *is* in Go (types, schemes, GVK/GVR, unstructured).
2. **client-go** — how you watch and change objects (informers → workqueue → reconcile).
3. **controller-runtime** — the modern framework that wraps all of that (Manager, Controller, Reconciler).
4. **code-generator / controller-tools** — how the `zz_generated.*` files and CRDs are produced (so you can ignore them correctly).
5. **Wrangler / Lasso** — Rancher's controller framework, built on top of client-go; this is what `harvester/harvester` and `rancher/rancher` actually use.

Learn these once and you can read **all** the upper repos. Skip them and you can't read any. It's also **100% doable today** on your guest cluster — no host/BMC access needed.

---

## What you'll be able to do at the end of Phase 1

- Explain the full control loop **informer → DeltaFIFO → workqueue → reconcile** from memory, pointing at the exact `client-go` packages.
- **Write a controller from scratch** in *both* styles: raw `client-go` (sample-controller style) **and** `controller-runtime` (Kubebuilder style).
- Open any `zz_generated.*.go` file and instantly say **which generator produced it and from what** — and never waste time reading it.
- Read a **Wrangler `OnChange` controller** fluently and explain its desired-vs-actual logic with no analogies.
- Use **Delve** to set a breakpoint inside a live reconcile and inspect the object.

When all five are true, you graduate to Phase 2 (Kubernetes core).

---

## Files in this folder

| File | What it is |
|---|---|
| [`repos.md`](./repos.md) | **Exactly which repos & packages to read**, in order, with per-package checklists (so you miss no component) + official docs. |
| [`labs.md`](./labs.md) | **9 hands-on labs** (all `[GUEST]` — doable now) that build the skills above, each ending with what it proves + Anki/recall hooks. |

---

## How to work this phase (the daily loop)

1. **Read the official doc first** for the package you're studying (links in `repos.md`) — the source of truth, not a blog.
2. **Do the matching lab** (`labs.md`) — write code, run it on the guest cluster, paste real output into `../journal/`.
3. **Then read the real source** of what you just used — trace it with the method in `repos.md`.
4. **End of day:** 3–5 Anki cards + a `## Recall` block (per the Retention System in the curriculum).

Behavior first → then its source, the same day. Always.

---

## Prerequisites (finish before Lab 1)

- [ ] Go installed (`go version` ≥ 1.22) and the Go tutorial finished (you're nearly there).
- [ ] Delve installed: `go install github.com/go-delve/delve/cmd/dlv@latest` (the Go source-level debugger).
- [ ] `kubectl` working against the guest cluster (`KUBECONFIG=~/Downloads/registry-controller-cluster.yaml`).
- [ ] A `~/src/` directory to clone the foundation repos into (see `repos.md`).
- [ ] Anki + a `phase-1` deck created.

---

## Definition of Mastery — Phase 1 (your exit exam)
- [ ] From memory, draw informer → workqueue → reconcile and name the `client-go` package for each box.
- [ ] Your from-scratch `client-go` controller runs against the guest cluster (Lab 2/9).
- [ ] Your `controller-runtime` controller reconciles a CRD on the guest cluster (Lab 4/9).
- [ ] You can open a `zz_generated.deepcopy.go` and say what made it and why you skip it (Lab 5).
- [ ] You traced a Wrangler `.OnChange` down into Lasso's shared controller (Lab 6) — written in `../journal/code-traces/`.
- [ ] You hit a Delve breakpoint inside a reconcile and inspected the object (Lab 7).
