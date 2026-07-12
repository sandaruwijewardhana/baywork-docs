# 02 — VLANs and Trunk Ports
## Splitting One Physical Switch Into Multiple Isolated Networks

---

## The Story

Imagine a huge open-plan office. Everyone can hear everyone else — no privacy, constant noise.
Now the company puts up glass walls dividing it into departments: Engineering, Finance, HR.
People in Engineering can shout to each other freely, but Finance cannot hear Engineering,
and HR cannot hear Finance. The walls are "virtual" (glass, not concrete) but they work.

VLANs are those glass walls inside your physical switch.

One switch. Multiple isolated networks. No extra hardware.

---

## 1. Why VLANs Exist

```
WITHOUT VLANs:
═══════════════════════════════════════════════════════════════════════

  All devices on the same switch → same broadcast domain → everyone
  hears everything.

  Problems:
  ├── Security: server01 can sniff traffic from server02
  ├── Noise: ARP broadcasts from all 48 ports flood to all 48 ports
  ├── Management: can't separate "management traffic" from "storage traffic"
  └── One broadcast storm can take down the whole switch

WITH VLANs:
═══════════════════════════════════════════════════════════════════════

  Same physical switch → multiple virtual switches, each isolated.
  
  VLAN 1  (management): node01-eth0, node02-eth0
  VLAN 10 (storage):    node01-eth1, node02-eth1
  VLAN 700 (vm-net):    node01-eth2, node02-eth2

  A broadcast in VLAN 1 NEVER reaches VLAN 700.
  A device on VLAN 1 CANNOT send to a device on VLAN 700.
  (Unless a router explicitly routes between them — Layer 3.)
```

---

## 2. How VLANs Work — The 802.1Q Tag

The switch needs to know which VLAN a frame belongs to. It does this with a **4-byte tag** inserted into the Ethernet frame.

```
NORMAL ETHERNET FRAME (no VLAN):
┌──────────┬──────────┬──────┬──────────────────────────────┐
│ Dest MAC │ Src  MAC │ Type │ Payload                      │
│ 6 bytes  │ 6 bytes  │ 2B   │ up to 1500 bytes             │
└──────────┴──────────┴──────┴──────────────────────────────┘

802.1Q TAGGED FRAME (with VLAN):
┌──────────┬──────────┬─────────────────────┬──────┬──────────────────┐
│ Dest MAC │ Src  MAC │   802.1Q TAG         │ Type │ Payload          │
│ 6 bytes  │ 6 bytes  │   4 bytes            │ 2B   │ up to 1496 bytes │
└──────────┴──────────┴─────────────────────┴──────┴──────────────────┘
                               │
                               ├── TPID:  0x8100  (2 bytes — "I am tagged")
                               └── TCI:   (2 bytes)
                                    ├── PCP: 3 bits  (priority, QoS)
                                    ├── DEI: 1 bit   (drop eligible)
                                    └── VID: 12 bits (VLAN ID, 0-4094)

VID = 700 means: "this frame belongs to VLAN 700"

The switch reads VID → only forwards to ports configured for VLAN 700.
```

---

## 3. Access Ports vs Trunk Ports

```
ACCESS PORT:
═══════════════════════════════════════════════════════════════════════
  Carries ONE VLAN only.
  Connected to: end devices (servers with single-purpose NICs, desktops)
  
  HOW IT WORKS:
  ├── Incoming from device: frame has NO tag → switch adds VLAN tag internally
  ├── Outgoing to device:   switch STRIPS the tag → device receives plain frame
  └── Device doesn't know it's on a VLAN (transparent to the device)

  Example: node01-eth0 is on VLAN 1 (mgmt).
  node01 sends normal Ethernet frame → switch adds "VLAN 1" tag inside switch
  Switch only sends to other VLAN 1 ports → node01 never sees VLAN 700 traffic


TRUNK PORT:
═══════════════════════════════════════════════════════════════════════
  Carries MULTIPLE VLANs simultaneously.
  Connected to: other switches, or server NICs that handle multiple VLANs
  
  HOW IT WORKS:
  ├── Frames arrive WITH tags (VLAN 699, 700, or 701)
  ├── Switch keeps tags intact, forwards based on tag
  ├── Outgoing frames KEEP their tags
  └── The connected device must understand 802.1Q tags

  Example: node01-eth2 is a trunk port carrying VLANs 699, 700, 701.
  A VM on VLAN 700 sends a frame → frame is tagged 700 → eth2 → switch
  Switch sees tag 700 → forwards to other VLAN 700 ports only
  → another VM's tap device on node02 (also VLAN 700) receives it


VISUAL COMPARISON:
═══════════════════════════════════════════════════════════════════════

  ACCESS PORT (node01-eth0, VLAN 1):
  
  node01 eth0          Switch (internally)           Switch port
  ┌──────────┐         ┌──────────────────────┐      to node02 eth0
  │ NO TAG   │──────►  │ + VLAN 1 tag added   │────►  VLAN 1 port  │
  └──────────┘         └──────────────────────┘      │ tag stripped │
                                                      │ → NO TAG     │
                                                      └──────────────┘
  
  TRUNK PORT (node01-eth2, VLANs 699/700/701):
  
  VM tap0 (VLAN 700)  harvester-br0    node01 eth2    Switch
  ┌──────────────┐    ┌────────────┐   ┌──────────┐   ┌──────────────┐
  │ TAG: VLAN700 │──► │ passes     │──►│ TAGGED   │──►│ reads tag    │
  └──────────────┘    │ through    │   │ frame    │   │ 700 → fwd to │
                      └────────────┘   └──────────┘   │ VLAN 700     │
                                                       │ ports only   │
                                                       └──────────────┘
```

---

## 4. VLAN Membership Table (Switch Config)

This table is **configured by an admin** — not learned automatically like the MAC table.

```
VLAN MEMBERSHIP TABLE (your Harvester switch):
┌──────┬──────────────────────┬───────────┬─────────────────────────────┐
│ Port │ Connected To         │ Mode      │ VLANs                       │
├──────┼──────────────────────┼───────────┼─────────────────────────────┤
│ p01  │ node01 eth0          │ ACCESS    │ VLAN 1   (mgmt)             │
│ p02  │ node02 eth0          │ ACCESS    │ VLAN 1   (mgmt)             │
│ p03  │ node01 eth1          │ ACCESS    │ VLAN 10  (storage)          │
│ p04  │ node02 eth1          │ ACCESS    │ VLAN 10  (storage)          │
│ p05  │ node01 eth2          │ TRUNK     │ VLAN 699, 700, 701          │
│ p06  │ node02 eth2          │ TRUNK     │ VLAN 699, 700, 701          │
│ p48  │ WSO2 WAN router      │ TRUNK     │ ALL VLANs (uplink)          │
└──────┴──────────────────────┴───────────┴─────────────────────────────┘

ISOLATION RULES:
  p01 ↔ p02:  ALLOWED   (both VLAN 1)
  p01 ↔ p05:  BLOCKED   (VLAN 1 vs VLAN 700 — different VLANs)
  p05 ↔ p06:  ALLOWED for VLAN 700 frames  (both trunk, both carry 700)
  p03 ↔ p04:  ALLOWED   (both VLAN 10 storage)
  p03 ↔ p05:  BLOCKED   (VLAN 10 vs VLANs 699/700/701)
```

---

## 5. Sub-interfaces — How One NIC Talks on Multiple VLANs

When a server NIC is connected to a trunk port, you create **sub-interfaces** (also called VLAN interfaces) — one per VLAN.

```
LINUX COMMANDS TO CREATE VLAN SUB-INTERFACES:

# node01 eth2 is a trunk port carrying VLANs 699, 700, 701
# Create sub-interface for VLAN 700:
sudo ip link add link eth2 name eth2.700 type vlan id 700
sudo ip link set eth2.700 up
sudo ip addr add 172.22.100.100/24 dev eth2.700

# Create sub-interface for VLAN 699:
sudo ip link add link eth2 name eth2.699 type vlan id 699
sudo ip link set eth2.699 up
sudo ip addr add 10.99.0.100/24 dev eth2.699

# Show sub-interfaces:
ip link show type vlan

# Output shows something like:
# eth2.700@eth2: <BROADCAST,MULTICAST,UP>
#     link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
#     vlan protocol 802.1Q id 700

# Frames sent on eth2.700 automatically get VLAN 700 tag
# Frames received on eth2 with VLAN 700 tag go to eth2.700
```

In Harvester, this is done by the kernel automatically when you configure network-attachment-definitions for VMs.

---

## 6. How VMs Get Tagged — Linux Bridge + VLAN Filter

Harvester uses a Linux bridge (`harvester-br0`) on top of `eth2`. Each VM's tap device gets a VLAN assignment.

```
HARVESTER BRIDGE VLAN SETUP:

# Create the bridge
sudo ip link add name harvester-br0 type bridge vlan_filtering 1
sudo ip link set harvester-br0 up

# Add eth2 (trunk port) to the bridge
sudo ip link set eth2 master harvester-br0

# Set eth2 as a trunk carrying VLANs 699, 700, 701
sudo bridge vlan add dev eth2 vid 699 trunk
sudo bridge vlan add dev eth2 vid 700 trunk
sudo bridge vlan add dev eth2 vid 701 trunk

# Create a tap device for VM1 (this VM will be on VLAN 700)
sudo ip tuntap add name tap-vm1 mode tap
sudo ip link set tap-vm1 master harvester-br0
sudo ip link set tap-vm1 up

# Assign VLAN 700 to this tap (access mode — VM sends untagged, bridge adds tag)
sudo bridge vlan add dev tap-vm1 vid 700 pvid untagged

# Now: VM1 sends a normal Ethernet frame →
#      tap-vm1 receives it → bridge sees tap-vm1 is PVID 700
#      bridge adds VLAN 700 tag → forwards to eth2 trunk → switch
# Result: VM1's traffic is isolated on VLAN 700 without VM knowing about VLANs


# Show bridge VLAN assignments:
bridge vlan show

# Output:
# port    vlan ids
# eth2     699
#          700
#          701
# tap-vm1  700 PVID Egress Untagged
```

---

## 7. VLANs in Your Harvester Setup — Full Picture

```
NODE01 (harvester-dev-lk-dc-node01)
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  eth0 ──── ACCESS port on switch (VLAN 1/mgmt)                      │
│  │         IP: 192.168.10.15                                         │
│  │         Traffic: Harvester API, K8s API, Rancher agent            │
│  │                                                                   │
│  eth1 ──── ACCESS port on switch (VLAN storage)                     │
│  │         No IP (raw block device traffic)                          │
│  │         Traffic: Longhorn replication between nodes               │
│  │                                                                   │
│  eth2 ──── TRUNK port on switch (VLANs 699, 700, 701)               │
│            │                                                         │
│            harvester-br0 (Linux bridge — software switch)            │
│            ├── eth2 (uplink trunk)                                   │
│            ├── tap-rke2-node1  ─── VLAN 700 ─── VM: 172.22.100.3   │
│            ├── tap-rke2-node2  ─── VLAN 700 ─── VM: 172.22.100.x   │
│            └── tap-other-vm    ─── VLAN 699 ─── some other VM       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

PACKET JOURNEY: provisioner pod (in VM 172.22.100.3) sends to internet

  pod writes bytes
  → Calico routes to VM's eth0 (172.22.100.3)
  → VM eth0 → tap-rke2-node1 (VLAN 700 applied here)
  → harvester-br0 bridge → eth2 (frame now tagged VLAN 700)
  → physical cable → physical switch
  → switch reads VLAN 700, forwards to other VLAN 700 ports
  → uplink port p48 (trunk) → WSO2 router
  → router routes 172.22.100.x to internet
```

---

## LAB 3 — Create Two Isolated VLANs on Your Linux Machine

This simulates two VLANs. Devices on VLAN 10 cannot reach devices on VLAN 20 without a router.

```bash
#!/bin/bash
# lab3-vlans.sh — Run with sudo

# Create a bridge with VLAN filtering
ip link add name testbr type bridge vlan_filtering 1
ip link set testbr up

# Create 4 "devices" using veth pairs (simulated servers)
for i in 1 2 3 4; do
  ip link add dev veth-$i type veth peer name peer-$i
  ip link set veth-$i master testbr
  ip link set veth-$i up
  ip link set peer-$i up
done

# Assign VLAN 10 to veth-1 and veth-2 (access mode)
bridge vlan del dev veth-1 vid 1  # remove default VLAN 1
bridge vlan add dev veth-1 vid 10 pvid untagged
bridge vlan del dev veth-2 vid 1
bridge vlan add dev veth-2 vid 10 pvid untagged

# Assign VLAN 20 to veth-3 and veth-4 (access mode)
bridge vlan del dev veth-3 vid 1
bridge vlan add dev veth-3 vid 20 pvid untagged
bridge vlan del dev veth-4 vid 1
bridge vlan add dev veth-4 vid 20 pvid untagged

# Assign IPs to the peer ends (simulating server NICs)
ip addr add 10.10.0.1/24 dev peer-1
ip addr add 10.10.0.2/24 dev peer-2
ip addr add 10.20.0.1/24 dev peer-3
ip addr add 10.20.0.2/24 dev peer-4

echo "Testing VLAN isolation..."
echo "--- peer-1 to peer-2 (same VLAN 10) should WORK ---"
ping -c 2 -I peer-1 10.10.0.2

echo "--- peer-1 to peer-3 (different VLAN) should FAIL ---"
ping -c 2 -W 1 -I peer-1 10.20.0.1 || echo "BLOCKED as expected ✅"

# Cleanup
echo "Cleaning up..."
for i in 1 2 3 4; do
  ip link del veth-$i 2>/dev/null
done
ip link del testbr
```

---

## Check Your Understanding

**Q1:** Your switch has VLAN 1 on ports 1-10 and VLAN 2 on ports 11-20. Can port 5 (VLAN 1) send an ARP broadcast that reaches port 15 (VLAN 2)?
> No. ARP broadcast sent from port 5 in VLAN 1 is ONLY forwarded to other VLAN 1 ports (1-10). VLAN 2 ports (11-20) are completely isolated.

**Q2:** A VM sends an untagged Ethernet frame out its virtual NIC. How does the physical switch know which VLAN this frame belongs to?
> The Linux bridge attached to the VM's tap device has a PVID (Port VLAN ID) configured. When the untagged frame arrives at the bridge, it adds the PVID tag (e.g. VLAN 700). The tagged frame then goes out the trunk NIC (eth2) to the physical switch, which reads the tag.

**Q3:** Why does node01's eth2 use a TRUNK port instead of ACCESS?
> Because eth2 carries traffic for multiple VLANs (699, 700, 701) simultaneously. KubeVirt VMs on different VLANs all share the same physical NIC but need to be isolated from each other.

---

## Summary

```
KEY FACTS:
  ├── VLAN = virtual isolated L2 network within one physical switch
  ├── 802.1Q tag = 4-byte tag in Ethernet frame identifying the VLAN (VID)
  ├── Access port = one VLAN, tag added/stripped by switch transparently
  ├── Trunk port = multiple VLANs, tags kept on wire, device must understand them
  ├── Linux bridge + vlan_filtering = software switch with VLAN support
  └── PVID = the default VLAN for untagged frames arriving on a port

YOUR SETUP:
  eth0 → ACCESS VLAN 1/mgmt → 192.168.10.x (Harvester nodes)
  eth1 → ACCESS VLAN strg   → Longhorn replication
  eth2 → TRUNK 699/700/701  → VMs on different VLANs
         └── VLAN 700       → registry-controller-cluster VMs (172.22.100.x)

NEXT: File 03 — IP addresses, subnets, routing, ARP — how packets travel beyond the local LAN.
```
