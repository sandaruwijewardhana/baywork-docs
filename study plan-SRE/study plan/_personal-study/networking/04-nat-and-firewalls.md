# 04 — NAT and Firewalls
## How Addresses Are Rewritten and Traffic Is Filtered

---

## The Story

You live in a block of flats. The building has ONE front door with ONE street address.
Inside, each flat has its own number (10.42.5.1, 10.42.5.2...).
When you send a letter out, the concierge stamps it with the building's address (the public IP).
When a reply arrives at the building's front door, the concierge looks at their notepad
and delivers it to the right flat.

That concierge is NAT (Network Address Translation).

The building also has a security guard (iptables/firewall) who decides:
"Only let in letters addressed to flat 8080. No other visitors allowed in."

---

## 1. Why NAT Exists

```
PROBLEM:
  You have 100 devices inside your network (pods, VMs, servers).
  You have only ONE public IP address from your ISP.
  All 100 devices need to reach the internet simultaneously.

SOLUTION: NAT
  NAT hides all internal IPs behind one public IP.
  It tracks outgoing connections and routes replies back to the right internal device.

  Internal:    10.42.5.8:49321  ──NAT──►  Public: 1.2.3.4:49321 ──► Google 8.8.8.8
  Internal:    10.42.5.9:52000  ──NAT──►  Public: 1.2.3.4:52000 ──► GitHub 140.82.x.x
  Internal:    10.42.5.10:44001 ──NAT──►  Public: 1.2.3.4:44001 ──► AWS 52.x.x.x

  Google sees all three as coming from 1.2.3.4 (the public IP).
  NAT table tracks port numbers to distinguish them.
```

---

## 2. SNAT — Source NAT (outbound traffic)

```
SNAT = change the SOURCE IP address of outgoing packets.

USE CASE: Pod (private IP 10.42.5.8) sends to internet (8.8.8.8).

BEFORE SNAT:          AFTER SNAT (at node eth0):
src: 10.42.5.8  ──►   src: 172.22.100.3
dst: 8.8.8.8          dst: 8.8.8.8

SNAT also records the translation in a "conntrack" table:

CONNTRACK TABLE (connection tracking):
┌──────────────────┬─────────────────┬──────────────────────────┐
│ Original         │ Translated      │ State                    │
├──────────────────┼─────────────────┼──────────────────────────┤
│ 10.42.5.8:49321  │ 172.22.100.3:49321 │ ESTABLISHED          │
│ 10.42.5.9:52000  │ 172.22.100.3:52000 │ TIME_WAIT            │
└──────────────────┴─────────────────┴──────────────────────────┘

When reply arrives (src=8.8.8.8 dst=172.22.100.3:49321):
→ conntrack matches: 172.22.100.3:49321 = originally 10.42.5.8:49321
→ reverse SNAT: change dst back to 10.42.5.8:49321
→ deliver to pod ✅

Kubernetes uses SNAT (called "masquerade") on every node for pod → internet traffic.
```

---

## 3. DNAT — Destination NAT (inbound traffic)

```
DNAT = change the DESTINATION IP of incoming packets.

USE CASE: External client reaches service via NodePort or LoadBalancer.

BEFORE DNAT:              AFTER DNAT (by kube-proxy):
src: 203.0.113.5          src: 203.0.113.5
dst: 172.22.100.3:31343   dst: 10.42.5.8:443  (ingress-nginx pod)

This is how ALL Kubernetes services work internally:
  ClusterIP 10.43.x.x is NOT a real interface anywhere.
  kube-proxy programs iptables DNAT rules:
  "if dst = 10.43.x.x:443 → change dst to 10.42.5.8:443 (ingress pod)"

EXAMPLE DNAT RULE (what kube-proxy programs):
  -t nat -A KUBE-SERVICES \
    -d 10.43.225.228/32 --dport 443 \
    -j KUBE-SVC-XXXXX

  -t nat -A KUBE-SVC-XXXXX \
    -j KUBE-SEP-YYYYY  ← selects a backend pod

  -t nat -A KUBE-SEP-YYYYY \
    -j DNAT --to-destination 10.42.5.8:443
```

---

## 4. iptables — The Linux Firewall and NAT Engine

iptables is the Linux kernel's packet processing framework. It handles both firewalling (drop/allow) and NAT (rewrite addresses). Every packet passes through "chains" which contain "rules".

```
IPTABLES TABLES AND CHAINS:

  TABLE: raw          (very early, before conntrack)
  TABLE: mangle       (modify packet headers)
  TABLE: nat          (address translation — SNAT/DNAT)
  TABLE: filter       (drop/allow packets) ← the "firewall" table

  CHAINS in filter table:
  ├── INPUT    → packets DESTINED for this machine
  ├── OUTPUT   → packets ORIGINATING from this machine
  └── FORWARD  → packets PASSING THROUGH this machine (routing)

  CHAINS in nat table:
  ├── PREROUTING  → DNAT (before routing decision, change dst)
  └── POSTROUTING → SNAT (after routing decision, change src)


PACKET FLOW THROUGH IPTABLES:
═══════════════════════════════════════════════════════════════════════

  Incoming packet arrives on eth0:
                    │
                    ▼
             raw PREROUTING
                    │
                    ▼
             mangle PREROUTING
                    │
                    ▼
             nat PREROUTING  ← DNAT happens here (change dst IP)
                    │
                    ▼
           [routing decision]
           ┌─────┴──────────┐
           │                │
    For this machine    For other machine (forward)
           │                │
           ▼                ▼
    mangle INPUT     mangle FORWARD
           │                │
           ▼                ▼
    filter INPUT     filter FORWARD  ← allow/drop forwarded packets
           │                │
           ▼                ▼
    [delivered      nat POSTROUTING  ← SNAT happens here (change src IP)
     to process]          │
                          ▼
                    [out via eth1]

  Outgoing packet from local process:
    [process] → mangle OUTPUT → nat OUTPUT → filter OUTPUT
    → routing decision → nat POSTROUTING → [out via interface]
```

---

## 5. iptables Rules — Syntax and Examples

```bash
# BASIC SYNTAX:
# iptables -t <table> -A <chain> [match options] -j <action>
#
# -t table:   filter (default), nat, mangle, raw
# -A chain:   append to chain (INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING)
# -j action:  ACCEPT, DROP, REJECT, DNAT, SNAT, MASQUERADE, LOG


# ═══ FIREWALL RULES (filter table) ════════════════════════════════

# Allow established connections (very important — without this, replies are blocked)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from specific IP only
iptables -A INPUT -p tcp --dport 22 -s 172.22.100.3 -j ACCEPT

# Block all other SSH
iptables -A INPUT -p tcp --dport 22 -j DROP

# Allow HTTP and HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Block ICMP (ping) from outside
iptables -A INPUT -p icmp -j DROP

# Allow all traffic from specific subnet
iptables -A INPUT -s 10.42.0.0/16 -j ACCEPT

# Log and drop everything else
iptables -A INPUT -j LOG --log-prefix "DROPPED: "
iptables -A INPUT -j DROP


# ═══ NAT RULES (nat table) ════════════════════════════════════════

# MASQUERADE (SNAT to the outgoing interface's IP — dynamic SNAT):
# "All pods going to internet, change source to this machine's IP"
iptables -t nat -A POSTROUTING -s 10.42.0.0/16 ! -d 10.42.0.0/16 -j MASQUERADE

# SNAT to specific IP (static — good for fixed public IPs):
iptables -t nat -A POSTROUTING -s 10.42.0.0/16 -o eth0 -j SNAT --to-source 172.22.100.3

# DNAT — forward port 8080 to a specific pod:
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.42.5.8:8080

# DNAT — Kubernetes NodePort (this is what kube-proxy does):
iptables -t nat -A PREROUTING -p tcp --dport 31343 \
  -j DNAT --to-destination 10.42.5.8:443


# ═══ SEE EXISTING RULES ═══════════════════════════════════════════

# Show filter rules:
iptables -L -n -v

# Show NAT rules:
iptables -t nat -L -n -v

# Show NAT rules in a more readable format:
iptables -t nat -S  # shows as commands

# Count packets matching each rule:
iptables -t nat -L -n -v --line-numbers


# ═══ CONNTRACK — CONNECTION TRACKING TABLE ═══════════════════════

# Show all tracked connections:
conntrack -L

# Output example:
# tcp 6 431999 ESTABLISHED
#   src=10.42.5.8 dst=8.8.8.8 sport=49321 dport=443
#   src=8.8.8.8   dst=172.22.100.3 sport=443 dport=49321
#   [ASSURED] mark=0 use=1
#
# The two lines: original direction and reply direction
```

---

## 6. How kube-proxy Uses iptables

```
KUBERNETES SERVICE: registry-provisioner ClusterIP 10.43.x.x:80 → pod 10.42.5.8:8080

kube-proxy programs these rules automatically when Service is created:

  # KUBE-SERVICES chain (added to PREROUTING and OUTPUT):
  -A KUBE-SERVICES -d 10.43.225.228/32 -p tcp --dport 80 -j KUBE-SVC-ABCDEF

  # KUBE-SVC-ABCDEF (load balancing — one rule per backend pod):
  -A KUBE-SVC-ABCDEF -j KUBE-SEP-POD1   # if only one backend

  # KUBE-SEP-POD1 (the actual DNAT):
  -A KUBE-SEP-POD1 -j DNAT --to-destination 10.42.5.8:8080

  # For multiple backends (e.g. 2 provisioner pods), load balance 50/50:
  -A KUBE-SVC-ABCDEF -m statistic --mode random --probability 0.5 -j KUBE-SEP-POD1
  -A KUBE-SVC-ABCDEF                                              -j KUBE-SEP-POD2


PACKET JOURNEY: CoreDNS → provisioner service → pod

  1. Pod A sends to 10.43.225.228:80
  2. iptables PREROUTING:
     KUBE-SERVICES: dst=10.43.225.228:80 → jump to KUBE-SVC-ABCDEF
     KUBE-SVC-ABCDEF: 50% probability → jump to KUBE-SEP-POD1
     KUBE-SEP-POD1: DNAT → dst becomes 10.42.5.8:8080
  3. Routing: 10.42.5.8 is on this node → local delivery
  4. iptables POSTROUTING: conntrack records the translation
  5. Pod 10.42.5.8 receives packet
  6. Pod replies to src=PodA:port
  7. iptables conntrack reverses the DNAT on reply


  THE RESULT: ClusterIP 10.43.225.228 appears real to callers
              but is just an iptables rule — no interface holds this IP
```

---

## 7. NetworkPolicy — Kubernetes Firewall

NetworkPolicy is iptables/eBPF rules managed by the CNI (Calico or Kube-OVN), expressed as K8s objects.

```yaml
# deny-cross-tenant NetworkPolicy (from your Harbor deployment):
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant
  namespace: acme-management
spec:
  podSelector: {}             # applies to ALL pods in acme-management
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx   # allow from nginx
    - podSelector: {}         # allow from same namespace pods
  egress:
  - ports:
    - protocol: UDP
      port: 53                # DNS
  - to:
    - podSelector: {}         # to same namespace pods
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8          # block RFC1918 (other tenants)
        - 172.16.0.0/12
        - 192.168.0.0/16
```

```
WHAT CALICO/KUBE-OVN DOES WITH THIS POLICY:

  Translates to iptables (Calico) or OVN ACLs (Kube-OVN):

  [harbor-acme-core pod] trying to reach [harbor-beta-core pod] (10.52.x.x):

  Calico egress check for acme-management:
    Rule: block 10.0.0.0/8 (RFC1918) except to same namespace pods
    10.52.x.x is in 10.0.0.0/8
    Target pod is NOT in acme-management namespace
    → BLOCKED ✅ (tenant isolation enforced)

  [harbor-acme-core pod] trying to reach Trivy DB (8.8.8.8):
    Rule: allow 0.0.0.0/0 except RFC1918
    8.8.8.8 is public → NOT in RFC1918 exception list
    → ALLOWED ✅
```

---

## LAB 5 — iptables NAT in Action

```bash
#!/bin/bash
# lab5-nat.sh
# Creates: client (10.10.0.2) → NAT machine → server (10.20.0.2)
# Client uses private IP, server sees NAT machine's IP

sudo ip netns add client
sudo ip netns add nat-box
sudo ip netns add server

# client ↔ nat-box link
sudo ip link add veth-client type veth peer name veth-nat-left
sudo ip link set veth-client netns client
sudo ip link set veth-nat-left netns nat-box

# nat-box ↔ server link
sudo ip link add veth-nat-right type veth peer name veth-server
sudo ip link set veth-nat-right netns nat-box
sudo ip link set veth-server netns server

# Configure client
sudo ip netns exec client ip link set veth-client up
sudo ip netns exec client ip addr add 10.10.0.2/24 dev veth-client
sudo ip netns exec client ip route add default via 10.10.0.1

# Configure nat-box
sudo ip netns exec nat-box ip link set veth-nat-left up
sudo ip netns exec nat-box ip link set veth-nat-right up
sudo ip netns exec nat-box ip addr add 10.10.0.1/24 dev veth-nat-left
sudo ip netns exec nat-box ip addr add 10.20.0.1/24 dev veth-nat-right
sudo ip netns exec nat-box sysctl -w net.ipv4.ip_forward=1
# SNAT: clients become the nat-box's right IP
sudo ip netns exec nat-box iptables -t nat -A POSTROUTING \
  -s 10.10.0.0/24 -o veth-nat-right -j SNAT --to-source 10.20.0.1

# Configure server
sudo ip netns exec server ip link set veth-server up
sudo ip netns exec server ip addr add 10.20.0.2/24 dev veth-server
sudo ip netns exec server ip route add default via 10.20.0.1

# Test: ping from client to server
echo "=== Ping from client to server (through NAT) ==="
sudo ip netns exec client ping -c 3 10.20.0.2

# See conntrack table on nat-box:
echo ""
echo "=== Conntrack table on nat-box ==="
sudo ip netns exec nat-box conntrack -L 2>/dev/null || \
  echo "(conntrack not available — install conntrack package)"

# See from server's perspective: what IP contacted it?
echo ""
echo "=== Server sees connections FROM (should be 10.20.0.1, not 10.10.0.2) ==="
sudo ip netns exec server ss -tn 2>/dev/null | head -5

# Cleanup
sudo ip netns del client
sudo ip netns del nat-box
sudo ip netns del server
```

---

## Check Your Understanding

**Q1:** kube-proxy creates a Service with ClusterIP 10.43.225.228. Which server actually holds the IP 10.43.225.228?
> No server holds it. It's a virtual IP that exists only in iptables DNAT rules on every node. Any packet destined to 10.43.225.228 hits an iptables rule that changes the destination to the actual pod IP before the packet is even routed.

**Q2:** A pod (10.42.5.8) makes a request to Google. Google replies to 172.22.100.3 (the node IP). How does the reply reach the pod?
> The SNAT rule that changed 10.42.5.8→172.22.100.3 also created a conntrack entry. When the reply arrives with dst=172.22.100.3, conntrack finds the entry and reverses the SNAT (DNAT on reply path): changes dst back to 10.42.5.8 and delivers to the pod.

**Q3:** What is the difference between MASQUERADE and SNAT?
> SNAT uses a fixed specified IP (`--to-source 172.22.100.3`). MASQUERADE automatically uses the outgoing interface's current IP (dynamic). MASQUERADE is used when the source IP might change (DHCP). SNAT is slightly faster (no interface lookup).

---

## Summary

```
KEY FACTS:
  ├── NAT = translate IP addresses on packets passing through a router/firewall
  ├── SNAT = change SOURCE IP (outbound, hides private IPs behind public IP)
  ├── DNAT = change DESTINATION IP (inbound, redirects to real backend)
  ├── Conntrack = tracks connections to allow reverse NAT on replies
  ├── MASQUERADE = SNAT using outgoing interface's IP (dynamic)
  ├── iptables = Linux kernel packet processing: filter (firewall) + nat (SNAT/DNAT)
  ├── kube-proxy = creates iptables DNAT rules for every Kubernetes Service
  └── NetworkPolicy = K8s object → CNI translates to iptables/eBPF/OVN ACLs

IN YOUR SETUP:
  Pod → internet:   Calico SNAT (10.42.x.x → 172.22.100.3)
                    then WSO2 router SNAT (172.22.100.3 → public IP)
  
  External → Harbor: MetalLB receives on 192.168.10.6
                      kube-proxy DNAT → ingress-nginx pod
                      ingress-nginx proxies → harbor-acme-core
  
  Pod → Service:    kube-proxy DNAT (10.43.x.x → real pod IP)

NEXT: File 05 — Linux networking internals: bridges, veth pairs, network namespaces.
```
