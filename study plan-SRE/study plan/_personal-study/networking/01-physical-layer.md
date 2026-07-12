# 01 — Physical Layer
## Cables, NICs, Switches, MAC Addresses

---

## The Story

Imagine a neighbourhood where everyone communicates by sending letters.
Each house has a unique address painted on the door — that's a MAC address.
The postman (the switch) walks every street and remembers which house is on which street.
When a letter arrives, the postman reads the destination address and delivers it only to that house.

Now imagine if instead of one postman there was an early naive system where every letter was
shoved under every single door in the neighbourhood — that was a "hub" (now obsolete).
The switch replaced the hub by being smarter: it only delivers to the right door.

---

## 1. What is a NIC?

A Network Interface Card is the hardware that connects your computer to a network.

```
YOUR COMPUTER / SERVER
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  CPU ──── RAM ──── MOTHERBOARD                          │
│                         │                              │
│                    PCIe slot                           │
│                         │                              │
│                    ┌────┴──────────────────┐           │
│                    │  NIC  (Network Card)   │           │
│                    │  ┌──────────────────┐  │           │
│                    │  │  RJ45 PORT (eth0) │◄─┼─── cable  │
│                    │  └──────────────────┘  │           │
│                    │  ┌──────────────────┐  │           │
│                    │  │  RJ45 PORT (eth1) │◄─┼─── cable  │
│                    │  └──────────────────┘  │           │
│                    └───────────────────────┘           │
└─────────────────────────────────────────────────────────┘

NIC responsibilities:
  ├── Convert digital bits from CPU → electrical/optical signals on the wire
  ├── Convert incoming signals from wire → digital bits for CPU
  ├── Has a permanent hardware address (MAC address) burned in at factory
  └── Can run at speeds: 1 Gbps, 10 Gbps, 25 Gbps, 100 Gbps
```

---

## 2. What is a MAC Address?

MAC = Media Access Control. It is a 6-byte (48-bit) hardware address.

```
MAC ADDRESS FORMAT:
  aa:bb:cc:dd:ee:ff
  │          │
  first 3 bytes = OUI     last 3 bytes = device-specific
  (manufacturer ID,        (unique within that manufacturer)
   globally assigned)

EXAMPLES:
  00:1A:2B:3C:4D:5E  ← a real NIC MAC address
  ff:ff:ff:ff:ff:ff  ← BROADCAST (send to everyone on this LAN)
  01:xx:xx:xx:xx:xx  ← MULTICAST (send to a group)

KEY PROPERTIES:
  ├── Globally unique (no two NICs in the world have the same MAC — in theory)
  ├── Layer 2 (Ethernet) uses MACs — switches speak MAC
  ├── Layer 3 (IP) uses IP addresses — routers speak IP
  └── MACs do NOT cross routers (they only live within one L2 network)
```

```bash
# See your NIC's MAC address
ip link show

# Output:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
#                ─────────────────
#                This is the MAC address
```

---

## 3. What is an Ethernet Frame?

When your computer wants to send data, it wraps it in an "Ethernet frame" — think of it as an envelope.

```
ETHERNET FRAME STRUCTURE:
┌──────────────────────────────────────────────────────────────────────┐
│ Preamble │ Dest MAC │ Src MAC  │ EtherType│ Payload (data)  │ FCS   │
│ 8 bytes  │ 6 bytes  │ 6 bytes  │ 2 bytes  │ 46-1500 bytes   │ 4 bytes│
└──────────────────────────────────────────────────────────────────────┘

Preamble:  "attention, a frame is starting" — synchronisation bytes
Dest MAC:  who should receive this (like the TO: address on an envelope)
Src MAC:   who sent this (like the FROM: address)
EtherType: what type of payload? 0x0800=IPv4, 0x0806=ARP, 0x86DD=IPv6
Payload:   the actual data (usually an IP packet inside)
FCS:       checksum — receiver verifies no corruption in transit

EXAMPLE FRAME (sending an IP packet):
  Dest MAC:  aa:bb:cc:dd:ee:ff  (the switch/router's MAC)
  Src MAC:   11:22:33:44:55:66  (my NIC's MAC)
  EtherType: 0x0800             (IPv4)
  Payload:   [IP packet from me to google.com]
```

---

## 4. What Does a Switch Do?

A switch connects multiple devices on the same local network. Its job is simple: **forward frames to the right port**.

```
BEFORE SWITCHES — THE HUB (still used in tiny setups):
═══════════════════════════════════════════════════════════════════════

  Hub = a "dumb repeater"
  Every frame received → broadcast to ALL other ports

  Server A sends frame to Server B:
  Hub receives it → sends it to ports 2, 3, 4, 5... (ALL of them)
  Server B, C, D, E ALL receive it → waste, collisions, security problem

  ┌──────────────────────────────────────────────────────────┐
  │                        HUB                               │
  │  p1──┐  p2──┐  p3──┐  p4──┐  p5──┐                      │
  │      │      │      │      │      │                      │
  │   Server A  B     C     D     E                          │
  │   sends →  receives (unwanted) ← same frame to all      │
  └──────────────────────────────────────────────────────────┘


THE SWITCH — SMART FORWARDING:
═══════════════════════════════════════════════════════════════════════

  Switch learns MAC addresses by WATCHING traffic.
  It builds a MAC table (also called CAM table).

  ┌───────────────────────────────────────────────────────────────┐
  │                         SWITCH                                │
  │                                                               │
  │  p1    p2    p3    p4    p5                                   │
  │  │     │     │     │     │                                    │
  │ SrvA  SrvB  SrvC  SrvD  SrvE                                  │
  └───────────────────────────────────────────────────────────────┘

  MAC TABLE (CAM table) — starts EMPTY:
  ┌──────────────────┬───────┬─────────┐
  │  MAC Address     │ Port  │ Age (s) │
  ├──────────────────┼───────┼─────────┤
  │  (empty)         │       │         │
  └──────────────────┴───────┴─────────┘

  Step 1: SrvA (MAC aa:aa:aa) sends frame TO SrvB (MAC bb:bb:bb)
    Switch sees: frame FROM aa:aa:aa arrived on port p1
    Switch learns: "aa:aa:aa lives on p1" → adds to table
    Switch looks up bb:bb:bb → NOT in table yet → FLOODS to all ports (p2,3,4,5)
    SrvB sees its own MAC in Dest → accepts. Others ignore.

  MAC TABLE now:
  ┌──────────────────┬───────┬─────────┐
  │  MAC Address     │ Port  │ Age (s) │
  ├──────────────────┼───────┼─────────┤
  │  aa:aa:aa:aa:aa  │  p1   │   0     │
  └──────────────────┴───────┴─────────┘

  Step 2: SrvB replies to SrvA
    Switch sees: frame FROM bb:bb:bb on port p2 → learns bb:bb:bb lives on p2
    Switch looks up aa:aa:aa → IN TABLE → sends ONLY to port p1

  MAC TABLE now:
  ┌──────────────────┬───────┬─────────┐
  │  MAC Address     │ Port  │ Age (s) │
  ├──────────────────┼───────┼─────────┤
  │  aa:aa:aa:aa:aa  │  p1   │   30    │
  │  bb:bb:bb:bb:bb  │  p2   │   0     │
  └──────────────────┴───────┴─────────┘

  After a few minutes of traffic, table is fully populated.
  All frames go ONLY to the right port. No more flooding. Efficient. ✅
```

---

## 5. Your Physical Setup (Harvester)

```
DATA CENTER RACK

  ┌───────────────────────────────────────────────────────────────────┐
  │               TOP-OF-RACK SWITCH                                   │
  │                                                                   │
  │  MAC TABLE (excerpt — fully populated after boot):                │
  │  ┌──────────────────────┬──────┬───────────────────────────────┐  │
  │  │ MAC                  │ Port │ Owner                         │  │
  │  ├──────────────────────┼──────┼───────────────────────────────┤  │
  │  │ aa:bb:cc:11:22:33    │ p01  │ harvester-node01 eth0 (mgmt)  │  │
  │  │ aa:bb:cc:44:55:66    │ p02  │ harvester-node02 eth0 (mgmt)  │  │
  │  │ aa:bb:cc:77:88:99    │ p03  │ harvester-node01 eth1 (strg)  │  │
  │  │ aa:bb:cc:aa:bb:cc    │ p04  │ harvester-node02 eth1 (strg)  │  │
  │  │ aa:bb:cc:dd:ee:ff    │ p05  │ harvester-node01 eth2 (vm)    │  │
  │  │ aa:bb:cc:00:11:22    │ p06  │ harvester-node02 eth2 (vm)    │  │
  │  └──────────────────────┴──────┴───────────────────────────────┘  │
  └──────────────────────────────┬────────────────────────────────────┘
                   ┌─────────────┼─────────────┐
          ┌────────┴──────┐      │      ┌───────┴───────┐
          │  NODE01       │      │      │  NODE02       │
          │  eth0: mgmt   │      │      │  eth0: mgmt   │
          │  eth1: strg   │      │      │  eth1: strg   │
          │  eth2: vm net │      │      │  eth2: vm net │
          └───────────────┘      │      └───────────────┘
                             uplink to
                           WSO2 network
```

---

## 6. What Happens at Hardware Level When a Packet Travels

```
STORY: harbor-acme-core pod sends a response to a docker push client

  Inside pod:
  ├── OS writes bytes into network buffer
  └── Kernel wraps in Ethernet frame

  Kernel → OVS bridge (software):
  ├── OVS adds GENEVE tunnel header (see file 06)
  └── Sends out the physical NIC as electrical/optical signal

  Physical NIC → copper cable → physical switch:
  ├── Electrical signal travels at ~60% speed of light
  ├── Switch receives signal, decodes bits
  ├── Reads Dest MAC in frame header
  ├── Looks up MAC table → forwards to correct port
  └── Sends as new electrical signal on that port's cable

  Cable → destination NIC:
  ├── NIC receives electrical signal
  ├── Decodes bits, checks FCS checksum
  ├── If FCS valid → interrupt the CPU
  └── CPU reads frame from NIC buffer

  All this happens in <1 millisecond for local LAN.
```

---

## LAB 1 — See Your Network Interfaces

Run these on any Linux machine:

```bash
# Show all network interfaces and their MACs
ip link show

# Show a specific interface in detail
ip link show eth0

# Show NIC speed, duplex, cable status
ethtool eth0
# (install with: sudo apt install ethtool)

# Show MAC address table of a Linux bridge (software switch)
bridge fdb show

# Watch frames arriving on an interface (raw capture)
sudo tcpdump -i eth0 -e -n    # -e shows MAC addresses
# Press Ctrl+C to stop

# See ALL frames including non-IP (ARP, etc.)
sudo tcpdump -i eth0 -e -n ether
```

Expected output of `ip link show`:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
```

---

## LAB 2 — Watch ARP in Action

```bash
# Terminal 1: watch ARP packets on the wire
sudo tcpdump -i eth0 -n arp

# Terminal 2: try to reach a new host (triggers ARP)
ping -c 1 192.168.1.1

# You will see in Terminal 1:
# ARP, Request who-has 192.168.1.1 tell 192.168.1.100
# ARP, Reply 192.168.1.1 is-at aa:bb:cc:11:22:33

# See your ARP cache (IP → MAC mappings your machine knows)
ip neigh show
# or:
arp -n
```

---

## Check Your Understanding

**Q1:** Your switch has 48 ports. Server A on port 1 sends a frame to Server B on port 5. The MAC table is empty. What does the switch do?
> It floods the frame to ALL ports (2–48). Server B receives it. All others ignore it (wrong MAC). Switch learns Server A's MAC is on port 1.

**Q2:** What is the difference between a hub and a switch?
> Hub: sends every frame to every port (broadcasts). Switch: learns MAC→port mapping, sends frames only to the right port.

**Q3:** Can a MAC address travel from London to Tokyo?
> No. MACs only live within one L2 broadcast domain (one LAN segment). Routers strip the Ethernet frame and use a new one on the next hop. The MAC only carries the packet to the next router.

**Q4:** In your Harvester setup, node01 has eth0, eth1, eth2. Why three separate cables?
> Each NIC serves a different purpose (mgmt, storage replication, VM traffic) so they can be on different VLANs with different bandwidth allocations and security rules. Physical separation prevents storage traffic from competing with VM traffic.

---

## Summary

```
LAYER 1 (Physical):  electrical/optical signals, cables, RJ45 connectors
LAYER 2 (Data Link): Ethernet frames, MAC addresses, switches, hubs
                     ↑ This is what this file covered ↑

KEY FACTS:
  ├── NIC = hardware that puts bits on the wire
  ├── MAC address = permanent hardware ID of a NIC
  ├── Ethernet frame = the envelope that wraps data for L2 delivery
  ├── Switch = learns MAC→port, forwards frames to correct port only
  ├── Hub = sends to everyone (obsolete, don't use)
  └── MACs do NOT cross routers (Layer 3 boundary replaces them)

NEXT: File 02 — How VLANs split one physical switch into multiple isolated networks.
```
