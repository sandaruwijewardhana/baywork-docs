# 09 — Harvester + Rancher Networking
## Your Actual Infrastructure — All Layers Combined

---

## The Story

You now know every building block. This file puts them all together
in your specific setup: two physical servers, Harvester HCI, KubeVirt VMs,
registry-controller-cluster inside those VMs, and Rancher managing everything.

Read this file with all 8 previous files in mind.

---

## 1. The Big Picture — Two Kubernetes Clusters on Two Physical Servers

```
YOUR INFRASTRUCTURE — FULL VIEW:
═══════════════════════════════════════════════════════════════════════════════════

  [Internet / WSO2 WAN]
         │
         │  BGP/OSPF routes, ISP connection
         ▼
  [WSO2 CORE ROUTER]
  ├── Routing table entry: 192.168.10.0/24 → VLAN 1 (mgmt) interface
  ├── Routing table entry: 172.22.100.0/24 → VLAN 700 interface
  └── Default route: → ISP
         │                    │
    mgmt VLAN           VLAN 700 trunk
    192.168.10.x        172.22.100.x
         │                    │
  ┌──────┴────────────────────┴───────────────────────────────────┐
  │              TOP-OF-RACK SWITCH                                │
  │  p01: node01-eth0 (access, VLAN 1/mgmt)                       │
  │  p02: node02-eth0 (access, VLAN 1/mgmt)                       │
  │  p03: node01-eth1 (access, strg VLAN)                         │
  │  p04: node02-eth1 (access, strg VLAN)                         │
  │  p05: node01-eth2 (trunk, VLAN 699/700/701)                   │
  │  p06: node02-eth2 (trunk, VLAN 699/700/701)                   │
  └──────────────┬────────────────────┬──────────────────────────┘
                 │                    │
    ┌────────────┴──────┐  ┌──────────┴──────────────┐
    │  PHYSICAL NODE01  │  │  PHYSICAL NODE02          │
    │  192.168.10.15    │  │  192.168.10.17            │
    └────────────────────┘  └────────────────────────────┘


  HARVESTER CLUSTER (K8s, runs on bare metal of both nodes):
  API Server: https://192.168.10.15:6443
  CNI: Kube-OVN (OVS + OVN)
  Pod CIDR:     10.52.0.0/16
  Service CIDR: 10.53.0.0/16
  MetalLB VIP:  192.168.10.6 (ingress-nginx LoadBalancer)

  Namespaces running Harbor:
  acme-management:  harbor-acme-* pods (10.52.x.x)
  beta-management:  harbor-beta-* pods (10.52.x.x)
  ...

  KubeVirt VMs (run ON Harvester, appear in VLAN 700):
  VM: registry-ctrl-node01  →  172.22.100.3  (VLAN 700)
  VM: registry-ctrl-node02  →  172.22.100.x  (VLAN 700)


  REGISTRY-CONTROLLER-CLUSTER (K8s, runs inside KubeVirt VMs):
  API Server: https://172.22.100.3:6443  (or via Rancher proxy)
  CNI: Calico
  Pod CIDR:     10.42.0.0/16
  Service CIDR: 10.43.0.0/16

  Namespaces:
  registry-controller:  provisioner pods (10.42.x.x)
  platform-db:          postgres pod
  ingress-nginx:        ingress-nginx pod
  cert-manager:         cert-manager pods
  cattle-system:        Rancher agent pod


  RANCHER (management plane, external to your datacenter):
  URL: rancher-lk-dev.wso2.com
  Manages: Harvester cluster + registry-controller-cluster
  Access method: reverse websocket tunnel (cattle-cluster-agent)
```

---

## 2. Physical Node Internals — Both Clusters Live Here

```
PHYSICAL NODE01 (192.168.10.15) — DETAILED INTERNAL LAYOUT:
═══════════════════════════════════════════════════════════════════════

  HARDWARE:
  ├── NIC eth0: connected to switch port p01 (ACCESS, mgmt VLAN)
  ├── NIC eth1: connected to switch port p03 (ACCESS, storage VLAN)
  └── NIC eth2: connected to switch port p05 (TRUNK, VLAN 699/700/701)

  HARVESTER OS (Linux kernel):
  │
  ├── MANAGEMENT BRIDGE (mgmt-br):
  │   ├── member: eth0
  │   └── IP: 192.168.10.15  ← Harvester node's management IP
  │       This IP is where:
  │       • Harvester K8s API server listens (port 6443)
  │       • Rancher cattle-agent sends heartbeats
  │       • MetalLB sends gratuitous ARP (for 192.168.10.6)
  │       • Longhorn health checks
  │
  ├── STORAGE BRIDGE (strg-br):
  │   ├── member: eth1
  │   └── No IP (raw L2 — Longhorn opens its own connections)
  │
  ├── VM BRIDGE (harvester-br0):
  │   ├── member: eth2 (trunk — VLANs 699/700/701 pass through)
  │   ├── member: tap-vm1  [VLAN 700] ←→ QEMU ←→ VM: 172.22.100.3
  │   └── member: tap-vm2  [VLAN 700] ←→ QEMU ←→ VM: 172.22.100.x
  │
  ├── OVS BRIDGE (br-int) — MANAGED BY KUBE-OVN:
  │   ├── GENEVE tunnel port: → 192.168.10.17 (other node)
  │   ├── patch port: → br-ex
  │   ├── Pod port: harbor-acme-core (10.52.5.8, OVS veth)
  │   ├── Pod port: harbor-acme-redis (10.52.5.9)
  │   ├── Pod port: harbor-acme-database (10.52.5.12)
  │   └── ... more Harbor pods ...
  │
  ├── OVS BRIDGE (br-ex) — EXTERNAL TRAFFIC:
  │   ├── patch port: ← br-int
  │   └── member: eth0 (or mgmt-br — Kube-OVN routes external traffic here)
  │
  └── QEMU PROCESSES (VMs):
      qemu-vm1: reads/writes tap-vm1 → provides "hardware" to VM1
      qemu-vm2: reads/writes tap-vm2 → provides "hardware" to VM2
```

---

## 3. Harvester Cluster — Network Tables

```
HARVESTER K8s API — ovn-nbctl show (logical topology):

  switch acme-management (10.52.5.0/24):
  ├── harbor-acme-core       10.52.5.8  node01
  ├── harbor-acme-redis      10.52.5.9  node01
  ├── harbor-acme-database   10.52.5.12 node02
  ├── harbor-acme-jobservice 10.52.5.13 node01
  ├── harbor-acme-registry   10.52.5.14 node01
  └── acme-to-cluster-router  (uplink to router)

  switch beta-management (10.52.6.0/24):
  ├── harbor-beta-core       10.52.6.8  node02
  └── beta-to-cluster-router (uplink to router)

  router cluster-router:
  ├── route: 10.52.5.0/24 → acme-management switch
  ├── route: 10.52.6.0/24 → beta-management switch
  └── ACL: block inter-tenant cross-subnet traffic

  loadbalancer: postgres-acme
  ├── VIP: 10.53.x.x:5432
  └── backends: 10.52.5.12:5432

HARVESTER METALLB — ADDRESS POOL:
  Pool: 192.168.10.6-192.168.10.10
  Assignment: ingress-nginx-controller → 192.168.10.6
  Mode: L2 (ARP)
  Owner: node01 (until failover)
  ARP announcement: "192.168.10.6 is at node01's MAC" every 30 seconds

HARVESTER KERNEL ROUTING TABLE (on node01):
  10.52.5.0/24     dev br-int (directly connected — local pods)
  10.52.0.0/16     dev genev_sys_6081 (other node pods via GENEVE)
  192.168.10.0/24  dev eth0 (mgmt VLAN)
  0.0.0.0/0        via 192.168.10.1 dev eth0 (default gateway)
```

---

## 4. VM Layer — Registry-Controller-Cluster

```
INSIDE VM (172.22.100.3) — registry-ctrl-node01:
═══════════════════════════════════════════════════════════════════════

  WHAT THE VM SEES (from its perspective):
  ├── eth0: 172.22.100.3/24  (the tap device on the host, seen as NIC)
  │   Gateway: 172.22.100.1 (WSO2 router — via VLAN 700)
  │
  ├── lo: 127.0.0.1
  │
  ├── tunl0 (Calico IPIP tunnel):
  │   Used for pod-to-pod traffic between VMs
  │   Local: 172.22.100.3, remote: 172.22.100.x
  │
  └── cali-XXXXXXXX (many):
      One veth end per pod — pod on other side has its eth0 here

  CALICO ROUTING TABLE (inside the VM):
  ┌──────────────────────┬─────────────────────┬───────────┐
  │ Destination          │ Via/Device          │ Source    │
  ├──────────────────────┼─────────────────────┼───────────┤
  │ 172.22.100.0/24      │ direct, eth0        │ kernel    │
  │ 0.0.0.0/0            │ 172.22.100.1, eth0  │ DHCP      │
  │ 10.42.5.8/32         │ cali-aaa, veth      │ calico    │
  │ 10.42.5.9/32         │ cali-bbb, veth      │ calico    │
  │ 10.42.96.0/26        │ direct, tunl0       │ calico    │← local pods
  │ 10.42.97.0/26        │ 172.22.100.x, tunl0 │ calico BGP│← remote pods
  └──────────────────────┴─────────────────────┴───────────┘

  ARP TABLE (inside VM):
  ┌─────────────────┬───────────────────┬───────────┐
  │ IP              │ MAC               │ State     │
  ├─────────────────┼───────────────────┼───────────┤
  │ 172.22.100.1    │ router's MAC      │ REACHABLE │← WSO2 router
  │ 172.22.100.x    │ node2 VM's MAC    │ REACHABLE │← other RKE2 VM
  │ 169.254.1.1     │ cali-aaa's MAC    │ REACHABLE │← Calico proxy ARP
  └─────────────────┴───────────────────┴───────────┘

  KUBE-PROXY iptables DNAT TABLE (inside VM):
  ┌──────────────────────────────┬───────────────────────────────┐
  │ Service ClusterIP:Port       │ DNAT to Pod:Port              │
  ├──────────────────────────────┼───────────────────────────────┤
  │ 10.43.0.1:443                │ 172.22.100.3:6443 (K8s API)   │
  │ 10.43.0.10:53                │ 10.42.x.x:53 (CoreDNS)        │
  │ 10.43.225.228:443            │ 10.42.5.3:443 (ingress)       │
  │ 10.43.74.93:5432             │ 10.42.5.9:5432 (postgres)     │
  │ 10.43.x.x:80                 │ 10.42.5.8:8080 (provisioner)  │
  └──────────────────────────────┴───────────────────────────────┘

  COREDNS (inside VM):
  Runs as pod 10.42.x.x, Service at 10.43.0.10:53
  Resolves: *.svc.cluster.local → K8s services
  Forwards: *.wso2.com → WSO2 DNS server (via 172.22.100.1)
  Forwards: everything else → 8.8.8.8
```

---

## 5. Rancher — The Management Layer

```
RANCHER ARCHITECTURE:
═══════════════════════════════════════════════════════════════════════

  [rancher-lk-dev.wso2.com] — runs somewhere in WSO2 infra
  ├── Has its own K8s cluster (management cluster)
  ├── API server: HTTPS on rancher-lk-dev.wso2.com:443
  ├── Manages "downstream" clusters via cattle-cluster-agent
  └── No direct access to your datacenter — clusters phone home


  CLUSTER REGISTRATION (how a cluster appears in Rancher):
  
  1. Admin imports cluster in Rancher UI
  2. Rancher generates a YAML with cattle-cluster-agent deployment
  3. Admin applies YAML to the cluster
  4. cattle-cluster-agent pod starts:
     ├── Pulls image from Rancher
     ├── Opens OUTBOUND websocket to rancher-lk-dev.wso2.com:443
     ├── Authenticates with cluster token
     └── Registers: "I am cluster c-m-zkklbjnb, node count 1, version v1.28"

  5. Rancher stores cluster in its database
  6. Rancher can now send commands to cluster via the websocket tunnel


  WEBSOCKET TUNNEL — HOW KUBECTL THROUGH RANCHER WORKS:
  
  Developer runs:
  kubectl --kubeconfig=rancher-rke2.yaml get pods -n acme-management
  
  kubeconfig:
    server: https://rancher-lk-dev.wso2.com/k8s/clusters/c-m-zkklbjnb
  
  1. kubectl → HTTPS to rancher-lk-dev.wso2.com
     URL: /k8s/clusters/c-m-zkklbjnb/api/v1/pods?namespace=acme-management
  
  2. Rancher receives request, looks up cluster c-m-zkklbjnb
     Finds: websocket tunnel connection from cattle-cluster-agent
  
  3. Rancher sends request through tunnel to cattle-cluster-agent
  
  4. cattle-cluster-agent (inside cluster) forwards to local K8s API:
     https://10.43.0.1:443 (K8s API ClusterIP, same as kubernetes.default)
  
  5. K8s API responds with pod list
  
  6. Response travels back:
     K8s API → cattle-cluster-agent → websocket → Rancher → developer kubectl ✅
  
  DEVELOPER NEVER CONNECTS DIRECTLY TO 172.22.100.3:6443
  Everything goes through rancher-lk-dev.wso2.com


  RANCHER KUBECONFIG TYPES:
  
  Type A (via Rancher proxy) — default:
    server: https://rancher-lk-dev.wso2.com/k8s/clusters/c-m-zkklbjnb
    Works from: anywhere with internet (Rancher as proxy)
    Downside: slower (extra hop through Rancher), fails if Rancher is down
  
  Type B (direct kubeconfig) — for automation:
    server: https://172.22.100.3:6443  (for RKE2)
    server: https://192.168.10.15:6443 (for Harvester)
    Works from: only if you have IP routing to these addresses
    Used by: provisioner pod → Harvester API (stored as Secret)


  HARVESTER KUBECONFIG (the one your provisioner uses):
  
  apiVersion: v1
  kind: Config
  clusters:
  - cluster:
      server: https://192.168.10.15:6443
      insecure-skip-tls-verify: true   ← dev bypass (see dev-bypasses.md)
  contexts:
  - context:
      cluster: harvester
      user: harvester-sa
  users:
  - name: harvester-sa
    user:
      token: <ServiceAccount token from harvester-rbac.yaml>
```

---

## 6. MetalLB Layer 2 — Deep Dive

```
METALLB L2 ARP ANNOUNCEMENT:

  MetalLB pods run on each Harvester node.
  MetalLB controller: watches K8s Services, assigns VIPs from pool.
  MetalLB speaker: runs on each node, sends ARP for owned VIPs.

  STARTUP SEQUENCE:
  1. ingress-nginx Service created with type: LoadBalancer
  2. MetalLB controller: "192.168.10.6 is available, assign to ingress-nginx"
  3. MetalLB speaker on NODE01:
     Sends gratuitous ARP on eth0 (mgmt VLAN):
     "ARP Reply: 192.168.10.6 is at [NODE01's eth0 MAC]"
  4. Physical switch and WSO2 router receive ARP:
     Update ARP cache: 192.168.10.6 → node01's MAC
  5. Any device on mgmt VLAN 192.168.10.x can now reach 192.168.10.6 ✅

  FAILOVER (if node01 fails):
  1. MetalLB speaker on node02 detects node01 is down
  2. node02 speaker sends new gratuitous ARP:
     "192.168.10.6 is at [NODE02's eth0 MAC]"
  3. Switch and router update ARP cache
  4. Traffic now flows to node02 ✅
  (Takes ~5-30 seconds for ARP caches to update)

  WHY 192.168.10.6 IS UNREACHABLE FROM 172.22.100.x:
  
  ┌──────────────────────────────────────────────────────────────────┐
  │  ARP broadcast: "192.168.10.6 is at node01's MAC"               │
  │  This ARP is sent on eth0 (mgmt VLAN = 192.168.10.x VLAN)        │
  │                                                                  │
  │  VLAN 700 (172.22.100.x):  ←─── VLAN BARRIER ──── mgmt VLAN     │
  │  NEVER receives this ARP                                         │
  │                                                                  │
  │  VM (172.22.100.3) tries to reach 192.168.10.6:                 │
  │  "Not in my subnet 172.22.100.0/24 → send to gateway 172.22.100.1" │
  │                                                                  │
  │  WSO2 Router receives from VLAN 700:                             │
  │  "Packet to 192.168.10.6 — check routing table"                 │
  │  Route: 192.168.10.0/24 → mgmt VLAN interface                   │
  │  "Need MAC for 192.168.10.6 — check ARP cache for mgmt interface"│
  │  ARP cache: may or may not have 192.168.10.6                    │
  │    If MetalLB recently sent ARP → router has it → packet works  │
  │    If ARP expired → router re-ARPs → MetalLB replies → might work│
  │    But: 192.168.10.6 responds to ARP only when MetalLB actively  │
  │         sends it → timing-dependent → unreliable → times out     │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 7. All Routers in Your Infrastructure

```
EVERY ROUTER IN YOUR SYSTEM:
═══════════════════════════════════════════════════════════════════════

  ROUTER 1: WSO2 Physical Top-of-Rack Router
  ┌─────────────────────────────────────────────────────────────────┐
  │ Routing Table:                                                  │
  │   192.168.10.0/24  → mgmt VLAN interface  (node01, node02)     │
  │   172.22.100.0/24  → VLAN 700 interface   (RKE2 VMs)           │
  │   10.0.0.0/8       → ? (may not exist — RFC1918 dropped)        │
  │   0.0.0.0/0        → ISP gateway                               │
  │                                                                 │
  │ ARP Table (on mgmt VLAN interface):                             │
  │   192.168.10.15    → node01's MAC                               │
  │   192.168.10.17    → node02's MAC                               │
  │   192.168.10.6     → node01's MAC (MetalLB VIP, may be stale)  │
  │                                                                 │
  │ ARP Table (on VLAN 700 interface):                              │
  │   172.22.100.3     → VM1's MAC                                  │
  │   172.22.100.1     → gateway/router's own MAC                  │
  └─────────────────────────────────────────────────────────────────┘

  ROUTER 2: Harvester Linux Kernel (on each physical node)
  ┌─────────────────────────────────────────────────────────────────┐
  │ Routing Table (on node01, 192.168.10.15):                       │
  │   10.52.5.0/24  → br-int (local pods — OVS)                    │
  │   10.52.0.0/16  → genev_sys_6081 (remote pods via GENEVE)      │
  │   192.168.10.0/24 → eth0 (mgmt VLAN)                            │
  │   0.0.0.0/0     → 192.168.10.1 via eth0                        │
  │                                                                 │
  │ iptables (kube-proxy for Harvester K8s):                        │
  │   DNAT: 10.53.x.x:5432 → 10.52.5.12:5432 (postgres service)   │
  │   DNAT: 192.168.10.6:443 → 10.52.x.x:443 (ingress-nginx pod)  │
  └─────────────────────────────────────────────────────────────────┘

  ROUTER 3: OVN Logical Router (virtual, inside Harvester K8s)
  ┌─────────────────────────────────────────────────────────────────┐
  │ Logical Routes (in OVN NB DB):                                  │
  │   10.52.5.0/24  → acme-management logical switch               │
  │   10.52.6.0/24  → beta-management logical switch               │
  │   0.0.0.0/0     → external gateway                             │
  │                                                                 │
  │ ACLs (NetworkPolicy):                                           │
  │   DENY cross-tenant (10.52.5.x → 10.52.6.x and vice versa)    │
  │   ALLOW same namespace                                          │
  │   ALLOW from ingress-nginx                                      │
  └─────────────────────────────────────────────────────────────────┘

  ROUTER 4: RKE2 VM Linux Kernel (inside VM, 172.22.100.3)
  ┌─────────────────────────────────────────────────────────────────┐
  │ Routing Table (Calico managed):                                 │
  │   172.22.100.0/24  → eth0 (VM's physical NIC = tap device)     │
  │   10.42.96.0/26    → tunl0 (local pod subnet on this node)     │
  │   10.42.97.0/26    → 172.22.100.x via tunl0 (remote node pods) │
  │   0.0.0.0/0        → 172.22.100.1 via eth0 (WSO2 router)       │
  │                                                                 │
  │ iptables (kube-proxy for registry-controller-cluster):          │
  │   DNAT: 10.43.0.1:443  → 172.22.100.3:6443 (own K8s API)      │
  │   DNAT: 10.43.0.10:53  → 10.42.x.x:53  (CoreDNS pod)          │
  │   DNAT: 10.43.74.93:5432 → 10.42.x.x:5432 (postgres pod)      │
  │   DNAT: 10.43.225.228:443 → 10.42.x.x:443 (ingress-nginx)     │
  └─────────────────────────────────────────────────────────────────┘

  ROUTER 5: Calico (virtual, inside RKE2 VMs)
  ┌─────────────────────────────────────────────────────────────────┐
  │ BGP routes (from other nodes):                                  │
  │   10.42.97.0/26 via 172.22.100.x dev tunl0                     │
  │                                                                 │
  │ Per-pod routes:                                                 │
  │   10.42.96.62/32 dev cali-aaaa (provisioner pod 1)             │
  │   10.42.96.63/32 dev cali-bbbb (provisioner pod 2)             │
  │   10.42.96.1/32  dev cali-cccc (ingress-nginx pod)             │
  └─────────────────────────────────────────────────────────────────┘

  ROUTER 6: ingress-nginx (L7 proxy — not IP routing but HTTP routing)
  ┌─────────────────────────────────────────────────────────────────┐
  │ nginx upstream config (generated from Ingress objects):         │
  │   registry-api.lkdc.wso2.com/api/v1 → 10.43.225.228:80        │
  │                                                                 │
  │ nginx on Harvester (Harbor ingress):                            │
  │   registry.acme.192.168.10.6.nip.io → harbor-acme-core:80     │
  │   registry.beta.192.168.10.6.nip.io → harbor-beta-core:80     │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Check Your Understanding

**Q1:** A provisioner pod in registry-controller-cluster wants to write to postgres. List all the routing decisions and table lookups that happen.
> 1. Pod DNS query → CoreDNS → returns 10.43.74.93. 2. Pod sends to 10.43.74.93:5432 → kube-proxy iptables DNAT → rewrites to 10.42.x.x:5432 (postgres pod). 3. Calico routes: same node → veth; different node → IPIP tunnel → VM eth0 → tap → harvester-br0 → eth2 → switch VLAN 700 → node2 VM.

**Q2:** Someone on the internet does a docker push to registry.acme.192.168.10.6.nip.io. Trace the path from their machine to the harbor-acme-registry pod.
> Internet → WSO2 WAN → WSO2 router → ARP for 192.168.10.6 on mgmt VLAN → node01 MetalLB replies → node01 eth0 receives frame → kube-proxy DNAT 192.168.10.6:443 → ingress-nginx pod (10.52.x.x) via OVS br-int → nginx L7 match → harbor-acme-core service (10.53.x.x) → kube-proxy DNAT → harbor-acme-core pod → harbor-core calls harbor-registry service → harbor-registry pod writes to PVC.

**Q3:** Why can the provisioner pod reach 192.168.10.15:6443 (Harvester API) but NOT 192.168.10.6:443 (MetalLB VIP)?
> 192.168.10.15 is a real physical NIC. Its ARP is announced regularly, the WSO2 router's ARP cache is always fresh for it, and IP routing 172.22.100.x → 192.168.10.15 works. 192.168.10.6 is a MetalLB VIP announced via ARP only on the mgmt VLAN. The ARP announcement only reaches devices on that L2 segment. Whether the WSO2 router forwards it across VLANs depends on router config and ARP cache freshness — which is unreliable in practice.

---

## Summary

```
YOUR INFRASTRUCTURE HAS 6 ROUTERS:
  1. WSO2 Physical Router       — connects VLANs to internet
  2. Harvester Linux Kernel     — routes Harvester pod traffic + MetalLB DNAT
  3. OVN Logical Router         — virtual L3 for Harvester pods + ACLs
  4. RKE2 VM Linux Kernel       — routes pod traffic + kube-proxy for RKE2
  5. Calico                     — BGP-based routing between RKE2 VMs
  6. ingress-nginx (L7)         — HTTP routing to services

THE TWO CLUSTER BOUNDARY:
  Provisioner cluster (172.22.100.x / 10.42.x.x)
  cannot reach Harbor pods (10.52.x.x) directly.
  The only reliable path between clusters is:
  Provisioner → 192.168.10.15:6443 (Harvester K8s API) → K8s proxy → Harbor pod

RANCHER:
  Everything managed via reverse websocket tunnel.
  You never need direct access to cluster API servers.
  But the provisioner DOES need direct kubeconfig (for Helm deployments).

NEXT: File 10 — All packet flow stories. Every hop, every table, every decision.
      The grand tour of your entire infrastructure.
```
