# The Full Territory & Coverage Map — what "know everything" really means

> You said: *"I want to be the one who knows anything down this — full code understanding, not simple parts."* This document shows the **entire territory** (bottom of the machine to the top of Rancher), exactly **what the curriculum covers**, and — honestly — **what it will NOT cover and why.** Read this so you're choosing the boundary with open eyes, not discovering it later.

---

## The honest definition of the goal

Nobody has read 100% of this stack — it is **tens of millions of lines across Go, C, and the Linux kernel**, and large parts (QEMU, the kernel, BIRD) are separate multi-year specialties. The people who *can answer anything* are not people who memorized every line. They are people who have:

1. **Architecture-complete** understanding of every subsystem (what each one is, how they connect), **and**
2. **Critical-path-deep** reading of the major Go subsystems (the controllers/loops/request paths that actually run), **and**
3. **Navigation speed** — they can `grep` to any answer in any repo in minutes, and dive into kernel/QEMU *through their interfaces* when a problem demands it.

That tier **is** achievable and **is** rare. This plan targets exactly that tier. The "long tail" (every one of 40 controllers, every scheduler plugin, every containerd snapshotter) is read **on demand**, not pre-memorized — because that's how the experts actually operate, and pretending otherwise would be a lie.

**Coverage legend:** ✅ read in depth (critical paths) · 🟡 architecture + on-demand deep dives (extended scope, beyond 3 months) · 🔵 **idea-level** (docs + a few key functions + live observation — **Phase 8**, added 2026-06-16) · ❌ truly out of scope (and why)

> **Update (2026-06-16):** the C/kernel layers that were ❌ are now **🔵 idea-level (Phase 8)** — you asked to cover QEMU, the kernel, libvirt, etc. via **docs + reading some key functions** (you know C), not full code reads. See [`phase-8/`](./phase-8/). Only the truly separate specialties (Go runtime, the JS/TS UIs, cloud CCMs) remain ❌.

---

## THE FULL FLOW — bottom of the machine → top of the platform

```
┌─ L13  Rancher / Harvester UIs (Vue/TS, JS)                                  ❌ frontend specialty
├─ L12  Rancher mgmt plane (rancher/rancher, fleet, steve, remotedialer…)     ✅ Phase 5
├─ L11  Harvester control plane (harvester + ~10 sub-repos)                   ✅ Phase 6
├─ L10  KubeVirt (virt-*) ── drives ──▶ libvirt (C) ─▶ QEMU (C) ─▶ KVM (kernel) ✅ KubeVirt / 🔵 QEMU+libvirt (Phase 8)
├─ L9   Longhorn (manager/engine/instance-manager/backupstore)               ✅ Phase 7
├─ L8   The distro: RKE2 ─ wraps ─ k3s (supervisor, bootstrap)               ✅ Phase 4
├─ L7   Cluster API (CAPI) + rke2 providers                                  ✅ Phase 5
│
│        ==== core Kubernetes (the heart — kubernetes/kubernetes, ~2.5M LOC) ====
├─ L6   kube-apiserver: generic apiserver · authn/authz · admission chain ·
│        REST storage/registry · aggregation · apiextensions(CRD) · watch cache  ✅ Phase 2 (paths) / 🟡 full breadth
├─ L6   kube-controller-manager: ~40 built-in controllers                    ✅ Phase 2 (key ones) / 🟡 all 40
├─ L6   kube-scheduler: scheduling framework, filter/score/bind plugins,
│        preemption, the scheduling queue                                    ✅ Phase 2 (framework) / 🟡 every plugin
├─ L6   kubelet: syncLoop · PLEG · volume mgr · device mgr · cgroup mgr ·
│        image mgr · eviction · probes · cAdvisor stats                      ✅ Phase 2 (syncLoop+CRI) / 🟡 every subsystem
├─ L6   kube-proxy: iptables / ipvs / nftables proxiers                      ✅ Phase 2/3
│
├─ L5   The controller FRAMEWORK: apimachinery · client-go · controller-runtime
│        · code-generator/controller-tools · wrangler/lasso                  ✅ Phase 1 (the foundation)
│
├─ L4   etcd: raft consensus · MVCC store · bbolt backend · watch/lease       ✅ Phase 2 (raft+mvcc)
│
├─ L3   Container runtime: CRI ─ containerd (daemon, snapshotters, content,
│        shim) ─ runc ─ OCI spec ; CNI spec ; CSI spec                        ✅ Phase 3
│
├─ L2   Networking dataplane: kube-vip · multus · Calico (felix Go +
│        BIRD C + typha + BPF/iptables dataplane) · Flannel/Canal             ✅ kube-vip/multus, 🟡 Calico-Go / 🔵 BIRD(C) (Phase 8)
│
├─ L1   LINUX KERNEL: namespaces · cgroups v2 · netfilter/nftables/conntrack ·
│        eBPF · KVM · block layer/device-mapper/iSCSI · overlayfs · veth/VXLAN 🔵 idea-level: docs + key functions (Phase 8)
│
├─ L0b  Immutable OS: SLE Micro / Leap Micro · Elemental · A/B updates        🟡 Parallel track (when access lands)
└─ L0a  Hardware/firmware: BMC · Redfish/IPMI · UEFI · NIC/disk firmware      🟡 Parallel track (not source — firmware)
```

Cross-cutting (used by everything): **Helm** (Go) ✅ as needed · **CoreDNS** (Go) 🟡 · **cert-manager** (Go) 🟡 · **gRPC/protobuf** ✅ as the RPC layer · **OCI image/distribution spec** ✅ as needed.

---

## Coverage table — what you WILL master

| Layer | Subsystem | Phase | Depth |
|---|---|---|---|
| L5 | apimachinery, client-go, controller-runtime, codegen, wrangler/lasso | **1** | ✅ in depth — the literacy layer |
| L4 | etcd (raft, mvcc, bbolt) | 2 | ✅ core paths |
| L6 | kube-apiserver (request pipeline, auth, admission, storage, CRD/aggregation) | 2 | ✅ critical paths · 🟡 every admission plugin |
| L6 | kube-controller-manager (deployment, replicaset, node, service, endpointslice, GC, PV binder…) | 2 | ✅ ~8 key controllers · 🟡 all ~40 |
| L6 | kube-scheduler (framework + core plugins + preemption) | 2 | ✅ framework · 🟡 every plugin |
| L6 | kubelet (syncLoop, PLEG, CRI client, volume/device/cgroup managers, eviction) | 2 | ✅ syncLoop + CRI · 🟡 every manager |
| L6 | kube-proxy (iptables/ipvs/nftables) | 2/3 | ✅ |
| L3 | containerd, runc, CRI, CNI, CSI | 3 | ✅ in depth |
| L8 | rke2 + k3s (supervisor, bootstrap, embedded components) | 4 | ✅ in depth |
| L7/12 | Cluster API + Rancher (rancher, steve, norman, fleet, remotedialer, system-agent, dynamiclistener) | 5 | ✅ provisioning + tunnel + agents |
| L10/11 | Harvester (+ sub-repos) + KubeVirt + CDI | 6 | ✅ in depth |
| L9 | Longhorn (manager, engine, instance-manager, backupstore) | 7 | ✅ in depth |
| L2 | kube-vip, multus, Calico (Go: felix, typha, ipam, cni-plugin) | 7 | ✅ kube-vip/multus · 🟡 full Calico-Go |

> "✅ critical paths" means you read the code that actually executes for the common flows + can navigate to the rest on demand. The 🟡 long tail is read **as problems require it**, using your Phase-1 reading skill — which is how real experts cover it too.

---

## What this plan will NOT cover — and why (read this carefully)

Most of these are now **🔵 idea-level (Phase 8)** — covered via **docs + a few key functions + live observation** (you know C). Only the genuinely separate disciplines remain ❌.

| Layer | What it is | How it's covered now |
|---|---|---|
| 🔵 **Linux kernel** (namespaces, cgroups, netfilter, eBPF, KVM, block layer, overlayfs) | The C implementation of what containers/VMs/networking *actually are* | **Phase 8:** docs (`man 7 namespaces`, cgroup-v2.rst) + named functions (`kernel_clone`, `nf_hook_slow`, `kvm_cpu_exec`) + live (`strace`, `/proc`, `nft`). Not the full ~30M-line source — no K8s engineer reads that. |
| 🔵 **QEMU** + **libvirt** | The C virtualizer KubeVirt drives | **Phase 8:** read `kvm_cpu_exec()` (the KVM run loop) + virtio + libvirt's `qemuBuildCommandLine()`; observe via `virsh dumpxml` + the real QEMU argv. Not QEMU's full device-emulation source. |
| 🔵 **BIRD** (C) | The BGP daemon Calico uses for routes | **Phase 8:** docs + its *role*; peek `proto/bgp/` only if curious. Read Calico's **Go** (felix) for the dataplane. |
| 🔵 **nginx** core (C) | Under ingress-nginx | **Phase 8 (idea-level):** docs (master/worker, request phases) + read the **generated `nginx.conf`**; the Go ingress controller is the real code. |
| ❌ **The Go runtime** (scheduler, GC) | The language you write in | You *use* Go; reading its runtime source is a separate rabbit hole, not needed to read these repos. |
| ❌ **Rancher/Harvester/Longhorn UIs** (Vue/TS) | The web frontends | Frontend is a separate discipline; the *backend* (steve API) is in scope, the JS UI is not. |
| ❌ **Cloud-provider CCMs** (AWS/GCP/Azure) | Cloud integration code | Irrelevant to bare-metal Harvester; skip entirely. |
| 🟡 **The full long tail** of K8s (every 1 of ~40 controllers, every scheduler/admission plugin, every containerd snapshotter, every Calico-Go corner) | Breadth within in-scope repos | Covered **architecture-wide + on-demand**, not exhaustively pre-read. This is the honest difference between "3-month plan" and "read literally everything," and it's how experts actually work. |

---

## So: can you "answer anything and resolve any problem"?

**Yes — within the boundary above, which is the real boundary the best people operate in:**

- ✅ Any Kubernetes question — apiserver, scheduling, kubelet, networking, storage — **from the code.**
- ✅ Any Rancher/Harvester/KubeVirt/Longhorn behavior or bug — **traced to the exact controller.**
- ✅ Any container-runtime / etcd / CNI / CSI problem — **from the source.**
- ✅ Kernel/QEMU/BIRD problems — diagnosed through their **interfaces, behavior, docs, and the key functions you read in Phase 8** (`kvm_cpu_exec`, `nf_hook_slow`, `kernel_clone`, `qemuBuildCommandLine`), diving deeper into their C only when a specific problem forces it.

What you will **not** be is "a person who has memorized the entire Linux kernel and all of QEMU line-by-line." **Neither is anyone** — that's not the bar. The bar is: full *idea* of the C layers (Phase 8) + full *code* of the Go layers (Phases 1–7) + the navigation speed to dive anywhere on demand. **That** is what this plan is built to clear.

---

## How to read this map going forward
- The **✅ rows are the spine** — Phases 1→7, in order. That's the 3-month-and-beyond path.
- The **🟡 long tail** you attack on demand using Phase-1 skills, and it deepens for years (that's the multi-year craft).
- The **❌ rows** you learn at the **interface/behavior** level only — enough to debug, not to maintain.

If you want to push *any* ❌ into scope (e.g. you decide you DO want kernel-networking source, or QEMU), tell me and I'll add a dedicated phase for it — but go in knowing each one is a multi-month-to-multi-year addition on its own.

→ Back to the plan: [`00-curriculum.md`](./00-curriculum.md) · Foundation: [`phase-1/`](./phase-1/)
