# 07 — Kubernetes Networking
## CNI, Pods, Services, kube-proxy, CoreDNS — The Complete K8s Network Stack

---

## The Story

Kubernetes is a distributed system. Pods start and stop constantly, move between nodes,
get new IPs on every restart. Yet applications need to find each other reliably.

Kubernetes solves this with a layered system:
- **CNI** gives each pod a stable IP (within its lifetime)
- **Services** give applications a STABLE IP that never changes even when pods restart
- **kube-proxy** makes that service IP work (via iptables DNAT)
- **CoreDNS** lets you use names instead of IPs ("postgres" instead of "10.43.5.8")
- **NetworkPolicy** controls which pods can talk to which

These four layers work together. Each depends on the ones below it.

---

## 1. The CNI Specification — "Give This Pod a Network"

CNI (Container Network Interface) is a standard. Kubernetes doesn't care WHICH CNI you use
(Calico, Flannel, Kube-OVN...) as long as it follows the spec.

```
WHEN A POD STARTS — THE CNI CONTRACT:
═══════════════════════════════════════════════════════════════════════

  1. kubelet creates the pod (starts containers)
  2. kubelet creates an empty network namespace for the pod
  3. kubelet calls the CNI plugin's ADD command with JSON config:
  
     {
       "cniVersion": "0.4.0",
       "name": "k8s-pod-network",
       "type": "calico",
       "ipam": {
         "type": "calico-ipam"    ← IP address management
       }
     }

  4. CNI plugin (Calico/Kube-OVN/Flannel) must:
     ├── Allocate an IP for this pod from the pod CIDR
     ├── Create veth pair (or OVS port)
     ├── Put one end in the pod namespace
     ├── Configure routing so pod can reach other pods
     └── Return the allocated IP to kubelet in JSON response

  5. kubelet injects /etc/resolv.conf (CoreDNS address) into pod
  6. Pod is ready to receive traffic

  WHEN A POD STOPS — CNI DEL command:
     CNI must release the IP and remove the veth/OVS port


KUBERNETES CNI REQUIREMENTS:
  ✅ All pods can communicate with all other pods without NAT
  ✅ Nodes can communicate with all pods without NAT
  ✅ Pods see their own IP the same way other pods see it
  (These are the Kubernetes network model rules — every CNI must follow them)
```

---

## 2. Kubernetes Services — Stable Virtual IPs

```
THE PROBLEM:
  Pod "provisioner" needs to reach "postgres".
  postgres pod IP: 10.42.5.8 today.
  postgres pod restarts → new IP: 10.42.5.15.
  provisioner can't hard-code pod IPs — they change.

THE SOLUTION — SERVICE:
  Create a Service object for postgres.
  Service gets a ClusterIP: 10.43.x.x — this NEVER CHANGES.
  provisioner connects to 10.43.x.x:5432 — always works.
  When postgres pod restarts with new IP, Service automatically updates
  its backend list. Provisioner code changes nothing.


SERVICE YAML:
  apiVersion: v1
  kind: Service
  metadata:
    name: postgres
    namespace: platform-db
  spec:
    selector:
      app: postgres     ← find pods with this label
    ports:
    - port: 5432         ← service port (what callers use)
      targetPort: 5432   ← pod port (what postgres listens on)

  K8s assigns ClusterIP: 10.43.74.93  (stable, never changes)

  ENDPOINT SLICE (auto-managed by K8s, tracks current pod IPs):
  10.43.74.93:5432 → [10.42.5.8:5432, 10.42.5.9:5432]
                       (current pod IPs — updated when pods start/stop)


SERVICE TYPES:
╔══════════════════╦═══════════════════════════════════════════════════╗
║ ClusterIP        ║ Only accessible inside the cluster.               ║
║ (default)        ║ kube-proxy creates iptables DNAT rules.           ║
║                  ║ Use: pod-to-pod communication.                    ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ NodePort         ║ ClusterIP + opens a port on every node (30000-   ║
║                  ║ 32767). External traffic: NodeIP:NodePort.        ║
║                  ║ Use: dev/test access from outside cluster.       ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ LoadBalancer     ║ NodePort + requests external LB from cloud/       ║
║                  ║ MetalLB. Gets external VIP (e.g. 192.168.10.6).  ║
║                  ║ Use: production external access.                 ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ ExternalName     ║ DNS CNAME to an external hostname.                ║
║                  ║ Use: external services (not relevant here).      ║
╚══════════════════╩═══════════════════════════════════════════════════╝
```

---

## 3. kube-proxy — Making Services Real

```
kube-proxy runs as a DaemonSet (one pod per node).
It watches the K8s API for Service and EndpointSlice changes.
When a Service is created, kube-proxy programs iptables rules on EVERY node.

FULL IPTABLES CHAIN FOR ONE SERVICE:
═══════════════════════════════════════════════════════════════════════

Service: postgres ClusterIP=10.43.74.93:5432
Endpoints: 10.42.5.8:5432 (only one backend for simplicity)

Rule 1 (hook into PREROUTING and OUTPUT):
  -A PREROUTING  -j KUBE-SERVICES
  -A OUTPUT      -j KUBE-SERVICES

Rule 2 (KUBE-SERVICES — match ClusterIP):
  -A KUBE-SERVICES \
    -d 10.43.74.93/32 -p tcp --dport 5432 \
    -m comment --comment "platform-db/postgres cluster IP" \
    -j KUBE-SVC-POSTGRES

Rule 3 (KUBE-SVC-POSTGRES — select backend, load balance):
  With 1 backend:
  -A KUBE-SVC-POSTGRES -j KUBE-SEP-POSTGRES1

  With 2 backends (round-robin):
  -A KUBE-SVC-POSTGRES \
    -m statistic --mode random --probability 0.50 \
    -j KUBE-SEP-POSTGRES1
  -A KUBE-SVC-POSTGRES -j KUBE-SEP-POSTGRES2

Rule 4 (KUBE-SEP — the actual DNAT):
  -A KUBE-SEP-POSTGRES1 \
    -p tcp -j DNAT --to-destination 10.42.5.8:5432

PACKET WALK-THROUGH:
  pod sends: src=10.42.5.20  dst=10.43.74.93:5432
  
  iptables PREROUTING:
    KUBE-SERVICES match: dst=10.43.74.93:5432 ✓
    Jump to KUBE-SVC-POSTGRES
    Jump to KUBE-SEP-POSTGRES1
    DNAT: dst becomes 10.42.5.8:5432

  After DNAT: src=10.42.5.20  dst=10.42.5.8:5432
  Routing decision: 10.42.5.8 is local → deliver via veth
  
  postgres pod receives connection from 10.42.5.20 ✅
  (postgres sees src=10.42.5.20, not the service IP — transparent proxy)
```

```bash
# See kube-proxy's iptables chains:
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers

# Find the chain for a specific service:
sudo iptables -t nat -L -n | grep "10.43.74.93"

# Count packets through a service (see actual traffic):
sudo iptables -t nat -L KUBE-SERVICES -n -v | head -20

# kube-proxy can also use IPVS mode (faster for many services):
# ipvsadm -Ln
# Shows virtual server table — same concept but kernel IPVS instead of iptables
```

---

## 4. NodePort — Exposing Services Externally

```
NodePort opens the SAME port on EVERY node in the cluster.
External traffic can reach the service via ANY node's IP.

YAML:
  spec:
    type: NodePort
    ports:
    - port: 80          ← service ClusterIP port
      targetPort: 8080  ← pod port
      nodePort: 31343   ← port opened on every node (30000-32767)

IPTABLES RULES FOR NODEPORT (added ON TOP of ClusterIP rules):

  -A KUBE-NODEPORTS \
    -p tcp --dport 31343 \
    -j KUBE-SVC-PROVISIONER

  (KUBE-SVC-PROVISIONER then DNAT to pod, same as ClusterIP)

TRAFFIC FLOW:
  External client → 172.22.100.3:31343
  
  Linux kernel: port 31343 → KUBE-NODEPORTS rule
  KUBE-NODEPORTS: jump to KUBE-SVC-PROVISIONER
  KUBE-SVC-PROVISIONER: select backend pod
  KUBE-SEP-XXXX: DNAT to 10.42.5.8:8080
  
  10.42.5.8 receives: src=external-IP dst=10.42.5.8:8080
  pod replies: src=10.42.5.8 dst=external-IP
  
  Wait — external client connected to 172.22.100.3, but reply comes from 10.42.5.8?
  That would break TCP! SNAT needed:
  
  iptables adds SNAT for NodePort traffic so pod replies via the node:
  -A KUBE-POSTROUTING: SNAT source → 172.22.100.3 (the node IP)
  External client sees replies from 172.22.100.3 ✅
```

---

## 5. CoreDNS — Names Instead of IPs

```
CoreDNS is a DNS server that runs inside the cluster.
Every pod is configured (by kubelet) to use CoreDNS for DNS resolution.

COREDNS CONFIG (Corefile):
  .:53 {
      kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
      }
      forward . /etc/resolv.conf {   ← forward external names to host's DNS
          max_concurrent 1000
      }
      cache 30
      loop
      reload
      loadbalance
  }


WHAT GOES INTO /etc/resolv.conf IN EVERY POD:
  nameserver 10.43.0.10          ← CoreDNS ClusterIP
  search registry-controller.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5

  ndots:5 means: if name has fewer than 5 dots, try search domains first


DNS QUERY RESOLUTION PATH:
═══════════════════════════════════════════════════════════════════════

NAME: "postgres"  (simple name, no dots)
  Pod sends to 10.43.0.10:53 (CoreDNS)
  CoreDNS tries:
    1. postgres.registry-controller.svc.cluster.local  → NOT FOUND
    2. postgres.svc.cluster.local                      → NOT FOUND
    3. postgres.cluster.local                          → NOT FOUND
    4. postgres (absolute)                             → NOT FOUND
  RESULT: NXDOMAIN ← SLOW (4 queries!) — avoid simple names

NAME: "platform-postgres-postgresql.platform-db"
  Pod sends to 10.43.0.10:53
  ndots check: 1 dot < 5 → try search domains first
    1. platform-postgres-postgresql.platform-db.registry-controller.svc.cluster.local
       → NOT FOUND
    2. platform-postgres-postgresql.platform-db.svc.cluster.local
       → FOUND! Service "platform-postgres-postgresql" in ns "platform-db"
       → return ClusterIP 10.43.74.93 ✅

NAME: "platform-postgres-postgresql.platform-db.svc.cluster.local"
  5 dots → use as absolute name directly
  CoreDNS: looks up Service → 10.43.74.93 ✅  (FAST — single query)
  
  Best practice: always use full name for speed.


COREDNS SERVICE DISCOVERY DATABASE:
  CoreDNS watches K8s API for Services.
  Internal index:
  ┌─────────────────────────────────────────┬──────────────────┐
  │ FQDN                                    │ ClusterIP        │
  ├─────────────────────────────────────────┼──────────────────┤
  │ postgres.platform-db.svc.cluster.local  │ 10.43.74.93      │
  │ provisioner.registry-ctrl.svc.clust... │ 10.43.225.228    │
  │ kubernetes.default.svc.cluster.local   │ 10.43.0.1        │
  └─────────────────────────────────────────┴──────────────────┘


COREDNS FOR PODS (pod DNS records):
  If you have a pod with IP 10.42.5.8 in namespace foo:
  DNS name: 10-42-5-8.foo.pod.cluster.local → 10.42.5.8
  (IPs in pod names use dashes, not dots)
```

```bash
# See what DNS server pods use (inside a pod):
cat /etc/resolv.conf

# DNS lookup from inside a pod:
nslookup platform-postgres-postgresql.platform-db
# or
dig platform-postgres-postgresql.platform-db.svc.cluster.local

# Debug DNS from inside a pod (troubleshoot slow DNS):
dig +stats platform-postgres-postgresql.platform-db.svc.cluster.local @10.43.0.10

# From a node — check CoreDNS logs:
kubectl logs -n kube-system -l k8s-app=kube-dns -f

# Test DNS resolution with a temporary pod:
kubectl run dns-test --image=busybox --restart=Never --rm -it -- \
  nslookup postgres.platform-db.svc.cluster.local
```

---

## 6. Ingress — L7 HTTP Routing Into the Cluster

```
Services handle L4 (TCP/UDP). Ingress handles L7 (HTTP/HTTPS).
Ingress controller (ingress-nginx) reads Ingress rules and
configures nginx to route based on hostname and path.

INGRESS RESOURCE:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: registry-provisioner
    namespace: registry-controller
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
  spec:
    ingressClassName: nginx
    tls:
    - hosts: [registry-api.lkdc.wso2.com]
      secretName: registry-api-tls
    rules:
    - host: registry-api.lkdc.wso2.com
      http:
        paths:
        - path: /api/v1
          pathType: Prefix
          backend:
            service:
              name: registry-provisioner
              port:
                name: http

WHAT INGRESS-NGINX DOES WITH THIS:
  Translates to nginx.conf:
  
  server {
      listen 443 ssl;
      server_name registry-api.lkdc.wso2.com;
      ssl_certificate /etc/ingress-nginx/ssl/registry-api-tls.pem;
      
      location /api/v1 {
          proxy_pass http://10.43.225.228:80;  ← provisioner Service ClusterIP
      }
  }

TRAFFIC FLOW WITH INGRESS:
  Client → 172.22.100.3:31343  (NodePort for ingress-nginx)
  → kube-proxy DNAT → ingress-nginx pod
  → nginx: read Host header "registry-api.lkdc.wso2.com"
  → nginx: match Ingress rule → proxy to 10.43.225.228:80
  → kube-proxy DNAT: 10.43.225.228:80 → provisioner pod 10.42.5.x:8080
  → provisioner pod handles request
  Response follows reverse path ✅
```

---

## 7. NetworkPolicy — K8s Firewall Rules

```yaml
# Complete example: acme-management isolation

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant
  namespace: acme-management
spec:
  podSelector: {}           # applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress

  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    - podSelector: {}       # from same namespace

  egress:
  - ports:                  # DNS allowed
    - protocol: UDP
      port: 53
  - to:
    - podSelector: {}       # to same namespace pods
  - to:
    - ipBlock:              # internet OK (for Trivy DB)
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8        # but NOT other internal namespaces
        - 172.16.0.0/12
        - 192.168.0.0/16
```

```
HOW CNI ENFORCES NetworkPolicy:
═══════════════════════════════════════════════════════════════════════

CALICO (registry-controller-cluster):
  Translates NetworkPolicy → iptables rules using Felix agent
  On each node, creates chains like:
  
  -A cali-pi-POLICYNAME -s 10.52.0.0/16 -j DROP   (block other Harvester pods)
  -A cali-pi-POLICYNAME -s 172.16.0.0/12 -j DROP  (block private ranges)
  -A cali-pi-POLICYNAME -j ACCEPT                  (allow rest)

  Each pod's veth has these rules applied.
  Packets enter caliXXXXXX → Calico iptables → ACCEPT or DROP

KUBE-OVN (Harvester cluster):
  Translates NetworkPolicy → OVN ACLs in OVN northbound DB
  OVN compiles ACLs → OpenFlow rules in OVS
  
  ovn-nbctl acl-add acme-management from-lport 2000 \
    'inport == "harbor-core" && ip4.dst == 10.52.0.0/16 \
     && ip4.dst != 10.52.5.0/24' drop
  
  OVS enforces at the vswitch level — before packet even leaves the OVS port.
  (See file 08 for OVN/OVS details)
```

---

## LAB 8 — Build a Mini Kubernetes Network by Hand

```bash
#!/bin/bash
# lab8-k8s-network.sh
# Simulates: 2 "pods", 1 "service" (via iptables), 1 "CoreDNS" (via /etc/hosts)

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1 >/dev/null

# Create namespaces (pods)
sudo ip netns add pod-provisioner
sudo ip netns add pod-postgres

# Create bridge (like kubelet/CNI bridge)
sudo ip link add name cni0 type bridge
sudo ip link set cni0 up
sudo ip addr add 10.42.5.1/24 dev cni0

# Connect pods to bridge via veth pairs
sudo ip link add cali-prov type veth peer name eth0-prov
sudo ip link add cali-pg   type veth peer name eth0-pg

sudo ip link set eth0-prov netns pod-provisioner
sudo ip link set eth0-pg   netns pod-postgres

sudo ip link set cali-prov master cni0
sudo ip link set cali-pg   master cni0
sudo ip link set cali-prov up
sudo ip link set cali-pg   up

# Configure pods
sudo ip netns exec pod-provisioner ip link set lo up
sudo ip netns exec pod-provisioner ip link set eth0-prov name eth0
sudo ip netns exec pod-provisioner ip link set eth0 up
sudo ip netns exec pod-provisioner ip addr add 10.42.5.8/24 dev eth0
sudo ip netns exec pod-provisioner ip route add default via 10.42.5.1

sudo ip netns exec pod-postgres ip link set lo up
sudo ip netns exec pod-postgres ip link set eth0-pg name eth0
sudo ip netns exec pod-postgres ip link set eth0 up
sudo ip netns exec pod-postgres ip addr add 10.42.5.9/24 dev eth0
sudo ip netns exec pod-postgres ip route add default via 10.42.5.1

# Create "Service" ClusterIP via iptables DNAT (what kube-proxy does)
SERVICE_IP="10.43.74.93"
sudo ip route add $SERVICE_IP/32 via 10.42.5.1  # make Service IP reachable
sudo iptables -t nat -A OUTPUT -d $SERVICE_IP -p tcp --dport 5432 \
  -j DNAT --to-destination 10.42.5.9:5432
sudo iptables -t nat -A PREROUTING -d $SERVICE_IP -p tcp --dport 5432 \
  -j DNAT --to-destination 10.42.5.9:5432

echo "=== Provisioner pod's view ==="
sudo ip netns exec pod-provisioner ip addr
sudo ip netns exec pod-provisioner ip route

echo ""
echo "=== Direct pod-to-pod connectivity ==="
sudo ip netns exec pod-provisioner ping -c 2 10.42.5.9

echo ""
echo "=== Through service IP (iptables DNAT rewrites to 10.42.5.9) ==="
# Start a simple "postgres" server in pod-postgres
sudo ip netns exec pod-postgres nc -l 5432 &
PG_PID=$!
sleep 0.5
# Connect from provisioner pod to Service IP
echo "hello postgres" | sudo ip netns exec pod-provisioner \
  nc -w 2 $SERVICE_IP 5432
kill $PG_PID 2>/dev/null

echo ""
echo "=== iptables DNAT rules (Service) ==="
sudo iptables -t nat -L PREROUTING -n -v | grep "10.43.74.93"

# Cleanup
sudo iptables -t nat -D OUTPUT -d $SERVICE_IP -p tcp --dport 5432 \
  -j DNAT --to-destination 10.42.5.9:5432 2>/dev/null
sudo iptables -t nat -D PREROUTING -d $SERVICE_IP -p tcp --dport 5432 \
  -j DNAT --to-destination 10.42.5.9:5432 2>/dev/null
sudo ip route del $SERVICE_IP/32 2>/dev/null
sudo ip netns del pod-provisioner
sudo ip netns del pod-postgres
sudo ip link del cni0
```

---

## Check Your Understanding

**Q1:** A Service has ClusterIP 10.43.74.93. You run `ip addr show` on any node and don't see this IP anywhere. How is it still reachable?
> It's a virtual IP that exists only in iptables DNAT rules. kube-proxy programs `DNAT --to-destination <pod-IP>` on every node. The IP is never assigned to any interface — the kernel intercepts packets destined for it via iptables before routing, rewrites the destination, and routes to the real pod.

**Q2:** CoreDNS returns IP 10.43.74.93 for "postgres". Where does it get this IP from?
> CoreDNS watches the Kubernetes API. When the postgres Service is created, K8s assigns it a ClusterIP (10.43.74.93) stored in etcd. CoreDNS reads this from the API and adds it to its in-memory database, returning it for the DNS name `postgres.platform-db.svc.cluster.local`.

**Q3:** What happens if kube-proxy crashes on one node?
> The iptables rules it programmed REMAIN (iptables rules survive process restarts). New Service changes won't be applied to that node until kube-proxy restarts, but existing traffic continues to work. Old rules may become stale if pods change IPs.

---

## Summary

```
KUBERNETES NETWORK LAYERS:
  ├── CNI (Calico/Kube-OVN)  = gives each pod a unique IP + routing
  ├── Service + kube-proxy   = stable virtual IP via iptables DNAT
  ├── CoreDNS                = name resolution for stable discovery
  ├── Ingress (nginx)        = L7 HTTP/HTTPS routing from outside
  └── NetworkPolicy          = CNI-enforced firewall between pods

EVERYTHING TOGETHER:
  curl registry-api.lkdc.wso2.com/api/v1/users
    → DNS → 172.22.100.3 (node IP via /etc/hosts or real DNS)
    → TCP → 172.22.100.3:31343 (NodePort)
    → kube-proxy DNAT → 10.42.5.3:443 (ingress-nginx pod)
    → nginx L7 match → proxy to 10.43.225.228:80 (provisioner Service)
    → kube-proxy DNAT → 10.42.5.8:8080 (provisioner pod)
    → handler code runs ✅

YOUR TWO CLUSTERS:
  registry-controller-cluster:  10.42.x.x pods, 10.43.x.x services
  Harvester cluster:             10.52.x.x pods, 10.53.x.x services
  (These are SEPARATE — no shared CNI, no service discovery between them)

NEXT: File 08 — OVS and OVN — the virtual network OS inside Harvester.
```
