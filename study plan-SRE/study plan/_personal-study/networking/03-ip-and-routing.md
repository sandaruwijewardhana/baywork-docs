# 03 — IP Addresses, Subnets, ARP, and Routing
## How Packets Find Their Way Across the World

---

## The Story

VLANs solved isolation on one switch. But what if two devices are on completely different
cities or even countries? MACs and VLANs only work within one local network.

IP (Internet Protocol) is the postal system for the entire internet.
Every device gets an IP address — a logical address that works globally.
Routers read IP addresses (not MACs) and forward packets hop-by-hop
from source to destination, across any number of networks.

ARP is the glue: it translates IP addresses to MAC addresses so that
within each hop, the Ethernet frame knows which physical device to go to.

---

## 1. IP Addresses and Subnets

```
IPv4 ADDRESS:
  192.168.10.15
  ─────────────
  4 numbers, each 0-255, separated by dots.
  Each number = 1 byte (8 bits).
  Total: 32 bits.

IN BINARY:
  192   .168   .10    .15
  11000000.10101000.00001010.00001111

CIDR NOTATION (Classless Inter-Domain Routing):
  192.168.10.0/24
              ─┘
              prefix length = 24 bits are the "network" part
              remaining 8 bits are the "host" part

  /24 means:
  Network:  192.168.10.   (first 24 bits, fixed — all devices here are on same LAN)
  Hosts:    .0 to .255    (last 8 bits, unique per device)

  .0   = network address  (not assignable)
  .255 = broadcast        (send to all on this subnet)
  .1 to .254 = usable IPs

COMMON SUBNET SIZES:
  /8   → 16,777,214 hosts   (10.0.0.0/8 — entire "10." range)
  /16  → 65,534 hosts       (172.16.0.0/16)
  /24  → 254 hosts          (192.168.1.0/24 — typical home/office LAN)
  /26  → 62 hosts           (smaller office segment)
  /30  → 2 hosts            (point-to-point link between two routers)
  /32  → 1 host             (single IP, used for loopback or VIPs)
```

```
YOUR SUBNETS:
  192.168.10.0/24   → Harvester management VLAN
                      .15 = node01, .17 = node02, .6 = MetalLB VIP
  172.22.100.0/24   → KubeVirt VMs (VLAN 700)
                      .3 = rke2-node01, others = more VMs
  10.52.0.0/16      → Harvester pod IPs (Kube-OVN assigns)
  10.53.0.0/16      → Harvester service IPs
  10.42.0.0/16      → registry-controller pods (Calico assigns)
  10.43.0.0/16      → registry-controller services
```

---

## 2. ARP — Address Resolution Protocol

Before a device can send an Ethernet frame, it needs the **MAC address** of the next hop.
It knows the IP (from the routing table) but not the MAC. ARP resolves IP → MAC.

```
PROBLEM: Pod (10.42.5.8) wants to reach 10.42.5.12.
         It knows the IP but needs the MAC to write the Ethernet frame.

ARP PROCESS:
═══════════════════════════════════════════════════════════════════════

  Step 1: Pod broadcasts ARP Request
  ┌────────────────────────────────────────────────────────────────┐
  │ ARP REQUEST (broadcast — everyone on LAN receives it)          │
  │ "Who has 10.42.5.12? Tell 10.42.5.8"                          │
  │ Dest MAC: ff:ff:ff:ff:ff:ff  (broadcast)                       │
  │ Src  MAC: 0a:00:00:00:05:08  (my MAC)                          │
  └────────────────────────────────────────────────────────────────┘

  Step 2: Owner of 10.42.5.12 sends ARP Reply (unicast)
  ┌────────────────────────────────────────────────────────────────┐
  │ ARP REPLY (unicast — only to the requester)                    │
  │ "10.42.5.12 is at 0a:00:00:00:05:12"                          │
  │ Dest MAC: 0a:00:00:00:05:08  (back to requester)               │
  └────────────────────────────────────────────────────────────────┘

  Step 3: Requester caches the answer
  ARP CACHE (ip neigh show):
  ┌────────────────┬───────────────────┬───────────┐
  │ IP Address     │ MAC Address       │ State     │
  ├────────────────┼───────────────────┼───────────┤
  │ 10.42.5.12     │ 0a:00:00:00:05:12 │ REACHABLE │
  └────────────────┴───────────────────┴───────────┘
  Cache expires in ~30-60 seconds. Then ARP happens again.

  Step 4: Now pod can send Ethernet frame with correct MAC
  Dest MAC: 0a:00:00:00:05:12  ← filled in from ARP cache
  Src  MAC: 0a:00:00:00:05:08
  Payload:  [IP packet]
```

```bash
# See your ARP cache
ip neigh show

# Output example:
# 192.168.10.1  dev eth0  lladdr aa:bb:cc:dd:ee:ff  REACHABLE
# 192.168.10.17 dev eth0  lladdr 11:22:33:44:55:66  STALE

# States:
# REACHABLE = confirmed working recently
# STALE     = cache entry old, will re-verify on next use
# FAILED    = ARP failed, no response (host unreachable)

# Manually add a static ARP entry (useful for debugging):
sudo ip neigh add 192.168.10.20 lladdr aa:bb:cc:11:22:33 dev eth0

# Delete an ARP entry (force re-ARP):
sudo ip neigh del 192.168.10.20 dev eth0
```

---

## 3. The Routing Table

Every device (computer, router, even a pod) has a routing table.
It answers one question: **"For this destination IP, where do I send the packet?"**

```
ROUTING TABLE STRUCTURE:
┌─────────────────┬─────────────────┬──────────────┬────────────┬────────┐
│ Destination     │ Gateway         │ Interface    │ Metric     │ Source │
├─────────────────┼─────────────────┼──────────────┼────────────┼────────┤
│ 172.22.100.0/24 │ 0.0.0.0         │ eth0         │ 0          │ kernel │
│ 10.42.0.0/16    │ 0.0.0.0         │ tunl0        │ 0          │ calico │
│ 10.42.97.0/26   │ 172.22.100.x    │ tunl0        │ 0          │ calico │
│ 0.0.0.0/0       │ 172.22.100.1    │ eth0         │ 0          │ dhcp   │
└─────────────────┴─────────────────┴──────────────┴────────────┴────────┘

HOW ROUTING DECISIONS ARE MADE:

  Packet to 10.42.97.5 — which row matches?
  ├── 172.22.100.0/24 → "is 10.42.97.5 in 172.22.100.0-254?" NO
  ├── 10.42.0.0/16    → "is 10.42.97.5 in 10.42.0.0-255.255?" YES
  ├── 10.42.97.0/26   → "is 10.42.97.5 in 10.42.97.0-63?" YES ← more specific!
  └── 0.0.0.0/0       → matches everything (default route)

  LONGEST PREFIX MATCH wins: 10.42.97.0/26 (26 bits match) beats 10.42.0.0/16 (16 bits match)
  → Send via tunl0 to gateway 172.22.100.x

  Packet to 8.8.8.8 (Google DNS):
  ├── 172.22.100.0/24 → NO
  ├── 10.42.0.0/16    → NO
  ├── 10.42.97.0/26   → NO
  └── 0.0.0.0/0       → YES (default route — catch-all)
  → Send via eth0 to gateway 172.22.100.1 (WSO2 router)
```

```bash
# Show routing table
ip route show

# Show routing table with more detail
ip route show table main

# Trace which route would be used for a specific destination
ip route get 8.8.8.8
# Output: 8.8.8.8 via 172.22.100.1 dev eth0 src 172.22.100.3 uid 0

ip route get 10.42.5.8
# Output: 10.42.5.8 dev tunl0 src 10.42.96.1

# Add a static route:
sudo ip route add 192.168.10.0/24 via 172.22.100.1 dev eth0

# Add default gateway:
sudo ip route add default via 172.22.100.1

# Delete a route:
sudo ip route del 192.168.10.0/24

# Temporarily flush all routes (dangerous on production!):
sudo ip route flush table main
```

---

## 4. How a Packet Hops Across Multiple Routers

```
JOURNEY: Pod in registry-controller-cluster pings Google (8.8.8.8)

  POD (10.42.5.8) on VM (172.22.100.3) on NODE (192.168.10.15)

  ────────────────────────────────────────────────────────────────────
  HOP 0: Pod creates IP packet
  ────────────────────────────────────────────────────────────────────
  IP Header:
    Source: 10.42.5.8
    Destination: 8.8.8.8
    TTL: 64   ← decremented by 1 at each router

  Pod's routing table:
    0.0.0.0/0 via 169.254.1.1 dev eth0  ← Calico's special gateway
  → Send to 169.254.1.1

  ────────────────────────────────────────────────────────────────────
  HOP 1: Pod → Calico on node (same node, no physical hop)
  ────────────────────────────────────────────────────────────────────
  Calico intercepts: "10.42.5.8 is going to 8.8.8.8"
  Calico's routing table:
    10.42.0.0/16 = local pod range, but 8.8.8.8 is external
  Calico applies SNAT: changes source from 10.42.5.8 → 172.22.100.3
  (so replies can find their way back — 10.42.5.8 is not routable externally)
  Sends out VM's eth0 (172.22.100.3) towards 172.22.100.1

  Ethernet frame:
    Src  MAC: VM's MAC (aa:bb:cc...)
    Dest MAC: gateway MAC (ARP for 172.22.100.1)
    IP src: 172.22.100.3 (SNAT applied)
    IP dst: 8.8.8.8
    TTL: 63  (decremented by Calico)

  ────────────────────────────────────────────────────────────────────
  HOP 2: VM eth0 → harvester-br0 → eth2 → physical switch
  ────────────────────────────────────────────────────────────────────
  VM eth0 → tap device → harvester-br0
  Bridge tags frame with VLAN 700
  eth2 → physical switch (trunk port)
  Switch reads: VLAN 700, dst MAC = WSO2 router's MAC
  Forwards to uplink port → WSO2 router

  ────────────────────────────────────────────────────────────────────
  HOP 3: WSO2 Router (L3 routing)
  ────────────────────────────────────────────────────────────────────
  CRITICAL: Router strips the Ethernet frame (MAC layer dies here)
  Router reads IP packet: src=172.22.100.3, dst=8.8.8.8, TTL=63

  Router's routing table:
    172.22.100.0/24  → directly connected (VLAN 700 interface)
    192.168.10.0/24  → directly connected (mgmt VLAN interface)
    0.0.0.0/0        → via 203.x.x.1 (ISP gateway, WAN link)

  8.8.8.8 matches 0.0.0.0/0 → forward to WAN gateway 203.x.x.1
  Router builds NEW Ethernet frame:
    Src  MAC: router's WAN NIC MAC
    Dest MAC: ISP gateway MAC
    IP unchanged: src=172.22.100.3, dst=8.8.8.8, TTL=62

  ────────────────────────────────────────────────────────────────────
  HOP 4-N: Through internet (multiple ISP routers)
  ────────────────────────────────────────────────────────────────────
  Each router:
  ├── Strips incoming Ethernet frame (MAC dies)
  ├── Reads IP dest: 8.8.8.8
  ├── Looks up its routing table
  ├── Builds new Ethernet frame with next-hop MAC
  └── Decrements TTL (if TTL=0, drops packet and sends ICMP Time Exceeded)

  ────────────────────────────────────────────────────────────────────
  HOP N: Google's datacenter router → Google server
  ────────────────────────────────────────────────────────────────────
  Google server receives: src=172.22.100.3, dst=8.8.8.8 (its own IP)
  Google replies: src=8.8.8.8, dst=172.22.100.3
  Reply travels back through internet → WSO2 router → VLAN 700 → VM

  Calico on VM: "I sent the original, I need to un-SNAT this"
  Calico DNAT: dst=172.22.100.3 → 10.42.5.8 (original pod IP)
  Pod receives reply ✅

  KEY INSIGHT: The IP src/dst stays the same across all hops.
               Only the Ethernet src/dst (MAC) changes at each hop.
```

---

## 5. Default Gateway — The Exit Door

```
DEFAULT GATEWAY:
  Every device on a LAN has one router it talks to for everything outside the LAN.
  This is the default gateway.

  If destination is in my subnet → send directly (ARP for the device)
  If destination is outside my subnet → send to default gateway

EXAMPLE:
  Your pod is on 10.42.5.0/24.
  Pod wants to reach 10.42.5.12: same /24 → ARP directly → no router needed
  Pod wants to reach 8.8.8.8:   not in 10.42.5.0/24 → send to gateway (169.254.1.1)
  Pod wants to reach 10.42.97.5: not in 10.42.5.0/24 → Calico handles routing

CHECK:
  ip route show | grep default
  # default via 172.22.100.1 dev eth0

  ip route get 8.8.8.8
  # shows which gateway + interface will be used
```

---

## 6. Private vs Public IPs

```
PRIVATE IP RANGES (RFC 1918 — not routable on public internet):
  10.0.0.0/8         (16M addresses — e.g. Kubernetes pods)
  172.16.0.0/12      (1M addresses  — e.g. your VMs: 172.22.100.x)
  192.168.0.0/16     (65K addresses — e.g. Harvester mgmt: 192.168.10.x)

  These are free to use in private networks.
  CANNOT be used on the public internet.
  ISPs drop packets with RFC1918 destinations/sources.

PUBLIC IP RANGES:
  Everything else → assigned by IANA to ISPs → expensive, limited

WHY YOUR PODS CAN REACH GOOGLE:
  10.42.5.8 (private) → Calico SNAT → 172.22.100.3 (also private)
  → WSO2 router SNAT → <WSO2 public IP> (public IP from WSO2's ISP)
  Google sees source = WSO2's public IP
  Reply goes back to WSO2 public IP → WSO2 NAT → 172.22.100.3 → 10.42.5.8

  Multiple layers of NAT, each hiding the private addresses behind a public one.
```

---

## LAB 4 — Build a Two-Router Network in Linux

```bash
#!/bin/bash
# lab4-routing.sh
# Creates: Net-A (10.10.0.x) ──── Router ──── Net-B (10.20.0.x)
# Tests that Host-A can reach Host-B through the router

# Enable IP forwarding (makes this Linux act as a router)
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Create network namespaces (isolated network stacks — like mini VMs)
sudo ip netns add host-a
sudo ip netns add host-b
sudo ip netns add router

# Create veth pairs
# veth-a ↔ veth-router-a  (connects host-a to router)
# veth-b ↔ veth-router-b  (connects host-b to router)
sudo ip link add veth-a    type veth peer name veth-router-a
sudo ip link add veth-b    type veth peer name veth-router-b

# Put interfaces into their namespaces
sudo ip link set veth-a       netns host-a
sudo ip link set veth-router-a netns router
sudo ip link set veth-b       netns host-b
sudo ip link set veth-router-b netns router

# Configure host-a: 10.10.0.2, gateway = 10.10.0.1 (router)
sudo ip netns exec host-a ip link set veth-a up
sudo ip netns exec host-a ip addr add 10.10.0.2/24 dev veth-a
sudo ip netns exec host-a ip route add default via 10.10.0.1

# Configure host-b: 10.20.0.2, gateway = 10.20.0.1 (router)
sudo ip netns exec host-b ip link set veth-b up
sudo ip netns exec host-b ip addr add 10.20.0.2/24 dev veth-b
sudo ip netns exec host-b ip route add default via 10.20.0.1

# Configure router: has interfaces on both networks
sudo ip netns exec router ip link set veth-router-a up
sudo ip netns exec router ip link set veth-router-b up
sudo ip netns exec router ip addr add 10.10.0.1/24 dev veth-router-a
sudo ip netns exec router ip addr add 10.20.0.1/24 dev veth-router-b
sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1

echo ""
echo "=== Router routing table ==="
sudo ip netns exec router ip route show

echo ""
echo "=== Host-A routing table ==="
sudo ip netns exec host-a ip route show

echo ""
echo "=== Ping from host-a to host-b (through router) ==="
sudo ip netns exec host-a ping -c 3 10.20.0.2

echo ""
echo "=== Traceroute from host-a to host-b ==="
sudo ip netns exec host-a traceroute 10.20.0.2 2>/dev/null || \
  sudo ip netns exec host-a tracepath 10.20.0.2

echo ""
echo "=== ARP cache on host-a (only knows router, not host-b) ==="
sudo ip netns exec host-a ip neigh show

# Cleanup
sudo ip netns del host-a
sudo ip netns del host-b
sudo ip netns del router
```

Expected output shows:
- Router routing table has routes for both 10.10.0.0/24 and 10.20.0.0/24
- Ping works (host-a → router → host-b)
- Traceroute shows: host-a → 10.10.0.1 (router) → 10.20.0.2 (host-b)
- ARP cache on host-a shows only the ROUTER's MAC — not host-b (it never ARPs for host-b directly)

---

## Check Your Understanding

**Q1:** You have IP 172.22.100.3/24. Your routing table has only:
```
172.22.100.0/24 dev eth0
0.0.0.0/0 via 172.22.100.1 dev eth0
```
You ping 10.42.5.8. What happens?
> 10.42.5.8 is NOT in 172.22.100.0/24. The only match is 0.0.0.0/0 (default route). Packet goes to 172.22.100.1 (gateway). Gateway decides what to do next. If gateway has no route to 10.42.0.0/16, packet is dropped.

**Q2:** Why does a router STRIP the Ethernet frame and rebuild it at each hop?
> Ethernet frames (MACs) are local — a MAC address is meaningless outside the local LAN. Each hop is a different LAN, so a new Ethernet frame is needed with new source/dest MACs for that specific LAN segment. The IP header stays intact across all hops.

**Q3:** What is the purpose of TTL in the IP header?
> Prevents packets from looping forever. Each router decrements TTL by 1. When TTL reaches 0, the router drops the packet and sends an ICMP "Time Exceeded" message back to the source. `traceroute` exploits this to discover the path.

---

## Summary

```
KEY FACTS:
  ├── IP address = logical 32-bit address, globally meaningful
  ├── Subnet = group of IPs sharing a network prefix (e.g. /24 = same LAN)
  ├── ARP = translates IP → MAC within one LAN (broadcast, never crosses routers)
  ├── ARP cache = table of known IP→MAC mappings, expires after 30-60 seconds
  ├── Routing table = decides next hop for every destination (longest prefix match)
  ├── Default gateway = the router to use for everything outside local subnet
  ├── Router strips Ethernet frame at each hop, rebuilds with new MACs
  └── IP src/dst stays the same across all hops; only MACs change per hop

YOUR SETUP'S KEY ROUTING BOUNDARIES:
  172.22.100.x (VLAN 700) ←→ 192.168.10.x (mgmt) : WSO2 router connects them
  10.42.x.x (pods)                                 : Calico routes internally in RKE2
  10.52.x.x (Harvester pods)                       : Kube-OVN routes internally in Harvester

NEXT: File 04 — NAT and iptables — how addresses get rewritten and firewalls work.
```
