# 05 — Linux Networking Internals
## Network Namespaces, veth Pairs, Bridges — How Containers and VMs Get Networks

---

## The Story

A pod is just a Linux process. But it has its own private network — its own eth0, its own
IP address, its own routing table. How can one physical machine run 50 processes each
with completely separate networks?

The answer is **network namespaces**. Linux can create isolated copies of the entire
network stack. Each namespace has its own interfaces, routes, iptables, ARP tables.
Processes inside a namespace cannot see or affect other namespaces.

A **veth pair** is a virtual ethernet cable connecting two namespaces.
A **bridge** is a virtual switch connecting many namespaces together.

These three primitives (namespace, veth, bridge) are the building blocks for
every container runtime, every CNI plugin, and KubeVirt VMs.

---

## 1. Network Namespaces

```
THE CONCEPT:

  PHYSICAL MACHINE (one kernel):
  ┌─────────────────────────────────────────────────────────────────┐
  │  HOST NETWORK NAMESPACE (default — always exists)               │
  │  eth0: 172.22.100.3                                             │
  │  lo:   127.0.0.1                                                │
  │  routes, iptables, ARP table...                                 │
  │                                                                 │
  │  POD 1 NAMESPACE (created by CNI when pod starts)               │
  │  eth0: 10.42.5.8      ← different IP, different interface       │
  │  lo:   127.0.0.1                                                │
  │  routes, iptables, ARP table — ALL separate from host           │
  │                                                                 │
  │  POD 2 NAMESPACE                                                │
  │  eth0: 10.42.5.9                                                │
  │  lo:   127.0.0.1                                                │
  │  routes, iptables, ARP table — ALL separate from pod 1          │
  └─────────────────────────────────────────────────────────────────┘

  Pod 1 can ONLY see its own eth0 (10.42.5.8).
  It cannot see the host's eth0 (172.22.100.3).
  It cannot see Pod 2's eth0 (10.42.5.9).
  Even if Pod 1 runs `ip link show`, it sees only its own interfaces.
```

```bash
# Create a new network namespace:
sudo ip netns add my-pod

# List network namespaces:
ip netns list
# Output: my-pod

# Run a command INSIDE the namespace:
sudo ip netns exec my-pod ip link show
# Output: only lo (loopback), nothing else — completely isolated

# Compare with host:
ip link show
# Output: eth0, lo, cali* (many), docker0, etc.

# Run a shell inside the namespace:
sudo ip netns exec my-pod bash
# Now you're "inside the pod"
# ip link show → only lo
# ip route show → empty (no routes yet)
# exit

# Delete namespace:
sudo ip netns del my-pod
```

---

## 2. veth Pairs — Virtual Ethernet Cable

A veth pair is two virtual NICs connected to each other. Whatever you send into one end comes out the other. It's like a virtual ethernet cable connecting two network namespaces.

```
VETH PAIR CONCEPT:

  HOST NAMESPACE          POD NAMESPACE
  ┌─────────────┐         ┌──────────────────┐
  │             │         │                  │
  │  caliXXXXXX ◄─────────► eth0             │
  │             │  virtual│  10.42.5.8        │
  │             │  cable  │                  │
  └─────────────┘         └──────────────────┘

  caliXXXXXX: lives in HOST namespace (Calico manages it)
  eth0:        lives in POD namespace (pod sees this as its NIC)

  Packet sent into caliXXXXXX → comes out eth0 inside pod
  Packet sent into eth0 → comes out caliXXXXXX on host
  Think of it as: two tin cans connected by a string


CREATING A VETH PAIR:

  # Create the pair (both ends on host initially):
  ip link add caliXXXXXX type veth peer name eth0-pod1

  # Move one end into a namespace (making it the pod's eth0):
  ip link set eth0-pod1 netns my-pod

  # Now caliXXXXXX is on host, eth0-pod1 is in my-pod namespace
  
  # Rename inside namespace (optional — CNI usually names it eth0):
  ip netns exec my-pod ip link set eth0-pod1 name eth0

  # Bring both up and assign IPs:
  ip link set caliXXXXXX up
  ip netns exec my-pod ip link set eth0 up
  ip netns exec my-pod ip addr add 10.42.5.8/24 dev eth0
```

---

## 3. Linux Bridge — Virtual Switch

A bridge connects multiple interfaces into the same broadcast domain. It works exactly like a physical switch (learns MACs, forwards frames) but in software.

```
LINUX BRIDGE CONCEPT:

                         BRIDGE (br0)
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐│
  │  │cali-pod1 │  │cali-pod2 │  │cali-pod3 │  │eth0 (uplink to   ││
  │  │(veth end)│  │(veth end)│  │(veth end)│  │ physical switch) ││
  │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘│
  └──────────────────────────────────────────────────────────────────┘
         │                │                │
         │  virtual cables │                │
         ▼                ▼                ▼
   [pod1 ns]          [pod2 ns]          [pod3 ns]
   eth0: 10.42.5.8   eth0: 10.42.5.9   eth0: 10.42.5.10


MAC TABLE of bridge (learned same as physical switch):
  cali-pod1 → pod1's MAC
  cali-pod2 → pod2's MAC
  cali-pod3 → pod3's MAC
  eth0      → gateway's MAC (physical switch)
```

```bash
# Create a bridge
sudo ip link add name br0 type bridge
sudo ip link set br0 up

# Add interfaces to the bridge
sudo ip link set cali-pod1 master br0
sudo ip link set cali-pod2 master br0
sudo ip link set eth0 master br0

# Assign IP to the bridge itself (makes it the gateway for pods):
sudo ip addr add 10.42.5.1/24 dev br0

# Show bridge details
bridge link show
bridge fdb show  # MAC table
bridge vlan show # VLAN assignments (if vlan_filtering enabled)

# Show which interfaces are in the bridge:
ip link show master br0
```

---

## 4. Complete Pod Network Setup — What CNI Does

When kubelet starts a pod, the CNI plugin (Calico or Kube-OVN) does all of this:

```bash
#!/bin/bash
# Simulates what Calico CNI does when a pod starts
# (simplified — real Calico uses more complex routing)

POD_NS="pod-harbor-acme-core"
POD_IP="10.42.5.8"
HOST_VETH="cali$(head -c 5 /dev/urandom | xxd -p)"
POD_VETH="eth0"
GATEWAY="169.254.1.1"  # Calico's special link-local gateway

echo "Step 1: Create pod network namespace"
ip netns add $POD_NS

echo "Step 2: Create veth pair"
ip link add $HOST_VETH type veth peer name $POD_VETH

echo "Step 3: Move pod end into namespace"
ip link set $POD_VETH netns $POD_NS

echo "Step 4: Bring up host end, no IP needed (Calico does proxy ARP)"
ip link set $HOST_VETH up
echo 1 > /proc/sys/net/ipv4/conf/$HOST_VETH/proxy_arp
echo 1 > /proc/sys/net/ipv4/conf/$HOST_VETH/forwarding

echo "Step 5: Configure pod inside namespace"
ip netns exec $POD_NS ip link set lo up
ip netns exec $POD_NS ip link set $POD_VETH up
ip netns exec $POD_NS ip addr add $POD_IP/32 dev $POD_VETH  # /32 not /24!

echo "Step 6: Pod needs a route — all traffic via special gateway"
ip netns exec $POD_NS ip route add $GATEWAY dev $POD_VETH
ip netns exec $POD_NS ip route add default via $GATEWAY

echo "Step 7: Host needs route — how to reach this pod"
ip route add $POD_IP/32 dev $HOST_VETH

echo ""
echo "=== Pod's view (ip link show in pod namespace) ==="
ip netns exec $POD_NS ip link show

echo ""
echo "=== Pod's routing table ==="
ip netns exec $POD_NS ip route show

echo ""
echo "=== Host routes now include this pod ==="
ip route show | grep $POD_IP

# THE MAGIC: WHY /32 AND 169.254.1.1?
# Calico uses /32 (single host route) so every pod route is explicit.
# The gateway 169.254.1.1 is link-local — Calico responds to ARP for this
# IP with the host's own MAC (proxy ARP). Pod thinks there's a real device
# at 169.254.1.1, but it's actually just the host veth responding.
# All pod traffic goes to the host veth → Calico processes it → routes correctly.
```

---

## 5. KubeVirt VM Networking — Tap Devices

VMs need network too, but a VM is a full operating system, not a Linux process. KubeVirt uses **tap devices** (TAP = network Tunnel, kernel level) to connect VMs to Linux bridges.

```
TAP DEVICE:
  A tap device appears as a regular network interface to the kernel.
  But instead of a physical NIC, reads/writes go to a process (QEMU).
  QEMU is the VM hypervisor — it creates the VM's "virtual NIC".

  QEMU ←→ tap device ←→ Linux bridge ←→ eth2 ←→ physical switch

VISUAL:
  ┌──────────────────────────────────────────────────────────────────┐
  │  HARVESTER NODE (bare metal Linux)                               │
  │                                                                  │
  │  PHYSICAL NICs:                                                  │
  │  eth0 → mgmt VLAN (192.168.10.15)                               │
  │  eth2 → vm-network trunk ──────────────────────────────────────┐ │
  │                                                                │ │
  │  SOFTWARE BRIDGES:                                             │ │
  │  harvester-br0 (bridge):                                       │ │
  │  ├── eth2 (trunk uplink)        ◄──────────────────────────────┘ │
  │  ├── tap0 (VM1's virtual cable) ──────────────────────┐         │
  │  └── tap1 (VM2's virtual cable) ──────────────────┐   │         │
  │                                                   │   │         │
  │  QEMU PROCESSES:                                  │   │         │
  │  qemu-vm1: reads/writes tap0 ◄────────────────────┘   │         │
  │  qemu-vm2: reads/writes tap1 ◄────────────────────────┘         │
  │                                                                  │
  │  VM1 kernel sees:    VM2 kernel sees:                            │
  │  eth0 (virtual NIC)  eth0 (virtual NIC)                         │
  │  IP: 172.22.100.3    IP: 172.22.100.x                           │
  │  Completely isolated — VMs think they have real NICs             │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Create a tap device (what KubeVirt does):
sudo ip tuntap add name tap-vm1 mode tap
sudo ip link set tap-vm1 master harvester-br0
sudo ip link set tap-vm1 up

# KubeVirt assigns VLAN 700 to this tap (PVID):
sudo bridge vlan add dev tap-vm1 vid 700 pvid untagged

# QEMU uses this tap as the VM's backend NIC:
# qemu-system-x86_64 ... \
#   -netdev tap,id=net0,ifname=tap-vm1,script=no \
#   -device virtio-net-pci,netdev=net0

# Show tap devices:
ip link show type tuntap

# Show bridge members:
bridge link show
```

---

## 6. Full Picture — Pod, Bridge, VM, Physical NIC

```
COMPLETE PICTURE ON ONE HARVESTER NODE:
═══════════════════════════════════════════════════════════════════════

  PHYSICAL LAYER:
  eth0 ──────────────── physical switch (mgmt VLAN, access port)
  eth1 ──────────────── physical switch (strg VLAN, access port)
  eth2 ──────────────── physical switch (vm trunk, VLAN 699/700/701)


  OS + BRIDGES:
  mgmt-br:
  └── eth0 (IP: 192.168.10.15) ← Harvester node IP
      Harvester K8s API server listens here (:6443)
      Harbor pods get traffic here (via MetalLB on this IP)

  strg-br:
  └── eth1 ← Longhorn uses this for replica traffic

  harvester-br0 (vm bridge, VLAN filtering):
  ├── eth2 (trunk uplink, VLANs 699/700/701)
  ├── tap-rke2-node1 (VLAN 700) → QEMU → VM: 172.22.100.3
  └── tap-rke2-node2 (VLAN 700) → QEMU → VM: 172.22.100.x


  INSIDE VM (172.22.100.3) — registry-controller-cluster node:
  eth0: 172.22.100.3  (VM's NIC, attached to tap-rke2-node1 via QEMU)
  │
  ├── RKE2 K8s running inside VM
  │
  ├── Calico CNI creates:
  │   tunl0 (IPIP tunnel) ← pod-to-pod across nodes
  │   cali-XXXXXXXX (veth for each pod)
  │
  ├── Pod namespaces:
  │   provisioner-pod1 ns: eth0=10.42.5.8, routes via 169.254.1.1
  │   provisioner-pod2 ns: eth0=10.42.5.9, routes via 169.254.1.1
  │   ingress-nginx ns:    eth0=10.42.5.3, routes via 169.254.1.1
  │
  └── kube-proxy: iptables DNAT rules for 10.43.x.x services


  HARVESTER K8s POD (directly on Harvester, NOT in VM):
  Kube-OVN CNI creates:
  OVS bridge (br-int):
  ├── GENEVE tunnel port (to node02)
  ├── harbor-acme-core port (OVS internal, 10.52.x.x)
  ├── harbor-acme-redis port
  └── harbor-acme-database port

  Each harbor pod gets a port in the OVS bridge.
  OVN programs OpenFlow rules for forwarding and ACLs.
  (See file 08 for OVS/OVN details)
```

---

## LAB 6 — Full Pod Network Setup

```bash
#!/bin/bash
# lab6-pod-network.sh
# Simulates a complete two-pod network:
# pod1 and pod2 connect via a bridge, pod1 can ping pod2

# --- Setup ---
sudo ip netns add pod1
sudo ip netns add pod2

# Create veth pairs (simulate what CNI does)
sudo ip link add veth-pod1-host type veth peer name veth-pod1-pod
sudo ip link add veth-pod2-host type veth peer name veth-pod2-pod

# Create bridge (simulate Linux bridge / OVS br-int)
sudo ip link add name cni0 type bridge
sudo ip link set cni0 up
sudo ip addr add 10.42.5.1/24 dev cni0

# Attach host ends to bridge
sudo ip link set veth-pod1-host master cni0
sudo ip link set veth-pod2-host master cni0
sudo ip link set veth-pod1-host up
sudo ip link set veth-pod2-host up

# Move pod ends into namespaces
sudo ip link set veth-pod1-pod netns pod1
sudo ip link set veth-pod2-pod netns pod2

# Configure pod1
sudo ip netns exec pod1 ip link set lo up
sudo ip netns exec pod1 ip link set veth-pod1-pod name eth0
sudo ip netns exec pod1 ip link set eth0 up
sudo ip netns exec pod1 ip addr add 10.42.5.8/24 dev eth0
sudo ip netns exec pod1 ip route add default via 10.42.5.1

# Configure pod2
sudo ip netns exec pod2 ip link set lo up
sudo ip netns exec pod2 ip link set veth-pod2-pod name eth0
sudo ip netns exec pod2 ip link set eth0 up
sudo ip netns exec pod2 ip addr add 10.42.5.9/24 dev eth0
sudo ip netns exec pod2 ip route add default via 10.42.5.1

echo "=== Bridge MAC table (empty until traffic flows) ==="
bridge fdb show dev cni0

echo ""
echo "=== Ping pod1 → pod2 ==="
sudo ip netns exec pod1 ping -c 3 10.42.5.9

echo ""
echo "=== Bridge MAC table (populated after ping) ==="
bridge fdb show dev cni0

echo ""
echo "=== ARP cache in pod1 (knows pod2's MAC) ==="
sudo ip netns exec pod1 ip neigh show

echo ""
echo "=== pod1's view — only sees its own interface ==="
sudo ip netns exec pod1 ip link show
sudo ip netns exec pod1 ip addr show
sudo ip netns exec pod1 ip route show

# Cleanup
sudo ip netns del pod1
sudo ip netns del pod2
sudo ip link del cni0
```

---

## Check Your Understanding

**Q1:** A pod runs `ip link show` and sees only `eth0` and `lo`. But the host has 20 interfaces. Why?
> The pod is in its own network namespace which is completely isolated from the host's network namespace. Each namespace has its own independent view of network interfaces.

**Q2:** veth pairs have two ends. If you delete one end, what happens to the other?
> When one end of a veth pair is deleted, the other end is automatically deleted too. If a pod is killed (its namespace deleted), the host-side veth (caliXXXXXX) is also automatically removed.

**Q3:** KubeVirt creates a tap device for each VM. What is the difference between a tap device and a veth pair?
> veth pair: both ends are in-kernel, used to connect two network namespaces. tap device: one end is in-kernel (visible as a network interface), the other is a file descriptor that a userspace program (QEMU) reads/writes. tap is used to connect the kernel network stack to a VM hypervisor.

---

## Summary

```
KEY PRIMITIVES OF LINUX NETWORKING:
  ├── Network namespace  = isolated copy of entire network stack
  │                        each pod gets one, each VM gets one (via QEMU)
  ├── veth pair          = virtual ethernet cable between two namespaces
  │                        CNI uses these to connect pod ns to host ns
  ├── tap device         = kernel interface connected to userspace process
  │                        KubeVirt/QEMU uses these for VM NICs
  ├── Linux bridge       = software switch (forwards by MAC, supports VLAN)
  │                        harvester-br0 connects VMs to physical NIC
  └── ip/bridge commands = everything you need to inspect and build networks

YOUR SETUP:
  Harbor pods: [pod ns] ←veth→ [OVS bridge br-int] ←GENEVE tunnel→ [other nodes]
  RKE2 pods:   [pod ns] ←veth→ [Calico routes on node] ←IPIP tunnel→ [other VMs]
  VMs:         [QEMU] ←tap→ [harvester-br0] ←eth2→ [physical switch VLAN 700]

NEXT: File 06 — Overlay networks (VXLAN, GENEVE) — how pods on different
      physical machines communicate as if on the same LAN.
```
