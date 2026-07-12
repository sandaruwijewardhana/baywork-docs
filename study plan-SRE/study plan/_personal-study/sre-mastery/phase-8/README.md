# Phase 8 — The C Frontier (idea-level: docs + key functions, NOT full code reads)

> Part of the **Code-Mastery program** ([`../00-curriculum.md`](../00-curriculum.md)). These are the **C / kernel layers beneath the Go** — what containers, VMs, packets, and volumes *actually are*. You will **not** read them wholesale (each is a multi-year codebase). Instead, because **you already know C**, you'll do three things per topic:
> 1. **Read the official docs** for the complete mental model.
> 2. **Read a handful of named key functions** — the 5–10 that explain the whole thing.
> 3. **Observe it live** through interfaces (`strace`, `/proc`, `/sys`, `virsh`, `nft`, `dmsetup`).
>
> **Best interleaved**, not done as a block: read the kernel namespace functions *during Phase 3*, QEMU's run loop *during Phase 6*, the storage dataplane *during Phase 7*. Each section says when.

> ⚠️ Exact file paths in C projects move between versions (e.g. QEMU moved `vl.c` → `system/vl.c`). **Confirm with `git grep <function>` in the cloned repo.** Names of functions are stable; locations drift.

---

## 1 · Linux kernel — containers *are* kernel features (interleave with Phase 3)

**Mental model (docs first):**
- `man 7 namespaces`, `man 7 cgroups`, `man 2 clone`, `man 2 unshare`, `man 2 setns`
- kernel.org: `Documentation/admin-guide/cgroup-v2.rst` (the definitive cgroup v2 doc)

**Key functions to peek at** (`git clone --depth 1 https://github.com/torvalds/linux`, then `git grep`):
- Namespaces: `kernel/nsproxy.c` → `create_new_namespaces()`, `copy_namespaces()`. Per-type: `net/core/net_namespace.c`, `kernel/pid_namespace.c`, `fs/namespace.c` (mount), `kernel/user_namespace.c`.
- cgroups v2: `kernel/cgroup/cgroup.c` → `cgroup_attach_task()`; the `struct cgroup_subsys`; memory controller `mm/memcontrol.c`.
- The syscall path: `kernel/fork.c` → `kernel_clone()` (how `clone()` with `CLONE_NEW*` flags makes a namespaced process — this is literally container creation).

**Observe live (labs):**
- [ ] `strace -f -e trace=clone,unshare,setns,mount runc run ...` (or watch a pod start) → see the exact syscalls. Map each to the functions above.
- [ ] `ls -l /proc/<pid>/ns/` (the namespace handles), `cat /proc/<pid>/cgroup`, explore `/sys/fs/cgroup/...` — read `memory.max`, `cpu.max` for a real pod.
- [ ] **Be the runtime:** `sudo unshare --pid --net --mount --fork bash` → you just created a container by hand. `nsenter` into another process's namespaces.

**Idea-level mastery:** explain, with a named function + live evidence, how a pod becomes an isolated process — no hand-waving, no kernel-maintenance claim.

---

## 2 · runc's C core — the bridge from Go to namespaces (interleave with Phase 3)

runc is mostly Go, but the namespace setup *must* happen in C before the Go runtime starts (you can't `clone(CLONE_NEWUSER)` cleanly from multithreaded Go).
- **Read:** `libcontainer/nsexec.c` in `opencontainers/runc` — the famous C shim that does `clone`/`setns` in the right order. ~1 file, perfect for a C reader. This is *the* seam between container-Go-land and the kernel.

---

## 3 · KVM + QEMU — how a VM actually runs (interleave with Phase 6 / KubeVirt)

**Mental model (docs):**
- Linux: `Documentation/virt/kvm/api.rst` (the KVM ioctl API — `KVM_CREATE_VM`, `KVM_CREATE_VCPU`, `KVM_RUN`).
- QEMU docs: qemu.org/docs — the "System Emulation" + "virtio" sections.

**The ONE function to read** (`git clone --depth 1 https://gitlab.com/qemu-project/qemu`):
- `accel/kvm/kvm-all.c` → **`kvm_cpu_exec()`** — the heart: the loop that calls the `KVM_RUN` ioctl, the CPU runs guest code natively until a **VM exit**, then QEMU handles the exit (I/O, MMIO) and loops. Read this one function and you understand hardware virtualization. 
- Then: `kvm_init()` (same file) — VM/vCPU setup via ioctls.

**Then virtio (paravirtualized devices — what your VMs actually use):**
- `hw/virtio/virtio.c` → `virtqueue_pop()` (how the guest and host exchange buffers via a ring). `hw/block/virtio-blk.c`, `hw/net/virtio-net.c` for disk/net.

**Observe live (labs):**
- [ ] On Harvester, pick a KubeVirt VM → `virsh dumpxml <domain>` (the libvirt XML).
- [ ] `ps aux | grep qemu-system` → the **actual QEMU argv**. Map each `-device virtio-...`, `-drive`, `-netdev` flag to the XML and to the virtio source above.
- [ ] `ls -l /dev/kvm`; understand it's the ioctl handle QEMU opens.

**Idea-level mastery:** explain VM exits, why virtio is fast, and trace a VM from `VirtualMachine` CRD → virt-launcher → libvirt XML → QEMU argv → `kvm_cpu_exec`.

---

## 4 · libvirt — XML → a running QEMU process (interleave with Phase 6)

libvirt is the C daemon KubeVirt's `virt-handler` talks to. It turns domain XML into a QEMU command line and manages it via QMP.
- **Read** (`git clone --depth 1 https://gitlab.com/libvirt/libvirt`):
  - `src/qemu/qemu_process.c` → `qemuProcessStart()` — the top-level "start a VM" flow.
  - `src/qemu/qemu_command.c` → `qemuBuildCommandLine()` — **how domain XML becomes QEMU argv** (the function that produces what you saw in `ps`).
  - `src/qemu/qemu_monitor.c` / `qemu_monitor_json.c` — QMP, the JSON protocol libvirt uses to control a live QEMU.
  - `src/conf/domain_conf.c` → `virDomainDefParseXML()` — XML → internal struct.

**Idea-level mastery:** "domain XML → `qemuBuildCommandLine` → QEMU argv → QMP control" with the named functions.

---

## 5 · The kernel networking dataplane (interleave with Phase 2/3/7)

What kube-proxy, Calico, and NetworkPolicy ultimately program.
**Docs:** netfilter.org docs; `man nft`; `Documentation/networking/` in the kernel.
**Key functions / areas:**
- netfilter hooks: `net/netfilter/core.c` → `nf_hook_slow()` (every packet passes the hooks); conntrack `net/netfilter/nf_conntrack_core.c`.
- nftables: `net/netfilter/nf_tables_api.c`. iptables (legacy): `net/ipv4/netfilter/`.
- eBPF (Calico's BPF dataplane): `kernel/bpf/` → `bpf_prog_run`; read with `bpftool prog`/`bpftool map`.
- Devices: `drivers/net/veth.c` (the pod↔host pair), `net/bridge/`, `drivers/net/vxlan/`.

**Observe live (labs):**
- [ ] `nft list ruleset` (or `iptables-save`) on a node → find the kube-proxy/Calico rules; trace a Service VIP DNAT.
- [ ] `conntrack -L` → live connection tracking table. `bpftool prog show` if Calico-BPF.
- [ ] `ip -d link show` a `cali*`/`veth` interface; map to `drivers/net/veth.c`.

**BIRD (Calico BGP) — idea-level only:** docs at bird.network.cz; concept = it advertises/learns pod-subnet routes via BGP. Peek `proto/bgp/` in `projectcalico/bird` only if curious. Understand the *role*, skip the C.

---

## 6 · The kernel storage dataplane (interleave with Phase 7 / Longhorn)

What Longhorn's engine ultimately rides on.
**Docs:** `Documentation/admin-guide/device-mapper/`, open-iscsi docs.
**Key areas:** block layer `block/blk-core.c` → `submit_bio()`; device-mapper `drivers/md/dm.c`; iSCSI initiator `drivers/scsi/`, target framework `drivers/target/`.
**Observe live (labs):**
- [ ] `lsblk`, `dmsetup ls --tree`, `iscsiadm -m session` on a node running a Longhorn volume → see the iSCSI/dm stack Longhorn builds.
- [ ] Map a Longhorn `Volume` → its `/dev/longhorn/...` block device → the engine process serving it.

**Idea-level mastery:** "PVC → Longhorn volume → engine → iSCSI/blockdev → `submit_bio` → disk" with the named layers.

---

## 7 · nginx (idea-level only — interleave with ingress topics)

For ingress-nginx, the real artifact is the **generated `nginx.conf`** (the Go controller writes it); nginx-the-C is idea-level.
- **Docs:** nginx.org "Development guide" + "How nginx processes a request".
- **Concept to know:** master/worker process model, the event loop (`ngx_event.c`), the 11 request-processing phases. Read `nginx.conf` that ingress-nginx generates and map directives to behavior. **Skip the C internals** unless a specific problem forces it.

---

## Phase 8 — Definition of "idea-level mastery"
For each ❌-turned-🔵 layer (kernel, QEMU, libvirt, dataplane, storage), you can:
- [ ] Explain the mechanism **from official docs** (no analogies).
- [ ] Name and have **read the 1–3 key functions** that embody it.
- [ ] **Observe it live** on the real cluster through its interface (`strace`/`/proc`/`virsh`/`nft`/`dmsetup`).
- [ ] Trace the **full handoff** from the Go layer into the C layer (e.g. KubeVirt → libvirt → QEMU → KVM).

That's "full idea + targeted code," which is exactly the right depth for these — and more than 99% of platform engineers ever reach.
