# 13 — Everything Together: Full Flow End-to-End

> **Goal:** Trace the complete journey from "user clicks Create Registry" to "user pushes an image" — showing the exact file, exact function, exact Pod, and exact namespace/layer for every step.

---

## What is a "Tenant"?

Before reading the flow, you must understand what "tenant" means in this system.

A **tenant** is NOT a Kubernetes namespace. A tenant is a **LKDC customer company** — e.g. "Acme Corp", "Beta Labs", "Gamma Inc".

In this platform (Harvester + Rancher), a tenant is represented as a **logically separated set of namespaces** at the Harvester level. Think of it as a virtual boundary drawn around everything that belongs to one customer.

```
HARVESTER LEVEL — what a "tenant" looks like
══════════════════════════════════════════════════════════════════════

  Tenant: Acme Corp
  ┌─────────────────────────────────────────────────────────────────┐
  │  This entire box = "Acme's space" in Harvester/Rancher          │
  │  Acme can ONLY see inside this box.                             │
  │  No other tenant can see inside this box.                       │
  │                                                                 │
  │  management namespace  ← default, exists for every tenant       │
  │  ┌────────────────────────────────────────────────────────────┐ │
  │  │  Harbor pods will be deployed here by the provisioner      │ │
  │  │  (this is the tenant's existing namespace — not new)       │ │
  │  └────────────────────────────────────────────────────────────┘ │
  │                                                                 │
  │  project-1 namespace   ← tenant creates this for their apps    │
  │  ┌────────────────────────────────────────────────────────────┐ │
  │  │  cluster-A  (RKE2 K8s cluster inside Harvester VMs)        │ │
  │  │    pod: frontend     pod: backend-api                      │ │
  │  │  cluster-B  (another K8s cluster)                          │ │
  │  │    pod: worker-service                                      │ │
  │  └────────────────────────────────────────────────────────────┘ │
  │                                                                 │
  │  project-2 namespace   ← tenant creates this for staging        │
  │  ┌────────────────────────────────────────────────────────────┐ │
  │  │  cluster-C                                                  │ │
  │  │    pod: staging-app                                         │ │
  │  └────────────────────────────────────────────────────────────┘ │
  └─────────────────────────────────────────────────────────────────┘

  Tenant: Beta Labs
  ┌─────────────────────────────────────────────────────────────────┐
  │  management namespace  ← Harbor pods will go here               │
  │  project-1 namespace                                            │
  │  project-2 namespace                                            │
  └─────────────────────────────────────────────────────────────────┘

  (each tenant is logically separated — identical structure, fully isolated)
```

**Key point:** The word "tenant" in the code (e.g. `tenantID = "acme"`) refers to this whole logical grouping — not a single namespace. When the provisioner deploys Harbor "for tenant acme", it deploys into `acme`'s existing `management` namespace inside that tenant's space.

---

## Full Infrastructure Stack

```
╔══════════════════════════════════════════════════════════════════════╗
║  PHYSICAL LAYER                                                      ║
║  bare-metal Server 1    Server 2    Server 3    Server 4 ...         ║
╚══════════════════════════════════════════════════════════════════════╝
                           ↓
╔══════════════════════════════════════════════════════════════════════╗
║  HARVESTER HCI                                                       ║
║  Runs directly on bare metal.                                        ║
║  Virtualises servers → VMs + Longhorn storage + virtual networking   ║
║  Harvester itself IS a Kubernetes cluster.                           ║
║  VMs, volumes, networks are Kubernetes objects inside Harvester.     ║
╚══════════════════════════════════════════════════════════════════════╝
                           ↓
╔══════════════════════════════════════════════════════════════════════╗
║  RANCHER                                                             ║
║  Multi-cluster Kubernetes management running on Harvester.           ║
║  Provisions RKE2/K3s clusters inside Harvester VMs.                  ║
║  Provides the Projects + Namespaces model per tenant.                ║
║  Each tenant gets their own Project (logical namespace grouping).    ║
╚══════════════════════════════════════════════════════════════════════╝
                           ↓
╔══════════════════════════════════════════════════════════════════════╗
║  NAMESPACE LAYER                                                     ║
║                                                                      ║
║  registry-controller namespace  ← OUR PROVISIONER LIVES HERE        ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  Pod: provisioner-pod-1   (HTTP goroutine + Worker goroutine) │   ║
║  │  Pod: provisioner-pod-2   (same — 2 replicas for HA)          │   ║
║  │  ClusterRole → can reach into ANY tenant's management ns      │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  acme-management namespace  ← HARBOR FOR TENANT ACME LIVES HERE     ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  (this namespace already exists — it is acme's default ns)   │   ║
║  │  harbor-harbor-core        harbor-harbor-portal              │   ║
║  │  harbor-harbor-registry    harbor-harbor-jobservice          │   ║
║  │  harbor-harbor-trivy       harbor-harbor-database            │   ║
║  │  harbor-harbor-redis                                         │   ║
║  │  Ingress: registry.acme.lkdc.wso2.com                      │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  beta-management namespace  ← HARBOR FOR TENANT BETA LIVES HERE     ║
║  (same structure — separate pods, separate PVCs, separate Ingress)   ║
║                                                                      ║
║  acme-project-1 namespace   ← acme's own workload clusters/pods     ║
║  acme-project-2 namespace   ← acme's own workload clusters/pods     ║
║  beta-project-1 namespace   ← beta's own workload clusters/pods     ║
║  ...                                                                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Code File Map

```
implementation-of-auto-harbor-deployment/
│
├── k8s/
│   ├── namespace-rbac.yaml          YOU apply once:
│   │                                  creates: registry-controller namespace
│   │                                  creates: ServiceAccount for provisioner
│   │                                  creates: ClusterRole (lets provisioner reach
│   │                                            into any tenant's management ns)
│   │
│   ├── provisioner-deployment.yaml  YOU apply once:
│   │                                  Deployment (2 replicas) in registry-controller
│   │                                  Service (ClusterIP) in registry-controller
│   │                                  Ingress → registry-api.lkdc.wso2.com
│   │                                  ConfigMap (env vars: BASE_DOMAIN, CERT_ISSUER...)
│   │
│   └── cert-issuer.yaml             YOU apply once:
│                                      ClusterIssuer: letsencrypt-prod (cluster-wide)
│
├── helm/
│   └── harbor-values-template.yaml  NOT deployed directly.
│                                     Provisioner pulls Harbor chart + fills in this
│                                     template with per-tenant values at runtime.
│
├── db/migrations/
│   └── 001_initial.sql              YOU run once against platform PostgreSQL.
│                                     Creates: registry_deployments
│                                              registry_credentials
│                                              audit_log
│
└── backend/
    ├── Dockerfile                   YOU build → wso2/registry-provisioner:latest
    │                                Runs inside provisioner pods in registry-controller
    │
    ├── cmd/
    │   └── main.go                  ENTRY POINT
    │                                WHERE: provisioner-pod-1/-2 | registry-controller
    │                                Wires all components, starts two goroutines:
    │                                  go deployWorker.Run()   ← Worker goroutine
    │                                  httpServer.ListenAndServe() ← HTTP goroutine
    │
    └── internal/
        ├── config/config.go         Reads env vars at pod startup.
        │                            WHERE: provisioner pods | registry-controller
        │
        ├── api/
        │   ├── server.go            Gin engine, middleware chain, route registration.
        │   │                        WHERE: provisioner pods | registry-controller
        │   │
        │   ├── handlers/
        │   │   └── registry.go      HTTP handlers: Create, Get, GetCredentials,
        │   │                        RotateCredentials, GetPullSecret, Delete.
        │   │                        WHERE: provisioner pods | registry-controller
        │   │                        ACTS ON: platform PostgreSQL (writes job row)
        │   │
        │   └── middleware/
        │       ├── auth.go          JWT validation + TenantGuard + DevAuthBypass.
        │       │                    WHERE: provisioner pods | registry-controller
        │       ├── logging.go       Zap structured request/response logger.
        │       │                    WHERE: provisioner pods | registry-controller
        │       └── ratelimit.go     Token bucket: 1 create/10 min per tenant.
        │                            WHERE: provisioner pods | registry-controller
        │
        ├── worker/
        │   └── deploy_worker.go     THE 7-STEP DEPLOY LOOP.
        │                            WHERE (pod):  provisioner pods | registry-controller
        │                            ACTS ON:      tenant's management namespace
        │
        ├── k8s/
        │   └── client.go            EnsureNamespace: ensures acme-management exists
        │                                             (namespace may already exist in
        │                                              Harvester — just ensures labels)
        │                            ApplyNetworkPolicy: isolation inside the ns
        │                            WaitForAllReady: watches Harbor pods in tenant ns
        │                            WHERE (pod):  provisioner pods | registry-controller
        │                            ACTS ON:      acme-management namespace
        │
        ├── helm/
        │   ├── deployer.go          Install/Upgrade/Uninstall via Helm Go SDK.
        │   │                        WHERE (pod):  provisioner pods | registry-controller
        │   │                        INSTALLS INTO: acme-management namespace
        │   │
        │   └── values_generator.go  PlanFor(), GenerateValues().
        │                            WHERE (pod):  provisioner pods | registry-controller
        │
        ├── harbor/
        │   └── bootstrap.go         Harbor REST client: Configure, CreateProject,
        │                            CreateRobotAccount, WaitReady.
        │                            WHERE (code runs): provisioner pods | registry-controller
        │                            CALLS INTO:        harbor-harbor-core pod | acme-management
        │                            ROUTE: provisioner → Ingress → harbor-core service → pod
        │
        ├── crypto/aes.go            AES-256-GCM Encrypt/Decrypt, GeneratePassword.
        │                            WHERE: provisioner pods | registry-controller
        │
        ├── db/postgres.go           All SQL (platform DB, not Harbor's DB).
        │                            WHERE: provisioner pods | registry-controller
        │                            TALKS TO: platform PostgreSQL (outside cluster)
        │
        ├── audit/logger.go          Async audit log via channel.
        │                            WHERE: provisioner pods | registry-controller
        │
        └── metrics/prometheus.go    Prometheus counters, histograms, gauges.
                                     WHERE: provisioner pods | registry-controller
```

---

## Phase 0: Platform Setup — Deploying the Registry Controller

> This happens ONCE before any tenant ever clicks a button.
> Platform/infra team does this manually. After this is done, the system is live.

```
YOU (platform engineer, on your laptop)
    │
    ├── docker build + push        ← builds the Go binary into a container image
    ├── psql < migration           ← creates DB tables in platform PostgreSQL
    ├── kubectl apply (×4 files)   ← creates all K8s objects
    └── kubectl create secret (×2) ← injects DB password + master encryption key
```

---

### Setup Step A — Build and Push the Provisioner Image

```
FILE: backend/Dockerfile
RUNS ON: your laptop / CI pipeline
PRODUCES: Docker image wso2/registry-provisioner:latest
```

```dockerfile
# FILE: backend/Dockerfile
# Multi-stage build — final image is distroless (no shell, minimal attack surface)

FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download                          # download all dependencies

COPY . .
RUN CGO_ENABLED=0 GOOS=linux \
    go build -o provisioner ./cmd/main.go    # compiles cmd/main.go → binary

FROM gcr.io/distroless/static:nonroot        # tiny base image, no shell
COPY --from=builder /app/provisioner /provisioner
USER nonroot:nonroot
ENTRYPOINT ["/provisioner"]                  # runs cmd/main.go's main()
```

```bash
# Commands you run:
docker build -t wso2/registry-provisioner:latest ./backend
docker push wso2/registry-provisioner:latest
# ↑ image is now in your container registry, ready to be pulled by K8s
```

---

### Setup Step B — Run Database Migrations

```
FILE: db/migrations/001_initial.sql
RUNS ON: your laptop (psql client)
ACTS ON: platform PostgreSQL  (outside the K8s cluster, shared across all services)
```

```bash
psql "postgresql://provisioner:pass@postgres-host:5432/registry_provisioner" \
  -f db/migrations/001_initial.sql
```

```sql
-- FILE: db/migrations/001_initial.sql
-- Creates the 3 tables the provisioner needs:

CREATE TABLE IF NOT EXISTS registry_deployments (
    tenant_id     TEXT PRIMARY KEY,
    namespace     TEXT NOT NULL,           -- "acme-management"
    status        TEXT NOT NULL CHECK (status IN ('PENDING','DEPLOYING','READY','FAILED','DELETING','DELETED')),
    registry_url  TEXT,                    -- "https://registry.acme.lkdc.wso2.com"
    helm_release  TEXT,                    -- "harbor-acme"
    plan          TEXT,                    -- "starter" / "professional" / "enterprise"
    progress      JSONB DEFAULT '{}',      -- {"namespace":"READY","helm_install":"STARTING",...}
    error_message TEXT,
    worker_lock   TEXT,                    -- which provisioner pod is processing this row
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    ready_at      TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS registry_credentials (
    tenant_id          TEXT PRIMARY KEY REFERENCES registry_deployments(tenant_id),
    robot_username     TEXT NOT NULL,      -- "robot$ci-default"
    encrypted_token    BYTEA NOT NULL,     -- AES-256-GCM ciphertext
    token_nonce        BYTEA NOT NULL,     -- GCM nonce (random, used once)
    admin_username     TEXT NOT NULL DEFAULT 'admin',
    encrypted_admin_pw BYTEA NOT NULL,
    admin_pw_nonce     BYTEA NOT NULL,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    rotated_at         TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS audit_log (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   TEXT NOT NULL,
    action      TEXT NOT NULL,             -- "REGISTRY_CREATE", "GET_CREDENTIALS", ...
    actor_id    TEXT,
    actor_email TEXT,
    source_ip   TEXT,
    result      TEXT NOT NULL,             -- "SUCCESS" / "FAILURE"
    details     JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_audit_log_tenant_id ON audit_log(tenant_id);
CREATE INDEX IF NOT EXISTS idx_registry_status     ON registry_deployments(status);
-- ↑ idx_registry_status speeds up: WHERE status = 'PENDING' (worker's hot query)
```

---

### Setup Step C — Create K8s Secrets

```
RUNS ON: your laptop (kubectl)
CREATES IN: registry-controller namespace
```

```bash
# Secret 1: platform DB connection string
# Used by: db/postgres.go  New()  → sql.Open("postgres", cfg.DSN)
kubectl create secret generic registry-provisioner-db \
  --from-literal=dsn="postgresql://provisioner:pass@postgres-host:5432/registry_provisioner" \
  -n registry-controller

# Secret 2: AES-256-GCM master encryption key
# Used by: crypto/aes.go  NewCipher() → reads this file at pod startup
# Must be 32 random bytes encoded as hex (64 hex chars)
openssl rand -hex 32 > /tmp/master-key
kubectl create secret generic registry-master-key \
  --from-file=master-key=/tmp/master-key \
  -n registry-controller
rm /tmp/master-key    # never keep plaintext key on disk
```

---

### Setup Step D — Apply RBAC and Namespace

```
FILE: k8s/namespace-rbac.yaml
RUNS ON: your laptop (kubectl apply)
CREATES IN: cluster-wide (ClusterRole, ClusterRoleBinding) + registry-controller namespace
```

```bash
kubectl apply -f k8s/namespace-rbac.yaml
```

```yaml
# FILE: k8s/namespace-rbac.yaml  — what this file contains:

---
# 1. The namespace where provisioner pods live
apiVersion: v1
kind: Namespace
metadata:
  name: registry-controller
  labels:
    lkdc.wso2.com/component: registry-provisioner

---
# 2. The identity the provisioner pods run as
apiVersion: v1
kind: ServiceAccount
metadata:
  name: registry-provisioner
  namespace: registry-controller

---
# 3. What that identity is ALLOWED to do (cluster-wide permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: registry-provisioner-role
rules:
  # Can create/patch namespaces → used by k8s/client.go EnsureNamespace()
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "create", "patch", "update"]

  # Can manage Secrets → Helm stores releases as Secrets; cert-manager TLS Secrets
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "create", "update", "delete"]

  # Can manage all core resources Helm creates in tenant namespaces
  - apiGroups: [""]
    resources: ["services", "configmaps", "persistentvolumeclaims",
                "serviceaccounts", "pods", "endpoints"]
    verbs: ["get", "list", "create", "update", "delete", "patch", "watch"]

  # Can manage Deployments and StatefulSets → Harbor pods
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "create", "update", "delete", "patch", "watch"]

  # Can create Ingress + NetworkPolicy → used in Steps 1 and 3
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["get", "list", "create", "update", "delete", "patch"]

  # cert-manager certificates → needed for TLS auto-provisioning
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates", "issuers", "clusterissuers"]
    verbs: ["get", "list", "create", "update", "delete", "patch"]

  # Pod logs → used by k8s/client.go for status checking
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]

  # StorageClass listing → validate "longhorn" exists before provisioning
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list"]

---
# 4. Bind the ServiceAccount to the ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: registry-provisioner-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: registry-provisioner-role
subjects:
  - kind: ServiceAccount
    name: registry-provisioner
    namespace: registry-controller
# ↑ this is what gives provisioner pods the ability to reach into ANY tenant namespace
```

---

### Setup Step E — Apply TLS Certificate Issuers

```
FILE: k8s/cert-issuer.yaml
RUNS ON: your laptop (kubectl apply)
CREATES: ClusterIssuer objects (cluster-wide, no namespace)
USED BY: Harbor Ingress annotation cert-manager.io/cluster-issuer: letsencrypt-prod
```

```bash
kubectl apply -f k8s/cert-issuer.yaml
```

```yaml
# FILE: k8s/cert-issuer.yaml

---
# Production: auto-TLS from Let's Encrypt for every tenant subdomain
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod      # ← referenced in harbor-values-template.yaml CertIssuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@wso2.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx      # uses nginx Ingress Controller for ACME challenge

---
# Dev/testing: self-signed certs (no Let's Encrypt rate limits)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

**What cert-manager does when it sees a Harbor Ingress:**
```
Harbor Helm chart creates Ingress with annotation:
  cert-manager.io/cluster-issuer: letsencrypt-prod

cert-manager sees that annotation and:
  1. Contacts Let's Encrypt ACME API
  2. Let's Encrypt sends back a challenge token
  3. cert-manager creates a temporary HTTP route via nginx:
       http://registry.acme.lkdc.wso2.com/.well-known/acme-challenge/<token>
  4. Let's Encrypt verifies domain ownership via that route
  5. Let's Encrypt issues TLS certificate
  6. cert-manager stores it as:
       Secret: harbor-harbor-ingress-tls  in acme-management namespace
  7. nginx Ingress loads the cert → HTTPS works
```

---

### Setup Step F — Deploy the Provisioner

```
FILE: k8s/provisioner-deployment.yaml
RUNS ON: your laptop (kubectl apply)
CREATES IN: registry-controller namespace
```

```bash
kubectl apply -f k8s/provisioner-deployment.yaml
```

```yaml
# FILE: k8s/provisioner-deployment.yaml  — what this file contains:

---
# ConfigMap: all non-secret env vars for the provisioner pods
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-provisioner-config
  namespace: registry-controller
data:
  BASE_DOMAIN:          "lkdc.wso2.com"     # → used in values_generator.go
  STORAGE_CLASS:        "longhorn"             # → used in values_generator.go
  INGRESS_CLASS:        "nginx"                # → used in values_generator.go
  CERT_ISSUER:          "letsencrypt-prod"     # → used in values_generator.go
  HARBOR_CHART_VERSION: "1.14.0"              # → used in helm/deployer.go
  HARBOR_HELM_REPO:     "https://helm.goharbor.io"
  JWKS_ENDPOINT:        "https://accounts.lkdc.wso2.com/.well-known/jwks.json"
  JWT_ISSUER:           "https://accounts.lkdc.wso2.com"
  JWT_AUDIENCE:         "registry-provisioner"
  PORT:                 "8080"
  ENV:                  "production"
  KUBERNETES_IN_CLUSTER: "true"
  MASTER_KEY_PATH:      "/etc/secrets/master-key"  # → path inside pod (mounted Secret)
  DB_MAX_OPEN_CONNS:    "25"
  DB_MAX_IDLE_CONNS:    "5"

---
# Deployment: runs 2 replicas of the provisioner pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry-provisioner
  namespace: registry-controller
spec:
  replicas: 2           # 2 replicas → HA; DB locking prevents double-deploy
  selector:
    matchLabels:
      app: registry-provisioner
  template:
    spec:
      serviceAccountName: registry-provisioner   # ← the ServiceAccount with ClusterRole
      containers:
        - name: provisioner
          image: wso2/registry-provisioner:latest # ← the image built in Step A
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: registry-provisioner-config  # ← all env vars from ConfigMap above
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: registry-provisioner-db    # ← the Secret created in Step C
                  key: dsn
          volumeMounts:
            - name: master-key
              mountPath: /etc/secrets              # ← master-key file mounted here
              readOnly: true
            - name: helm-cache
              mountPath: /tmp/helm-charts          # ← Harbor chart downloaded here
            - name: helm-values
              mountPath: /tmp/helm-values          # ← per-tenant values written here
          livenessProbe:
            httpGet:
              path: /healthz                       # ← api/server.go healthz route
              port: 8080
          readinessProbe:
            httpGet:
              path: /readyz                        # ← api/server.go readyz route (pings DB)
              port: 8080
      volumes:
        - name: master-key
          secret:
            secretName: registry-master-key        # ← Secret created in Step C
        - name: helm-cache
          emptyDir: {}                             # in-pod disk for downloaded charts
        - name: helm-values
          emptyDir:
            medium: Memory                         # ← VALUES IN RAM (contain passwords!)

---
# Service: stable ClusterIP for the provisioner pods
apiVersion: v1
kind: Service
metadata:
  name: registry-provisioner
  namespace: registry-controller
spec:
  selector:
    app: registry-provisioner
  ports:
    - port: 80
      targetPort: 8080

---
# Ingress: exposes the API to the outside world via HTTPS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: registry-provisioner
  namespace: registry-controller
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - registry-api.lkdc.wso2.com
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
                  number: 80
# ↑ after this: https://registry-api.lkdc.wso2.com/api/v1/... works

---
# NetworkPolicy: limits what the provisioner pods can talk to
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
              kubernetes.io/metadata.name: ingress-nginx  # only nginx can call us
  egress:
    - ports: [{port: 53, protocol: UDP}]    # DNS lookups
    - ports: [{port: 443, protocol: TCP}]   # K8s API, Helm repo, JWKS, Let's Encrypt
    - ports: [{port: 5432, protocol: TCP}]  # platform PostgreSQL
    - to: [{}]                              # all tenant namespaces (for Helm deploy + Harbor bootstrap)
```

---

### Setup Step G — Apply Monitoring (optional)

```
FILES: monitoring/service-monitor.yaml, monitoring/prometheus-rules.yaml
RUNS ON: your laptop (kubectl apply)
```

```bash
kubectl apply -f monitoring/service-monitor.yaml
kubectl apply -f monitoring/prometheus-rules.yaml
```

```yaml
# FILE: monitoring/service-monitor.yaml
# Tells Prometheus: "scrape /metrics from provisioner pods every 30 seconds"
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: registry-provisioner
  namespace: registry-controller
spec:
  selector:
    matchLabels:
      app: registry-provisioner
  endpoints:
    - port: http
      path: /metrics        # ← metrics/prometheus.go exposes this
      interval: 30s
```

---

### Setup Step H — Pod Startup: What cmd/main.go does

After `kubectl apply -f provisioner-deployment.yaml`, K8s pulls the image and starts the pods.
This is what happens inside each pod when it starts — all in `cmd/main.go`:

```
Pod starts → /provisioner binary runs → main() executes
```

```go
// FILE: backend/cmd/main.go  main()
// RUNS IN: provisioner-pod-1 (and pod-2 simultaneously)
// NAMESPACE: registry-controller

func main() {
    // 1. Logger setup
    logger, _ := zap.NewProduction()
    // FILE: go.uber.org/zap — structured JSON logging

    // 2. Load config from environment variables (injected by ConfigMap + Secrets)
    // FILE: backend/internal/config/config.go  Load()
    cfg, err := config.Load()
    // reads: PORT, DATABASE_URL, BASE_DOMAIN, STORAGE_CLASS, JWKS_ENDPOINT, ...
    // reads: MASTER_KEY_PATH = "/etc/secrets/master-key"

    // 3. Connect to platform PostgreSQL
    // FILE: backend/internal/db/postgres.go  New()
    store, err := db.New(cfg.DB, logger)
    // sql.Open("postgres", DATABASE_URL)
    // db.Ping() → verifies connection is alive
    // if this fails → pod crashes → K8s restarts it → /readyz stays unhealthy

    // 4. Run DB schema migration (idempotent — safe to run on every startup)
    // FILE: backend/internal/db/postgres.go  Migrate()
    store.Migrate()
    // runs: CREATE TABLE IF NOT EXISTS registry_deployments ...
    // (same SQL as db/migrations/001_initial.sql, embedded in schemaSQL const)

    // 5. Load master encryption key from mounted Secret
    // FILE: backend/internal/crypto/aes.go  NewCipher()
    masterKey, _ := os.ReadFile("/etc/secrets/master-key")
    cipher, _ := crypto.NewCipher(masterKey)
    // AES-256-GCM cipher ready for encrypt/decrypt

    // 6. Connect to Kubernetes API server (in-cluster config)
    // FILE: backend/internal/k8s/client.go  NewClient()
    k8sClient, _ := k8s.NewClient(cfg.Kubernetes)
    // rest.InClusterConfig() → reads /var/run/secrets/kubernetes.io/serviceaccount/token
    // kubernetes.NewForConfig() → creates client-go Clientset
    // this is how the pod knows it's "registry-provisioner" ServiceAccount
    // and can use its ClusterRole permissions

    // 7. Initialize Helm deployer
    // FILE: backend/internal/helm/deployer.go  NewDeployer()
    helmDeployer, _ := helm.NewDeployer(cfg.Helm, logger)
    // cli.New() → Helm env settings
    // ensureRepo() → downloads Harbor chart index from helm.goharbor.io
    //                stores in /tmp/helm-charts/ (the emptyDir volume)

    // 8. Metrics registry
    // FILE: backend/internal/metrics/prometheus.go  NewRegistry()
    reg := metrics.NewRegistry()
    // registers: provision_total, provision_duration_seconds, active_deployments
    // these appear at GET /metrics

    // 9. Audit logger
    // FILE: backend/internal/audit/logger.go  NewLogger()
    auditLog := audit.NewLogger(store, logger)
    // creates a channel + goroutine for async DB writes

    // 10. Deploy worker
    // FILE: backend/internal/worker/deploy_worker.go  New()
    deployWorker := worker.New(store, helmDeployer, k8sClient, cipher,
                               cfg.Helm, auditLog, reg, logger)

    // 11. Start Worker goroutine (runs forever in background)
    // FILE: backend/internal/worker/deploy_worker.go  Run()
    go deployWorker.Run(context.Background())
    // ticks every 5 seconds
    // calls GetOldestPending() → picks up PENDING jobs
    // calls deploy() in a new goroutine per job

    // 12. Start HTTP server (blocks until shutdown signal)
    // FILE: backend/internal/api/server.go  NewServer() + Router()
    srv := api.NewServer(cfg, store, cipher, auditLog, reg, logger)
    httpServer := &http.Server{
        Addr:    ":8080",
        Handler: srv.Router(),  // Gin engine with all routes + middleware
    }
    httpServer.ListenAndServe()  // ← BLOCKS HERE (serves requests)

    // 13. Graceful shutdown on SIGTERM (K8s sends this before killing the pod)
    <-quit  // wait for SIGINT or SIGTERM
    httpServer.Shutdown(ctx)  // finish in-flight requests
}
```

**What `api/server.go  Router()` registers:**
```go
// FILE: backend/internal/api/server.go  Router()
// Sets up the Gin engine with all middleware and routes

engine.Use(
    middleware.Recovery(),           // middleware/logging.go  — catch panics
    middleware.RequestLogger(logger),// middleware/logging.go  — log every request
    cors.New(...),                   // gin-contrib/cors      — CORS headers
)

tenants := engine.Group("/api/v1/tenants/:tenantId")
tenants.Use(
    middleware.RateLimiter(store),   // middleware/ratelimit.go — 1 create/10min
    middleware.JWTValidator(cfg.Auth),// middleware/auth.go    — validate JWT
    middleware.TenantGuard(),        // middleware/auth.go    — URL == JWT tenantId
)
tenants.POST  ("/registry",                      h.Create)
tenants.GET   ("/registry",                      h.Get)
tenants.DELETE("/registry",                      h.Delete)
tenants.GET   ("/registry/credentials",          h.GetCredentials)
tenants.POST  ("/registry/credentials/rotate",   h.RotateCredentials)
tenants.GET   ("/registry/pull-secret",          h.GetPullSecret)

engine.GET("/healthz", func(c *gin.Context) {
    c.JSON(200, gin.H{"status": "ok"})
})
engine.GET("/readyz", func(c *gin.Context) {
    if err := store.Ping(); err != nil {
        c.JSON(503, gin.H{"status": "db unavailable"})
        return
    }
    c.JSON(200, gin.H{"status": "ok"})
})
engine.GET("/metrics", gin.WrapH(promhttp.Handler()))
```

---

### Setup Timeline: What K8s state looks like after all steps

```
After kubectl apply -f k8s/namespace-rbac.yaml:
  namespace/registry-controller                    ← created
  serviceaccount/registry-provisioner              ← created in registry-controller
  clusterrole/registry-provisioner-role            ← created cluster-wide
  clusterrolebinding/registry-provisioner-binding  ← created cluster-wide

After kubectl create secret (×2):
  secret/registry-provisioner-db   ← in registry-controller
  secret/registry-master-key       ← in registry-controller

After kubectl apply -f k8s/cert-issuer.yaml:
  clusterissuer/letsencrypt-prod    ← cluster-wide
  clusterissuer/selfsigned-issuer   ← cluster-wide

After kubectl apply -f k8s/provisioner-deployment.yaml:
  configmap/registry-provisioner-config   ← in registry-controller
  deployment/registry-provisioner         ← in registry-controller (2 replicas)
  service/registry-provisioner            ← in registry-controller (ClusterIP)
  ingress/registry-provisioner            ← in registry-controller
  networkpolicy/provisioner-network-policy← in registry-controller
  pod/provisioner-pod-1                   ← RUNNING
  pod/provisioner-pod-2                   ← RUNNING
    ↓ each pod runs: HTTP goroutine + Worker goroutine
    ↓ /healthz → 200 OK
    ↓ /readyz  → 200 OK (DB connected)

After kubectl apply -f monitoring/:
  servicemonitor/registry-provisioner     ← in registry-controller
  prometheusrule/registry-provisioner     ← in registry-controller

System is now LIVE. Ready to accept tenant requests.
```

---

### Full Setup Flow in One Diagram

```
YOUR LAPTOP
───────────
│
├─ docker build  (backend/Dockerfile)
│  └─► wso2/registry-provisioner:latest  pushed to container registry
│
├─ psql < db/migrations/001_initial.sql
│  └─► platform PostgreSQL
│       registry_deployments table ✓
│       registry_credentials table ✓
│       audit_log table ✓
│
├─ kubectl apply -f k8s/namespace-rbac.yaml
│  └─► registry-controller namespace ✓
│       ServiceAccount ✓
│       ClusterRole (cross-tenant perms) ✓
│       ClusterRoleBinding ✓
│
├─ kubectl create secret (×2)
│  └─► Secret: registry-provisioner-db  (DATABASE_URL) ✓
│       Secret: registry-master-key     (AES key) ✓
│
├─ kubectl apply -f k8s/cert-issuer.yaml
│  └─► ClusterIssuer: letsencrypt-prod ✓
│       ClusterIssuer: selfsigned-issuer ✓
│
├─ kubectl apply -f k8s/provisioner-deployment.yaml
│  └─► ConfigMap ✓
│       Deployment → 2 pods start in registry-controller:
│         pod starts → cmd/main.go main() runs:
│           config.Load()         reads env vars from ConfigMap
│           db.New()              connects to platform PostgreSQL
│           store.Migrate()       runs CREATE TABLE IF NOT EXISTS
│           crypto.NewCipher()    loads master key from Secret mount
│           k8s.NewClient()       InClusterConfig → uses ServiceAccount token
│           helm.NewDeployer()    downloads Harbor chart index
│           go worker.Run()       starts 5-sec polling goroutine
│           httpServer.Serve()    starts HTTP on :8080
│         /healthz → 200 OK ✓
│         /readyz  → 200 OK ✓
│
└─ kubectl apply -f monitoring/
   └─► ServiceMonitor ✓   (Prometheus starts scraping /metrics)
        PrometheusRule ✓   (alerts configured)

SYSTEM IS LIVE
registry-api.lkdc.wso2.com is reachable
Worker goroutines are polling for PENDING jobs every 5 seconds
→ ready for Phase 1 (the button click)
```

---

## The Big Picture Flow

```
Physical servers → Harvester HCI → Rancher → namespaces
                                                  │
                   ┌──────────────────────────────┴──────────────────────────────┐
                   │ registry-controller namespace                                │
                   │   provisioner-pod-1     provisioner-pod-2                   │
                   │   [HTTP goroutine]       [HTTP goroutine]                    │
                   │   [Worker goroutine]     [Worker goroutine]                  │
                   └────────────────────────────────────────────────────────────┘
                              │  HTTP                     │  Worker
                              │                           │
User Browser                  │                           │
────────────                  │                           │
                              ▼                           ▼
[Create Registry] ──► handlers/registry.go     deploy_worker.go
    button            writes PENDING to DB     polls DB every 5s
                      ← 202 in < 10ms          picks up PENDING job
                                               runs 7-step deploy
                                                    │
                                    ┌───────────────▼────────────────────────┐
                                    │  acme-management namespace             │
                                    │  (Acme's existing management ns        │
                                    │   in their Harvester tenant space)     │
                                    │                                        │
                                    │  Step 1: ensure ns + NetworkPolicy     │
                                    │  Step 3: helm install → Harbor pods    │
                                    │    harbor-harbor-core                  │
                                    │    harbor-harbor-portal                │
                                    │    harbor-harbor-registry              │
                                    │    harbor-harbor-jobservice            │
                                    │    harbor-harbor-trivy                 │
                                    │    harbor-harbor-database              │
                                    │    harbor-harbor-redis                 │
                                    │    PVCs (50Gi + 5Gi + 1Gi)            │
                                    │    Ingress + TLS cert                  │
                                    │  Step 4: WaitForAllReady (watches here)│
                                    │  Step 5: bootstrap calls harbor-core   │
                                    └────────────────────────────────────────┘
```

---

## Phase 1: The Button Click

**Time: T+0s** | **Code file:** `handlers/registry.go` | **Pod:** provisioner-pod-1 | **Namespace:** registry-controller

```
Request path through the middleware stack:
  middleware/logging.go    → logs: "POST /api/v1/tenants/acme/registry"
  middleware/ratelimit.go  → checks 1 create / 10 min for this tenant
  middleware/auth.go       → validates JWT, extracts tenantId = "acme"
  middleware/auth.go       → TenantGuard: URL "acme" == JWT claim "acme" ✓
  handlers/registry.go     → Create() handler executes
```

```go
// FILE: backend/internal/api/handlers/registry.go  Create()
// POD:  provisioner-pod-1 (HTTP goroutine)
// NS:   registry-controller

tenantID  := c.Param("tenantId")          // "acme"
namespace := tenantID + "-management"     // "acme-management"
//            ↑ this is acme's EXISTING management namespace in Harvester
//              the provisioner will deploy Harbor INTO this namespace

helmRelease := "harbor-" + tenantID       // "harbor-acme"

// FILE: backend/internal/db/postgres.go  CreateDeployment()
// Writes a row to the PLATFORM PostgreSQL (not Harbor's DB, not in the cluster)
h.store.CreateDeployment(ctx, &db.RegistryDeployment{
    TenantID:    "acme",
    Namespace:   "acme-management",   // ← stored; worker will use this
    Status:      db.StatusPending,
    HelmRelease: "harbor-acme",
    Plan:        "professional",
})

// FILE: backend/internal/audit/logger.go  Log()
h.auditLog.Log(ctx, audit.Event{
    TenantID: "acme", Action: "REGISTRY_CREATE",
    ActorID: claims.UserID, Result: "SUCCESS",
})
// Returns 202 Accepted in < 10ms
```

**DB state after Phase 1:**
```sql
registry_deployments:
  tenant_id = 'acme'
  namespace = 'acme-management'    ← acme's existing management namespace
  status    = 'PENDING'
  plan      = 'professional'
```

---

## Phase 2: Worker Picks Up the Job

**Time: T+5s** | **Code file:** `worker/deploy_worker.go` | **Pod:** provisioner-pod-1 | **Namespace:** registry-controller

```go
// FILE: backend/internal/worker/deploy_worker.go  Run()
// POD:  provisioner-pod-1 (Worker goroutine)
// NS:   registry-controller

// Both pod-1 and pod-2 run this query at the same time.
// FOR UPDATE SKIP LOCKED ensures only ONE wins the job.

// FILE: backend/internal/db/postgres.go  GetOldestPending()
UPDATE registry_deployments
SET worker_lock = 'provisioner-pod-1',   // this pod's hostname
    status      = 'DEPLOYING'
WHERE tenant_id = (
    SELECT tenant_id FROM registry_deployments
    WHERE status = 'PENDING' AND worker_lock IS NULL
    ORDER BY created_at ASC LIMIT 1
    FOR UPDATE SKIP LOCKED   // provisioner-pod-2 skips this row entirely
)
RETURNING tenant_id, namespace, ...
// Returns: {TenantID:"acme", Namespace:"acme-management", Plan:"professional"}

// FILE: backend/internal/worker/deploy_worker.go  processPending()
go w.deploy(context.Background(), deployment)
// New goroutine — worker ticker keeps running for other tenants simultaneously
```

---

## Phase 3: The 7-Step Deploy

**All steps — provisioner code runs in registry-controller, resources land in acme-management**

---

### Step 1 — Ensure Management Namespace + NetworkPolicy (T+5s)

```
FILE: backend/internal/worker/deploy_worker.go   deploy()
FILE: backend/internal/k8s/client.go             EnsureNamespace(), ApplyNetworkPolicy()

Pod runs in:             provisioner-pod-1  |  registry-controller
Namespace it acts on:    acme-management  (acme's EXISTING management namespace in Harvester)
```

```go
// FILE: backend/internal/k8s/client.go  EnsureNamespace()
// acme-management already exists in Harvester — provisioner just ensures labels are applied
// (idempotent: uses Create with AlreadyExists check)
k8sClient.cs.CoreV1().Namespaces().Create(ctx, &corev1.Namespace{
    ObjectMeta: metav1.ObjectMeta{
        Name: "acme-management",        // ← acme's existing management namespace
        Labels: map[string]string{
            "lkdc.wso2.com/tenant":    "acme",
            "lkdc.wso2.com/component": "registry",
        },
    },
})
// if AlreadyExists → that's fine, continue

// FILE: backend/internal/k8s/client.go  ApplyNetworkPolicy()
// Applies deny-cross-tenant policy INSIDE acme-management:
//   ✅ ingress-nginx → harbor pods     (public HTTPS traffic in)
//   ✅ harbor pods   ↔ harbor pods     (internal calls within the namespace)
//   ✅ harbor pods   → internet        (Trivy CVE DB updates)
//   ❌ acme-management ↔ beta-management  (tenant isolation — blocked by policy)
k8sClient.cs.NetworkingV1().NetworkPolicies("acme-management").Create(...)

// FILE: backend/internal/db/postgres.go  UpdateProgress()
w.setProgress(ctx, tenantID, "namespace", "READY")
```

```
State after Step 1:
  acme-management namespace (already existed in Harvester)
    + label: lkdc.wso2.com/tenant=acme          ← added
    + networkpolicy/deny-cross-tenant             ← added, blocks cross-tenant traffic
```

---

### Step 2 — Generate Helm Values (T+6s)

```
FILE: backend/internal/worker/deploy_worker.go     deploy()
FILE: backend/internal/helm/values_generator.go    PlanFor(), GenerateValues()
FILE: backend/internal/crypto/aes.go               GeneratePassword()
FILE: helm/harbor-values-template.yaml             template (in repo, loaded at runtime)

Pod runs in:    provisioner-pod-1  |  registry-controller
No K8s changes in this step — only in-memory computation
```

```go
// FILE: backend/internal/helm/values_generator.go  PlanFor()
plan, _ := helm.PlanFor("professional")
// TenantPlan{RegistryStorage:"50Gi", DBStorage:"5Gi", CoreCPUReq:"200m", ...}

// FILE: backend/internal/crypto/aes.go  GeneratePassword()
adminPass, _ := crypto.GeneratePassword(32)  // random 32-char password
dbPass,    _ := crypto.GeneratePassword(32)  // different random password

// FILE: backend/internal/helm/values_generator.go  GenerateValues()
// reads harbor-values-template.yaml, fills in {{ }} placeholders
values, _ := helm.GenerateValues(helm.ValuesInput{
    TenantID:     "acme",
    AdminPass:    adminPass,
    DBPass:       dbPass,
    BaseDomain:   "lkdc.wso2.com",
    StorageClass: "longhorn",
    IngressClass: "nginx",
    CertIssuer:   "letsencrypt-prod",
    Plan:         plan,
})
```

**Rendered YAML stored in pod memory at `/tmp/helm-values/harbor-acme-XXXX.yaml`:**
```yaml
externalURL: https://registry.acme.lkdc.wso2.com
harborAdminPassword: "xK9!mP2@..."
expose:
  ingress:
    hosts:
      core: registry.acme.lkdc.wso2.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
persistence:
  persistentVolumeClaim:
    registry:
      storageClass: longhorn
      size: 50Gi           # professional plan
    database:
      size: 5Gi
```

---

### Step 3 — Helm Install (T+8s)

```
FILE: backend/internal/worker/deploy_worker.go   deploy()
FILE: backend/internal/helm/deployer.go          Install()

Pod runs in:          provisioner-pod-1  |  registry-controller
Creates resources in: acme-management  (acme's management namespace in Harvester)
Talks to:             K8s API server + https://helm.goharbor.io (chart download)
```

```go
// FILE: backend/internal/helm/deployer.go  Install()
// Uses Helm Go SDK (action.NewInstall) — no helm CLI needed in the pod
// Downloads harbor chart from helm.goharbor.io → /tmp/helm-charts/harbor/ (in pod)
w.helm.Install(ctx, "acme", "acme-management", values)
// helm install harbor-acme goharbor/harbor --namespace acme-management -f values.yaml
```

**What Helm creates INSIDE `acme-management` (acme's management namespace):**
```
deployment.apps/harbor-harbor-core          0/1   (Harbor REST API + auth)
deployment.apps/harbor-harbor-portal        0/1   (Web UI)
deployment.apps/harbor-harbor-registry      0/1   (OCI image blob store)
deployment.apps/harbor-harbor-jobservice    0/1   (background scan/GC jobs)
deployment.apps/harbor-harbor-trivy         0/1   (CVE vulnerability scanner)
statefulset.apps/harbor-harbor-database     0/1   (Harbor's own PostgreSQL)
statefulset.apps/harbor-harbor-redis-master 0/1   (Redis cache)

service/harbor-harbor-core      ClusterIP  (internal)
service/harbor-harbor-portal    ClusterIP  (internal)
service/harbor                  ClusterIP  (internal)

persistentvolumeclaim/harbor-harbor-registry   Pending→Bound  50Gi (Longhorn)
persistentvolumeclaim/harbor-harbor-database   Pending→Bound  5Gi  (Longhorn)
persistentvolumeclaim/harbor-harbor-redis      Pending→Bound  1Gi  (Longhorn)

ingress/harbor-harbor
  host:       registry.acme.lkdc.wso2.com → service/harbor-harbor-core
  annotation: cert-manager.io/cluster-issuer: letsencrypt-prod
              ↓ cert-manager (cluster-wide) sees this
              ↓ requests cert from Let's Encrypt for registry.acme.lkdc.wso2.com
              ↓ stores cert as secret/harbor-harbor-tls in acme-management
```

---

### Step 4 — Wait for Harbor Pods (T+8s → T+7m)

```
FILE: backend/internal/worker/deploy_worker.go   deploy()
FILE: backend/internal/k8s/client.go             WaitForAllReady(), AllDeploymentsReady()
FILE: backend/internal/db/postgres.go            UpdateProgress() (via progressCb)

Pod runs in:    provisioner-pod-1  |  registry-controller
Watches pods in: acme-management  (polls Deployment.Status every 10s via K8s API)
```

```go
// FILE: backend/internal/k8s/client.go  WaitForAllReady()
// Queries: GET /apis/apps/v1/namespaces/acme-management/deployments/harbor-harbor-core
// Checks: ReadyReplicas >= Replicas
err = w.k8s.WaitForAllReady(podCtx, "acme-management", func(status map[string]bool) {
    // FILE: backend/internal/db/postgres.go  UpdateProgress()
    // written every 10 seconds → UI sees live component-by-component progress
    progress["postgresql"] = "READY"
    progress["redis"]      = "STARTING"
    w.store.UpdateProgress(ctx, tenantID, progress)
})
```

```
Pods in acme-management during this phase:
  T+30s:  harbor-harbor-database-0        1/1 Running  ← postgresql: READY
          harbor-harbor-redis-master-0    1/1 Running  ← redis: READY
          harbor-harbor-core-abc123       0/1 Running  ← core: STARTING (DB migrations...)

  T+7m:   ALL 7 pods 1/1 Running → WaitForAllReady() returns nil
```

---

### Step 5 — Harbor Bootstrap (T+7m → T+9m)

```
FILE: backend/internal/worker/deploy_worker.go   deploy()
FILE: backend/internal/harbor/bootstrap.go       NewClient(), WaitReady(), Bootstrap()

Code runs in pod:  provisioner-pod-1  |  registry-controller
Calls Harbor API:  harbor-harbor-core pod  |  acme-management
Route:             provisioner-pod-1 → nginx Ingress → harbor-harbor-core (acme-management)
```

```go
// FILE: backend/internal/worker/deploy_worker.go
registryURL := "https://registry.acme.lkdc.wso2.com"
// This URL goes through: nginx Ingress → harbor-harbor-core service → harbor-core pod
// All of those are in: acme-management namespace (acme's Harvester management ns)

// FILE: backend/internal/harbor/bootstrap.go  NewClient()
harborClient := harbor.NewClient(registryURL, adminPass)

// FILE: backend/internal/harbor/bootstrap.go  WaitReady()
// polls GET https://registry.acme.lkdc.wso2.com/api/v2.0/ping  every 10s
// T+7m10s: 200 OK

// FILE: backend/internal/harbor/bootstrap.go  Bootstrap()
//   Configure()         PUT /api/v2.0/configurations
//                       Sets: auth_mode=db_auth, robot_token_duration=365, ...
//                       Runs on: harbor-harbor-core pod in acme-management
//
//   CreateProject()     POST /api/v2.0/projects
//                       Creates: {"project_name":"library","public":false}
//                       Runs on: harbor-harbor-core pod in acme-management
//
//   CreateRobotAccount() POST /api/v2.0/robots
//                        Creates: {name:"ci-default", permissions:[push,pull,scan,...]}
//                        Returns: {Name:"robot$ci-default", Secret:"eyJ...250chars..."}
//                        Runs on: harbor-harbor-core pod in acme-management
robot, _ := harborClient.Bootstrap(ctx)

// FILE: backend/internal/db/postgres.go  UpdateProgress()
w.setProgress(ctx, tenantID, "bootstrap", "READY")
```

---

### Step 6 — Encrypt and Store Credentials (T+9m)

```
FILE: backend/internal/worker/deploy_worker.go   deploy()
FILE: backend/internal/crypto/aes.go             EncryptString()
FILE: backend/internal/db/postgres.go            SaveCredentials()

Pod runs in:    provisioner-pod-1  |  registry-controller
Master key from: Secret registry-master-key (mounted into pod at /etc/secrets/master-key)
Writes to:      platform PostgreSQL  (outside K8s cluster)
```

```go
// FILE: backend/internal/crypto/aes.go  EncryptString()
encToken,  tokenNonce,  _ := w.cipher.EncryptString(robot.Secret)
// binary ciphertext — useless without the master key
encAdmin, adminNonce, _ := w.cipher.EncryptString(adminPass)

// FILE: backend/internal/db/postgres.go  SaveCredentials()
w.store.SaveCredentials(ctx, &db.RegistryCredentials{
    TenantID:         "acme",
    RobotUsername:    "robot$ci-default",
    EncryptedToken:   encToken,
    TokenNonce:       tokenNonce,
    EncryptedAdminPW: encAdmin,
    AdminPWNonce:     adminNonce,
})
```

---

### Step 7 — Mark READY (T+9m)

```
FILE: backend/internal/worker/deploy_worker.go   deploy()
FILE: backend/internal/db/postgres.go            SetDeploymentReady()
FILE: backend/internal/metrics/prometheus.go     ProvisionResult()

Pod runs in:  provisioner-pod-1  |  registry-controller
```

```go
// FILE: backend/internal/db/postgres.go  SetDeploymentReady()
w.store.SetDeploymentReady(ctx, "acme", "https://registry.acme.lkdc.wso2.com")
// UPDATE registry_deployments SET status='READY', registry_url=..., ready_at=now()

// FILE: backend/internal/metrics/prometheus.go
w.metrics.ProvisionResult("acme", "success")
timer.ObserveDuration()
// provision_duration_seconds{tenant="acme"} = 540.3
```

---

## Phase 4: Get Credentials

**Pod:** provisioner-pod-1 or pod-2 | **Namespace:** registry-controller

```go
// FILE: backend/internal/api/handlers/registry.go  GetCredentials()
creds, _ := h.store.GetCredentials(ctx, "acme")

// FILE: backend/internal/crypto/aes.go  DecryptString()
// decrypted in-memory only — never written anywhere
robotToken, _ := h.cipher.DecryptString(creds.EncryptedToken, creds.TokenNonce)

// FILE: backend/internal/audit/logger.go  Log()
h.auditLog.Log(ctx, audit.Event{
    TenantID: "acme", Action: "GET_CREDENTIALS",
    ActorID: claims.UserID, SourceIP: c.ClientIP(),
})
```

---

## Full Timeline: File → Pod → Namespace → Acts On

```
TIME     EVENT                         FILE                               POD (NS)                     ACTS ON
──────   ──────────────────────────    ─────────────────────────────────  ──────────────────────────── ─────────────────────────
T+0s     JWT validated                 middleware/auth.go                 provisioner-pod-1 (registry-controller)  —
T+0s     Rate limit checked            middleware/ratelimit.go            provisioner-pod-1 (registry-controller)  —
T+0s     DB INSERT PENDING             db/postgres.go CreateDeployment    provisioner-pod-1 (registry-controller)  platform PostgreSQL
T+0s     202 Accepted                  handlers/registry.go Create()      provisioner-pod-1 (registry-controller)  —

T+5s     FOR UPDATE SKIP LOCKED        db/postgres.go GetOldestPending    provisioner-pod-1 (registry-controller)  platform PostgreSQL
T+5s     go w.deploy() goroutine       worker/deploy_worker.go            provisioner-pod-1 (registry-controller)  —

T+5s     Ensure management namespace   k8s/client.go EnsureNamespace      provisioner-pod-1 (registry-controller)  acme-management (Harvester tenant ns)
T+5s     Apply NetworkPolicy           k8s/client.go ApplyNetworkPolicy   provisioner-pod-1 (registry-controller)  acme-management
T+6s     GeneratePassword ×2           crypto/aes.go                      provisioner-pod-1 (registry-controller)  —
T+6s     GenerateValues (fill template) helm/values_generator.go          provisioner-pod-1 (registry-controller)  —

T+8s     helm install harbor-acme      helm/deployer.go Install()         provisioner-pod-1 (registry-controller)  acme-management → creates 7 pods, PVCs, Ingress
T+8s     WaitForAllReady polling       k8s/client.go                      provisioner-pod-1 (registry-controller)  watches acme-management pods
T+30s    UpdateProgress postgresql:READY db/postgres.go                   provisioner-pod-1 (registry-controller)  platform PostgreSQL
T+7m     All pods in acme-management 1/1 Running                          —                                         acme-management

T+7m     WaitReady polling ping        harbor/bootstrap.go WaitReady      provisioner-pod-1 (registry-controller)  harbor-core pod (acme-management)
T+7m10s  /api/v2.0/ping → 200 OK       —                                  harbor-harbor-core (acme-management)      —
T+7m     Configure()                   harbor/bootstrap.go                provisioner-pod-1 (registry-controller)  harbor-core pod (acme-management)
T+8m     CreateProject()               harbor/bootstrap.go                provisioner-pod-1 (registry-controller)  harbor-core pod (acme-management)
T+8m     CreateRobotAccount()          harbor/bootstrap.go                provisioner-pod-1 (registry-controller)  harbor-core pod (acme-management)

T+9m     EncryptString ×2              crypto/aes.go                      provisioner-pod-1 (registry-controller)  in-memory only
T+9m     SaveCredentials               db/postgres.go                     provisioner-pod-1 (registry-controller)  platform PostgreSQL
T+9m     SetDeploymentReady            db/postgres.go                     provisioner-pod-1 (registry-controller)  platform PostgreSQL
T+9m     ProvisionResult metric        metrics/prometheus.go              provisioner-pod-1 (registry-controller)  —

T+9m05s  GET /registry/acme → READY   handlers/registry.go Get()         provisioner-pod-1 or -2 (registry-controller) platform PostgreSQL
T+9m30s  GET /credentials             handlers/registry.go GetCredentials provisioner-pod-1 or -2 (registry-controller) platform PostgreSQL + decrypt
T+9m30s  Audit: GET_CREDENTIALS       audit/logger.go                    provisioner-pod-1 or -2 (registry-controller)  platform PostgreSQL
```

---

## Where Each "Tenant" Thing Lives in Harvester

```
HARVESTER + RANCHER
════════════════════════════════════════════════════════════════════════

  Tenant: Acme Corp  (logically separated set of namespaces in Harvester)
  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  acme-management namespace  ← Acme's default/management namespace  │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │  After registry provisioned:                                 │   │
  │  │    harbor-harbor-core-xxx        (Harbor REST API)           │   │
  │  │    harbor-harbor-portal-xxx      (Harbor Web UI)             │   │
  │  │    harbor-harbor-registry-xxx    (image blob store)          │   │
  │  │    harbor-harbor-jobservice-xxx  (scan + GC jobs)            │   │
  │  │    harbor-harbor-trivy-xxx       (CVE scanner)               │   │
  │  │    harbor-harbor-database-0      (Harbor's PostgreSQL)       │   │
  │  │    harbor-harbor-redis-master-0  (Redis cache)               │   │
  │  │    PVC: 50Gi (images), 5Gi (DB), 1Gi (Redis) ← Longhorn      │   │
  │  │    Ingress: registry.acme.lkdc.wso2.com                    │   │
  │  │    Secret: harbor-harbor-tls (TLS cert from Let's Encrypt)   │   │
  │  │    NetworkPolicy: deny-cross-tenant                          │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  │                                                                     │
  │  acme-project-1 namespace  ← Acme creates for their app workloads  │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │  cluster-A (RKE2 K8s cluster inside Harvester VMs)           │   │
  │  │    pod: frontend    pod: backend-api                         │   │
  │  │  cluster-B                                                   │   │
  │  │    pod: worker-service                                       │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  │                                                                     │
  │  acme-project-2 namespace  ← Acme creates for staging               │
  │  ...                                                                │
  └─────────────────────────────────────────────────────────────────────┘

  Tenant: Beta Labs  (completely separate logically isolated space)
  ┌─────────────────────────────────────────────────────────────────────┐
  │  beta-management namespace  ← Beta's Harbor pods live here          │
  │  beta-project-1 namespace   ← Beta's app workloads                  │
  │  ...                                                                │
  └─────────────────────────────────────────────────────────────────────┘

  Controller space  (NOT a tenant — this is our provisioner)
  ┌─────────────────────────────────────────────────────────────────────┐
  │  registry-controller namespace                                      │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │  provisioner-pod-1   provisioner-pod-2                       │   │
  │  │  Has ClusterRole → can reach into ANY tenant's management ns  │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## 🏋️ Exercises

### Exercise 1 — Tenant vs Namespace
Explain in one sentence each:
- What is a tenant in Harvester terms?
- What is a namespace in Kubernetes terms?
- Why does a tenant have MORE than one namespace?
- Which namespace does the provisioner deploy Harbor into and why?

### Exercise 2 — Follow the Code for One Request
For `POST /api/v1/tenants/acme/registry`:
1. Which file validates the JWT? Which namespace is that code in?
2. Which file writes to the DB? What table?
3. Which file actually creates Harbor pods? In which namespace does it create them?
4. Which file calls the Harbor REST API? Which namespace does the API run in?

### Exercise 3 — The HA Race Condition
Two provisioner pods both run `GetOldestPending()` at the same millisecond:
- Which file contains this query?
- Which SQL keyword prevents double-deployment?
- Which column in the DB records which pod won?
- What happens to provisioner-pod-2 after pod-1 wins?

### Exercise 4 — Namespace Boundary Tracing
For each action below, answer: which namespace does the code run in, and which namespace does it act on?
- `EnsureNamespace()` in `k8s/client.go`
- `Install()` in `helm/deployer.go`
- `WaitForAllReady()` in `k8s/client.go`
- `Bootstrap()` in `harbor/bootstrap.go`
- `SaveCredentials()` in `db/postgres.go`

### Exercise 5 — docker push end-to-end
Starting from `docker push registry.acme.lkdc.wso2.com/library/myapp:v1`:
- Which K8s object receives the request first? In which namespace?
- Which Harbor pod actually stores the image layers? In which namespace?
- Which Harbor pod triggers the Trivy scan? In which namespace?
- Can anyone in `beta-management` see this image? Why not?
