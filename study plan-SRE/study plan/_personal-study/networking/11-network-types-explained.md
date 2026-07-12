# 11 — Management, VM, Storage, and Overlay Networks Explained

> **Question answered here:** When does traffic use the management network vs VM network vs storage network vs overlay — and why is each one separate?

---

## The Core Idea: Why Separate Networks At All?

Imagine one fat pipe carrying everything: VM disk writes, Kubernetes API calls, tenant app traffic, and storage replication — all at once. One tenant doing a large image push would starve the control plane. Storage replication would compete with user traffic. A network blip would break Kubernetes AND VMs AND storage simultaneously.

Separation gives you:
- **Isolation** — a problem in one network doesn't affect others
- **Bandwidth guarantees** — storage gets its own pipe, not fighting VMs
- **Security** — management plane not reachable from tenant VMs
- **Performance** — each network tuned for its traffic type

In Harvester, four network types serve four completely different purposes:

```
Physical Harvester Node
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  NIC 1 (bond0) ─── Management VLAN  192.168.10.x       │
│                 ─── Storage VLAN    192.168.20.x        │
│                                                         │
│  NIC 2 (bond0) ─── VM Network VLAN 172.22.100.x        │
│                 ─── (more VM VLANs per tenant)          │
│                                                         │
│  Overlay        ─── virtual, rides ON TOP of mgmt/VM   │
│                     VXLAN tunnels (Flannel) by default  │
│                     GENEVE only if Kube-OVN add-on ON   │
└─────────────────────────────────────────────────────────┘
```

---

## 1. Management Network (192.168.10.x)

### What it carries

- Harvester OS → OS communication between the two physical nodes
- Kubernetes control plane: etcd, kube-apiserver, kube-scheduler, kube-controller-manager
- `kubectl` commands (your `use-harvester` → talks to 192.168.10.15:6443)
- Rancher → Harvester: Rancher manages Harvester through this network
- Provisioner → Harvester K8s API: your provisioner deploys Harbor here
- Harvester built-in LB announces VIPs (e.g. 192.168.10.6) on this network via ARP
  (Harvester has its own LB — NOT MetalLB. Uses IP pools configured in Harvester UI)
- KubeVirt VM live migration control signals
- Harvester Web UI access

### Who uses it

```
Your laptop          → 192.168.10.15:6443  (kubectl to Harvester)
Rancher              → 192.168.10.15:6443  (managing Harvester cluster)
Provisioner pod      → 192.168.10.15:6443  (deploying Harbor via Helm)
External client      → 192.168.10.6:443    (Harbor via Harvester built-in LB VIP)
Node 1               → Node 2               (K8s control plane gossip, etcd)
```

### Why it's separate

The management network is the **brain** of the cluster. If this gets saturated or goes down, the entire cluster becomes unmanageable — you can't deploy VMs, can't run kubectl, Rancher loses contact. Keeping it isolated means a tenant doing a 100GB image push can't accidentally starve your control plane of bandwidth.

### What does NOT use it

- VM-to-VM traffic between tenants → uses VM network
- Longhorn disk replication → uses storage network
- Pod-to-pod traffic across nodes → uses overlay

---

## 2. VM Network / Tenant Network (172.22.100.x, VLAN 699/700)

### What it carries

- **North-south traffic**: external world ↔ tenant VMs (user requests, internet access)
- VM NICs — the IP a VM's `eth0` gets is on this network
- RKE2 cluster nodes (your registry-controller-cluster VMs) live here on 172.22.100.x
- Inter-VM traffic between VMs on the same tenant network
- Docker push/pull traffic from tenant developers to Harbor

### Who uses it

```
Developer laptop     → 172.22.100.3:6443   (kubectl to registry-controller-cluster)
Provisioner pod      → 192.168.10.6        (reaches Harbor VIP via router)
registry-controller  → Harvester API       (cross-VLAN via WSO2 router)
VM (tenant workload) → internet            (goes through WSO2 router SNAT)
```

### Why it's separate

VMs generate **bursty, high-bandwidth, unpredictable traffic** — a tenant streaming logs, running CI builds, doing image pulls. You never want that competing with the management plane. Also, different tenants get different VLANs (699, 700, 701...) for isolation — tenant A's VM traffic cannot be seen by tenant B even though they share the same physical NICs.

```
VLAN 699 → Tenant A's VMs  (completely isolated from VLAN 700)
VLAN 700 → Tenant B's VMs  (registry-controller-cluster lives here)
VLAN 701 → Tenant C's VMs
```

### What does NOT use it

- Kubernetes control plane → uses management network
- Longhorn replication → uses storage network
- Pod overlay traffic → uses VXLAN tunnels (Flannel, default) or GENEVE (Kube-OVN, if enabled)

---

## 3. Storage Network (192.168.20.x)

### What it carries

- **Longhorn replication**: when you write to a PVC, Longhorn replicates the blocks to 2-3 other nodes. This is pure node-to-node I/O.
- iSCSI/NVMe-over-TCP: block device data between the Longhorn engine and its replicas
- Longhorn volume migration: moving a replica from one node to another

### Who uses it

```
Node 1 Longhorn engine → Node 2 Longhorn replica  (replicate 3 copies of each PVC)
Node 2 Longhorn engine → Node 3 Longhorn replica
(all internal, never leaves the datacenter)
```

### Why it's the most critical to separate

Storage replication is **extremely high bandwidth and latency-sensitive**. A single 50GB Harbor registry PVC writes might generate 150GB of replication traffic (3 replicas). If this runs on the management network:

```
WITHOUT storage network:
Management NIC at 1Gbps
├── etcd heartbeats (needs <10ms, or nodes think each other dead)
├── K8s API calls
├── Longhorn replication (150GB burst) ← saturates the NIC
└── Result: etcd times out → cluster thinks nodes are down → pods evicted
```

```
WITH storage network:
Management NIC: K8s control plane (no storage traffic at all)
Storage NIC:    Longhorn replication (full bandwidth dedicated)
Result: control plane stable even during heavy storage I/O
```

Longhorn specifically has a setting `Storage Network` in Harvester UI — you point it at the storage VLAN and Longhorn moves all replica traffic there automatically.

### What does NOT use it

- Anything that's not Longhorn block replication
- User-facing reads from a mounted PVC go through the pod's overlay network

---

## 4. Overlay Network (virtual — no dedicated physical NIC)

### What it is

An overlay is not a physical network. It's a **virtual network built on top of** one of the physical networks (usually management). Packets are **encapsulated** — wrapped inside another packet — so a pod IP packet travels inside a node IP packet.

```
Pod A (10.52.0.5) → Pod B (10.52.1.7)  [different nodes]

Without overlay:
  The physical switch has no route to 10.52.0.0/16 — drops packet

With overlay (GENEVE tunnel):
  Inner packet:  src=10.52.0.5   dst=10.52.1.7
  Outer packet:  src=192.168.10.15  dst=192.168.10.16  ← node IPs the switch knows
  Switch forwards the outer packet normally
  Node 2 unwraps it, delivers inner packet to Pod B
```

### In Harvester: Canal (Calico + Flannel) uses VXLAN — DEFAULT

> ⚠️ **Important:** Kube-OVN is NOT the default CNI in Harvester.
> Harvester ships with **Canal** (Calico + Flannel) as the built-in CNI.
> Kube-OVN is an **experimental optional add-on** — must be manually enabled via the `kubeovn-operator` add-on in Harvester settings.

Default setup (Canal/Flannel — what most Harvester clusters actually use):

```
Node 1 (192.168.10.15)                Node 2 (192.168.10.16)
┌─────────────────────────┐            ┌─────────────────────────┐
│ Pod A: 10.52.0.5        │            │ Pod B: 10.52.1.7        │
│   │                     │            │   │                     │
│ mgmt-br (Linux bridge)  │            │ mgmt-br (Linux bridge)  │
│   │                     │            │   │                     │
│ VXLAN tunnel (Flannel)  │────────────│ VXLAN tunnel (Flannel)  │
│   src: 192.168.10.15    │            │   dst: 192.168.10.16    │
│   overhead: 50 bytes    │            │   pod MTU: 1450         │
└─────────────────────────┘            └─────────────────────────┘
         physical management network (192.168.10.x)
```

The VXLAN tunnel rides on top of the management network. The physical switch only sees 192.168.10.x traffic.

If Kube-OVN add-on is enabled (experimental):
- Replaces Flannel VXLAN with OVS + OVN pipeline
- Default tunnel type: VXLAN (configurable to GENEVE)
- Adds VPC, subnet, ACL capabilities

### In registry-controller-cluster: Calico uses IPIP

```
Node 1 (172.22.100.3)                 Node 2 (172.22.100.4)
┌─────────────────────────┐            ┌─────────────────────────┐
│ Pod: 10.42.0.5          │            │ Pod: 10.42.1.5          │
│ Calico IPIP tunnel      │────────────│ Calico IPIP tunnel      │
│   outer: 172.22.100.3   │            │   outer: 172.22.100.4   │
└─────────────────────────┘            └─────────────────────────┘
         VM network (172.22.100.x)
```

Calico tunnels ride on top of the VM network. The physical switch only sees 172.22.100.x.

### Who uses it

```
Harbor pod (10.52.x.x) → Longhorn CSI  (pod accessing its PVC)
Harbor-core → harbor-database           (pod to pod, same cluster)
Provisioner → any K8s service           (ClusterIP resolved by kube-proxy)
OVN ACL drops cross-tenant pod traffic  (NetworkPolicy enforcement)
```

### Why it exists

Without an overlay, every pod subnet would need to be routable on the physical network. With 50 tenants × 256 pods each = 12,800 pod IPs that the physical router would need routes for. That doesn't scale. The overlay hides all pod IPs inside node IPs — the physical network only needs to know about a handful of nodes.

---

## Decision Map: Which Network For What?

```
Is it Kubernetes control plane traffic?
  → Management network (192.168.10.x)

Is it VM traffic from a tenant (external access, internet, app traffic)?
  → VM/tenant network (172.22.100.x, VLAN 700/699/...)

Is it Longhorn block replication between nodes?
  → Storage network (192.168.20.x)

Is it pod-to-pod traffic inside a Kubernetes cluster?
  → Overlay network (GENEVE/IPIP on top of mgmt or VM network)

Is it a user hitting Harbor from outside?
  → Enters via VM network → MetalLB VIP on management network
    → nginx ingress → Harbor pod via overlay
```

---

## Your Setup Mapped Out

```
                         INTERNET
                             │
                    WSO2 border router
                    (SNAT to public IP)
                      │           │
            ┌─────────┘           └──────────┐
     VLAN 700 (172.22.100.x)    VLAN mgmt (192.168.10.x)
       VM Network                 Management Network
            │                           │
     ┌──────┴───────┐           ┌───────┴──────┐
     │              │           │              │
   RKE2 VM        RKE2 VM    Harvester    Harvester
 (node1)         (node2)      Node 1       Node 2
 172.22.100.3  172.22.100.4  .10.15        .10.16
     │              │           │              │
  Calico         Calico      Kube-OVN      Kube-OVN
  IPIP overlay             GENEVE overlay
     │                           │
  Pod IPs                    Pod IPs
  10.42.x.x                 10.52.x.x
  (provisioner)             (Harbor pods)
                                 │
                    Storage VLAN (192.168.20.x)
                    Longhorn replication
                    (node1 ↔ node2, 3 replicas)
```

---

## Why Each Traffic Type Belongs Where It Does

| Traffic | Network | Reason |
|---------|---------|--------|
| `kubectl apply` to Harvester | Management | Control plane must be isolated and always reachable |
| Rancher manages Harvester | Management | Same — management operations |
| Provisioner deploys Harbor | Management | Targeting Harvester K8s API on 192.168.10.15:6443 |
| Harbor VIP (MetalLB) | Management | MetalLB announces ARP on the management L2 |
| Developer `docker push` to Harbor | VM network → management | Enters from internet via VM VLAN, hits MetalLB VIP on management |
| Harbor core → harbor-db | Overlay | Pod-to-pod inside Harvester K8s, stays in GENEVE tunnel |
| Harbor registry → Longhorn PVC | Overlay + Storage | Pod access via overlay; node-to-node replication via storage VLAN |
| etcd between Harvester nodes | Management | K8s control plane — needs dedicated low-latency path |
| Longhorn replica sync | Storage | Dedicated bandwidth, never competes with control plane |
| Provisioner pod → any service | Overlay (IPIP) | Calico tunnels on VM network |
| VM live migration | Management | KubeVirt sends VM memory pages between nodes via mgmt |
