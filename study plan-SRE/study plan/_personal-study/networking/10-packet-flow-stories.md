# 10 — Complete Packet Flow Stories
## Every Hop, Every Table, Every Router in Your Infrastructure

---

## How to Read This File

Each story shows:
- **Bold** = routing decision or table lookup
- `[COMPONENT]` = which piece of infrastructure the packet is at
- `→` = packet moves forward
- ❌ = packet dropped here
- ✅ = packet delivered successfully

Read file 09 first — this file assumes you know the IP map and architecture.

---

## Story 1: Provisioner Pod → Postgres Database (Same Cluster)
**Path: pod in RKE2 cluster → service → pod in same cluster**

```
ACTORS:
  provisioner-pod-1  IP: 10.42.96.62  (in VM 172.22.100.3)
  postgres pod       IP: 10.42.96.80  (in VM 172.22.100.3, same node)
  postgres Service   ClusterIP: 10.43.74.93

════════════════════════════════════════════════════════════════════

[PROVISIONER POD] Go code calls:
  db.Connect("platform-postgres-postgresql.platform-db:5432")

  Step 1: DNS Resolution
  ──────────────────────────────────────────────────────
  Pod's /etc/resolv.conf:
    nameserver 10.43.0.10  (CoreDNS Service)
    search registry-controller.svc.cluster.local ...

  Pod sends UDP packet:
    src=10.42.96.62:49000  dst=10.43.0.10:53
    Query: platform-postgres-postgresql.platform-db.svc.cluster.local

  [POD'S ROUTING TABLE]:
    10.43.0.10 — not in 10.42.0.0/16 → use default: via 169.254.1.1
    ARP: who is 169.254.1.1?
    [CALICO PROXY ARP on cali-XXXX]: I am 169.254.1.1 (replies with caliXXX MAC)
    Packet leaves pod via eth0 → enters cali-XXXX on host

  [VM KERNEL iptables PREROUTING]:
    KUBE-SERVICES: dst=10.43.0.10:53 → DNAT → 10.42.96.20:53 (CoreDNS pod)
    Packet now: dst=10.42.96.20:53

  [VM KERNEL ROUTING]:
    10.42.96.20 is on this node → deliver via cali-coredns veth

  [COREDNS POD]:
    Receives DNS query
    Looks up K8s Service "platform-postgres-postgresql" in namespace "platform-db"
    Finds: ClusterIP 10.43.74.93
    Returns DNS answer: 10.43.74.93
    UDP reply travels back same path

  Step 2: TCP Connection to Service IP
  ──────────────────────────────────────────────────────
  Provisioner connects:
    src=10.42.96.62:50000  dst=10.43.74.93:5432

  [VM KERNEL iptables PREROUTING]:
    KUBE-SERVICES: dst=10.43.74.93:5432 →
    KUBE-SVC-POSTGRES: (one backend) →
    KUBE-SEP-PG1: DNAT → dst=10.42.96.80:5432
    conntrack records: {10.43.74.93:5432 ↔ 10.42.96.80:5432}

  [VM KERNEL ROUTING]:
    dst=10.42.96.80 → on this node → via cali-pg veth

  [POSTGRES POD]:
    Receives: src=10.42.96.62:50000  dst=10.42.96.80:5432  ✅
    (postgres sees provisioner's real IP, not service IP)

  Step 3: Reply back
  ──────────────────────────────────────────────────────
  Postgres replies: src=10.42.96.80  dst=10.42.96.62

  [VM KERNEL conntrack]:
    Matches existing connection → reverse DNAT:
    src=10.42.96.80 → src=10.43.74.93 (restore service IP for symmetry)
    (NOTE: actually conntrack restores for the REPLY path)

  [PROVISIONER POD]: receives SQL response ✅

════════════════════════════════════════════════════════════════════
ROUTERS INVOLVED: Calico proxy ARP, kube-proxy iptables (DNAT ×2)
TABLES HIT: routing table, iptables PREROUTING, conntrack
PHYSICAL HOPS: 0 (all inside same VM kernel)
════════════════════════════════════════════════════════════════════
```

---

## Story 2: Provisioner Pod → Harvester K8s API (Cross-Cluster Helm Deploy)
**Path: pod in RKE2 cluster → physical network → Harvester API server**

```
ACTORS:
  provisioner-pod-1   IP: 10.42.96.62  (in VM 172.22.100.3)
  Harvester K8s API   IP: 192.168.10.15:6443

════════════════════════════════════════════════════════════════════

[PROVISIONER POD] Helm Go SDK builds HTTP request:
  HTTPS POST https://192.168.10.15:6443/api/v1/namespaces

  Step 1: No DNS needed (IP address directly)
  ──────────────────────────────────────────────────────

  Step 2: Routing in pod namespace
  ──────────────────────────────────────────────────────
  [POD ROUTING TABLE]:
    192.168.10.15 — not in 10.42.0.0/16 → default: via 169.254.1.1
    Packet exits pod via eth0 → cali-prov-veth on host

  Step 3: VM kernel routing + Calico SNAT
  ──────────────────────────────────────────────────────
  [VM KERNEL iptables PREROUTING]:
    No DNAT match (192.168.10.15 is external, not a service IP)
    Packet continues to routing

  [VM KERNEL ROUTING TABLE]:
    192.168.10.15 — not in 172.22.100.0/24 → default: via 172.22.100.1 dev eth0
    Next hop: 172.22.100.1

  [VM KERNEL iptables POSTROUTING — Calico SNAT]:
    MASQUERADE: src=10.42.96.62 → src=172.22.100.3 (VM's eth0 IP)
    Calico conntrack: {10.42.96.62:50001 ↔ 172.22.100.3:50001 → 192.168.10.15:6443}

  [VM KERNEL ARP]:
    Need MAC for 172.22.100.1 → ARP cache has it (gateway's MAC)
    Ethernet frame: src=VM's MAC  dst=gateway's MAC

  Step 4: VM → tap device → bridge → physical switch
  ──────────────────────────────────────────────────────
  Packet: src=172.22.100.3  dst=192.168.10.15:6443

  [VM eth0] → packet bytes sent to kernel
  [QEMU process]: reads bytes from tap-vm1 file descriptor
                  injects into VM's vNIC output buffer
  Actually: VM writes to eth0 → tap-vm1 device in host kernel
  [HOST KERNEL tap-vm1]: receives from VM
  [harvester-br0]: tap-vm1 has PVID=700 → frame tagged VLAN 700
  [eth2 trunk]: frame sent with VLAN 700 tag

  Step 5: Physical switch
  ──────────────────────────────────────────────────────
  [SWITCH MAC TABLE]:
    dst MAC = gateway's MAC → which port?
    Gateway is connected to uplink port p48
    Forwards to p48 (trunk port, keeps VLAN 700 tag)

  Step 6: WSO2 Router
  ──────────────────────────────────────────────────────
  [WSO2 ROUTER] receives frame on VLAN 700 interface:
    VLAN 700 interface strips VLAN tag (L3 routing starts)
    IP packet: src=172.22.100.3  dst=192.168.10.15:6443

  [WSO2 ROUTER ROUTING TABLE]:
    192.168.10.15 → matches 192.168.10.0/24 → mgmt VLAN interface

  [WSO2 ROUTER ARP TABLE] (for mgmt VLAN interface):
    192.168.10.15 → aa:bb:cc:11:22:33 (node01's eth0 MAC) — REACHABLE

  New Ethernet frame built:
    src MAC = router's mgmt interface MAC
    dst MAC = node01 eth0 MAC
    IP unchanged: src=172.22.100.3  dst=192.168.10.15

  Step 7: Frame travels to node01
  ──────────────────────────────────────────────────────
  [PHYSICAL SWITCH]: dst MAC = node01's MAC → port p01 (mgmt, access port)
  Strip VLAN 1 tag → plain Ethernet frame to node01 eth0

  Step 8: Harvester node01 receives
  ──────────────────────────────────────────────────────
  [NODE01 eth0] → mgmt-br → kernel
  [NODE01 KERNEL ROUTING]:
    dst=192.168.10.15 → that's me (local delivery)
    Port 6443 → kube-apiserver process

  Step 9: Harvester K8s API Server
  ──────────────────────────────────────────────────────
  [kube-apiserver]:
    TLS: connection from 172.22.100.3 (insecure-skip-tls-verify: true — dev bypass)
    Auth: Bearer token from kubeconfig → ServiceAccount "provisioner" on Harvester
    RBAC: check permission for POST /api/v1/namespaces
    ✅ RBAC allows (harvester-rbac.yaml grants this)
    Create namespace in etcd → schedule pods → return 201 Created

  Step 10: Response path (reverse)
  ──────────────────────────────────────────────────────
  Node01 → mgmt VLAN → switch → router → VLAN 700 → VM eth0 → tap
  [VM KERNEL iptables conntrack]:
    Reverse SNAT: src=172.22.100.3 → src=10.42.96.62 (restore pod IP)
  [PROVISIONER POD]: receives 201 Created ✅

════════════════════════════════════════════════════════════════════
ROUTERS INVOLVED: Calico (SNAT), VM kernel, WSO2 physical router,
                  Harvester node kernel
TABLES HIT: pod routing, iptables POSTROUTING (SNAT), conntrack,
            VM routing, ARP, WSO2 routing table, WSO2 ARP table
PHYSICAL HOPS: 2 (VM → switch → router → node01)
════════════════════════════════════════════════════════════════════
```

---

## Story 3: External Tenant → Harbor (docker push)
**Path: docker client on internet → MetalLB → ingress → Harbor pod**

```
ACTORS:
  Tenant's CI/CD runner  (external IP, anywhere)
  MetalLB VIP            192.168.10.6
  ingress-nginx pod      10.52.x.x  (on Harvester node01)
  harbor-acme-core pod   10.52.5.8  (on Harvester node01)
  harbor-acme-registry   10.52.5.14 (on Harvester node02)

════════════════════════════════════════════════════════════════════

[CI/CD RUNNER] runs: docker push registry.acme.192.168.10.6.nip.io/app:v1

  Step 1: DNS for registry.acme.192.168.10.6.nip.io
  ──────────────────────────────────────────────────────
  CI/CD runner queries public DNS
  [nip.io DNS SERVER]: "192.168.10.6.nip.io → always returns 192.168.10.6"
  Returns: 192.168.10.6

  Step 2: TCP connection to 192.168.10.6:443
  ──────────────────────────────────────────────────────
  TCP SYN: src=<CI IP>  dst=192.168.10.6:443

  [INTERNET]: packet routed to WSO2 WAN IP
  [WSO2 WAN router]: "192.168.10.6 → 192.168.10.0/24 → mgmt VLAN"

  [WSO2 ROUTER ARP TABLE]:
    192.168.10.6 → aa:bb:cc:11:22:33 (node01's eth0 MAC)
    (MetalLB speaker has sent ARP announcement recently)

  Ethernet frame: dst=node01 eth0 MAC
  [SWITCH]: MAC table → port p01 → node01 eth0

  Step 3: Harvester node01 receives
  ──────────────────────────────────────────────────────
  [NODE01 KERNEL]:
    dst=192.168.10.6 → not a local interface IP, but MetalLB bound this IP
    MetalLB creates iptables rule (via kube-proxy) for this VIP:

  [NODE01 iptables PREROUTING]:
    KUBE-SERVICES: dst=192.168.10.6:443 →
    KUBE-SVC-INGRESS: select backend →
    KUBE-SEP-INGRESS1: DNAT → 10.52.x.x:443 (ingress-nginx pod IP)

  Step 4: OVS routing to ingress-nginx pod
  ──────────────────────────────────────────────────────
  [OVN LOGICAL PIPELINE in OVS br-int]:
    Packet to 10.52.x.x:443
    Ingress-nginx pod is on this node (node01)
    OVS port: ingress-nginx-port-xxx
    Pipeline: table 0 → table 8 ACL (ingress allowed) → table 64 deliver
    Deliver to ingress-nginx veth

  Step 5: ingress-nginx processes request
  ──────────────────────────────────────────────────────
  [INGRESS-NGINX POD]:
    TLS termination: reads self-signed cert for *.192.168.10.6.nip.io
    SNI: registry.acme.192.168.10.6.nip.io → matched Ingress rule
    Ingress rule: → harbor-acme-core Service

    HTTP/2 proxy to: 10.53.x.x:80 (harbor-acme-core ClusterIP)
    nginx builds upstream connection to service IP

  Step 6: Service DNAT for harbor-acme-core
  ──────────────────────────────────────────────────────
  [NODE01 iptables]:
    dst=10.53.x.x:80 → DNAT → 10.52.5.8:80 (harbor-acme-core pod)

  [OVS br-int]:
    10.52.5.8 is on this node → deliver via OVS port directly

  [HARBOR-ACME-CORE POD]:
    Docker Registry v2 API: POST /v2/app/blobs/uploads/
    Authenticates docker credentials (robot account)
    Accepts upload request

  Step 7: harbor-core calls harbor-registry for blob storage
  ──────────────────────────────────────────────────────
  [HARBOR-ACME-CORE POD]:
    internal RPC to harbor-acme-registry service (10.53.x.x)
    → DNAT → harbor-acme-registry pod (10.52.5.14 on NODE02)

  [OVN PIPELINE on node01]:
    dst=10.52.5.14 → on node02 (192.168.10.17)
    table 64: output via GENEVE tunnel

  [GENEVE PACKET]:
    outer src=192.168.10.15  outer dst=192.168.10.17  UDP:6081
    GENEVE metadata: datapath=acme-mgmt, egress-port=harbor-registry
    inner: harbor-core → harbor-registry

  [PHYSICAL NETWORK]:
    Same mgmt VLAN as both nodes → switch delivers directly
    node02 receives on eth0

  [NODE02 OVS br-int]:
    Decapsulate GENEVE → read metadata → deliver to harbor-registry veth

  [HARBOR-ACME-REGISTRY POD]:
    Receives image layer
    Writes to /storage (Longhorn PVC)

  Step 8: Longhorn writes to disk
  ──────────────────────────────────────────────────────
  [LONGHORN CSI driver]:
    Write to primary replica on node02 (local disk)
    Replicate to node01 via storage VLAN:
    node02 eth1 → switch strg VLAN → node01 eth1 → disk
    Acknowledge write when both replicas confirmed

  [CI/CD RUNNER]: docker push completes → digest: sha256:... ✅

════════════════════════════════════════════════════════════════════
ROUTERS INVOLVED: nip.io DNS, WSO2 router, node01 kernel (MetalLB DNAT),
                  OVN logical router, ingress-nginx (L7), node01 kube-proxy,
                  OVS (GENEVE to node02), node02 OVS
TABLES HIT: nip.io DNS, WSO2 routing, WSO2 ARP, switch MAC,
            node01 iptables (DNAT ×2), OVN pipeline (table 0,8,64),
            GENEVE encap, node02 OVS pipeline
PHYSICAL HOPS: internet → WSO2 router → node01 → node02 (Longhorn replication)
════════════════════════════════════════════════════════════════════
```

---

## Story 4: Developer kubectl via Rancher
**Path: developer laptop → Rancher → websocket tunnel → cluster API**

```
ACTORS:
  Developer laptop         anywhere with internet
  Rancher                  rancher-lk-dev.wso2.com
  cattle-cluster-agent     inside registry-controller-cluster (pod 10.42.x.x)
  K8s API server           172.22.100.3:6443

════════════════════════════════════════════════════════════════════

[DEVELOPER] runs:
  kubectl --kubeconfig=rancher-rke2.yaml get pods -n acme-management

  kubeconfig server: https://rancher-lk-dev.wso2.com/k8s/clusters/c-m-zkklbjnb

  Step 1: DNS for Rancher
  ──────────────────────────────────────────────────────
  [DEV MACHINE DNS]: rancher-lk-dev.wso2.com → real public IP of Rancher
  (this is a real domain with real DNS — Rancher is on internet)

  Step 2: HTTPS to Rancher
  ──────────────────────────────────────────────────────
  kubectl → TLS handshake → HTTPS POST
  URL: /k8s/clusters/c-m-zkklbjnb/api/v1/pods?namespace=acme-management
  Header: Authorization: Bearer <developer's Rancher token>

  [INTERNET ROUTING]: packets travel to Rancher's data center normally

  Step 3: Rancher processes request
  ──────────────────────────────────────────────────────
  [RANCHER]:
    Auth: verify developer's bearer token → has access to cluster c-m-zkklbjnb
    Proxy: find websocket tunnel for cluster c-m-zkklbjnb
    Tunnel was opened by cattle-cluster-agent inside the cluster ← LONG-LIVED connection

  Step 4: Rancher sends request through websocket tunnel
  ──────────────────────────────────────────────────────
  Rancher wraps kubectl request in websocket frame
  Sends through existing websocket connection to cattle-cluster-agent

  [WEBSOCKET TUNNEL]:
    Rancher's server → internet → ISP → WSO2 WAN → VLAN 700 →
    VM 172.22.100.3 → Calico → cattle-cluster-agent pod (10.42.x.x)
    This connection was OPENED BY CATTLE-AGENT (outbound from VM)
    So it works even though Rancher can't reach 172.22.100.3 directly

  Step 5: cattle-cluster-agent forwards to K8s API
  ──────────────────────────────────────────────────────
  [CATTLE-CLUSTER-AGENT POD]:
    Receives proxied kubectl request
    Forwards to local K8s API: https://kubernetes.default.svc.cluster.local
    = 10.43.0.1 → kube-proxy DNAT → 172.22.100.3:6443

  [K8s API SERVER on VM]:
    Auth: impersonates developer (via Rancher's service account)
    RBAC: check permissions for GET pods in acme-management
    Returns pod list (JSON)

  Step 6: Response travels back
  ──────────────────────────────────────────────────────
  K8s API → cattle-agent → websocket → Rancher → developer kubectl ✅

  [DEVELOPER] sees: NAME  READY  STATUS  ...
                    harbor-acme-core-xxx  1/1  Running

════════════════════════════════════════════════════════════════════
IMPORTANT: Developer NEVER has a direct TCP connection to 172.22.100.3
The only inbound connection is from cattle-agent to Rancher (outbound).
════════════════════════════════════════════════════════════════════
```

---

## Story 5: Provisioner Bootstrap → Harbor API (FAILS and Proposed Fix)
**Path: provisioner pod → Harbor API (why it fails + the fix)**

```
════════════════════════════════════════════════════════════════════
TODAY'S BEHAVIOR (FAILS):
════════════════════════════════════════════════════════════════════

[PROVISIONER POD] deploy_worker step 5:
  harbor.NewInsecureClient("https://registry.acme.192.168.10.6.nip.io", adminPass)
  client.Ping() → GET https://registry.acme.192.168.10.6.nip.io/api/v2.0/ping

  Step 1: DNS
  [COREDNS]: external name → forward to WSO2 DNS → nip.io DNS → 192.168.10.6 ✅
  Returns: 192.168.10.6

  Step 2: TCP to 192.168.10.6:443
  Packet: src=10.42.96.62  dst=192.168.10.6:443
  [CALICO]: not local → exit via VM eth0 (SNAT: 172.22.100.3)
  [VM eth0] → tap → bridge → eth2 → switch VLAN 700 → WSO2 router

  Step 3: WSO2 Router checks routing
  [WSO2 ROUTER ROUTING TABLE]:
    192.168.10.6 → matches 192.168.10.0/24 → mgmt VLAN interface

  [WSO2 ROUTER ARP TABLE for mgmt VLAN]:
    192.168.10.6 → ??? 
    ARP cache may have expired (MetalLB only re-announces periodically)
    Router sends ARP request on mgmt VLAN: "who is 192.168.10.6?"
    MetalLB may or may not respond in time

    IF ARP FOUND:
    Router forwards packet → node01 eth0 → MetalLB kube-proxy DNAT → ingress-nginx
    (This MIGHT work occasionally — timing dependent)
    
    IF ARP EXPIRED/MISSING:
    Router: no ARP entry → ICMP Host Unreachable back to 172.22.100.3
    OR: packet just dropped

  [PROVISIONER POD]: connection timeout after 8 minutes ❌
  deploy_worker: FAILED status "harbor_api_ready timed out"

════════════════════════════════════════════════════════════════════
PROPOSED FIX: K8s API Proxy
════════════════════════════════════════════════════════════════════

Instead of connecting to MetalLB VIP, use Harvester K8s API as a proxy.

URL FORMAT:
  https://192.168.10.15:6443/api/v1/namespaces/acme-management/
         services/harbor-acme-core:80/proxy/api/v2.0/ping

[PROVISIONER POD]:
  HTTP GET to 192.168.10.15:6443/api/v1/namespaces/acme-management/...
  (Same path as Helm deploy — proven to work, Story 2)

  [WSO2 ROUTER]: 192.168.10.15 → real IP → ARP always fresh → works ✅

[HARVESTER K8s API SERVER] receives request:
  Auth: same ServiceAccount token as Helm (already works)
  URL path: /api/v1/namespaces/acme-management/services/harbor-acme-core:80/proxy/api/v2.0/ping
  K8s API recognizes /proxy/ path: "proxy this to the service"

  [K8s API SERVER internally]:
    Looks up Service "harbor-acme-core" in "acme-management"
    ClusterIP: 10.53.x.x:80
    Opens internal HTTP connection to 10.53.x.x:80
    → kube-proxy DNAT → 10.52.5.8:80 (harbor-acme-core pod)
    All internal to Harvester cluster, no external network needed

[HARBOR-ACME-CORE POD]:
  Receives GET /api/v2.0/ping
  Returns 200 OK {"ping": "pong"}

Response travels back:
  harbor-core → K8s API → HTTP response body → provisioner pod ✅

No MetalLB, no ARP issues, no cross-VLAN problem.
Same authentication already in use. Clean solution.

════════════════════════════════════════════════════════════════════
```

---

## Story 6: Harbor Pod → Internet (Trivy DB Update)
**Path: pod in Harvester cluster → through NAT → public internet**

```
ACTORS:
  harbor-acme-trivy pod  10.52.5.20 (on Harvester, node01)
  Trivy DB server        ghcr.io (GitHub Container Registry)

════════════════════════════════════════════════════════════════════

[HARBOR-ACME-TRIVY POD] downloads vulnerability DB:
  GET https://ghcr.io/aquasecurity/trivy-db:2

  Step 1: DNS
  ──────────────────────────────────────────────────────
  [HARVESTER COREDNS] (10.53.0.10):
    ghcr.io → external → forward to WSO2 upstream DNS
    Returns: 140.82.x.x (GitHub's CDN IP)

  Step 2: NetworkPolicy check (happens before routing)
  ──────────────────────────────────────────────────────
  [OVN ACL for acme-management egress]:
    Rule: ALLOW 0.0.0.0/0 EXCEPT (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
    140.82.x.x → NOT in RFC1918 → ALLOWED ✅
    OVS pipeline proceeds

  Step 3: OVS pipeline routes to external
  ──────────────────────────────────────────────────────
  [OVN LOGICAL PIPELINE]:
    table 0: trivy pod port (port 11)
    table 8: ACL ALLOW for external traffic
    table 32: logical router — 140.82.x.x is external → send to gateway
    table 64: output via patch port → br-ex → eth0

  [NODE01 KERNEL iptables POSTROUTING]:
    MASQUERADE: src=10.52.5.20 → src=192.168.10.15 (node01's mgmt IP)
    conntrack records: {10.52.5.20:PORT ↔ 192.168.10.15:PORT → 140.82.x.x:443}

  Step 4: Node01 eth0 → physical switch → WSO2 router → internet
  ──────────────────────────────────────────────────────
  Packet: src=192.168.10.15  dst=140.82.x.x:443
  [SWITCH]: dst MAC = WSO2 router's MAC → uplink
  [WSO2 ROUTER]: 140.82.x.x → 0.0.0.0/0 → ISP gateway
  [ISP]: SNAT again → public IP
  [INTERNET]: → GitHub CDN

  Step 5: Response
  ──────────────────────────────────────────────────────
  GitHub → ISP → WSO2 → node01 eth0
  [NODE01 conntrack]: reverse SNAT: src=10.52.5.20 restored
  [OVS br-int]: delivers to trivy pod ✅

  Trivy DB downloaded, vulnerability scan works ✅

════════════════════════════════════════════════════════════════════
ROUTERS INVOLVED: Harvester CoreDNS, OVN logical router, OVS pipeline,
                  node01 kernel (SNAT/MASQUERADE), WSO2 physical router,
                  ISP (another SNAT)
TABLES HIT: DNS, OVN ACL, OVN routing table, iptables POSTROUTING (SNAT),
            conntrack, WSO2 routing, WSO2 ARP
════════════════════════════════════════════════════════════════════
```

---

## Story 7: Cross-Tenant Block (Why Tenants Can't Reach Each Other)
**Path: one tenant's Harbor pod trying to reach another tenant's pod — BLOCKED**

```
ACTORS:
  harbor-acme-core  10.52.5.8  (acme-management namespace)
  harbor-beta-core  10.52.6.8  (beta-management namespace)

════════════════════════════════════════════════════════════════════

Hypothetical attack: acme-core tries to probe beta-core.
  TCP SYN: src=10.52.5.8:40000  dst=10.52.6.8:443

  Step 1: OVN ACL evaluation
  ──────────────────────────────────────────────────────
  [OVS br-int — OVN pipeline]:
    table 0: ingress from harbor-acme-core port (port 7)
    table 8: INGRESS ACL check for acme-management:
      Check egress direction from 10.52.5.8:
      Rule: DENY if dst in 10.52.0.0/16 AND dst NOT in 10.52.5.0/24
      10.52.6.8 is in 10.52.0.0/16 (Harvester pod range)
      10.52.6.8 is NOT in 10.52.5.0/24 (acme's own namespace)
      → ACTION: DROP ❌

  Packet is dropped at the OVS level.
  Never reaches the logical router.
  Never leaves node01.
  harbor-beta-core never sees the SYN.

  [HARBOR-ACME-CORE]: TCP connection timeout.
  attacker gets nothing.

════════════════════════════════════════════════════════════════════
ROUTERS INVOLVED: none (packet dropped before any routing)
TABLES HIT: OVN ACL (table 8)
PHYSICAL HOPS: 0
KEY INSIGHT: Isolation enforced at OVS OpenFlow, not at iptables or
             application level. Even if provisioner had a bug creating
             wrong permissions, OVN ACLs enforce the boundary.
════════════════════════════════════════════════════════════════════
```

---

## Story 8: Pod-to-Pod Across Nodes in Harvester (GENEVE Tunnel)
**Path: harbor-core (node01) → harbor-database (node02) within same tenant**

```
ACTORS:
  harbor-acme-core      10.52.5.8   (on node01: 192.168.10.15)
  harbor-acme-database  10.52.5.12  (on node02: 192.168.10.17)

════════════════════════════════════════════════════════════════════

  Step 1: OVS pipeline on node01
  ──────────────────────────────────────────────────────
  [OVN PIPELINE — node01 OVS br-int]:
    table 0: in_port=7 (harbor-core)
             set metadata = acme-management datapath ID
    table 8: ACL — same namespace → ALLOW
    table 10: L2 lookup: dst-MAC = harbor-database's MAC
              harbor-database is on node02 → set tunnel metadata
    table 48: egress ACL → ALLOW (same namespace)
    table 64: output to GENEVE tunnel port (genev_sys_6081)

  Step 2: GENEVE encapsulation
  ──────────────────────────────────────────────────────
  [OVS GENEVE TUNNEL PORT]:
    Outer IP: src=192.168.10.15  dst=192.168.10.17
    UDP dst: 6081
    GENEVE VNI: cluster-wide ID
    GENEVE OPTIONS:
      datapath = acme-management ID
      ingress_port = harbor-core port number
      egress_port = harbor-database port number
    Inner: original harbor-core → harbor-database frame

  Step 3: Physical delivery (same mgmt VLAN, same switch!)
  ──────────────────────────────────────────────────────
  Outer packet: src=192.168.10.15  dst=192.168.10.17
  Both IPs are on same 192.168.10.0/24 subnet → NO ROUTER NEEDED!

  [NODE01 KERNEL ARP]:
    192.168.10.17 → already in ARP cache (same LAN) → node02's eth0 MAC

  [NODE01 eth0] sends Ethernet frame directly to node02's MAC
  [PHYSICAL SWITCH]: MAC table → port p02 → node02 eth0
  No router hop! Pure L2 on mgmt VLAN.

  Step 4: node02 receives GENEVE
  ──────────────────────────────────────────────────────
  [NODE02 OVS br-int]:
    genev_sys_6081 receives packet
    Decapsulate GENEVE
    Read metadata: datapath=acme-management, egress_port=harbor-database-port

    OVN pipeline on receive side (egress pipeline):
    The GENEVE metadata tells OVS EXACTLY which port to deliver to.
    No MAC lookup, no ACL re-evaluation needed.
    table 64: output to harbor-database-port (port 9)

  [HARBOR-ACME-DATABASE POD]: receives query ✅

════════════════════════════════════════════════════════════════════
ROUTERS INVOLVED: OVN logical router (routing decision in pipeline),
                  but NO physical router (both nodes on same L2 VLAN)
TABLES HIT: OVN pipeline (table 0,8,10,48,64), ARP cache, switch MAC
PHYSICAL HOPS: 1 (node01 → switch → node02, pure L2)
KEY INSIGHT: GENEVE metadata lets receiving node skip table lookups.
             GENEVE on mgmt VLAN (physical) is L2 — no router needed.
════════════════════════════════════════════════════════════════════
```

---

## Master Topology Diagram — All Paths at Once

```
═══════════════════════════════════════════════════════════════════════════════
FULL TOPOLOGY WITH ALL PACKET PATHS ANNOTATED
═══════════════════════════════════════════════════════════════════════════════

[INTERNET/WAN]
      │
      │ Story 3: tenant docker push (inbound)
      │ Story 6: Trivy DB download (outbound, SNAT)
      ▼
[WSO2 WAN ROUTER]
  Routes: 192.168.10.0/24 → mgmt VLAN
          172.22.100.0/24 → VLAN 700
          0.0.0.0/0 → ISP
      │ mgmt VLAN              │ VLAN 700
      ▼                        ▼
[PHYSICAL SWITCH]─────────────────────────────────────────────────
  Port p01: node01-eth0 (mgmt, access)      │
  Port p02: node02-eth0 (mgmt, access)      │
  Port p03: node01-eth1 (strg, access)  Port p05: node01-eth2 (trunk 699/700/701)
  Port p04: node02-eth1 (strg, access)  Port p06: node02-eth2 (trunk 699/700/701)
      │ (mgmt VLAN)                          │ (VLAN 700)
   ┌──┴─────────────────────┐         ┌─────┴────────────────────┐
   │                        │         │                          │
   ▼                        ▼         ▼                          ▼
[NODE01 eth0]          [NODE02 eth0]  [NODE01 eth2]           [NODE02 eth2]
192.168.10.15          192.168.10.17  (trunk)                  (trunk)
   │                                       │
   │ MetalLB VIP: 192.168.10.6            │ harvester-br0
   │ (Story 3: inbound tenant traffic)     │
   │                                   ┌───┴───────────────────────┐
   │                                   │  tap-vm1 [VLAN 700]       │
   │                                   │  tap-vm2 [VLAN 700]       │
   │                                   │       │        │          │
   │                                   │  QEMU-vm1   QEMU-vm2      │
   │                                   └──────┬──────────┬─────────┘
   │                                          │          │
   │                                 [VM1:172.22.100.3] [VM2:...]
   │                                  registry-ctrl-node01
   │                                   │
   │◄───────────────────────────────── Story 2: provisioner → Harvester API
   │                                   │         (172.22.100.3 → 192.168.10.15)
   │                                   │
   │                                   ├── [Calico pods: 10.42.x.x]
   │                                   │     provisioner-pod-1 10.42.96.62
   │                                   │     provisioner-pod-2 10.42.96.63
   │                                   │     ingress-nginx     10.42.x.x
   │                                   │     coredns           10.42.x.x
   │                                   │     postgres          10.42.x.x
   │                                   │
   │                                   ├── [kube-proxy services: 10.43.x.x]
   │                                   │     K8s API    10.43.0.1
   │                                   │     CoreDNS    10.43.0.10
   │                                   │     postgres   10.43.74.93
   │                                   │     provisioner 10.43.225.228
   │                                   │
   │                                   └── Story 1: provisioner → postgres
   │                                         (10.42.96.62 → 10.43.74.93 → 10.42.x.x)
   │
   ├── [OVS br-int — Kube-OVN]
   │     harbor-acme-core     10.52.5.8  (port 7)
   │     harbor-acme-redis    10.52.5.9  (port 8)
   │     harbor-acme-database 10.52.5.12 (port 9) ─── Story 8: GENEVE → node02
   │     harbor-acme-registry 10.52.5.14 (port 10)─── Story 8: GENEVE → node02
   │     harbor-acme-trivy    10.52.5.20 (port 11)─── Story 6: → internet (SNAT)
   │     ingress-nginx         10.52.x.x (port 3) ─── Story 3: inbound traffic
   │     GENEVE tunnel ──────────────────────────────── to 192.168.10.17
   │
   └── Story 3: inbound path
         192.168.10.6:443 → MetalLB kube-proxy DNAT
         → ingress-nginx pod (10.52.x.x)
         → harbor-acme-core (10.52.5.8)
         → GENEVE → node02 → harbor-acme-registry (10.52.5.14)


[RANCHER: rancher-lk-dev.wso2.com]
      ▲
      │ outbound websocket (opened by cattle-agent)
      │ Story 4: developer kubectl flows through here
      │
[cattle-cluster-agent pod]
  inside registry-controller-cluster (10.42.x.x)
```

---

## Quick Reference: All Routing Tables in One Place

```
══════════════════════════════════════════════════════════════════════════
ROUTING/FORWARDING TABLES — ALL COMPONENTS
══════════════════════════════════════════════════════════════════════════

PHYSICAL SWITCH — MAC TABLE:
  node01-eth0 MAC → port p01
  node02-eth0 MAC → port p02
  node01-eth1 MAC → port p03
  node02-eth1 MAC → port p04
  node01-eth2 MAC → port p05
  node02-eth2 MAC → port p06
  (all VMs' MACs learned on p05/p06 as trunk port carries them)

WSO2 PHYSICAL ROUTER — ROUTING TABLE:
  192.168.10.0/24 → mgmt VLAN interface (directly connected)
  172.22.100.0/24 → VLAN 700 interface  (directly connected)
  0.0.0.0/0       → ISP gateway

WSO2 ROUTER — ARP TABLE:
  192.168.10.15 → node01 eth0 MAC (stable, always present)
  192.168.10.17 → node02 eth0 MAC (stable)
  192.168.10.6  → node01 eth0 MAC (MetalLB VIP — may expire)
  172.22.100.3  → VM1's MAC

HARVESTER NODE01 — KERNEL ROUTING:
  192.168.10.0/24 → eth0 (direct)
  10.52.5.0/24    → br-int (local OVS pods)
  10.52.0.0/16    → genev_sys_6081 (remote pods via GENEVE)
  0.0.0.0/0       → 192.168.10.1 (WSO2 router)

HARVESTER — OVN LOGICAL ROUTER:
  10.52.5.0/24 → acme-management switch
  10.52.6.0/24 → beta-management switch
  0.0.0.0/0    → external (via node kernel → WSO2 router)

HARVESTER — OVN ACLs:
  ALLOW: same namespace pods (ACL priority 2000)
  ALLOW: from ingress-nginx namespace (ACL priority 2000)
  ALLOW: egress to public internet (not RFC1918) (ACL priority 1000)
  DENY:  egress to RFC1918 except same namespace (ACL priority 1000)
  DENY:  everything else (ACL priority 1)

HARVESTER — kube-proxy DNAT (for Harvester K8s services):
  10.53.0.10:53    → CoreDNS pod
  10.53.x.x:5432   → postgres pod
  192.168.10.6:443 → ingress-nginx pod  ← MetalLB VIP DNAT

RKE2 VM (172.22.100.3) — KERNEL ROUTING (Calico):
  172.22.100.0/24 → eth0 (direct)
  10.42.96.0/26   → tunl0 (local pod CIDR)
  10.42.97.0/26   → 172.22.100.x tunl0 (remote pods via IPIP BGP route)
  0.0.0.0/0       → 172.22.100.1 (WSO2 router)

RKE2 VM — kube-proxy DNAT (for RKE2 services):
  10.43.0.1:443    → 172.22.100.3:6443 (own K8s API)
  10.43.0.10:53    → CoreDNS pod
  10.43.74.93:5432 → postgres pod
  10.43.225.228:443 → ingress-nginx pod

RKE2 VM — COREDNS ROUTING:
  *.svc.cluster.local → K8s Service registry (etcd)
  *.wso2.com          → WSO2 upstream DNS
  *                   → 8.8.8.8 (internet)

OVS br-int (Harvester) — OpenFlow PORT MAP:
  port 1: genev_sys_6081 (GENEVE tunnel to node02)
  port 3: patch-to-br-ex (external traffic)
  port 7: harbor-acme-core veth
  port 8: harbor-acme-redis veth
  port 9: harbor-acme-database veth
  ...
══════════════════════════════════════════════════════════════════════════
```

---

## You Can Now Explain Everything

After reading all 10 files, you should be able to explain:

```
QUESTIONS YOU CAN NOW ANSWER:

1. "How does a provisioner pod reach postgres?"
   → DNS → CoreDNS → ClusterIP → kube-proxy DNAT → Calico route → pod

2. "How does docker push reach Harbor?"
   → DNS (nip.io) → WSO2 router → MetalLB VIP DNAT → OVS → ingress-nginx
   → kube-proxy → harbor-core → GENEVE → harbor-registry → Longhorn

3. "Why can't provisioner bootstrap Harbor via 192.168.10.6?"
   → MetalLB L2 ARP doesn't reliably cross from VLAN 700 to mgmt VLAN.
   → Router ARP cache may expire. 192.168.10.15 (real IP) always works.

4. "How does Rancher manage your clusters without direct access?"
   → cattle-cluster-agent opens outbound websocket to Rancher.
   → kubectl commands proxied through websocket tunnel.
   → Rancher never initiates inbound connections to your clusters.

5. "Why do Harbor pods use 10.52.x.x but provisioner pods use 10.42.x.x?"
   → Different Kubernetes clusters, different CNIs.
   → Harvester K8s: Kube-OVN pod CIDR 10.52.0.0/16.
   → registry-controller RKE2: Calico pod CIDR 10.42.0.0/16.
   → No shared CNI, no cross-cluster service discovery.

6. "What happens at the physical switch when a GENEVE packet crosses nodes?"
   → GENEVE is encapsulated inside a normal UDP packet with physical node IPs.
   → Switch sees normal IP packet on mgmt VLAN (192.168.10.15→10.17).
   → Same L2 segment — ARP works, direct L2 delivery, no router needed.
   → Destination node OVS decapsulates GENEVE, reads metadata, delivers to pod.
```
