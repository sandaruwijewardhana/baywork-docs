# 04 — Kubernetes Fundamentals

> **Goal:** Understand the core Kubernetes objects this system uses: Namespaces, Pods, Deployments, Services, Secrets, PVCs, RBAC, and NetworkPolicy.

---

## 1. What Is Kubernetes?

Kubernetes (K8s) is a **container orchestrator** — it takes containers and decides where and how to run them across a cluster of servers.

```
Without Kubernetes:
  You: "run this container on server-3"
  Server-3 crashes → your app is down
  You: manually restart it on server-4

With Kubernetes:
  You: "I want 3 copies of this container always running"
  K8s: places them across servers
  Server-3 crashes → K8s detects it, reschedules on server-5
  You: do nothing — it just works
```

---

## 2. Cluster Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Control Plane (Master)                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │   │
│  │  │  API Server  │  │  Scheduler   │  │   etcd   │  │   │
│  │  │  (REST API)  │  │  (placement) │  │  (state) │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Node 1     │  │   Node 2     │  │   Node 3     │      │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │      │
│  │  │ Pod A  │  │  │  │ Pod B  │  │  │  │ Pod C  │  │      │
│  │  │ Pod D  │  │  │  │ Pod E  │  │  │  │ Pod F  │  │      │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │      │
│  │  kubelet     │  │  kubelet     │  │  kubelet     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

- **API Server** — everything talks to this; our Go code calls it via `client-go`
- **Scheduler** — decides which Node a Pod runs on
- **etcd** — distributed key-value store; the source of truth for all K8s state
- **kubelet** — agent on each Node; actually starts/stops containers
- **Node** — a physical or virtual machine in the cluster

---

## 3. The Core Objects

### Pod — the smallest unit

A Pod is **one or more containers that share a network and storage**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: harbor-core-abc123
  namespace: tenant-a-management
spec:
  containers:
    - name: core
      image: goharbor/harbor-core:v2.10.0
      ports:
        - containerPort: 8080
      env:
        - name: HARBOR_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: harbor-secret
              key: adminPassword
```

Pods are **ephemeral** — they can be killed and replaced. You almost never create Pods directly; you use Deployments.

### Deployment — "keep N pods running"

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: harbor-core
  namespace: tenant-a-management
spec:
  replicas: 1          # ← keep 1 copy running
  selector:
    matchLabels:
      app: harbor-core  # ← manages pods with this label
  template:            # ← the pod template
    metadata:
      labels:
        app: harbor-core
    spec:
      containers:
        - name: core
          image: goharbor/harbor-core:v2.10.0
```

```
Deployment "harbor-core"
  │
  ├── desires: 1 replica
  ├── current: 1 pod running ✓
  │
  └── if pod dies → creates a replacement automatically
```

The provisioner's own Deployment (in `k8s/provisioner-deployment.yaml`) has **2 replicas** for high availability.

### Service — stable network address

Pods come and go (crash, restart, upgrade). A **Service** gives a stable DNS name and IP:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: harbor-core
  namespace: tenant-a-management
spec:
  selector:
    app: harbor-core   # ← routes to pods with this label
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP       # ← internal-only (no external access)
```

```
Other pods call: http://harbor-core/api/v2.0/...
                        │
                        ▼
              Service "harbor-core"
                        │
              routes to pod: harbor-core-abc123:8080
              (even if pod restarts and gets new IP)
```

Service types:
- **ClusterIP** — internal only (default)
- **NodePort** — accessible on each node's IP
- **LoadBalancer** — creates a cloud load balancer
- **Ingress** — HTTP/HTTPS routing with host-based rules (used here!)

### Namespace — logical isolation

```
Cluster
├── namespace: kube-system           (K8s internal components)
├── namespace: registry-controller   (the provisioner API runs here)
├── namespace: tenant-a-management   (Harbor for tenant-a)
├── namespace: tenant-b-management   (Harbor for tenant-b)
└── namespace: ingress-nginx         (nginx ingress controller)
```

Each tenant gets their own namespace. This means:
- Separate DNS: `harbor-core.tenant-a-management.svc.cluster.local`
- Separate resource quotas (configurable)
- Separate NetworkPolicies
- Separate RBAC

The provisioner creates namespaces like this:

```go
// From backend/internal/k8s/client.go
func (c *Client) EnsureNamespace(ctx context.Context, namespace, tenantID string) error {
    ns := &corev1.Namespace{
        ObjectMeta: metav1.ObjectMeta{
            Name: namespace,                           // "tenant-a-management"
            Labels: map[string]string{
                "lkdc.wso2.com/tenant":    tenantID, // ← for NetworkPolicy selectors
                "lkdc.wso2.com/component": "registry",
            },
        },
    }
    _, err := c.cs.CoreV1().Namespaces().Create(ctx, ns, metav1.CreateOptions{})
    if errors.IsAlreadyExists(err) {
        return nil   // idempotent — OK if already exists
    }
    return err
}
```

---

## 4. Secrets — Sensitive Configuration

A **Secret** stores sensitive data (passwords, tokens, TLS certs) as base64-encoded values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-admin-secret
  namespace: tenant-a-management
type: Opaque
data:
  adminPassword: c3VwZXJzZWNyZXQ=   # base64("supersecret")
```

> ⚠️ base64 is **NOT encryption** — anyone who can read the Secret can decode it. Real security comes from K8s RBAC controlling who can read Secrets.

In this system:
- **`registry-master-key`** Secret stores the AES encryption key (mounted as a file)
- **`registry-provisioner-db`** Secret stores the PostgreSQL DSN (connection string)

```yaml
# From k8s/provisioner-deployment.yaml
volumeMounts:
  - name: master-key
    mountPath: /etc/secrets      # ← accessible as /etc/secrets/master-key
    readOnly: true
volumes:
  - name: master-key
    secret:
      secretName: registry-master-key
      defaultMode: 0400          # ← owner read-only (chmod 400)
```

### ConfigMap — Non-sensitive Configuration

Like a Secret but for non-sensitive values:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-provisioner-config
data:
  BASE_DOMAIN: "lkdc.wso2.com"
  STORAGE_CLASS: "longhorn"
  PORT: "8080"
```

---

## 5. PersistentVolumeClaims — Persistent Disk

Containers are **ephemeral** — their filesystem is deleted when they stop. To persist data (Harbor's images, PostgreSQL data), we use PersistentVolumeClaims (PVCs):

```
PVC Request:
  "I need 50 GiB of storage, ReadWriteOnce, class=longhorn"
                    │
                    ▼
StorageClass "longhorn"
  provisions a real disk (cloud volume, NFS, Ceph...)
                    │
                    ▼
PersistentVolume bound
  actual disk: /dev/sdb on some node
                    │
                    ▼
Pod mounts it at: /storage/docker  (Harbor's image blobs)
```

```yaml
# Harbor registry storage (from Helm values template)
persistence:
  persistentVolumeClaim:
    registry:
      storageClass: longhorn
      size: 50Gi                # ← professional plan
      accessMode: ReadWriteOnce # ← only one pod can write at a time
```

When a tenant is deleted, PVCs are kept by default (`resourcePolicy: keep`) — their data isn't lost unless you explicitly delete them. This is intentional: you don't want to accidentally destroy 100GB of production images.

---

## 6. RBAC — Who Can Do What

RBAC (Role-Based Access Control) in Kubernetes controls which **service accounts** can call which **API verbs** on which **resources**.

```
ServiceAccount "registry-provisioner"
  │
  └── ClusterRole "registry-provisioner"
        │
        ├── namespaces:        create, get, update, delete
        ├── deployments:       get, list, watch
        ├── pods:              get, list, watch
        ├── persistentvolumeclaims: delete
        ├── networkpolicies:   create, get, update
        └── secrets:           create, get (for TLS certs)
```

This follows the **principle of least privilege** — the provisioner can only do exactly what it needs. It cannot:
- Read secrets in other namespaces
- Modify nodes or the control plane
- Access other tenants' data

From `k8s/namespace-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: registry-provisioner
rules:
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["create", "get", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies"]
    verbs: ["create", "get", "update", "patch"]
```

---

## 7. Ingress — HTTP Routing

An **Ingress** is a rule that says "requests for this hostname → route to this Service":

```yaml
# Harbor gets its own Ingress automatically from the Helm chart
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod  # ← auto-provision TLS cert
spec:
  rules:
    - host: registry.tenant-a.lkdc.wso2.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: harbor     # ← routes to Harbor's nginx service
```

```
Internet
    │
    ▼
nginx Ingress Controller (runs in ingress-nginx namespace)
    │  matches host: registry.tenant-a.lkdc.wso2.com
    │
    ▼
Harbor nginx service (in tenant-a-management namespace)
    │
    ▼
Harbor nginx pod → Core / Registry
```

Each tenant's Harbor gets a unique subdomain:
- `registry.tenant-a.lkdc.wso2.com`
- `registry.tenant-b.lkdc.wso2.com`

---

## 8. Probes — Health Monitoring

Kubernetes knows if a pod is healthy via **probes**:

```yaml
# From k8s/provisioner-deployment.yaml
livenessProbe:
  httpGet:
    path: /healthz    # ← calls GET /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 30
  failureThreshold: 3    # ← 3 consecutive failures → restart the pod

readinessProbe:
  httpGet:
    path: /readyz     # ← calls GET /readyz (also checks DB)
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

```
liveness probe fails 3 times → K8s kills and restarts the pod
readiness probe fails         → K8s removes pod from Service endpoints
                                (stops sending it traffic)
```

This is why `server.go` has `/healthz` and `/readyz` endpoints — Kubernetes needs them.

---

## 9. The Go Client — How the Provisioner Talks to K8s

The provisioner uses `k8s.io/client-go` to call the Kubernetes API:

```go
// From backend/internal/k8s/client.go

// Build config from KUBECONFIG file (dev) or in-cluster service account (prod)
func NewClient(cfg config.KubeConfig) (*Client, error) {
    if cfg.InCluster {
        restCfg, _ = rest.InClusterConfig()  // reads /var/run/secrets/...
    } else {
        restCfg, _ = clientcmd.BuildConfigFromFlags("", cfg.KubeconfigPath)
    }
    cs, _ := kubernetes.NewForConfig(restCfg)
    return &Client{cs: cs}, nil
}

// Create a namespace
c.cs.CoreV1().Namespaces().Create(ctx, ns, metav1.CreateOptions{})

// Get all deployments
c.cs.AppsV1().Deployments(namespace).List(ctx, metav1.ListOptions{})

// Delete a PVC
c.cs.CoreV1().PersistentVolumeClaims(namespace).Delete(ctx, name, metav1.DeleteOptions{})
```

The pattern is always: `clientset.GroupVersion().ResourceType(namespace).Verb(...)`.

---

## 🏋️ Exercises

### Exercise 1 — Explore K8s Objects in Running Dev Stack
```bash
# The dev stack uses docker-compose, not real K8s
# But you can practice with kubectl against a local cluster (minikube/kind)
# If you have kubectl available:
kubectl get namespaces
kubectl get deployments --all-namespaces
kubectl get pods --all-namespaces
```

### Exercise 2 — Read the Provisioner Deployment YAML
Open [k8s/provisioner-deployment.yaml](../k8s/provisioner-deployment.yaml):
- How many replicas?
- What is the user ID the process runs as?
- Why is `readOnlyRootFilesystem: true`?
- Why is the `helm-values` volume type `Memory`?

### Exercise 3 — Trace a Namespace Creation
In [backend/internal/k8s/client.go](../backend/internal/k8s/client.go), find `EnsureNamespace`:
- What labels are applied to the namespace?
- What happens if the namespace already exists?
- Why is the `lkdc.wso2.com/tenant` label important? (Hint: NetworkPolicy)

### Exercise 4 — Understand In-Cluster vs. Out-of-Cluster Auth
```go
// Look at NewClient() in k8s/client.go
if cfg.InCluster {
    restCfg, _ = rest.InClusterConfig()
} else {
    restCfg, _ = clientcmd.BuildConfigFromFlags("", cfg.KubeconfigPath)
}
```
- What file does `InClusterConfig()` read? (Hint: it's a mounted Secret)
- For local dev, what env var points to the kubeconfig file?
- Look at `docker-compose.dev.yml` — what kubeconfig is mounted?

### Exercise 5 — Design an RBAC Rule
The provisioner needs to **read** (not write) all Pods in any namespace.
Write the RBAC rule:
```yaml
- apiGroups: [???]
  resources: [???]
  verbs: [???]
```

---

## Summary

| Object | Purpose |
|--------|---------|
| **Pod** | Smallest unit — one or more containers |
| **Deployment** | Keeps N pods running; handles rolling updates |
| **Service** | Stable network address for a set of pods |
| **Namespace** | Logical partition — each tenant gets one |
| **Secret** | Stores sensitive data (passwords, tokens) |
| **ConfigMap** | Stores non-sensitive configuration |
| **PVC** | Persistent disk — survives pod restarts |
| **Ingress** | HTTP/HTTPS routing by hostname |
| **NetworkPolicy** | Firewall rules between pods |
| **ClusterRole/Binding** | RBAC — controls who can do what |
| **ServiceAccount** | Identity for a running pod |

**Next:** [05 — Helm: Kubernetes Package Manager →](./05-helm-package-manager.md)
