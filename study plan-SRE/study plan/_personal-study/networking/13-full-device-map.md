# 13 — Full Device Map: Every Virtual and Physical Device

All V1–V10 and P1–P3 devices in one diagram, showing exactly where each one sits.

---

```
                              INTERNET
                                 │
                    ─────────────────────────────
                    P3  ISP ROUTERS  (multiple hops)
                        BGP routing, MAC rewrite each hop
                        drops all RFC1918 packets
                    ─────────────────────────────
                                 │
                    ┌────────────────────────────┐
                    │   P2  WSO2 BORDER ROUTER   │
                    │                            │
                    │  eth0.700 172.22.100.1/24  │ ← VM network VLAN
                    │  eth0.mgmt 192.168.10.1/24 │ ← management VLAN
                    │  eth1(WAN) 203.x.x.x       │ ← public internet
                    │                            │
                    │  V3-mirror: SNAT           │
                    │  172.22.x → 203.x.x.x      │ ← hides private IPs
                    │                            │
                    │  Routing table:            │
                    │  172.22.100.0/24 → eth0.700│
                    │  192.168.10.0/24 → eth0.mgmt│
                    │  0.0.0.0/0       → ISP     │
                    └────────────┬───────────────┘
                                 │  trunk (all VLANs)
                    ┌────────────────────────────┐
                    │   P1  PHYSICAL L2 SWITCH   │
                    │   VLAN-aware, no routing   │
                    │                            │
                    │  MAC table:                │
                    │  nodeMAC1 → port 1         │
                    │  nodeMAC2 → port 2         │
                    │  routerMAC → uplink        │
                    │                            │
                    │  VLANs: mgmt / vlan700 /   │
                    │         storage / vlan699  │
                    └──────┬──────────────┬──────┘
                           │              │
              VLAN trunk   │              │  VLAN trunk
                           │              │
        ╔══════════════════╧══╗      ╔═══╧═════════════════╗
        ║ HARVESTER NODE 1    ║      ║ HARVESTER NODE 2    ║
        ║ mgmt: 192.168.10.15 ║      ║ mgmt: 192.168.10.17 ║
        ║                     ║      ║                     ║
        ║ ┌─────────────────┐ ║      ║ ┌─────────────────┐ ║
        ║ │ V6  mgmt-br     │ ║      ║ │ V6  mgmt-br     │ ║
        ║ │ Linux bridge    │ ║      ║ │ Linux bridge    │ ║
        ║ │ connects:       │ ║      ║ │                 │ ║
        ║ │ - physical NIC  │ ║      ║ │                 │ ║
        ║ │ - tap devices   │ ║      ║ │                 │ ║
        ║ │ - VXLAN vtep    │ ║      ║ │                 │ ║
        ║ └────────┬────────┘ ║      ║ └────────┬────────┘ ║
        ║          │          ║      ║          │          ║
        ║ ┌────────┴────────┐ ║      ║ ┌────────┴────────┐ ║
        ║ │ V8 VXLAN tunnel │◄╬──────╬►│ V8 VXLAN tunnel │ ║
        ║ │ (Flannel/Canal) │ ║      ║ │ (Flannel/Canal) │ ║
        ║ │ pod overlay     │ ║      ║ │                 │ ║
        ║ │ MTU: 1450       │ ║      ║ │                 │ ║
        ║ │ outer: node IPs │ ║      ║ │                 │ ║
        ║ │ inner: pod IPs  │ ║      ║ │                 │ ║
        ║ └─────────────────┘ ║      ║ └─────────────────┘ ║
        ║                     ║      ║                     ║
        ║ ┌─────────────────┐ ║      ║ ┌─────────────────┐ ║
        ║ │ V10 Harvester   │ ║      ║ │  Harbor pods    │ ║
        ║ │ built-in LB     │ ║      ║ │  10.52.x.x      │ ║
        ║ │ VIP 192.168.10.6│ ║      ║ │                 │ ║
        ║ │ DNAT → nginx pod│ ║      ║ │ V1 veth pair    │ ║
        ║ └─────────────────┘ ║      ║ │  pod ↔ bridge   │ ║
        ║                     ║      ║ └─────────────────┘ ║
        ║ V9 tap device       ║      ║                     ║
        ║ VM NIC ↔ mgmt-br   ║      ╚═════════════════════╝
        ║     │               ║
        ║ ┌───┴─────────────────────────────────────┐     ║
        ║ │  RKE2 VM  (172.22.100.3)                │     ║
        ║ │                                         │     ║
        ║ │  ┌─────────────────────────────────┐   │     ║
        ║ │  │  V2  VM KERNEL ROUTER           │   │     ║
        ║ │  │  eth0: 172.22.100.3             │   │     ║
        ║ │  │                                 │   │     ║
        ║ │  │  Routing table:                 │   │     ║
        ║ │  │  10.42.0.0/16 dev cali*         │   │     ║
        ║ │  │  10.42.x.0/24 via tunl0 (IPIP)  │   │     ║
        ║ │  │  0.0.0.0/0 via 172.22.100.1     │   │     ║
        ║ │  │                                 │   │     ║
        ║ │  │  V3  Calico SNAT                │   │     ║
        ║ │  │  src=10.42.x.x → 172.22.100.3   │   │     ║
        ║ │  │  conntrack table                │   │     ║
        ║ │  │                                 │   │     ║
        ║ │  │  V4  kube-proxy DNAT            │   │     ║
        ║ │  │  ClusterIP → pod IP             │   │     ║
        ║ │  │  10.43.x.x → 10.42.x.x         │   │     ║
        ║ │  │                                 │   │     ║
        ║ │  │  V5  IPIP tunnel                │   │     ║
        ║ │  │  outer: 172.22.100.3→.4         │   │     ║
        ║ │  │  inner: 10.42.0.x→10.42.1.x     │   │     ║
        ║ │  └──────────────┬──────────────────┘   │     ║
        ║ │                 │                       │     ║
        ║ │    ┌────────────┴──────────────────┐   │     ║
        ║ │    │  V1  veth pair (per pod)       │   │     ║
        ║ │    │  cali11aa ═══════════ eth0     │   │     ║
        ║ │    │  (host end)       (pod end)    │   │     ║
        ║ │    │                                │   │     ║
        ║ │    │  Calico proxy ARP:             │   │     ║
        ║ │    │  answers 169.254.1.1 on veth   │   │     ║
        ║ │    └──────────────┬─────────────────┘   │     ║
        ║ │                   │                      │     ║
        ║ │    ┌──────────────┴─────────────────┐   │     ║
        ║ │    │  POD  (10.42.0.5)              │   │     ║
        ║ │    │                                │   │     ║
        ║ │    │  eth0: 10.42.0.5               │   │     ║
        ║ │    │  route: 0.0.0.0/0              │   │     ║
        ║ │    │    via 169.254.1.1 dev eth0    │   │     ║
        ║ │    │                                │   │     ║
        ║ │    │  (provisioner pod lives here)  │   │     ║
        ║ │    └────────────────────────────────┘   │     ║
        ║ └─────────────────────────────────────────┘     ║
        ╚═════════════════════════════════════════════════╝
```

---

## Which device handles what action

```
ACTION                          DEVICE USED
──────────────────────────────────────────────────────────────────
Pod sends packet                V1 veth → exits pod namespace
Pod finds next hop              V2 VM kernel routing table
                                  (169.254.1.1 → Calico proxy ARP)
Pod → Service IP (ClusterIP)    V4 kube-proxy DNAT (rewrites dst)
Pod → same cluster diff node    V5 IPIP tunnel (Calico)
Pod → internet                  V3 Calico SNAT (hide pod IP)
                                V2 VM kernel routes to gateway
                                V9 tap → exits VM into host
                                V6 mgmt-br → out physical NIC
                                P1 switch → uplink to router
                                P2 WSO2 router SNAT (hide 172.x)
                                P3 ISP routers → internet
Harbor pod → Harbor pod         V8 VXLAN tunnel (same Harvester)
  (diff Harvester node)         P1 physical switch (same VLAN)
External → Harbor               P2 WSO2 router (inter-VLAN route)
                                P1 switch → node NIC
                                V10 Harvester LB DNAT (VIP→pod)
                                V6 mgmt-br → pod
                                V1 veth → Harbor pod
```

---

## Packet state changes at key devices

```
DEVICE    SRC IP changes?   DST IP changes?   MAC changes?
────────────────────────────────────────────────────────────
V1 veth   no                no                yes (veth MACs)
V2 kernel no                no                yes (next-hop MAC)
V3 SNAT   YES pod→node IP   no                yes
V4 DNAT   no                YES svc→pod IP    yes
V5 IPIP   adds outer IP     adds outer IP     outer MACs
V6 bridge no                no                yes (L2 forward)
V8 VXLAN  adds outer IP     adds outer IP     outer MACs
V9 tap    no                no                no (same frame)
V10 DNAT  no                YES VIP→pod IP    yes
P1 switch no                no                yes (L2 forward)
P2 router no (or SNAT)      no                YES (new MACs)
P3 routers no               no                YES each hop
```

---

## Layer summary

```
Layer 7 (Application)  Harbor, provisioner, nginx
Layer 4 (Transport)    TCP/UDP ports — kube-proxy tracks these
Layer 3 (Network)      IP routing: V2, V5, V8, P2, P3
Layer 2 (Data Link)    MAC forwarding: V1, V6, V9, P1
NAT                    V3 (SNAT), V4 (DNAT), V10 (DNAT), P2 (SNAT)
Tunnels                V5 (IPIP), V8 (VXLAN)
```
