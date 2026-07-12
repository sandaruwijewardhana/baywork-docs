# 12 — Every Switch, Router, and NAT for Every Traffic Pattern

---

## Device Inventory First

Before counting hops, here are ALL devices that exist in the setup:

```
VIRTUAL / SOFTWARE DEVICES
┌─────────────────────────────────────────────────────────────────┐
│ V1  veth pair (Calico)      one per pod, point-to-point wire    │
│ V2  VM kernel router        routing table + iptables in RKE2 VM │
│ V3  Calico SNAT             iptables MASQUERADE in VM kernel     │
│ V4  kube-proxy DNAT         iptables DNAT for Service IPs       │
│ V5  IPIP tunnel (Calico)    pod overlay between RKE2 nodes      │
│ V6  mgmt-br Linux bridge    L2 bridge on each Harvester node    │
│     (default: Canal/Flannel) OVS only if Kube-OVN add-on on    │
│ V7  OVN logical router      L3 router — ONLY if Kube-OVN on    │
│     (experimental add-on)   Not present in default Harvester   │
│ V8  VXLAN tunnel (Flannel)  pod overlay between Harvester nodes │
│     default tunnel type     50-byte overhead, pod MTU 1450     │
│     (GENEVE only if Kube-OVN add-on explicitly configured)     │
│ V9  tap device              VM NIC ↔ host kernel bridge         │
│ V10 Harvester built-in LB   DNAT: VIP → actual pod IP          │
│     (NOT MetalLB)           uses IP pools configured in UI     │
└─────────────────────────────────────────────────────────────────┘

PHYSICAL DEVICES
┌─────────────────────────────────────────────────────────────────┐
│ P1  Physical L2 switch      VLAN-aware, no routing              │
│ P2  WSO2 border router      inter-VLAN routing + SNAT to WAN    │
│ P3  ISP routers (N hops)    BGP, internet backbone              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Traffic Pattern Count Table

```
PATTERN                          V-SW  V-RT  V-NAT  P-SW  P-RT  P-NAT  TOTAL HOPS
─────────────────────────────────────────────────────────────────────────────────
1. Pod → Pod  (same node,         1     1      0      0     0     0      2
   same RKE2 VM, same subnet)

2. Pod → Pod  (same node,         1     2      0      0     0     0      3
   same RKE2 VM, Service IP)      (+ kube-proxy DNAT counts as V-RT)

3. Pod → Pod  (diff node,         2     2      0      1     0     0      5
   same RKE2 cluster, diff VMs)   (IPIP tunnel across VLAN 700)

4. Pod → Pod  (diff node,         2     2      0      0     0     0      4
   same Harvester cluster,        (VXLAN tunnel/Flannel default,
   same VLAN)                      GENEVE only if Kube-OVN add-on)

5. Pod → Pod  (cross-cluster:     3     4      2      1     1     0      11
   RKE2 → Harvester K8s)         (via MetalLB VIP — only way)

6. Pod → Node IP                  1     1      0      0     0     0      2
   (same VM, 172.22.100.3)

7. Pod → Node IP                  2     3      1      1     1     0      8
   (diff VM, diff Harvester node)

8. Pod → Harvester K8s API        2     4      1      1     1     0      9
   (192.168.10.15:6443)

9. Pod → MetalLB VIP              2     5      2      1     1     0      11
   (192.168.10.6 → Harbor pod)

10. Pod → Internet (8.8.8.8)      2     3      2      1     N     1      9+N
    (N = ISP router hops)

11. Pod → ClusterIP Service       1     2      1      0     0     0      4
    (same node)

12. Pod → ClusterIP Service       2     3      1      1     0     0      7
    (diff node, same cluster)

─────────────────────────────────────────────────────────────────────────────────
V-SW = virtual switch    V-RT = virtual router    V-NAT = virtual NAT
P-SW = physical switch   P-RT = physical router   P-NAT = physical NAT
```

---

## Each Pattern Expanded

---

### Pattern 1 — Pod → Pod (same RKE2 VM, same /24 subnet)

```
Pod A (10.42.0.5) ──── Pod B (10.42.0.6)   both on VM 172.22.100.3
```

```
[Pod A eth0]
    │ V1: veth pair (cali11aa)
    ▼
[VM kernel] ── V2: routing table: 10.42.0.6 dev cali22bb
    │
    │ V1: veth pair (cali22bb)
    ▼
[Pod B eth0]

Devices: V1 (×2), V2
NAT: none — pod IPs unchanged
Physical: nothing — stays inside VM kernel
```

---

### Pattern 2 — Pod → Pod via Service IP (same node)

```
Pod A → ClusterIP 10.43.0.1:443 → Pod B (10.42.0.6)  same VM
```

```
[Pod A eth0]
    │ V1: veth
    ▼
[VM kernel iptables PREROUTING]
    │ V4: kube-proxy DNAT: 10.43.0.1:443 → 10.42.0.6:8080
    ▼
[VM kernel routing] ── V2: 10.42.0.6 dev cali22bb
    │ V1: veth
    ▼
[Pod B eth0]

Devices: V1 (×2), V2, V4
NAT: 1 DNAT (service → pod IP)
```

---

### Pattern 3 — Pod → Pod (different RKE2 VMs, different Harvester nodes)

```
Pod A (10.42.0.5) on VM1 (172.22.100.3) on Harvester-node1
  →
Pod B (10.42.1.5) on VM2 (172.22.100.4) on Harvester-node2
```

```
[Pod A eth0]
    │ V1: veth (cali11aa)
    ▼
[VM1 kernel] ── V2: route: 10.42.1.0/24 via tunl0 (IPIP tunnel)
    │ V5: IPIP encapsulate: inner(10.42.0.5→10.42.1.5) outer(172.22.100.3→172.22.100.4)
    ▼
[VM1 eth0] → V9: tap device → OVS/harvester-br0
    │ V6: OVS bridge (VLAN 700, L2 forward to trunk)
    ▼
[Harvester-node1 NIC] ── VLAN 700 tag
    │ P1: Physical switch (L2 only, same VLAN 700 → forwards to node2 port)
    ▼
[Harvester-node2 NIC]
    │ V6: OVS bridge → tap device → V9
    ▼
[VM2 eth0]
    │ V2: VM2 kernel: unwrap IPIP, route 10.42.1.5 dev cali55bb
    │ V1: veth (cali55bb)
    ▼
[Pod B eth0]

Devices: V1(×2), V2(×2), V5, V6(×2), V9(×2), P1
NAT: 0 — IPIP is encapsulation not NAT, pod IPs unchanged
Physical switch: 1 (same VLAN, no routing needed)
Physical router: 0
```

---

### Pattern 4 — Pod → Pod (same Harvester K8s cluster, different nodes)

```
Harbor-core (10.52.0.5) on Harvester-node1
  →
Harbor-db (10.52.1.5) on Harvester-node2
```

Default (Canal/Flannel — what most Harvester clusters use):

```
[harbor-core eth0]
    │ V1: veth
    ▼
[mgmt-br Linux bridge] ── V6: L2 forward
    │ V8: VXLAN encapsulate (Flannel): inner(10.52.0.5→10.52.1.5) outer(192.168.10.15→192.168.10.16)
    │     overhead: 50 bytes, pod MTU: 1450
    ▼
[Harvester-node1 NIC] ── management VLAN
    │ P1: Physical switch (L2, same management VLAN → forwards to node2)
    ▼
[Harvester-node2 NIC]
    │ V8: unwrap VXLAN (Flannel)
    │ V6: mgmt-br Linux bridge
    │ V1: veth
    ▼
[harbor-db eth0]

Devices: V1(×2), V6(×2), V8(VXLAN), P1
NAT: 0
Physical router: 0 (same VLAN, L2 only)

NOTE: If Kube-OVN add-on is enabled, V6 becomes OVS bridge (br-int),
      V8 becomes GENEVE/VXLAN tunnel controlled by OVN, and V7 (OVN
      logical router) is added to the path.
```

---

### Pattern 5 — Pod → Pod Cross-Cluster (RKE2 pod → Harvester pod)

```
Provisioner (10.42.0.5) → Harbor-core (10.52.0.5)
```

**Direct pod IP is NOT possible** — 10.42.x.x and 10.52.x.x are completely separate overlay networks with no route between them.

Only way: go through MetalLB VIP → nginx ingress → Harbor pod:

```
[Provisioner pod]
    │ V1: veth → V2: route 0.0.0.0/0 via 169.254.1.1
    │ V3: Calico SNAT: 10.42.0.5 → 172.22.100.3
    ▼
[VM1 eth0] → V9: tap → V6: OVS (VLAN 700)
    │ P1: Physical switch
    ▼
[WSO2 router] ── P2: inter-VLAN route: 192.168.10.6 on mgmt VLAN
    │ P1: Physical switch (mgmt VLAN)
    ▼
[Harvester-node1 NIC]
    │ V6: OVS → V10: MetalLB kube-proxy DNAT: 192.168.10.6 → nginx-pod IP
    │ V8: GENEVE (if nginx on different node)
    ▼
[nginx-ingress pod] → routes to harbor-core (another GENEVE hop)
    ▼
[harbor-core pod]

Devices: V1, V2, V3, V6(×3), V7, V8(×2), V9, V10, P1(×2), P2
NAT: 2 (Calico SNAT + MetalLB kube-proxy DNAT)
```

---

### Pattern 6 — Pod → Internet

```
Provisioner (10.42.0.5) → 8.8.8.8
```

```
[Pod eth0]
    │ V1: veth
    │ V2: route: 0.0.0.0/0 via 169.254.1.1 (Calico gateway)
    │ V3: SNAT: 10.42.0.5 → 172.22.100.3 (conntrack entry created)
    ▼
[VM eth0] → V9: tap → V6: OVS (VLAN 700)
    │ P1: Physical switch (VLAN 700 → uplink to WSO2 router)
    ▼
[WSO2 router]
    │ P2: route: 0.0.0.0/0 via ISP
    │ P2-NAT: SNAT: 172.22.100.3 → 203.x.x.x (public IP)
    ▼
[ISP routers] (N hops, BGP, MAC rewrite each hop)
    ▼
[8.8.8.8]

Devices: V1, V2, V3, V6, V9, P1, P2, P3(×N)
NAT: 2 (Calico SNAT + WSO2 SNAT)
Physical routers: 1 + N ISP hops
```

---

### Pattern 7 — Pod → Harvester K8s API (192.168.10.15:6443)

```
Provisioner (10.42.0.5) → kube-apiserver on Harvester node1
```

```
[Pod eth0]
    │ V1: veth
    │ V2: route: 192.168.10.15 not in 10.42.x.x → default via 169.254.1.1
    │ V3: SNAT: 10.42.0.5 → 172.22.100.3
    ▼
[VM eth0] → V9: tap → V6: OVS (VLAN 700)
    │ P1: Physical switch (VLAN 700 → uplink to WSO2 router)
    ▼
[WSO2 router]
    │ P2: route: 192.168.10.15 → 192.168.10.0/24 on mgmt VLAN interface
    │ No NAT — stays private
    │ ARP for 192.168.10.15 → Harvester-node1 MAC
    │ P1: Physical switch (mgmt VLAN → node1 port)
    ▼
[Harvester-node1 NIC]
    │ Kernel: dst=192.168.10.15 = me → local delivery
    │ Port 6443 → kube-apiserver process
    ▼
[kube-apiserver]

Devices: V1, V2, V3, V6, V9, P1(×2), P2
NAT: 1 (Calico SNAT only)
Physical routers: 1 (WSO2 inter-VLAN routing)
```

---

### Pattern 8 — Pod → MetalLB VIP (Harbor)

```
Provisioner (10.42.0.5) → 192.168.10.6:443 → Harbor
```

Same as Pattern 7 up to WSO2 router, then:

```
[WSO2 router]
    │ ARP for 192.168.10.6 → MetalLB speaker MAC (node1)
    │ P1: Physical switch (mgmt VLAN → node1 port)
    ▼
[Harvester-node1 NIC]
    │ V6: OVS ingress
    │ V10: kube-proxy DNAT: 192.168.10.6:443 → nginx-ingress-pod:443
    │ V8: GENEVE (if nginx pod on node2)
    ▼
[nginx-ingress pod] → routes by hostname → harbor-core
    │ V8: GENEVE to harbor-core node
    ▼
[harbor-core pod]

Devices: V1, V2, V3, V6(×2+), V7, V8(×2), V9, V10, P1(×2), P2
NAT: 2 (Calico SNAT + MetalLB DNAT)
```

---

## What Each Device Can Talk To

```
DEVICE              CAN TALK TO                          CANNOT TALK TO
──────────────────────────────────────────────────────────────────────────────
V1 veth pair        Only the two ends of the pair        Anything else directly
                    (pod namespace ↔ host namespace)

V2 VM kernel        Any IP reachable from VM eth0        Other cluster's pod IPs
   router           (172.22.100.x, 192.168.10.x,         (10.52.x.x — no route)
                    10.42.x.x same cluster via IPIP)

V3 Calico SNAT      Not a router — modifies src IP        —
                    on packets leaving VM for
                    external destinations

V4 kube-proxy DNAT  Not a router — rewrites dst IP        —
                    for ClusterIP and NodePort services

V5 IPIP tunnel      Any node in same RKE2 cluster         Different cluster nodes
                    (172.22.100.x range)                  Harvester pods directly

V6 mgmt-br          Pods on same Harvester node           Pods on other clusters
   Linux bridge     (via veth), pods on other              (no route to 10.42.x.x)
   (Canal default)  Harvester nodes (via VXLAN),
                    physical network via NIC uplink
   [If Kube-OVN     OVS bridge replaces mgmt-br,          Same limitations
    add-on enabled] adds OVN pipeline + VPC ACLs

V7 OVN logical      ONLY present if Kube-OVN experimental  Everything — does NOT
   router           add-on is enabled. Routes between       exist in default setup.
   (experimental)   OVN logical switches / namespaces.

V8 VXLAN tunnel     Any Harvester node (192.168.10.x)     Cross-cluster pods
   (Flannel/Canal)  carrying pod traffic — 50-byte         (10.42.x.x unreachable)
   DEFAULT tunnel   overhead, pod MTU forced to 1450
   [GENEVE only if  Same reach, different encapsulation,
    Kube-OVN on]    configurable in Kube-OVN settings

V9 tap device       VM eth0 (one end) ↔                   —
                    Linux bridge/OVS on host (other end)

V10 Harvester LB    Rewrites LB VIP to pod IP             —
    built-in DNAT   Works for any client that can
    (NOT MetalLB)   reach 192.168.10.x — uses IP pools

P1 Physical switch  Any device on same VLAN               Devices on different VLANs
                    (L2 only — same broadcast domain)      (needs router to cross VLANs)

P2 WSO2 router      ALL VLANs it has interfaces on:       —
                    - 172.22.100.0/24 (VM network)
                    - 192.168.10.0/24 (mgmt)
                    - 192.168.20.0/24 (storage)
                    - 0.0.0.0/0 (internet via WAN)

P3 ISP routers      Internet (BGP full table)              RFC1918 private ranges
                                                           (dropped at border)
```

---

## Summary Card

```
Traffic                          Virt  Phys  NAT   Notes
                                 hops  hops  count
──────────────────────────────────────────────────────────────
Pod → Pod same VM                 2     0     0    stays in VM kernel
Pod → Pod same VM via Service     3     0     1    +kube-proxy DNAT
Pod → Pod diff VM same cluster    7     1     0    IPIP tunnel + switch
Pod → Pod same Harvester cluster  5     1     0    VXLAN tunnel (Flannel default) + switch
Pod → Pod cross-cluster          11     3     2    Harvester LB + WSO2 router
Pod → Same node IP                2     0     0    loopback-ish
Pod → Diff node IP (diff VLAN)    5     2     1    crosses WSO2 router
Pod → Harvester K8s API (.15)     6     2     1    crosses WSO2 router
Pod → MetalLB VIP (.6)            8     3     2    crosses WSO2 + MetalLB DNAT
Pod → Internet                    5     1+N   2    2× SNAT
Pod → ClusterIP same node         3     0     1    kube-proxy DNAT
Pod → ClusterIP diff node         5     1     1    kube-proxy DNAT + switch
```

---

## The Golden Rule

```
Same VLAN + same overlay network  →  NO router needed (L2 only)
Different VLAN, same datacenter   →  WSO2 router (1 physical router hop)
Different overlay, same host      →  OVN/Calico routes in software
Different cluster pod IPs         →  IMPOSSIBLE direct — must use Service/VIP
Public internet                   →  2× SNAT (Calico + WSO2)
```
