# Networking — Zero to Harvester+Rancher
## Complete Self-Study Guide

Read in order. Each file builds on the previous one.
By the end you will be able to trace every hop a packet makes through your infrastructure.

---

## Reading Order

| File | Topic | What You Learn |
|------|-------|----------------|
| [01-physical-layer.md](01-physical-layer.md) | Cables, NICs, Switches, MACs | How bits travel on a wire. What a switch actually does. |
| [02-vlans.md](02-vlans.md) | VLANs, 802.1Q, Trunk Ports | How one wire carries multiple isolated networks. |
| [03-ip-and-routing.md](03-ip-and-routing.md) | IPs, Subnets, ARP, Routers | How packets find their destination across the world. |
| [04-nat-and-firewalls.md](04-nat-and-firewalls.md) | NAT, iptables, SNAT, DNAT | How addresses are rewritten. How firewalls work. |
| [05-linux-networking.md](05-linux-networking.md) | Linux bridges, veth, namespaces | Build a mini network on your laptop using only commands. |
| [06-overlay-networks.md](06-overlay-networks.md) | VXLAN, GENEVE, tunnels | How pods on different physical machines talk as if on same LAN. |
| [07-kubernetes-networking.md](07-kubernetes-networking.md) | CNI, pods, services, kube-proxy, CoreDNS | The complete K8s network stack. |
| [08-ovs-and-ovn.md](08-ovs-and-ovn.md) | OVS, OpenFlow, OVN logical topology | The virtual network OS inside Harvester. |
| [09-harvester-and-rancher.md](09-harvester-and-rancher.md) | Harvester bridges, KubeVirt VMs, MetalLB, Rancher proxy | Your actual infrastructure explained layer by layer. |
| [10-packet-flow-stories.md](10-packet-flow-stories.md) | Every packet flow, all router tables, complete paths | The grand tour — trace every hop in your system. |

---

## The Big Picture (Before You Start)

```
YOUR INFRASTRUCTURE HAS THESE LAYERS:

  PHYSICAL WORLD
  ──────────────────────────────────────────────────────────
  Two servers → cables → one physical switch → WSO2 WAN router

  HARVESTER OS LAYER  (runs on bare metal of both servers)
  ──────────────────────────────────────────────────────────
  Linux bridges (software switches inside each server)
  Harvester Kubernetes cluster (RKE2 + Kube-OVN CNI)
  KubeVirt (runs VMs as Kubernetes pods)
  MetalLB (assigns VIP 192.168.10.6 to ingress-nginx)
  Harbor pods run HERE directly (acme-management namespace)

  VM LAYER  (KubeVirt VMs on Harvester, VLAN 700)
  ──────────────────────────────────────────────────────────
  registry-controller-cluster (RKE2 inside VMs)
  Calico CNI (10.42.x.x pods, 10.43.x.x services)
  Provisioner pods run HERE

  MANAGEMENT LAYER  (cloud, outside your datacenter)
  ──────────────────────────────────────────────────────────
  Rancher (rancher-lk-dev.wso2.com) — manages all clusters via reverse tunnel

  IP MAP SUMMARY:
  192.168.10.x  → Harvester physical nodes (mgmt VLAN)
  192.168.10.6  → MetalLB VIP (Harvester ingress-nginx)
  172.22.100.x  → KubeVirt VMs (VLAN 700)
  10.52.x.x     → Harvester pods (Kube-OVN)
  10.53.x.x     → Harvester services
  10.42.x.x     → registry-controller-cluster pods (Calico)
  10.43.x.x     → registry-controller-cluster services
```

---

## Prerequisites

No prior networking knowledge needed. You need:
- A Linux terminal (Ubuntu/Debian preferred for exercises)
- `sudo` access for the lab exercises
- Curiosity

---

## How to Use This Guide

- **Read the story first** — every file opens with a story/analogy before the technical detail
- **Run the exercises** — they work on any Linux machine, no extra hardware needed
- **Check your understanding** — each file ends with questions; try answering before reading the answer
- **Come back to 10-packet-flow-stories.md** — re-read it after finishing all other files
