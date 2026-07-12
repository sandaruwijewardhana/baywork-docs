# SRE Code-Mastery Curriculum — Kubernetes / Rancher / Harvester, from the source

**Owner:** Sandaruw · **Started:** 2026-06-16 · **Target:** ~2026-09-16 (12 weeks, continues past that)
**Pace:** 4+ hrs/day, intense
**Mission:** **Master the actual source code** of Kubernetes, Rancher, and Harvester — and the stack beneath them — deep enough to **answer any question and debug any failure *from the code itself*.**

> **Learning rule:** every command's every flag gets explained, and we always look at the **real environment**
> (actual node tables / real output / real source), never just diagrams. **Always confirm against official docs +
> the actual cloned repo — never trust a remembered file path.**

> **One plan, one numbering.** This curriculum is the **code-mastery spine (Phases 1–8)** below. Everything is
> ordered by *code literacy*, not by hardware-up operations. The old "bare-metal-first" numbering has been
> retired; bare-metal + immutable-OS now live as a **Parallel Track** (near the end), done when host/BMC access
> lands. (The previous version is preserved in git history.)

---

## The mission — what "know everything" really means

Nobody has read 100% of this stack — it's tens of millions of lines across Go, C, and the kernel. The people who *can answer anything* have:
1. **Architecture-complete** understanding of every subsystem (what it is, how it connects), **and**
2. **Critical-path-deep** reading of the major Go subsystems (the controllers/loops/request paths that actually run), **and**
3. **Navigation speed** — `grep` to any answer in any repo in minutes; dive into kernel/QEMU *through their interfaces* when a problem demands it.

That tier is achievable and rare. This plan targets exactly it. The long tail is read **on demand**, not pre-memorized — because that's how experts actually operate. → Full coverage boundary in [`MAP.md`](./MAP.md); motivation in [`WHY.md`](./WHY.md).

---

## The complete subsystem inventory (so nothing is missed)

> Each row = a subsystem you will eventually read at component level. **Official doc + repo** given so you start
> from the source of truth. Treat URLs/paths as starting points and verify against the live repo.

**A · FOUNDATION — the literacy layer (Phase 1, start here, 100% unblocked)**
| Subsystem | Repo | Official docs |
|---|---|---|
| API machinery | `k8s.io/apimachinery` | pkg.go.dev/k8s.io/apimachinery |
| client-go (informers, listers, workqueue, clients) | `k8s.io/client-go` | pkg.go.dev/k8s.io/client-go |
| controller-runtime | `sigs.k8s.io/controller-runtime` | book.kubebuilder.io |
| codegen (deepcopy/client/informer/lister + CRDs) | `k8s.io/code-generator`, `sigs.k8s.io/controller-tools` | book.kubebuilder.io/reference/controller-gen |
| Wrangler / Lasso (Rancher's controller framework) | `rancher/wrangler`, `rancher/lasso` | the repos' READMEs |

**B · KUBERNETES CORE** — `kubernetes/kubernetes` (kubernetes.io/docs) + `etcd-io/etcd` (etcd.io/docs)
kube-apiserver · etcd · kube-scheduler · kube-controller-manager · kubelet · kube-proxy

**C · RUNTIME & INTERFACES**
containerd (`containerd/containerd`) · runc (`opencontainers/runc`) · CRI · CNI (`containernetworking/cni`) · CSI (`container-storage-interface/spec`)

**D · DISTRO** — `rancher/rke2` (docs.rke2.io) wrapping `k3s-io/k3s` (docs.k3s.io)

**E · RANCHER** — `rancher/rancher`
+ `rancher/steve` · `rancher/norman` · `rancher/fleet` · `rancher/remotedialer` · `rancher/system-agent` · `rancher/system-upgrade-controller` · `rancher/dynamiclistener` · Cluster API (`kubernetes-sigs/cluster-api`) + the rke2 CAPI providers

**F · HARVESTER + VIRTUALIZATION** — `harvester/harvester`
+ `harvester-network-controller` · `harvester-load-balancer` · `harvester-csi-driver` · `harvester-cloud-provider` · `node-manager` · `node-disk-manager` · `seeder` · `pcidevices` · `vm-import-controller`
KubeVirt (`kubevirt/kubevirt`) + CDI (`kubevirt/containerized-data-importer`)

**G · STORAGE** — Longhorn (longhorn.io)
`longhorn/longhorn-manager` · `longhorn-engine` · `longhorn-instance-manager` · `backupstore` · `go-iscsi-helper`

**H · NETWORKING DATAPLANE**
`kube-vip/kube-vip` · `k8snetworkplumbingwg/multus-cni` · Calico (`projectcalico/calico`) — felix/bird/typha/confd · Flannel/Canal

**I · C FRONTIER (idea-level — Phase 8, interleaved)** — Linux kernel (namespaces/cgroups/netfilter/KVM/block) · QEMU · libvirt · BIRD

**Parallel Track (when host/BMC access lands)** — Hardware (BMC/Redfish/IPMI) + immutable OS: SLE Micro / Leap Micro + Elemental (`rancher/elemental`)

---

## THE CODE-MASTERY SPINE — the one plan, in order

```
Phase 1  Foundation            apimachinery · client-go · controller-runtime · codegen · wrangler/lasso   ← START HERE
Phase 2  Kubernetes core       apiserver · etcd · scheduler · controller-manager · kubelet · kube-proxy
Phase 3  Runtime & interfaces  containerd · runc · CRI · CNI · CSI
Phase 4  Distro                rke2 · k3s
Phase 5  Rancher               rancher · steve/norman · provisioningv2 + CAPI · fleet · remotedialer · system-agent
Phase 6  Harvester + KubeVirt  harvester controllers · virt-* · CDI · network-controller · csi/cloud-provider · seeder · pcidevices
Phase 7  Longhorn + dataplane  longhorn-manager/engine/instance-manager · kube-vip · multus · calico
Phase 8  C Frontier (idea-level) kernel (namespaces/cgroups/netfilter/KVM/block) · QEMU · libvirt · BIRD
                                 — docs + a few key functions + live observation; INTERLEAVED with Phases 3/6/7
Parallel  Hardware + immutable OS  (when access lands — see "Parallel Track" below)
```

- **Phase 8 is not read as a block** — do its kernel sections while in Phase 3, QEMU/libvirt while in Phase 6, storage/network dataplane while in Phase 7. It's *idea-level* (docs + ~5–10 named functions + `strace`/`virsh`/`nft` observation), the correct depth for C/kernel/QEMU that no one reads wholesale.
- Each phase gets its own folder (`phase-1/`, `phase-2/`, …) with **repos to read (package checklists), official docs, and full labs.** **Phase 1 is built — see [`phase-1/`](./phase-1/).** Each phase ends with a **Definition of Mastery** you must do from memory on the real environment before advancing.

---

## Legend — lab access tags

| Tag | Meaning |
|---|---|
| `[GUEST]` | Doable **today** on `registry-controller-cluster` (full kubectl, `kubectl debug node`). |
| `[HOST]` | Needs **SSH to Harvester nodes** (currently blocked — read-only/no-SSH). |
| `[BMC]` | Needs **BMC/IPMI console + BIOS/firmware** access (currently blocked). |
| `[THROWAWAY]` | Needs a **disposable env you can destroy** (nested-VM Harvester or spare box). Not confirmed yet. |
| `[READ]` | Read-only observation, doable now even on Harvester at cluster level. |

## ⛔ Week-0 gating tasks (do first — they unblock the parallel track)

- [ ] Request in writing: **SSH** to `node01` (192.168.10.15) and `node02` (192.168.10.17).
- [ ] Request **BMC/IPMI** web-console + power-control (Dell iDRAC / HPE iLO / Supermicro / Redfish — find out which).
- [ ] Request **BIOS/firmware** read access (or a maintenance window).
- [ ] Decide a **`[THROWAWAY]`** lab: spare node, OR nested Harvester in a VM. Unlocks every destructive hardware/OS lab.
- [ ] Create the lab journal: `journal/` — one dated file per lab, paste REAL output, write what broke and how you fixed it.

---

## ⭐ MODULE 0 — the controller pattern (do BEFORE any upper repo)

The single highest-leverage thing in the curriculum. Skipping it makes the codebases look like noise; doing it makes them readable. **The whole ecosystem is the same shape:** a **CRD** (custom resource) + a **controller** whose `OnChange`/reconcile function runs every time that resource changes, driving *actual state → desired state*. Learn to read **one** controller and you can read them all.

### Read, in order
1. **The K8s controller/informer pattern** (client-go): `Informer` → local `cache`/`Lister` → `WorkQueue` → `reconcile`. Source: `k8s.io/client-go/tools/cache` (SharedIndexInformer) and `kubernetes/sample-controller` (the canonical minimal example — ~1 file of real reconcile code). **This is the Rosetta Stone.**
2. **Wrangler** (`rancher/wrangler`) — Rancher's wrapper that both Harvester and Rancher are built on. It code-generates typed clients + controllers from CRD Go structs. Read how `controller.SharedControllerFactory` works and what a generated `XxxController.OnChange(ctx, name, handler)` actually does (an informer + workqueue underneath). **This is THE key.**
3. **Lasso** (`rancher/lasso`) — the caching/controller engine under Wrangler. Skim.
4. **CRD codegen** — open any `pkg/apis/.../types.go` (the struct defining a CRD spec/status) and the generated `zz_generated_*` next to it. Read the struct, not the boilerplate.

### The trace method (your repeatable superpower)
```
1. Name the resource (VirtualMachine, Cluster, Volume, Node…).
2. grep for the Kind + "OnChange"/"Register"/"Reconcile":
     git grep -n "OnChange" pkg/ | grep -i <kind>
3. That handler IS the entry point. Read it.
4. Follow the typed clients it calls (.Get/.Update/.Create) to the next controller.
5. Stop at the apiserver, an external binary (libvirt, longhorn-engine), or a CRD another controller owns — note the handoff, jump there.
6. Confirm with reality: kubectl get events, controller pod logs, or a dlv breakpoint.
```
Keep each trace as a numbered note in `journal/code-traces/`. After ~10 traces the codebase stops being scary.

### The repo map (clone into `~/src/`)
| Repo | Role / what to read |
|---|---|
| `kubernetes/sample-controller` | The minimal reconcile example — read `controller.go` end to end (Module 0). |
| `rancher/wrangler`, `rancher/lasso` | The framework Harvester/Rancher are built from (Module 0). |
| `kubernetes/kubernetes` | Core: apiserver request pipeline, ~8 key controllers, scheduler framework, kubelet syncLoop+CRI. Trace, never linear-read. |
| `etcd-io/etcd` | raft consensus + MVCC store + bbolt. |
| `containerd/containerd`, `opencontainers/runc` | The runtime under K8s (CRI → containerd → runc → OCI). |
| `rancher/rke2` (+ `k3s-io/k3s`) | rke2 is *thin* — the real distro logic is k3s (`pkg/cluster`, `pkg/etcd`, `pkg/agent`). |
| `rancher/rancher` (+ steve/norman/fleet/remotedialer/system-agent) | Multi-cluster control plane. Target provisioningv2 + agents + tunnel; use the trace method. |
| `kubernetes-sigs/cluster-api` + rke2 CAPI providers | Provisioning v2 = CAPI under the hood. |
| `kubevirt/kubevirt` (+ CDI) | VMs as pods: `virt-api` / `virt-controller` / `virt-handler` / `virt-launcher`. Start `pkg/virt-controller/watch/`. |
| `longhorn/longhorn-manager` + `longhorn-engine` | Storage control plane (Volume/Replica/Engine controllers) + the block dataplane. |
| `harvester/harvester` (+ sub-repos) | The HCI brain: VM lifecycle, networking, upgrades, settings (`pkg/controller/`, `pkg/webhook/` — verify by grep). |
| `kube-vip/kube-vip`, `k8snetworkplumbingwg/multus-cni`, `projectcalico/calico` | The networking dataplane (VIP, extra NICs, CNI). |

> ⚠️ Exact file paths are version-dependent — **always confirm with `git grep`/`grep -rn` on the cloned repo.** Finding the current path *is* the skill.

---

## 🎯 PROOF-OF-KNOWLEDGE — the two PRs

> A merged PR into a project in this stack beats any certificate — public, permanent, signed by you.

- **PR #1 — "learn the machinery" (target ~Jul 6–12):** a low-risk **docs/example fix** in a repo you're already reading (Harvester/Longhorn/Rancher docs or `kube-vip`). The point is the *workflow*: **DCO `git commit -s`**, **CLA**, `good-first-issue`/`help-wanted` labels, small-PR etiquette. Do it early on something safe.
- **PR #2 — "the proof" (target ~Aug 1–9):** a **real code change** via `label:good-first-issue`. Best targets: **`kube-vip/kube-vip`** (small, focused — top pick), **`longhorn/longhorn-manager`** (your storage layer), or a small Harvester sub-repo. ❌ Avoid `rancher/rancher` and `kubevirt/kubevirt` for a *first* code PR — too big to land fast.
- **Optional, compounding:** a public writeup of one deep trace (e.g. *"How Longhorn rebuilds a replica, read from the code"*). Every extra merged PR compounds your GitHub into a credential no résumé bullet matches.

**Definition of "proven":** PR #1 merged by ~mid-July · PR #2 merged by ~mid-August · one public code-trace writeup by end of program.

---

## 🧠 RETENTION SYSTEM (use every single day — this is what beats the forgetting curve)

You lose most of what you learn in **5–10 days** unless you actively retrieve it. Re-reading feels productive but barely works (recognition, not recall). Countermeasure:

**Split what you learn** — spend memory only on the left, offload the right to grep + cheatsheets:

| MEMORIZE (durable) | MAKE FINDABLE (never memorize — it drifts) |
|---|---|
| Mental models (how a reconcile loop works) | Exact file paths in a repo |
| **The method to re-derive specifics** (`git grep -n "OnChange" pkg/ \| grep -i <kind>`) | Exact flags of a rare command |
| Muscle memory from redoing labs | Long YAML, ports, IPs, line numbers |
| The *shape* of a flow (resource → controller → handoff) | — |

**The daily system (~20 min on top of labs):**
- [ ] **Anki** (desktop + AnkiDroid) — after every lab make 3–5 cards that recall the **method/model**, not trivia. Do due reviews each morning (~10 min).
- [ ] **Journal entries end with a `## Recall` block** — 3 questions, no answers in the file (answers live in Anki).
- [ ] **Friday "redo from memory" ritual** — redo 1–2 earlier labs with the doc closed. What you can't reproduce is what didn't stick.
- [ ] **Teach it back (Feynman)** once per phase — explain a flow with no notes; where you stumble = the gap.
- [ ] **One cheatsheet per layer** in `cheatsheets/` — your external memory (OK to be path-specific).

**Retention hierarchy (best → worst):** doing/breaking it > teaching it > active recall (Anki) > re-reading.

**Folder layout:**
```
sre-mastery/
  00-curriculum.md   ← this file (the plan)     MAP.md   WHY.md
  phase-1/ … phase-8/  ← per-phase repos + labs (phase-1 built)
  journal/           ← dated labs: REAL output + what broke + ## Recall
    code-traces/     ← numbered flow traces through real source
  cheatsheets/       ← quick-ref per layer
  anki/              ← exported .apkg decks
```

---

## 📅 CALENDAR — dated timetable (mapped to the spine)

Anchored to **today = 2026-06-16**. Order matters more than exact dates, but keep the **two PR milestones** roughly on time (reviews take weeks — leave buffer).

| Phase | Weeks | Dates | Milestone |
|---|---|---|---|
| Week-0 prep | — | Jun 16–21 | Go done · tooling (Anki, Delve) · access requested · **Module 0 started** |
| **Phase 1** Foundation | 1–2 | Jun 22 – Jul 5 | from-scratch client-go + controller-runtime controllers run on guest cluster |
| 🎯 **PR #1** (docs/mechanics) | — | **~Jul 6–12** | first PR merged |
| **Phase 2** Kubernetes core | 3–4 | Jul 6 – Jul 19 | trace a request through apiserver; read raft+mvcc in etcd |
| **Phase 3** Runtime & interfaces | 5 | Jul 20 – Jul 26 | trace pod → containerd → runc → cgroup (Phase-8 kernel bits here) |
| **Phase 4** Distro (rke2/k3s) | 6 | Jul 27 – Aug 2 | explain how a node becomes a K8s node from k3s source |
| 🎯 **PR #2** (real code = the proof) | — | **~Aug 1–9** | code PR merged via good-first-issue |
| **Phase 5** Rancher + CAPI | 7–8 | Aug 3 – Aug 16 | trace guest-cluster provisioning to the exact controllers |
| **Phase 6** Harvester + KubeVirt | 9–10 | Aug 17 – Aug 30 | trace VM start end-to-end (QEMU/libvirt Phase-8 bits here) |
| **Phase 7** Longhorn + dataplane | 11–12 | Aug 31 – Sep 13 | storage traced all the way down; kube-vip/multus/calico dataplane |
| **Phase 8** C frontier | interleaved | in 3/6/7 | idea-level kernel/QEMU/BIRD via docs + key functions + observation |

### Daily shape (4+ hrs/day)
Morning **Anki** (~15 min) → **1 lab** with real output into `journal/` → **read the source** for what you just saw → end of day **3–5 Anki cards** + a `## Recall` block.

### Weekly shape
Mon–Thu new material · **Fri = redo earlier labs from memory** · weekend lighter (official docs + the current PR).

### Weekly cadence (rough split)
~40% real labs (output into `journal/`) · ~30% reading official docs for that layer · ~20% connecting it back to *this* platform · ~10% a written "what would page me, and how I'd fix it" for that layer.

---

## 🔧 PARALLEL TRACK — Hardware + immutable OS (NOT the spine; when access lands)

> This was the old "Phase 1/2." It's **not** the starting point anymore — it's a **side track** you run when
> `[BMC]`/`[HOST]`/`[THROWAWAY]` access is granted, because it's blocked now and least code-relevant. It stays here
> as **operational context** you pair with the matching code phase (hardware ↔ Phase 6 seeder/pcidevices; OS ↔ Phase 3 runtime).

### Bare metal
**Model:** before any Linux boots there's firmware (UEFI/BIOS), a BMC running independently of the OS, disks behind a controller, and PXE-capable NICs. An SRE who can't talk to the BMC can't recover a dead node.
- Server anatomy: CPU/NUMA, RAM channels, PCIe, disk backplane, NIC ports, the BMC chip.
- **BMC/IPMI/Redfish** out-of-band: power on/off, serial-over-LAN console, virtual media, sensors — how you fix a node with no OS.
- UEFI vs BIOS, Secure Boot, boot order, **PXE/iPXE** (how Harvester nodes install at scale).
- Disks: SATA/SAS/NVMe, HBA vs hardware RAID, why HCI wants **plain disks (JBOD)**, SMART.
- NICs: link speed, bonding/LACP, SR-IOV, MTU/jumbo frames.
- **Labs:** `[BMC]` log into node01 BMC (power/firmware/temps/MACs) · open a serial-over-LAN console and watch POST→UEFI→bootloader · `ipmitool -I lanplus -H <bmc> -U <u> -P <p> chassis status` then `sensor list`, `sel list` · `[READ]/[HOST]` `lscpu`, `lsblk -o NAME,SIZE,TYPE,MODEL`, `lspci | grep -i eth`, `dmidecode -t memory` · `[THROWAWAY]` mount the Harvester ISO via virtual media and reach the installer.
- **Mastery:** power-cycle + console into a node entirely via BMC (no OS) · state why hardware RAID is wrong for HCI · map node NICs (MACs/speeds/bonds) to Harvester `mgmt`/`strg`/`vm` networks.

### The immutable OS under Harvester
**Model:** Harvester runs **SLE Micro / Leap Micro** — an **immutable, transactional, A/B-updated** OS managed by **Elemental**. Root is read-only; config lives in specific persisted paths; upgrades swap the whole image. "Fix" a node the wrong way and it reverts on reboot.
- Immutable/transactional OS: read-only root, `transactional-update`, A/B partitions, auto-rollback; **Elemental**.
- The daily Linux the SRE leans on, seen on real nodes: **systemd** (`systemctl`, `journalctl`; rke2 runs as units), **cgroups v2 + namespaces** (the real container mechanism), **containerd** (`crictl ps`), `/proc` & `/sys`, `sysctl`, modules, chrony (etcd needs time sync).
- **Labs:** `[GUEST]` `kubectl debug node/<n> -it --image=busybox` → `chroot /host` → `systemctl status rke2-server`, `journalctl -u rke2-server`, `crictl ps` · find a pod's cgroup under `/sys/fs/cgroup`, read `memory.max`/`cpu.max`, correlate to its K8s limits · `[HOST]` `cat /etc/os-release`, `transactional-update --help`, A/B via `btrfs subvolume list /`.
- **Mastery:** explain (with real output) why an edit vanishes on reboot and the correct persistent way · trace a pod → containerd container → cgroup limits → systemd-managed kubelet/rke2 · read `journalctl` to find why a node went NotReady.

### Operational validation to fold into the matching code phase
These are real-cluster labs to do *alongside* the code reading (not separate phases):
- **With Phase 6 (Harvester/KubeVirt):** `kubectl get vmi -A -o wide`; find the `virt-launcher` pod behind a VM; on `[THROWAWAY]` drain a node and watch VM behavior + Longhorn rebuild + kube-vip VIP failover.
- **With Phase 7 (Longhorn/dataplane):** map one PVC → Longhorn volume → its 3 replicas → nodes/disks; fault-inject a replica and watch rebuild; reproduce the full external→pod hop trace with real `ip route`/`nft`/`ipvsadm` tables.
- **With Phase 5 (Rancher):** walk the real topology in the Rancher UI; etcd snapshot save/restore on `[THROWAWAY]`; provision a tiny guest RKE2 cluster from Terraform and watch CCM+CSI auto-install.
- **Observability (cross-cutting):** stand up/inspect Prometheus + Grafana; learn the **golden signals** (latency, traffic, errors, saturation); build one real dashboard for the registry cluster (node CPU/mem/disk, etcd health, API latency, Longhorn volume health).

---

## Primary sources (official, version-matched)
Kubernetes · etcd · containerd · RKE2 · k3s · Rancher · Harvester · Longhorn · KubeVirt · kube-vip · Calico · Cluster API docs · SUSE SLE Micro / Elemental docs · the Kubebuilder book (controller-runtime).
Your own repo notes: [[networking_deep_dive]], [[operational_notes]], [[harvester_rancher_harbor_official]], [[dev-cluster-state]].

---

## Progress tracker (the spine)

| Phase | Weeks | Status |
|---|---|---|
| Week-0 — access requests + tooling | prep | ☐ |
| **Module 0** — controller pattern + Wrangler + Delve | prep | ☐ |
| **Phase 1** — Foundation (literacy layer) — [`phase-1/`](./phase-1/) | 1–2 | ☐ |
| 🎯 PR #1 — mechanics (docs/example fix) | ~Jul 6–12 | ☐ |
| **Phase 2** — Kubernetes core | 3–4 | ☐ |
| **Phase 3** — Runtime & interfaces (+ Phase-8 kernel bits) | 5 | ☐ |
| **Phase 4** — Distro (rke2/k3s) | 6 | ☐ |
| 🎯 PR #2 — real code, the proof | ~Aug 1–9 | ☐ |
| **Phase 5** — Rancher + CAPI | 7–8 | ☐ |
| **Phase 6** — Harvester + KubeVirt (+ Phase-8 QEMU/libvirt) | 9–10 | ☐ |
| **Phase 7** — Longhorn + dataplane (+ Phase-8 BIRD) | 11–12 | ☐ |
| Parallel — Hardware + immutable OS | when access lands | ☐ |

Update this table and tick lab boxes as you go. Each session, tell me which phase you're in and I'll pull the matching labs (using your real access) and explain every command.
