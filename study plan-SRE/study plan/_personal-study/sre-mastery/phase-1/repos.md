# Phase 1 — Repos & packages to read (with checklists)

> Mission: **read every component/package of the foundation layer** — not every line (skip `zz_generated.*` and `vendor/`). Tick each package as you genuinely understand it. Official docs are the source of truth; **verify paths against the live cloned repo** (they drift between versions).

---

## Clone everything first

```bash
mkdir -p ~/src && cd ~/src
# Foundation repos
git clone --depth 1 https://github.com/kubernetes/sample-controller
git clone --depth 1 https://github.com/kubernetes/client-go
git clone --depth 1 https://github.com/kubernetes/apimachinery
git clone --depth 1 https://github.com/kubernetes-sigs/controller-runtime
git clone --depth 1 https://github.com/kubernetes/code-generator
git clone --depth 1 https://github.com/kubernetes-sigs/controller-tools
git clone --depth 1 https://github.com/rancher/wrangler
git clone --depth 1 https://github.com/rancher/lasso
```
`--depth 1` = a shallow clone (latest commit only, no history) — faster and smaller; you don't need git history to read code.

> Note: `client-go`, `apimachinery`, and `api` are also *mirrors* of staging dirs inside `kubernetes/kubernetes` (`staging/src/k8s.io/...`). Reading the standalone mirror repos is simpler.

---

## The repeatable reading method (use it on every repo, all 7 phases)

```
1. Name the resource/type involved (Pod, Foo CR, ConfigMap…).
2. Find the entry point:  grep -rn "func.*Reconcile" . | head
   or for client-go style: grep -rn "AddEventHandler\|OnChange\|workqueue" .
3. Read that handler/reconcile function top to bottom.
4. Follow the clients it calls (.Get/.Update/.Create) to the next layer.
5. Stop at: the apiserver, an external binary, or a generated file — note the handoff.
6. Confirm with reality: run it, kubectl get events, or a Delve breakpoint.
7. Write the trace into ../journal/code-traces/NN-name.md.
```

---

## 1 · `kubernetes/sample-controller` — your Rosetta Stone (read 100%)

The canonical minimal client-go controller. It is small enough to **read every file**. Everything else is this, scaled up.

Official: README in the repo · concepts at kubernetes.io/docs/concepts/architecture/controller

- [ ] `controller.go` — **read every function.** This is the whole control loop: informer event handlers → workqueue → `processNextWorkItem` → `syncHandler` (the reconcile). The single most important file in Phase 1.
- [ ] `main.go` — how the clientset + informer factory + controller are wired and started.
- [ ] `pkg/apis/samplecontroller/v1alpha1/types.go` — the CRD's Go struct (`Foo`). The hand-written part.
- [ ] `pkg/apis/.../zz_generated.deepcopy.go` — **skim once to recognize generated code, then ignore.** (Lab 5 shows what made it.)
- [ ] `pkg/generated/` — the generated clientset/informers/listers. Recognize, don't memorize.

---

## 2 · `k8s.io/apimachinery` — what a Kubernetes object *is*

Official: pkg.go.dev/k8s.io/apimachinery · kubernetes.io/docs/reference/using-api/api-concepts

Read these packages (under `pkg/`):
- [ ] `apis/meta/v1` — `TypeMeta`, `ObjectMeta`, `ListMeta`, `LabelSelector`, `OwnerReference`. *Every* K8s object embeds these.
- [ ] `runtime` — `Object` interface, `Scheme` (type registry), `Codec`/`Serializer` (encode/decode). How Go structs ↔ JSON/YAML/protobuf.
- [ ] `runtime/schema` — `GroupVersionKind` (GVK) and `GroupVersionResource` (GVR). The difference between Kind and Resource is foundational.
- [ ] `apis/meta/v1/unstructured` — `Unstructured` (a typeless `map[string]interface{}` object). How the dynamic client and tools like `kubectl` handle any resource.
- [ ] `labels`, `fields`, `selection` — selectors (how `kubectl get -l app=x` works in code).
- [ ] `watch` — the `Interface` + `Event` (Added/Modified/Deleted/Bookmark). The basis of informers.
- [ ] `api/errors` — `IsNotFound`, `IsConflict`, etc. You'll check these in *every* reconcile.
- [ ] `api/meta` — `RESTMapper` (maps GVK ↔ GVR). 
- [ ] `types` — `NamespacedName`, `UID`, `PatchType`.
- [ ] `util/wait` — `Backoff`, `PollUntilContextCancel` (retry/poll primitives used everywhere).
- [ ] `util/sets`, `util/runtime`, `util/intstr`.

---

## 3 · `k8s.io/client-go` — how you watch & change objects

Official: pkg.go.dev/k8s.io/client-go · examples in the repo's `examples/` dir.

Read these packages:
- [ ] `tools/cache` — **the heart.** Read in this order: `Reflector` (list+watch from apiserver) → `DeltaFIFO` (the queue of changes) → `Indexer`/`ThreadSafeStore` (local cache) → `SharedIndexInformer` (ties them together) → `Lister` (read from cache, not apiserver). This *is* "informer → reconcile."
- [ ] `util/workqueue` — `RateLimitingInterface`, the typed/rate-limited work queue. Why reconcile takes a *key* (namespace/name), not the object.
- [ ] `kubernetes` — the generated **typed clientset** (`clientset.CoreV1().Pods(ns).Get(...)`).
- [ ] `informers` + `listers` — the generated `SharedInformerFactory` and per-type listers.
- [ ] `dynamic` — the dynamic client (works with `Unstructured`, any GVR). How operators touch resources they weren't compiled against.
- [ ] `discovery` — API discovery (what GVRs the server supports) → feeds the RESTMapper.
- [ ] `rest` — `rest.Config`, the REST client, request building.
- [ ] `transport` — auth/TLS round-trippers.
- [ ] `tools/clientcmd` — load a kubeconfig (you use this every day).
- [ ] `tools/leaderelection` — how HA controllers elect one active leader (why your provisioner can run 2 replicas safely; also how every controller-manager works).
- [ ] `tools/record` — the `EventRecorder` (how controllers emit the events you see in `kubectl describe`).

---

## 4 · `sigs.k8s.io/controller-runtime` — the modern framework

Official: **book.kubebuilder.io** (the canonical tutorial — read the "Architecture" + "Quick Start" + "Implementing a controller" chapters) · pkg.go.dev/sigs.k8s.io/controller-runtime

Read these packages (under `pkg/`):
- [ ] `manager` — `Manager`: owns the shared cache, clients, leader election, and runs all controllers. The thing `main()` starts.
- [ ] `reconcile` — `Reconciler` interface (`Reconcile(ctx, Request) (Result, error)`), `Request`, `Result` (incl. `RequeueAfter`). The contract you implement.
- [ ] `controller` — `Controller`: wires a source → workqueue → your Reconciler.
- [ ] `builder` — the `ctrl.NewControllerManagedBy(mgr).For(&Foo{}).Owns(&ConfigMap{})...` fluent API most operators use.
- [ ] `client` — the cache-backed `Client` (`Get/List` from cache, `Create/Update/Patch/Delete` to apiserver). Note `client.Reader` vs `client.Writer`.
- [ ] `cache` — the informer cache behind the client.
- [ ] `source`, `handler`, `predicate` — where events come from, how they map to reconcile Requests, and how you filter them (`Owns`, `EnqueueRequestForOwner`).
- [ ] `webhook` (+ `webhook/admission`) — validating/mutating/conversion webhooks (the same admission pattern Harvester/Rancher use heavily).
- [ ] `manager/signals` — graceful shutdown context; `log` — the logging facade.
- [ ] `pkg/client/apiutil` + `scheme` — how GVK is resolved and types are registered.

> **client-go vs controller-runtime:** sample-controller (#1) is the *raw* loop; controller-runtime *wraps* it. Read raw first (Lab 2), then the wrapper (Lab 4) — you'll see exactly what the framework hides.

---

## 5 · `k8s.io/code-generator` + `sigs.k8s.io/controller-tools` — where `zz_generated` comes from

Official: book.kubebuilder.io/reference/controller-gen · READMEs in both repos.

Understand each generator (you don't read their output — you read their *purpose*, then run them in Lab 5):
- [ ] `deepcopy-gen` → `zz_generated.deepcopy.go` (every K8s object needs `DeepCopy()`).
- [ ] `client-gen` → typed clientsets.
- [ ] `informer-gen` → `SharedInformerFactory`.
- [ ] `lister-gen` → listers.
- [ ] `conversion-gen` / `defaulter-gen` → version conversion + defaulting.
- [ ] `controller-tools`' **`controller-gen`** → deepcopy **+ CRD YAML + RBAC + webhook manifests** from `// +kubebuilder:` markers. The one Kubebuilder/operators use.
- [ ] The driver scripts: `kube_codegen.sh` / `generate-groups.sh`.

**Payoff:** after this, every `zz_generated.*` in every repo (Harvester, Rancher, KubeVirt, Longhorn) is instantly recognizable as machine output you skip.

---

## 6 · `rancher/wrangler` + `rancher/lasso` — the framework Harvester & Rancher actually use ⭐

This is the **direct bridge to Phases 5–6.** Harvester and Rancher controllers are Wrangler controllers.

Official: the repo READMEs (Wrangler/Lasso are Rancher-internal; the README + code are the docs).

**`rancher/wrangler`** (`pkg/`):
- [ ] `generic` / `controller` — the generated `XxxController` with `.OnChange(ctx, name, handler)` and `.OnRemove(...)`. Map these to client-go's informer+workqueue (it's the same thing, wrapped).
- [ ] `generated` — example generated controllers (recognize the pattern).
- [ ] `apply` — Wrangler's powerful `apply` (desired-state apply with ownership/pruning); Harvester uses it constantly.
- [ ] `schemes` / `start` — scheme registration + starting all controllers.

**`rancher/lasso`** (`pkg/`):
- [ ] `controller` — `sharedController` / `sharedHandler`: the engine under Wrangler. Trace how `.OnChange` ends up here and creates an informer.
- [ ] `cache`, `client`, `dynamic`, `mapper` — caching + dynamic client + GVK/GVR mapping (Lasso's own RESTMapper).

**Lab 6** traces a Wrangler `.OnChange` all the way into Lasso's `sharedController` → informer. Do it — that trace is the key that opens `harvester/harvester`.

---

## Reference books (optional but excellent, official-adjacent)
- **"Programming Kubernetes"** (O'Reilly, Hausenblas & Schimanski) — the best deep treatment of client-go/apimachinery/custom controllers. Pairs perfectly with #2–#3.
- **The Kubebuilder Book** (book.kubebuilder.io) — official, free, the controller-runtime path (#4).

---

## Coverage checklist (Phase 1 "miss nothing")
- [ ] sample-controller — 100% read
- [ ] apimachinery — all packages above ticked
- [ ] client-go — all packages above ticked
- [ ] controller-runtime — all packages above ticked
- [ ] code-generator + controller-tools — every generator understood + run once
- [ ] wrangler + lasso — OnChange traced to the informer

Then → [`labs.md`](./labs.md) makes each of these real.
