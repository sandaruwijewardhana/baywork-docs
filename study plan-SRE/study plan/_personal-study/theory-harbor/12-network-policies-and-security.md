# 12 — Network Policies & Tenant Isolation

> **Goal:** Understand Kubernetes NetworkPolicy, why tenant namespaces need network isolation, and the full security model of this system.

---

## 1. The Problem: Flat Network

By default, **every Pod in a Kubernetes cluster can talk to every other Pod** — regardless of namespace:

```
WITHOUT NetworkPolicy:
  Harbor pod (tenant-a-management) ──────► Harbor pod (tenant-b-management)
  Harbor pod (tenant-b-management) ──────► Platform PostgreSQL
  Compromised pod (any namespace)  ──────► Registry-provisioner secrets
```

If a vulnerability exists in Harbor and one tenant's instance is compromised, the attacker could reach other tenants' data. This is the **lateral movement** attack.

---

## 2. NetworkPolicy — Kubernetes Firewall Rules

A `NetworkPolicy` object is a **namespace-scoped firewall**. It says which pods can send/receive traffic.

```yaml
# This NetworkPolicy applies to ALL pods in the namespace
spec:
  podSelector: {}   # ← {} means "select all pods"
  policyTypes:
    - Ingress        # ← control incoming traffic
    - Egress         # ← control outgoing traffic
```

**Default deny + explicit allow** — the safest approach:
- When a NetworkPolicy exists, all traffic NOT explicitly allowed is **denied**
- You start with "deny everything" and add allow rules

---

## 3. The Tenant Isolation Policy

Created in Step 1 of the deploy worker. Applied to every tenant's namespace:

```go
// From backend/internal/k8s/client.go
policy := &networkingv1.NetworkPolicy{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "deny-cross-tenant",
        Namespace: namespace,   // e.g. "tenant-a-management"
    },
    Spec: networkingv1.NetworkPolicySpec{
        PodSelector: metav1.LabelSelector{},  // all pods
        PolicyTypes: []PolicyType{Ingress, Egress},

        Ingress: []NetworkPolicyIngressRule{
            // ALLOW: traffic from ingress-nginx namespace
            From: NamespaceSelector{kubernetes.io/metadata.name: "ingress-nginx"},
            // ALLOW: traffic from pods within the SAME namespace
            From: PodSelector{},
        },

        Egress: []NetworkPolicyEgressRule{
            // ALLOW: DNS (port 53) — pods need to resolve hostnames
            Ports: [{port: 53, protocol: UDP}],
            // ALLOW: traffic to pods within the SAME namespace
            To: [PodSelector{}],
            // ALLOW: external internet (for Trivy CVE DB updates, image pulls)
            // But DENY private IP ranges (blocks other tenants, platform DB)
            To: IPBlock{CIDR: "0.0.0.0/0", Except: ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]}
        },
    },
}
```

### Traffic Analysis

```
ALLOWED:
  external user → ingress-nginx → harbor-nginx pod ✓
  harbor-core → harbor-postgres (same namespace) ✓
  harbor-core → harbor-redis (same namespace) ✓
  harbor-trivy → cve-database.trivy.io (external, not private IP) ✓
  harbor-registry → DNS (port 53) ✓

BLOCKED:
  harbor-core (tenant-a) → harbor-core (tenant-b) ✗  (cross-namespace)
  harbor-pod (tenant-a) → platform-postgres ✗         (private IP)
  harbor-pod (any) → provisioner secrets ✗             (different namespace)
  harbor-pod (any) → K8s API server ✗                  (HTTPS on private IP)
```

---

## 4. Visualising the Network Boundaries

```
Cluster Network
├── namespace: registry-controller
│     │  ← only reachable from ingress-nginx and itself
│     │  ← can reach: K8s API (port 443), platform DB (5432), ingress
│     └── provisioner pods
│
├── namespace: ingress-nginx
│     └── nginx-ingress pods
│           → routes HTTPS to tenant Harbor instances
│
├── namespace: tenant-a-management  ← NetworkPolicy: deny-cross-tenant
│     │  ← accepts: ingress-nginx traffic + internal pod traffic
│     │  ← denies: traffic from/to other tenant namespaces
│     │  ← allows: external internet (non-private IPs) for Trivy
│     ├── harbor-core
│     ├── harbor-registry
│     ├── harbor-postgres
│     └── harbor-redis
│
└── namespace: tenant-b-management  ← same NetworkPolicy
      ├── harbor-core
      ├── harbor-registry
      ├── harbor-postgres
      └── harbor-redis
```

```
tenant-a-management pods ─────────────────────────── ✗ → tenant-b-management
tenant-a-management pods ─────────────────────────── ✗ → platform-postgres
tenant-a-management pods ───────────────────────────  ✓ → CVE-database.trivy.io (external)
ingress-nginx ──────────────────────────────────────  ✓ → tenant-a harbor-nginx
```

---

## 5. The Provisioner's Own Network Policy

The provisioner itself has a NetworkPolicy restricting what it can reach:

```yaml
# From k8s/provisioner-deployment.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: provisioner-network-policy
  namespace: registry-controller
spec:
  podSelector:
    matchLabels:
      app: registry-provisioner
  policyTypes: [Ingress, Egress]

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx  # ← API traffic
        - podSelector: {}   # ← same namespace (metrics scraping)

  egress:
    - ports: [{port: 53, protocol: UDP}]    # DNS
    - ports: [{port: 443, protocol: TCP}]   # K8s API server, Helm repos, JWKS
    - ports: [{port: 5432, protocol: TCP}]  # Platform PostgreSQL
    - to: [{}]                              # All tenant namespaces for Helm
```

The provisioner can:
- Receive API requests from ingress
- Call K8s API (create namespaces, deploy Helm, etc.)
- Connect to platform PostgreSQL
- Deploy to tenant namespaces

The provisioner **cannot**:
- Be reached directly (only via ingress-nginx)
- Connect to random external services
- Access other internal services

---

## 6. TLS Everywhere

All traffic uses TLS (HTTPS):

```
External user → HTTPS → nginx Ingress (TLS terminates here) → HTTP → Harbor nginx
                         cert-manager auto-provisions TLS cert via Let's Encrypt
```

### cert-manager

`cert-manager` is a Kubernetes controller that automatically provisions TLS certificates:

```yaml
# From k8s/cert-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@lkdc.wso2.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

When an Ingress has the annotation `cert-manager.io/cluster-issuer: letsencrypt-prod`:
1. cert-manager sees the new Ingress
2. Requests a certificate from Let's Encrypt for `registry.tenant-a.lkdc.wso2.com`
3. Stores the certificate as a Kubernetes Secret
4. nginx Ingress uses the Secret for TLS

---

## 7. Defense in Depth — Security Layers

Security isn't one thing — it's layers. If one layer fails, others protect:

```
Layer 1: JWT Authentication
  All API requests must have a valid JWT
  → Unauthenticated access blocked

Layer 2: TenantGuard
  TENANT_ADMIN can only access their own tenant
  → Cross-tenant API access blocked

Layer 3: Rate Limiting
  Max 1 provision per 10 minutes per tenant
  → Denial-of-service attacks slowed

Layer 4: NetworkPolicy
  Pods can't reach other tenants' namespaces
  → Compromised pod can't reach other tenants

Layer 5: RBAC
  Provisioner ServiceAccount only has the permissions it needs
  → Even if provisioner is compromised, can't escalate

Layer 6: Encryption at Rest
  Credentials encrypted with AES-256-GCM
  → Stolen DB backup is useless without master key

Layer 7: Distroless Container
  No shell, no package manager in the container
  → Harder to exploit even with code execution

Layer 8: Non-Root Container
  Runs as user 65534 (nobody)
  → File system writes blocked without explicit mounts

Layer 9: Read-Only Root Filesystem
  readOnlyRootFilesystem: true
  → Can't write to container filesystem (except emptyDir mounts)

Layer 10: Audit Log
  Every action logged with actor, IP, timestamp
  → Accountability, forensics after incidents
```

---

## 8. The ExceptCIDR Pattern

The `Except` field in the egress IP block rule is critical:

```go
To: []NetworkPolicyPeer{
    {
        IPBlock: &networkingv1.IPBlock{
            CIDR: "0.0.0.0/0",        // ← allow all IPs...
            Except: []string{
                "10.0.0.0/8",         // ← ...except RFC 1918 private ranges
                "172.16.0.0/12",
                "192.168.0.0/16",
            },
        },
    },
},
```

Why? These IP ranges are used by:
- Kubernetes pod network (usually `10.x.x.x`)
- Platform databases and services
- Other tenants' namespaces

By blocking private IPs but allowing public IPs, Harbor can:
- ✓ Update Trivy's CVE database from the internet
- ✓ Pull base images from Docker Hub
- ✗ Reach platform PostgreSQL (10.x.x.x)
- ✗ Reach other tenant namespaces (10.x.x.x)

---

## 🏋️ Exercises

### Exercise 1 — Read the NetworkPolicy Code
Open [backend/internal/k8s/client.go](../backend/internal/k8s/client.go), find `ApplyNetworkPolicy`.

- What namespace does `ingressNS` refer to?
- What port 53 rule is needed? Why would pods break without it?
- The `PodSelector: metav1.LabelSelector{}` with empty selector matches ALL pods in the namespace — why is this needed for internal Harbor communication?

### Exercise 2 — Trace Cross-Tenant Blocking
Given this scenario:
- tenant-a's Harbor core pod is at `10.128.5.23`
- tenant-b's Harbor core pod is at `10.128.8.47`

With the NetworkPolicy applied to tenant-a's namespace:
1. Can tenant-a's core pod make a TCP connection to `10.128.8.47`? Why?
2. Can tenant-a's core pod make a TCP connection to `8.8.8.8` (Google DNS)? Why?
3. What about `172.16.0.1`?

### Exercise 3 — Security Layers Quiz
For each attack, which security layer stops it?

| Attack | Layer that stops it |
|--------|-------------------|
| Someone calls the API without a JWT | ? |
| TENANT_ADMIN tries to delete another tenant's registry | ? |
| Attacker steals the PostgreSQL DB dump | ? |
| Compromised Harbor pod tries to reach platform DB | ? |
| Script tries to create 100 registries per minute | ? |

### Exercise 4 — Design a Network Policy
Write (in pseudocode or YAML) a NetworkPolicy for a new component: the **Billing Service** in namespace `billing`.

Rules:
- Accepts traffic from: `api-gateway` namespace only
- Can reach: `platform-postgres` (port 5432), external payment gateway (internet), DNS
- Blocks: everything else

### Exercise 5 — What's Missing?
The current NetworkPolicy allows egress to all external (non-private) IPs. This means Harbor can connect to ANY internet server.

What risks does this create? How could you make it more restrictive?
(Hint: What's the minimum set of external IPs Harbor actually needs to reach?)

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Flat network** | Default K8s: every pod can reach every pod |
| **NetworkPolicy** | Namespace-scoped firewall rules |
| **Default deny** | No policy = allow all; policy = deny all except stated |
| **Ingress rule** | Controls who can send traffic TO the pods |
| **Egress rule** | Controls where pods can SEND traffic |
| **PodSelector** | Selects pods by label (empty `{}` = all pods) |
| **NamespaceSelector** | Selects pods in specific namespaces |
| **IPBlock + Except** | Allow internet but block private IP ranges |
| **cert-manager** | Auto-provisions TLS certs from Let's Encrypt |
| **TenantGuard** | API-level isolation (JWT claim vs URL param) |
| **Defense in depth** | Multiple security layers — one breach ≠ total compromise |

**Next:** [13 — Everything Together: Full Flow →](./13-full-flow-end-to-end.md)
