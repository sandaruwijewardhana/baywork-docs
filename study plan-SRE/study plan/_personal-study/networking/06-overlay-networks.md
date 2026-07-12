# 06 — Overlay Networks
## VXLAN, GENEVE, IPIP — How Pods Talk Across Physical Machines

---

## The Story

You have pods on Node1 and pods on Node2. A pod on Node1 has IP 10.42.5.8.
A pod on Node2 has IP 10.42.97.5. These are completely different subnets.
The physical network between the nodes only knows about 172.22.100.x addresses.
The physical switch has never heard of 10.42.x.x.

How can 10.42.5.8 reach 10.42.97.5?

The answer: wrap the pod's IP packet inside ANOTHER IP packet that the physical network DOES understand. Send it across the physical network. Unwrap at the other side.

This is called **tunneling** or **overlay networking** — a virtual network layered on top of the physical network.

---

## 1. The Problem Without Tunnels

```
WITHOUT OVERLAY:
═══════════════════════════════════════════════════════════════════════

  Pod1 (10.42.5.8) on Node1 (172.22.100.3)
  Pod2 (10.42.97.5) on Node2 (172.22.100.x)

  Pod1 sends packet: src=10.42.5.8  dst=10.42.97.5

  Pod1's routing table: "10.42.97.5 → default gateway"
  Packet reaches Node1's kernel: "10.42.97.5 → ??? I have no route"

  Physical switch: "10.42.97.5 → no MAC for this, flood or drop"
  Physical router: "10.42.97.5 → not in routing table, drop"

  RESULT: ❌ packet lost. Physical network knows NOTHING about 10.42.x.x.

WITH OVERLAY:
═══════════════════════════════════════════════════════════════════════

  Pod1 sends: src=10.42.5.8  dst=10.42.97.5  (INNER packet)

  Calico/Kube-OVN on Node1 intercepts:
  "Pod 10.42.97.5 is on Node2 (172.22.100.x) — I know this from BGP/OVSDB"
  WRAPS the inner packet in a new OUTER packet:

  OUTER packet: src=172.22.100.3  dst=172.22.100.x
  INNER packet: src=10.42.5.8     dst=10.42.97.5   (original, untouched)

  Physical network: "172.22.100.x → I know that! It's on this switch"
  Physical switch delivers to Node2's eth0

  Node2's CNI: "incoming OUTER packet from 172.22.100.3, let me unwrap..."
  Node2 removes outer header → reads inner: dst=10.42.97.5
  10.42.97.5 is a local pod → deliver via local veth → Pod2 receives ✅
```

---

## 2. IPIP — IP in IP (Used by Calico)

The simplest tunnel: wrap one IP packet inside another IP packet.

```
IPIP ENCAPSULATION:
═══════════════════════════════════════════════════════════════════════

ORIGINAL PACKET (from Pod1 10.42.5.8 to Pod2 10.42.97.5):
┌──────────────────────────────────────────────────────────────────┐
│  IP Header: src=10.42.5.8  dst=10.42.97.5  proto=TCP           │
│  TCP Header: sport=49000  dport=8080                            │
│  Data: "GET /api/v1/users HTTP/1.1"                             │
└──────────────────────────────────────────────────────────────────┘

AFTER IPIP ENCAPSULATION (Calico adds outer IP header):
┌──────────────────────────────────────────────────────────────────┐
│  OUTER IP: src=172.22.100.3  dst=172.22.100.x  proto=4(IPIP)   │
├──────────────────────────────────────────────────────────────────┤
│  INNER IP: src=10.42.5.8  dst=10.42.97.5  proto=TCP             │
│  TCP: sport=49000  dport=8080                                   │
│  Data: "GET /api/v1/users HTTP/1.1"                             │
└──────────────────────────────────────────────────────────────────┘

OVERHEAD: +20 bytes (size of one IP header)
PROTOCOL: proto=4 (IP-in-IP, IANA assigned)

WHAT HAPPENS AT EACH HOP:
  Node1 kernel: OUTER src=172.22.100.3, OUTER dst=172.22.100.x → route via eth0
  Physical switch: sees 172.22.100.x → delivers to Node2
  Node2 kernel: proto=4 (IPIP) → strip outer header → inner dst=10.42.97.5
  Node2 Calico: 10.42.97.5 is local → deliver via cali veth to Pod2
```

```bash
# IPIP in action — Calico uses the tunl0 interface:

# Show the IPIP tunnel interface created by Calico:
ip link show tunl0
# Output: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN
#         link/ipip 0.0.0.0 peer 0.0.0.0
# Note: MTU is 1480 = 1500 (ethernet) - 20 (IPIP overhead)

# Show Calico's tunnel routes:
ip route show | grep tunl0
# 10.42.97.0/26 via 172.22.100.x dev tunl0  (pod CIDR on Node2 → via Node2's IP through tunnel)
# 10.42.98.0/26 via 172.22.100.y dev tunl0  (pod CIDR on Node3 → via Node3's IP through tunnel)

# See IPIP packets on the wire (outer IP proto=4):
sudo tcpdump -i eth0 -n proto 4
# When a cross-node pod-to-pod packet happens, you'll see:
# 172.22.100.3 > 172.22.100.x: IP 10.42.5.8 > 10.42.97.5: TCP 8080
```

---

## 3. VXLAN — Virtual eXtensible LAN

VXLAN wraps Ethernet frames (not just IP packets) inside UDP packets. This allows L2 networks to span multiple physical locations. Used by Flannel (k3s default) and many cloud providers.

```
VXLAN ENCAPSULATION:
═══════════════════════════════════════════════════════════════════════

ORIGINAL: Pod1 sends Ethernet frame to Pod2's MAC (L2 frame, not L3 packet)
┌─────────────────────────────────────────────────────────────────┐
│  Ethernet: dst-MAC=0a:00:00:00:97:05  src-MAC=0a:00:00:00:05:08 │
│  IP: src=10.42.5.8  dst=10.42.97.5                              │
│  TCP: ...data...                                                 │
└─────────────────────────────────────────────────────────────────┘

AFTER VXLAN ENCAPSULATION:
┌─────────────────────────────────────────────────────────────────┐
│  OUTER Ethernet: dst=Node2-MAC   src=Node1-MAC                  │
│  OUTER IP:   src=172.22.100.3   dst=172.22.100.x                │
│  OUTER UDP:  sport=12345        dport=4789 (VXLAN)              │
│  VXLAN HEADER: VNI=42 (Virtual Network Identifier)              │
├─────────────────────────────────────────────────────────────────┤
│  INNER Ethernet: dst=Pod2-MAC  src=Pod1-MAC (ORIGINAL FRAME)    │
│  INNER IP: src=10.42.5.8   dst=10.42.97.5                       │
│  TCP: ...data...                                                 │
└─────────────────────────────────────────────────────────────────┘

OVERHEAD: +50 bytes (eth+IP+UDP+VXLAN headers)
PORT: UDP 4789 (must be open in firewalls!)

VNI (Virtual Network Identifier): 24 bits → 16 million virtual networks
Used to separate different overlay networks (like VLAN but over IP)

WHY UDP?
  UDP is stateless — routers forward it without tracking.
  The overlay protocol handles its own reliability (TCP inside handles it).
  Some implementations also use TCP for tunnels.
```

```bash
# Create a VXLAN interface (what Flannel does):
# This creates a VXLAN tunnel endpoint on this machine
sudo ip link add name vxlan0 type vxlan \
  id 42 \              # VNI = 42
  dstport 4789 \       # standard VXLAN port
  dev eth0 \           # use eth0 as the physical transport
  local 172.22.100.3   # this node's IP

sudo ip link set vxlan0 up
sudo ip addr add 10.42.5.1/24 dev vxlan0

# Add FDB entry (forward inner MACs to remote VTEP):
# "MAC 0a:00:00:00:97:05 (Pod2's MAC) lives at 172.22.100.x:4789"
sudo bridge fdb add 0a:00:00:00:97:05 dev vxlan0 dst 172.22.100.x

# Now packets to Pod2 will be VXLAN encapsulated and sent to 172.22.100.x

# Capture VXLAN traffic:
sudo tcpdump -i eth0 -n udp port 4789
# 172.22.100.3.12345 > 172.22.100.x.4789: VXLAN, flags [I], vni 42
#   IP 10.42.5.8 > 10.42.97.5: TCP
```

---

## 4. GENEVE — Generic Network Virtualization Encapsulation (Used by Kube-OVN/OVN)

GENEVE is the evolution of VXLAN. It's more flexible because it supports extensible metadata in the header. OVN (and therefore Kube-OVN on Harvester) uses GENEVE.

```
GENEVE vs VXLAN:
═══════════════════════════════════════════════════════════════════════

  VXLAN header:  fixed 8 bytes, VNI only
  GENEVE header: variable length, VNI + arbitrary TLV options

  TLV = Type-Length-Value options (like HTTP headers but in binary)
  OVN uses GENEVE options to carry:
    ├── Logical datapath (which virtual network/switch this belongs to)
    ├── Logical ingress port (which OVN logical port sent it)
    └── Logical egress port (which OVN logical port should receive it)

  This metadata lets the remote node's OVN know which logical switch/router
  this packet belongs to WITHOUT looking up any tables — the info is in the header.

GENEVE ENCAPSULATION (used by Kube-OVN on Harvester):
┌──────────────────────────────────────────────────────────────────┐
│  OUTER Ethernet: Node1-MAC → Node2-MAC                           │
│  OUTER IP: 192.168.10.15 → 192.168.10.17  (Harvester node IPs)  │
│  OUTER UDP: sport=random  dport=6081 (Geneve port)               │
│  GENEVE HEADER:                                                   │
│    VNI: logical network ID                                        │
│    Options:                                                       │
│      type=0x0101 value=[logical_datapath_id]  ← which OVN switch │
│      type=0x0102 value=[logical_ingress_port] ← source port      │
│      type=0x0103 value=[logical_egress_port]  ← dest port        │
├──────────────────────────────────────────────────────────────────┤
│  INNER Ethernet: Pod1-MAC → Pod2-MAC                             │
│  INNER IP: 10.52.5.8 → 10.52.97.5  (Harbor pod IPs)             │
│  TCP/UDP: actual data                                             │
└──────────────────────────────────────────────────────────────────┘

PORT: UDP 6081 (must be open between Harvester nodes!)
```

```bash
# Create a GENEVE tunnel (what OVN/Kube-OVN does):
sudo ip link add name genev0 type geneve \
  id 100 \
  remote 192.168.10.17 \
  dstport 6081

sudo ip link set genev0 up

# OVS uses GENEVE via ovs-vsctl (see file 08 for OVS details):
# ovs-vsctl add-port br-int genev0 -- \
#   set interface genev0 type=geneve \
#   options:remote_ip=192.168.10.17 \
#   options:key=flow \
#   options:dst_port=6081

# Capture GENEVE traffic between Harvester nodes:
sudo tcpdump -i eth0 -n udp port 6081
```

---

## 5. How CNI Knows Which Node Has Which Pod — Route Distribution

The tunnel solves HOW to deliver. But how does Node1 know Pod2 is on Node2?

```
CALICO: BGP (Border Gateway Protocol)
═══════════════════════════════════════════════════════════════════════
  Each node runs a BIRD daemon (BGP speaker).
  When Node2 starts pod 10.42.97.5, Node2's Calico:
    BGP announces: "I have subnet 10.42.97.0/26" to all other nodes
  
  Node1's Calico receives BGP update:
    Adds route: "10.42.97.0/26 via 172.22.100.x dev tunl0"
  
  Now Node1 knows to tunnel packets for 10.42.97.x to Node2.

  bird routing table on a node:
  BGP.origin: IGP
  BGP.next_hop: 172.22.100.x    ← Node2's physical IP
  10.42.97.0/26                 ← Pod CIDR on Node2

  Linux route table after Calico processes BGP:
  10.42.97.0/26 via 172.22.100.x dev tunl0 proto bird


KUBE-OVN/OVN: OVSDB (OVS Database)
═══════════════════════════════════════════════════════════════════════
  OVN centrally manages the logical topology.
  ovn-controller on each node connects to the central OVN southbound DB.
  When a pod starts on Node1:
    ovn-controller registers: "logical port harbor-pod1 is on Node1 (192.168.10.15)"
  
  ovn-controller on Node2 sees this in southbound DB:
    "harbor-pod1 is on 192.168.10.15"
    Programs OVS: "send packets to harbor-pod1 via GENEVE to 192.168.10.15"
  
  No BGP needed — central DB (OVN southbound) distributes the information.
```

---

## 6. MTU — The Hidden Problem with Tunnels

```
MTU = Maximum Transmission Unit = max size of one Ethernet frame payload

Standard Ethernet MTU: 1500 bytes

PROBLEM WITH TUNNELS:
  Each tunnel adds overhead (headers):
  IPIP:   +20 bytes → effective MTU = 1480
  VXLAN:  +50 bytes → effective MTU = 1450
  GENEVE: +58 bytes → effective MTU = 1442

  If pod sends 1500-byte IP packet → Calico encapsulates in IPIP
  → outer packet is 1520 bytes → BIGGER THAN ETHERNET MTU
  → physical NIC fragments or drops the packet ❌

SOLUTIONS:
  1. Set pod MTU to 1480 (IPIP) or 1450 (VXLAN) → packets fit in outer frame
     Calico configures this automatically.
     Inside pod: ip link show eth0 → MTU 1480 (not 1500)

  2. Enable "jumbo frames" on physical switch/NICs (MTU 9000+)
     Then overlay overhead doesn't matter.
     Requires switch config and NIC support.

  3. PMTUD (Path MTU Discovery):
     Packets with DF (Don't Fragment) bit → router sends ICMP "fragmentation needed"
     Sender reduces packet size.
     Works but adds latency; some firewalls block ICMP → black hole.

YOUR SETUP:
  Calico (in VMs): pod MTU = 1480 (IPIP mode)
  Kube-OVN (Harvester): pod MTU = 1442 (GENEVE mode)
```

---

## LAB 7 — See Overlay Tunneling in Action

```bash
#!/bin/bash
# lab7-overlay.sh
# Creates a simple IPIP tunnel between two namespaces simulating two nodes

# Namespaces represent: Node1 (172.22.100.3) and Node2 (172.22.100.4)
sudo ip netns add node1
sudo ip netns add node2

# "Physical" link between nodes
sudo ip link add veth-n1 type veth peer name veth-n2
sudo ip link set veth-n1 netns node1
sudo ip link set veth-n2 netns node2

sudo ip netns exec node1 ip link set veth-n1 up
sudo ip netns exec node1 ip addr add 172.22.100.3/24 dev veth-n1

sudo ip netns exec node2 ip link set veth-n2 up
sudo ip netns exec node2 ip addr add 172.22.100.4/24 dev veth-n2

# Enable routing in both nodes
sudo ip netns exec node1 sysctl -w net.ipv4.ip_forward=1 >/dev/null
sudo ip netns exec node2 sysctl -w net.ipv4.ip_forward=1 >/dev/null

# Create IPIP tunnels (Calico does this)
sudo ip netns exec node1 ip tunnel add tunl0 mode ipip \
  local 172.22.100.3 remote 172.22.100.4

sudo ip netns exec node2 ip tunnel add tunl0 mode ipip \
  local 172.22.100.4 remote 172.22.100.3

sudo ip netns exec node1 ip link set tunl0 up
sudo ip netns exec node2 ip link set tunl0 up

# Assign pod subnet routes through tunnel
# Node1 has pod subnet 10.42.5.0/24, Node2 has 10.42.97.0/24
sudo ip netns exec node1 ip addr add 10.42.5.1/24 dev tunl0
sudo ip netns exec node1 ip route add 10.42.97.0/24 dev tunl0

sudo ip netns exec node2 ip addr add 10.42.97.1/24 dev tunl0
sudo ip netns exec node2 ip route add 10.42.5.0/24 dev tunl0

echo "=== Routing table on Node1 ==="
sudo ip netns exec node1 ip route show

echo ""
echo "=== Ping from pod-on-Node1 (10.42.5.1) to pod-on-Node2 (10.42.97.1) ==="
sudo ip netns exec node1 ping -c 3 10.42.97.1

echo ""
echo "=== MTU on tunnel interface (should be 1480 = 1500 - 20 IPIP) ==="
sudo ip netns exec node1 ip link show tunl0 | grep mtu

# Capture tunnel traffic (IPIP packets look like IP proto=4):
echo ""
echo "=== Capturing tunnel traffic (run ping again to see it) ==="
echo "(Press Ctrl+C after a few seconds)"
sudo ip netns exec node1 tcpdump -i veth-n1 -n proto 4 -c 4 2>/dev/null &
sleep 0.5
sudo ip netns exec node1 ping -c 3 -q 10.42.97.1 >/dev/null
wait

# Cleanup
sudo ip netns del node1
sudo ip netns del node2
```

---

## Check Your Understanding

**Q1:** Why can't pods just use their IPs directly without tunnels?
> The physical network (switches, routers) only knows about node IPs (172.22.100.x). Pod IPs (10.42.x.x) are "invented" by Kubernetes. No physical routing table has entries for 10.42.x.x, so packets would be dropped. Tunnels solve this by wrapping pod packets inside node packets that the physical network understands.

**Q2:** VXLAN wraps Ethernet frames; IPIP wraps IP packets. Which carries more overhead and why?
> VXLAN carries more overhead (~50 bytes) vs IPIP (~20 bytes) because VXLAN adds outer Ethernet + outer IP + outer UDP + VXLAN headers, while IPIP only adds one IP header. VXLAN can emulate L2 (Ethernet), IPIP only does L3 (IP).

**Q3:** Why does Kube-OVN (Harvester) use GENEVE instead of VXLAN?
> GENEVE supports extensible metadata options in the header. OVN uses these options to carry logical topology metadata (which virtual switch, which logical port) without needing separate lookups. This makes the OVS pipeline more efficient — the remote node's OVS knows exactly what logical path to follow from the GENEVE header alone.

---

## Summary

```
OVERLAY TYPES:
  IPIP   → IP in IP. Calico uses this. +20 bytes. Simple, L3 only.
  VXLAN  → Ethernet in UDP. Flannel uses this. +50 bytes. L2 over L3.
  GENEVE → Ethernet in UDP with metadata. OVN/Kube-OVN uses this. +58 bytes.

IN YOUR SETUP:
  registry-controller-cluster (Calico):
    pod-to-pod cross-node = IPIP tunnel via tunl0
    Node IPs 172.22.100.x are the tunnel endpoints

  Harvester cluster (Kube-OVN/OVN):
    pod-to-pod cross-node = GENEVE tunnel via OVS br-int
    Node IPs 192.168.10.x are the tunnel endpoints
    OVN adds logical port metadata in GENEVE options

  Cross-cluster (provisioner → Harbor):
    No tunnel — must go through physical network / K8s API proxy
    These are different Kubernetes clusters with no shared CNI

NEXT: File 07 — Kubernetes networking: how CNI, Services, kube-proxy,
      CoreDNS all work together as a complete system.
```
