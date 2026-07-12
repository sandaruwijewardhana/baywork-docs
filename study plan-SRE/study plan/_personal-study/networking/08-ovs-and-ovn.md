# 08 — OVS and OVN
## Open vSwitch, OpenFlow, and OVN Logical Topology — The Virtual Network OS

---

## The Story

Regular Linux bridges and iptables work well for simple cases.
But when you have 50 tenants, each needing isolated networks, load balancers,
firewalls, and routing — managing hundreds of iptables rules manually is a nightmare.

OVS (Open vSwitch) replaces the simple Linux bridge with a programmable switch
that can run OpenFlow programs — tiny programs executed per-packet.

OVN (Open Virtual Network) is the "operating system" for OVS.
You tell OVN "I want a virtual network for acme-management with these ACLs"
and OVN automatically translates that into thousands of OpenFlow rules on every node.

This is what Harvester uses (via Kube-OVN).

---

## 1. Open vSwitch (OVS)

OVS is a software switch with superpowers. It replaces the simple Linux bridge.

```
LINUX BRIDGE (simple, what we've seen so far):
  Only does: MAC table → forward to correct port
  Can do: VLANs (with vlan_filtering)
  Cannot do: complex routing, statistics per-flow, OpenFlow programs

OVS (powerful):
  ✅ MAC learning and forwarding
  ✅ VLANs
  ✅ VXLAN/GENEVE tunnels built-in
  ✅ OpenFlow tables (programmable packet pipeline)
  ✅ QoS and bandwidth limiting per-flow
  ✅ Packet statistics per-flow (great for monitoring)
  ✅ Bonding / LACP
  ✅ DPDK support (kernel bypass for high performance)
  ✅ Remote control via OVSDB protocol (network controller can program it)


OVS ARCHITECTURE:
═══════════════════════════════════════════════════════════════════════

  USER SPACE:
  ┌─────────────────────────────────────────────────────────────────┐
  │  ovs-vswitchd  (OVS daemon — the main control process)          │
  │  ovsdb-server  (database storing OVS config)                    │
  └─────────────────────────────────────────────────────────────────┘
             │  communicates via OVSDB protocol
             ▼
  KERNEL SPACE:
  ┌─────────────────────────────────────────────────────────────────┐
  │  openvswitch kernel module                                      │
  │  ├── datapath: processes packets at line speed                  │
  │  ├── flow table cache (fast path — seen flows are cached)       │
  │  └── upcall: unknown flows → ask ovs-vswitchd for instructions  │
  └─────────────────────────────────────────────────────────────────┘

  PACKET PROCESSING:
  New packet arrives → kernel checks flow cache (fast path)
  Cache hit?  → execute cached actions immediately (microseconds)
  Cache miss? → send to ovs-vswitchd → vswitchd applies OpenFlow tables
              → compute actions → install in cache → execute
```

---

## 2. OVS Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# BASIC OVS MANAGEMENT
# ═══════════════════════════════════════════════════════════════════

# Show OVS version
ovs-vsctl show

# List all bridges:
ovs-vsctl list-br
# Output (on Harvester node):
# br-int
# br-ex

# Show all ports on a bridge:
ovs-vsctl list-ports br-int
# Output:
# genev_sys_6081   ← GENEVE tunnel port
# patch-br-int-to-br-ex
# harbor-acme-core-xxxx  ← pod port

# Show detailed port info:
ovs-vsctl list port harbor-acme-core-xxxx

# Show all interfaces with their types:
ovs-vsctl list interface | grep -E "(name|type|ofport)"

# ═══════════════════════════════════════════════════════════════════
# CREATE AND CONFIGURE BRIDGES (what Kube-OVN does at startup)
# ═══════════════════════════════════════════════════════════════════

# Create a bridge (integration bridge — all pod ports connect here):
ovs-vsctl add-br br-int

# Create external bridge (for traffic going to physical network):
ovs-vsctl add-br br-ex

# Add a patch port connecting two bridges:
ovs-vsctl add-port br-int patch-int-ex \
  -- set interface patch-int-ex type=patch options:peer=patch-ex-int
ovs-vsctl add-port br-ex patch-ex-int \
  -- set interface patch-ex-int type=patch options:peer=patch-int-ex

# Add physical NIC to external bridge:
ovs-vsctl add-port br-ex eth0

# Add GENEVE tunnel (for pod traffic to other nodes):
ovs-vsctl add-port br-int genev-node2 \
  -- set interface genev-node2 type=geneve \
     options:remote_ip=192.168.10.17 \
     options:key=flow \
     options:dst_port=6081

# ═══════════════════════════════════════════════════════════════════
# OPENFLOW — READ THE PACKET PIPELINE
# ═══════════════════════════════════════════════════════════════════

# Show ALL OpenFlow rules (the heart of OVS — can be thousands of lines):
ovs-ofctl dump-flows br-int

# Show rules formatted nicely:
ovs-ofctl dump-flows br-int | sort -k3 -t, | head -50

# Count flows per table:
ovs-ofctl dump-flows br-int | awk -F'table=' '{print $2}' | \
  awk '{print $1}' | sort -n | uniq -c

# Show ports and their OpenFlow port numbers:
ovs-ofctl show br-int

# ═══════════════════════════════════════════════════════════════════
# DEBUGGING
# ═══════════════════════════════════════════════════════════════════

# Trace a packet through OVS (very powerful debugging tool):
ovs-appctl ofproto/trace br-int \
  in_port=<PORT_NUM>,tcp,nw_src=10.52.5.8,nw_dst=10.52.5.12,tp_dst=443

# Show packet statistics per flow:
ovs-ofctl dump-flows br-int | grep npackets

# Show OVSDB records:
ovsdb-client dump | head -100
```

---

## 3. OpenFlow — Programmable Packet Pipeline

OpenFlow is the programming language for OVS. It defines rules: "if packet matches X, do action Y".

```
OPENFLOW RULE STRUCTURE:
  table=N, priority=P, match-fields → actions

  table=N:     which table (0-254) — packets flow through tables in order
  priority=P:  higher priority wins when multiple rules match (0-65535)
  match-fields: what to match (any combination):
    in_port=7           ← arrived on OVS port 7
    eth.src=0a:00:...   ← specific source MAC
    ip.dst=10.52.5.12   ← specific destination IP
    tcp.dst=443         ← TCP destination port
    metadata=0x42       ← metadata field (OVN uses this)
  actions:
    output:8            ← send to OVS port 8
    drop                ← discard packet
    goto_table:5        ← send to table 5 for more processing
    set_field:0x1->reg0 ← set a register (used by OVN for state)
    mod_nw_dst:10.52.5.8 ← DNAT (change destination IP)
    CONTROLLER          ← send to ovs-vswitchd for slow-path handling


EXAMPLE: Simple L2 forwarding table

  # Default: send all unknown packets to controller (slow path learning)
  table=0, priority=0                             → CONTROLLER

  # Known MAC → forward to correct port
  table=0, priority=10, eth.dst=0a:00:00:05:08   → output:7   (pod1)
  table=0, priority=10, eth.dst=0a:00:00:05:12   → output:9   (pod3)
  table=0, priority=10, eth.dst=ff:ff:ff:ff:ff:ff → output:ALL (broadcast)


OVN'S MULTI-TABLE PIPELINE (simplified — real OVN uses ~20 tables):
  Table 0:  Ingress port classification
  Table 8:  Ingress ACL (check: is this packet allowed IN?)
  Table 16: Ingress L2 forwarding (ARP proxy, DHCP, unicast lookup)
  Table 32: Logical Router — route between logical networks
  Table 48: Egress ACL (check: is this packet allowed OUT?)
  Table 64: Egress port delivery (send to correct output port)
```

---

## 4. OVN — Open Virtual Network

OVN is the brain above OVS. It provides a high-level API for virtual network topology.

```
OVN COMPONENTS AND THEIR JOBS:
═══════════════════════════════════════════════════════════════════════

  ovn-northbound DB   ← "what you WANT" — logical topology
  │  Stores: logical switches, logical routers, load balancers, ACLs
  │  Written to by: Kube-OVN controller (watching K8s API)
  │  Read by: ovn-northd
  │
  ovn-northd          ← TRANSLATOR
  │  Reads northbound DB (logical topology)
  │  Computes: how to implement it on real hardware
  │  Writes to southbound DB
  │
  ovn-southbound DB   ← "how to DO it" — physical binding
  │  Stores: which chassis (node) owns which logical port
  │          OpenFlow logical flows (not yet OpenFlow rules)
  │          Tunnel endpoints for each chassis
  │
  ovn-controller      ← runs on EVERY node
     Reads southbound DB
     Translates to actual OpenFlow rules for local OVS
     Manages GENEVE tunnels to other nodes


DATA FLOW:
  Kube-OVN creates logical port "harbor-acme-core"
              │
              ▼  writes to
  OVN Northbound DB: "harbor-acme-core on switch acme-mgmt, IP 10.52.5.8"
              │
              ▼  ovn-northd processes
  OVN Southbound DB: "harbor-acme-core is physically on chassis 192.168.10.15"
                     + logical flows for routing/ACL
              │
              ▼  ovn-controller on each node reads
  OVS OpenFlow rules (on node 192.168.10.15):
    "output to harbor-acme-core veth on this node (local)"
  OVS OpenFlow rules (on node 192.168.10.17):
    "output to 192.168.10.15 via GENEVE tunnel (remote)"
```

---

## 5. OVN Logical Topology — Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# OVN NORTHBOUND (logical topology — what you configure)
# ═══════════════════════════════════════════════════════════════════

# List logical switches (one per namespace in Kube-OVN):
ovn-nbctl ls-list
# Output:
# <uuid> (acme-management)
# <uuid> (beta-management)
# <uuid> (registry-controller)

# Show logical switch details:
ovn-nbctl show acme-management
# Output:
#   switch <uuid> (acme-management)
#       port harbor-acme-core-<uuid>
#           addresses: ["0a:00:00:00:05:08 10.52.5.8"]
#       port harbor-acme-redis-<uuid>
#           addresses: ["0a:00:00:00:05:09 10.52.5.9"]
#       port acme-management-to-router
#           type: router
#           addresses: ["router"]

# Show logical routers:
ovn-nbctl lr-list
# cluster-router  ← routes between all namespaces

# Show logical router routes:
ovn-nbctl lr-route-list cluster-router
# IPv4 Routes
# Route Table <main>:
#              10.52.5.0/24               10.52.5.1 dst-ip
#              10.52.6.0/24               10.52.6.1 dst-ip

# Show ACLs (NetworkPolicy rules):
ovn-nbctl acl-list acme-management
# Output:
#   to-lport  2000 (ip4.src == 10.52.5.0/24) allow  ← same namespace
#   to-lport  2000 (inport == "ingress-ns") allow   ← from ingress
#   to-lport  1     (ip4) drop                       ← default deny

# Show Load Balancers (Service ClusterIPs):
ovn-nbctl lb-list
# UUID                       LB             PROTO      VIP             IPs
# <uuid>  kube-svc-postgres  tcp    10.53.74.93:5432   10.52.5.9:5432


# ═══════════════════════════════════════════════════════════════════
# CREATE LOGICAL TOPOLOGY MANUALLY (educational)
# ═══════════════════════════════════════════════════════════════════

# Create a logical switch:
ovn-nbctl ls-add tenant-acme

# Add logical port (pod):
ovn-nbctl lsp-add tenant-acme harbor-core
ovn-nbctl lsp-set-addresses harbor-core "0a:00:00:00:05:08 10.52.5.8"
ovn-nbctl lsp-set-port-security harbor-core "0a:00:00:00:05:08 10.52.5.8"

# Add another port:
ovn-nbctl lsp-add tenant-acme harbor-db
ovn-nbctl lsp-set-addresses harbor-db "0a:00:00:00:05:09 10.52.5.9"

# Create a logical router and connect to switch:
ovn-nbctl lr-add tenant-router

# Router port on the switch side:
ovn-nbctl lrp-add tenant-router lrp-to-acme 0a:00:00:00:01:01 10.52.5.1/24

# Switch port connecting to router:
ovn-nbctl lsp-add tenant-acme acme-to-router
ovn-nbctl lsp-set-type acme-to-router router
ovn-nbctl lsp-set-addresses acme-to-router router
ovn-nbctl lsp-set-options acme-to-router router-port=lrp-to-acme

# Add ACL: deny traffic between switches (tenant isolation):
ovn-nbctl acl-add tenant-acme to-lport 1000 \
  "ip4.dst == 10.52.0.0/16 && ip4.dst != 10.52.5.0/24" drop

# Add Load Balancer (Service):
ovn-nbctl lb-add postgres-svc 10.53.74.93:5432 10.52.5.9:5432
ovn-nbctl ls-lb-add tenant-acme postgres-svc

# ═══════════════════════════════════════════════════════════════════
# OVN SOUTHBOUND (physical implementation — read-only, managed by OVN)
# ═══════════════════════════════════════════════════════════════════

# Show chassis (physical nodes):
ovn-sbctl show
# Chassis "node01-chassis-uuid"
#     hostname: harvester-dev-lk-dc-node01
#     Encap geneve
#         ip: "192.168.10.15"
#     Port_Binding harbor-acme-core-<uuid>  ← this pod is on node01

# Show all port bindings:
ovn-sbctl list Port_Binding | grep -E "(logical_port|chassis|mac)"

# Show logical flows (translated but not yet OpenFlow):
ovn-sbctl lflow-list | head -50
# table=0 (ls_in_port_sec_l2), priority=50,
#   match=(inport == "harbor-acme-core" && eth.src == {0a:00:00:00:05:08}),
#   action=(next;)
```

---

## 6. Full OVN Packet Walk-Through

```
PACKET: harbor-acme-core (10.52.5.8) → harbor-acme-database (10.52.5.12)
Both pods on SAME node (192.168.10.15)

NODE 192.168.10.15 — OVS bridge br-int:

  Port numbers (example):
  OVS port 7: harbor-acme-core veth
  OVS port 9: harbor-acme-database veth

  OPENFLOW PIPELINE (OVN generated):

  Packet arrives from harbor-acme-core veth (port 7):

  table=0  [Ingress port classification]
    match: in_port=7
    action: set metadata=<acme-management-datapath-id>, set reg14=7
    → goto_table:8

  table=8  [Ingress port security L2]
    match: in_port=7, eth.src==0a:...:05:08
    action: next (allowed — port security match)
    → goto_table:10

  table=10 [Ingress L2 lookup]
    match: eth.dst==0a:...:05:12  (harbor-database's MAC)
    action: set reg15=9 (database's port number)
    → goto_table:32

  table=32 [ARP proxy / DHCP — skipped for unicast]
    → goto_table:48

  table=48 [Egress ACL check]
    match: metadata=acme-management, ip4.dst==10.52.5.12
    ACL rule: same namespace (10.52.5.0/24) → ALLOW
    action: next
    → goto_table:64

  table=64 [Egress delivery]
    match: reg15=9
    action: output:9  (send to port 9 = database veth)

  DATABASE POD RECEIVES PACKET ✅
  No physical hop — all in-kernel via OVS!


PACKET: harbor-acme-core (10.52.5.8) → harbor-beta-core (10.52.6.8)
Pods on DIFFERENT nodes (acme on node01, beta on node02):

  table=10 [Ingress L2 lookup — destination MAC is unknown locally]
    match: eth.dst==beta-core-MAC, metadata=acme-management
    ... no local port match ...
    → go to logical router (cross-switch routing)

  table=32 [Logical router]
    match: ip4.dst==10.52.6.8 (in different subnet 10.52.6.0/24)
    action: set eth.src=router-MAC, set eth.dst=beta-core-MAC
            set reg15=beta-core-port, load_balance
    → goto_table:48

  table=48 [ACL check on beta-management switch]
    ACL: deny traffic from 10.52.0.0/16 EXCEPT same namespace
    10.52.5.8 is NOT in 10.52.6.0/24 → BLOCKED
    action: drop

  PACKET DROPPED — CROSS-TENANT TRAFFIC BLOCKED ✅


PACKET: harbor-acme-core (10.52.5.8) → harbor-acme-database (10.52.5.12)
But database is on DIFFERENT node (node02):

  table=64 [Egress delivery]
    reg15 = database-port (on node02)
    OVN knows: database port is on chassis 192.168.10.17
    action: output to GENEVE tunnel to 192.168.10.17
             with GENEVE metadata = {datapath, ingress-port, egress-port}

  GENEVE packet:
    outer src: 192.168.10.15
    outer dst: 192.168.10.17
    UDP dst: 6081
    GENEVE options: datapath=acme-mgmt, ingress=core-port, egress=db-port
    inner: original harbor-core → harbor-db packet

  NODE 192.168.10.17 receives GENEVE:
    OVS decapsulates
    Reads GENEVE metadata → knows logical egress port = harbor-db
    Delivers directly to harbor-db veth (skip all lookup tables — metadata tells it!)
    DATABASE POD RECEIVES ✅
```

---

## Check Your Understanding

**Q1:** What is the difference between an OVN logical switch and an OVN logical router?
> Logical switch = L2 virtual LAN (like a VLAN). Devices on the same switch can reach each other directly (no routing). Logical router = L3 virtual router, connects multiple logical switches, routes packets between them based on IP subnet. In Kube-OVN, each namespace gets one logical switch; one cluster-wide logical router connects all of them.

**Q2:** OVS has a "slow path" and a "fast path". What is the difference?
> Fast path: packet matches a cached flow rule in the kernel datapath → executed in microseconds without process context switch. Slow path: no cached rule → packet sent to ovs-vswitchd process in userspace → vswitchd applies OpenFlow tables → installs result in cache → future packets for same flow take fast path. First packet of each flow is slow, all subsequent are fast.

**Q3:** How does GENEVE metadata help OVN avoid table lookups on the receiving node?
> When the sending node encapsulates the packet in GENEVE, it adds metadata fields (logical datapath ID, ingress port, egress port) that OVN computed. The receiving node's OVS reads these from the GENEVE header and knows exactly which logical port to deliver to — no MAC lookup, no ACL evaluation needed at the egress side. This is more efficient than VXLAN which carries no metadata.

---

## Summary

```
KEY COMPONENTS:
  OVS      = programmable software switch, replaces Linux bridge
  OpenFlow = the "program" OVS executes per packet (match + action)
  OVN      = control plane that generates OpenFlow rules automatically
             from high-level logical topology (switches, routers, ACLs, LBs)
  Kube-OVN = makes OVN/OVS Kubernetes-aware (watches K8s API, creates
             OVN objects for pods/services/NetworkPolicies)

HARVESTER USES:
  br-int (OVS bridge):
    ├── pod veth ports (Harbor pods in acme-management etc.)
    ├── GENEVE tunnel ports (to other Harvester nodes)
    └── patch port (to br-ex for external traffic)
  
  br-ex (OVS bridge):
    ├── patch port (from br-int)
    └── physical NIC port (eth0 → mgmt VLAN)

  OVN runs on Harvester's own K8s and manages all tenant pod networks.
  NetworkPolicy ACLs are compiled into OpenFlow rules in OVS.
  No iptables — everything is OVN ACLs + OVS OpenFlow.

NEXT: File 09 — Harvester + Rancher networking — your actual infrastructure
      explained with everything you've learned so far.
```
