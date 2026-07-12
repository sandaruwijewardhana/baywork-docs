# Phase 1 — Labs (hands-on, all `[GUEST]`, doable today)

> Every lab runs against your guest cluster (`KUBECONFIG=~/Downloads/registry-controller-cluster.yaml`) — **no host/BMC access needed.** Each lab: do it → paste real output into `../journal/` → read the source for what you just used → make Anki cards. Order matters: 1→9 builds the foundation in dependency order.

> Reminder on `kubectl debug`/RBAC: you have full kubectl on this cluster, so all of these work. If a `Create`/CRD apply is denied, you may need the namespace you own — use a namespace you control.

---

## Lab 1 — Run the Rosetta Stone (sample-controller)

**Goal:** see the canonical client-go control loop fire on a real cluster, then read every line of it.

1. `cd ~/src/sample-controller && go build -o sample-controller .` — compile it.
2. Apply its CRD: `kubectl apply -f artifacts/examples/crd.yaml` (defines the `Foo` resource). *(Confirm the path with `ls artifacts/examples/` — names drift.)*
3. Run it: `./sample-controller -kubeconfig=$HOME/Downloads/registry-controller-cluster.yaml`
4. In another terminal, create a `Foo`: `kubectl apply -f artifacts/examples/example-foo.yaml`
5. Watch the controller log: it reconciles the `Foo` and creates a `Deployment`. Edit the Foo's replicas, watch it re-reconcile. Delete the Deployment by hand, watch it recreate it (that's the control loop — desired vs actual).
6. **Add your own line:** in `controller.go`'s `syncHandler`, add `klog.Infof("RECONCILING foo=%s/%s", namespace, name)`, rebuild, rerun, watch it fire.

**Read:** `controller.go` top to bottom — `processNextWorkItem` → `syncHandler`. Map each piece to the diagram in `repos.md`.
**Proves:** you can run, modify, and read a real controller.
**Anki:** "control loop = observe desired (spec) vs actual (status) → make them match → repeat"; "reconcile gets a *key* (ns/name), then re-fetches the object — why? (the cached object may be stale)".

---

## Lab 2 — Build an informer from scratch (raw client-go)

**Goal:** reproduce informer → workqueue → handler yourself, so the framework later hides nothing.

Write a program that:
1. Loads kubeconfig (`clientcmd.BuildConfigFromFlags`).
2. Creates a clientset + a `SharedInformerFactory` (`informers.NewSharedInformerFactory(clientset, 30*time.Second)`).
3. Gets the Pod informer, adds event handlers (`AddFunc/UpdateFunc/DeleteFunc`) that push the object's key onto a `workqueue.RateLimitingInterface`.
4. Runs a worker loop: `queue.Get()` → log "reconciling pod X" → `queue.Forget()`/`Done()`.
5. Starts the factory and blocks.

Run it; create/delete a pod in any namespace; watch your worker log.

**Read:** while it runs, open `client-go/tools/cache/shared_informer.go` and `util/workqueue/`.
**Proves:** you understand the loop at the raw level.
**Anki:** "why a workqueue between informer and worker? → decouple event rate from processing; dedupe; rate-limit/retry"; name the 3 cache pieces: Reflector, DeltaFIFO, Indexer.

---

## Lab 3 — Trace the cache internals (reading, not writing)

**Goal:** know exactly how `tools/cache` turns a watch into a local store.

Trace and write a note in `../journal/code-traces/01-informer.md`:
- `Reflector.ListAndWatch` → where does it call the apiserver?
- How does a watch `Event` become a `Delta` in `DeltaFIFO`?
- How does `processDeltas` update the `Indexer` (the thread-safe store)?
- How does a `Lister` read from that store (and why is it cheap / apiserver-free)?

**Proves:** you can read core Kubernetes library code unaided.
**Anki:** "Lister reads from the local informer cache, not the apiserver — that's why `List` in a reconcile is cheap".

---

## Lab 4 — Build a controller-runtime controller (Kubebuilder)

**Goal:** the modern framework, and see what it wraps from Labs 2–3.

1. `go install sigs.k8s.io/kubebuilder/v4/cmd/kubebuilder@latest` (verify current install command on book.kubebuilder.io).
2. Scaffold: `kubebuilder init --domain lkdc.wso2.com --repo lkdc.dev/echo` then `kubebuilder create api --group demo --version v1 --kind Echo`.
3. In the generated `internal/controller/echo_controller.go`, implement `Reconcile`: fetch the `Echo`, create/update a ConfigMap named after it with its `spec.message`, set `status.ready=true`. Handle `apierrors.IsNotFound` (deleted) by returning `nil`.
4. `make install` (applies the CRD), `make run` (runs against your guest cluster).
5. `kubectl apply` an `Echo` CR; watch the ConfigMap appear; edit the message; watch it update.

**Read:** the generated `cmd/main.go` — find the `Manager`, the `NewControllerManagedBy(mgr).For(&Echo{}).Owns(&ConfigMap{})` builder. Compare to Lab 2: the Manager *is* your informer factory + workqueue + leader election, wrapped.
**Proves:** you can build a real operator.
**Anki:** "`Owns(&ConfigMap{})` → re-reconciles the parent when an owned ConfigMap changes (owner-reference watch)"; "`Result{RequeueAfter: t}` → retry later without an error".

---

## Lab 5 — Run the code generators (demystify `zz_generated`)

**Goal:** never be confused by generated code again.

1. In your Lab-4 project, look at `zz_generated.deepcopy.go` — note the `// +kubebuilder:` and `// +k8s:deepcopy-gen` markers in `echo_types.go` that produced it.
2. Delete `zz_generated.deepcopy.go`, then run `make generate` (which runs `controller-gen object`). Watch it regenerate identically.
3. Run `make manifests` (runs `controller-gen crd rbac`) — watch it regenerate the CRD YAML + RBAC from your markers.
4. (Optional) Clone a real repo (e.g. `harvester/harvester`), find a `zz_generated.deepcopy.go`, and confirm you recognize it as the same machine output.

**Proves:** you can tell hand-written from generated instantly — and skip generated correctly.
**Anki:** "`controller-gen object` → deepcopy; `controller-gen crd` → CRD YAML; `controller-gen rbac` → Role/ClusterRole — all from `// +kubebuilder:` markers".

---

## Lab 6 — Trace Wrangler `.OnChange` into Lasso ⭐ (the key to Harvester/Rancher)

**Goal:** read the exact framework `harvester/harvester` and `rancher/rancher` are built on.

In `~/src/wrangler` and `~/src/lasso`, trace and write `../journal/code-traces/02-wrangler-onchange.md`:
- Find a generated controller's `OnChange(ctx, name, SyncFunc)` (in `wrangler/pkg/generic` or a `generated` controller).
- Follow it: Wrangler registers your handler → hands to **Lasso's `sharedController`** (`lasso/pkg/controller`) → which creates an **informer + workqueue** (same as Lab 2!) and calls your `SyncFunc` on each change.
- Note `OnRemove` (finalizer-based delete handling) and `wrangler/pkg/apply` (desired-state apply).

**Proves:** you can now open `harvester/harvester`'s controllers and read them — they're all `OnChange` handlers.
**Anki:** "Harvester/Rancher controller entry point = `XxxController.OnChange(ctx, name, func)` → Lasso sharedController → informer+workqueue (it's client-go, wrapped)".

---

## Lab 7 — Delve into a live reconcile

**Goal:** add the debugger to your toolkit — "read the code" now includes "watch the values."

1. Run your Lab-4 controller under Delve: `dlv debug ./cmd/main.go` (or attach). Set a breakpoint: `break (*EchoReconciler).Reconcile`.
2. `continue`, then `kubectl apply` an `Echo`. When it breaks: `print req`, `print ctx`, step with `next`/`step`, inspect the fetched object after the `Get`.
3. Do the same on **sample-controller**'s `syncHandler` (Lab 1).

**Proves:** you can inspect real controller state at runtime — invaluable in Phases 5–7 when reading huge repos.
**Anki:** "dlv: `break pkg.(*Type).Method`, `continue`, `print var`, `next`, `step`".

---

## Lab 8 — apimachinery: GVK ↔ GVR + the dynamic client

**Goal:** understand types/schemes/mapping — the "what is an object" layer.

Write a program that:
1. Builds a `discovery` client + `RESTMapper`, and maps GVK `apps/v1, Kind=Deployment` → its GVR (`deployments`). Print both. (Internalize Kind vs Resource.)
2. Uses the **dynamic client** (`dynamic.NewForConfig`) to `Get` a Deployment as `*unstructured.Unstructured`, and prints `obj.Object["spec"]["replicas"]` via `unstructured.NestedInt64`.

**Read:** `apimachinery/pkg/runtime/schema` (GVK/GVR) and `pkg/apis/meta/v1/unstructured`.
**Proves:** you understand how `kubectl` and operators touch *any* resource generically.
**Anki:** "GVK (Group/Version/**Kind**) = the type; GVR (Group/Version/**Resource**) = the REST path; RESTMapper converts between them".

---

## Lab 9 — Capstone: a real little operator (uses everything)

**Goal:** prove Phase 1 by building a controller that exercises every concept.

Build (in controller-runtime) a CRD `DemoRegistry{ spec: { name, replicas }, status: { ready, message } }` whose Reconcile:
- [ ] Creates a ConfigMap + a Deployment owned by the CR (`controllerutil.SetControllerReference` / `Owns`).
- [ ] Adds a **finalizer**; on delete, cleans up + removes the finalizer (read `controllerutil.AddFinalizer`/`RemoveFinalizer`).
- [ ] Sets **status conditions** (`Ready=True/False`) and uses `Result{RequeueAfter}` to retry when not ready.
- [ ] Handles `IsNotFound`/`IsConflict` correctly.

Run it on the guest cluster; create/update/delete the CR; verify owned objects and finalizer behavior.

**Proves:** you can build a production-shaped operator — the exact shape of every controller in Phases 5–7.
**This is your Phase-1 graduation artifact.** Save the repo; you'll reference its patterns when reading Harvester/Rancher.

---

## After the labs
- [ ] Tick the **Definition of Mastery** in [`README.md`](./README.md).
- [ ] Commit your Lab-9 operator to a personal repo — it's also good practice for **PR #1** (workflow muscle memory).
- [ ] Move to Phase 2 (Kubernetes core) — you can now read controllers, so the controller-manager will read like a (big) book of Lab-9s.
