# 05 — Helm: Kubernetes Package Manager

> **Goal:** Understand what Helm is, how charts and values work, and how this system installs a full Harbor stack with one function call.

---

## 1. The Problem Helm Solves

Deploying Harbor to Kubernetes manually requires creating ~25 YAML files:
- Deployments for core, registry, portal, jobservice, trivy, nginx
- Services for each
- ConfigMaps, Secrets
- PersistentVolumeClaims
- Ingress rules
- ServiceAccounts, RBAC
- ...and they all need to reference each other correctly

**Helm turns all of this into one command:**

```bash
helm install harbor-tenantA harbor/harbor \
  --namespace tenant-a-management \
  --values my-tenant-values.yaml
```

Think of Helm like **apt** or **npm** — but for Kubernetes apps.

---

## 2. Key Concepts

```
Chart           = a package (all the YAML templates for an app)
Release         = a specific installation of a chart
Values          = configuration injected into the templates
Repository      = a server hosting charts (like npm registry)

harbor/harbor   = chart "harbor" from repository "harbor"
harbor-tenantA  = release name (our installation's unique name)
```

```
Helm Chart: harbor/harbor
┌─────────────────────────────────────────────────┐
│  Chart.yaml           (metadata: name, version) │
│  values.yaml          (default configuration)   │
│  templates/           (parameterised YAML)       │
│    core-deployment.yaml                         │
│    registry-deployment.yaml                     │
│    portal-deployment.yaml                       │
│    ...24 more files...                          │
└─────────────────────────────────────────────────┘
          │
          │  helm install --values custom.yaml
          ▼
Rendered YAML (templates + your values merged)
          │
          ▼
kubectl apply (Helm sends to Kubernetes API)
          │
          ▼
Kubernetes creates all the objects
```

---

## 3. Values and Templates

A Helm template is a YAML file with `{{ .Values.something }}` placeholders:

```yaml
# templates/core-deployment.yaml (simplified)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: harbor-core
  namespace: {{ .Release.Namespace }}
spec:
  containers:
    - name: core
      image: goharbor/harbor-core:{{ .Chart.AppVersion }}
      resources:
        requests:
          cpu: {{ .Values.core.resources.requests.cpu }}
          memory: {{ .Values.core.resources.requests.memory }}
```

Your `values.yaml` provides the actual values:

```yaml
# your custom values (what this system generates per-tenant)
core:
  resources:
    requests:
      cpu: 200m       # ← professional plan
      memory: 256Mi
```

Result after rendering:
```yaml
spec:
  containers:
    - name: core
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
```

---

## 4. How This System Uses Helm

The system uses the **Helm Go SDK** — not the `helm` CLI — to install/upgrade/uninstall Harbor programmatically from Go code.

### The Deployer

```go
// From backend/internal/helm/deployer.go

type Deployer struct {
    cfg    config.HelmConfig   // harbor repo URL, chart version, etc.
    logger *zap.Logger
    env    *cli.EnvSettings    // Helm's environment (like ~/.helm)
}
```

### Installing Harbor

```go
// Simplified from deployer.go
func (d *Deployer) Install(ctx context.Context, tenantID, namespace string, values []byte) error {
    // 1. Get Helm action config (knows how to talk to K8s)
    cfg, _ := d.actionConfig(namespace)

    // 2. Write values to a temp file (contains passwords!)
    valuesPath, cleanup, _ := writeTempValues(tenantID, values)
    defer cleanup()  // ← always delete the temp file

    // 3. Load the Harbor chart from /tmp/helm-charts/harbor
    chart, _ := d.loadChart()

    // 4. Parse values YAML into a map
    vals, _ := parseValuesFile(valuesPath)

    // 5. Run the install
    install := action.NewInstall(cfg)
    install.ReleaseName = "harbor-" + tenantID  // e.g. "harbor-tenant-a"
    install.Namespace = namespace
    install.CreateNamespace = false             // we already created it in Step 1
    install.Wait = false                        // don't block; we poll pods ourselves
    install.Timeout = 10 * time.Minute

    install.RunWithContext(ctx, chart, vals)
    return nil
}
```

### The Release Name Pattern

Each tenant's Harbor release is named `harbor-{tenantID}`:

```
tenant: acme-corp
  release name: harbor-acme-corp
  namespace:    acme-corp-management

tenant: startup-xyz
  release name: harbor-startup-xyz
  namespace:    startup-xyz-management
```

This lets Helm track each installation independently. The function:

```go
func releaseName(tenantID string) string {
    return fmt.Sprintf("harbor-%s", tenantID)
}
```

---

## 5. The Values Generator

This is where the system customizes Harbor per tenant. Look at [backend/internal/helm/values_generator.go](../backend/internal/helm/values_generator.go):

```go
// Three plans with different resource allocations
var plans = map[string]TenantPlan{
    "starter": {
        RegistryStorage: "20Gi", DBStorage: "2Gi",
        CoreCPUReq: "100m", CoreMemReq: "128Mi",
    },
    "professional": {
        RegistryStorage: "50Gi", DBStorage: "5Gi",
        CoreCPUReq: "200m", CoreMemReq: "256Mi",
    },
    "enterprise": {
        RegistryStorage: "200Gi", DBStorage: "10Gi",
        CoreCPUReq: "500m", CoreMemReq: "512Mi",
    },
}
```

The template renders these into Harbor values:

```go
const harborValuesTemplate = `
expose:
  ingress:
    hosts:
      core: registry.{{.TenantID}}.{{.BaseDomain}}

harborAdminPassword: "{{.AdminPass}}"

persistence:
  persistentVolumeClaim:
    registry:
      size: {{.Plan.RegistryStorage}}    # ← 20Gi / 50Gi / 200Gi
    database:
      size: {{.Plan.DBStorage}}

core:
  resources:
    requests:
      cpu: {{.Plan.CoreCPUReq}}
      memory: {{.Plan.CoreMemReq}}
`
```

Final generated YAML for a `professional` plan tenant named `acme`:

```yaml
expose:
  ingress:
    hosts:
      core: registry.acme.lkdc.wso2.com

harborAdminPassword: "xK9!mP2@qR5#nL8$"    ← randomly generated

persistence:
  persistentVolumeClaim:
    registry:
      size: 50Gi
    database:
      size: 5Gi

core:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
```

---

## 6. Chart Loading — Pull and Cache

The Helm chart is downloaded from the Harbor chart repository before installing:

```go
func (d *Deployer) loadChart() (*chart.Chart, error) {
    pull := action.NewPullWithOpts(action.WithConfig(&action.Configuration{}))
    pull.RepoURL = d.cfg.HarborRepoURL  // https://helm.goharbor.io
    pull.Version = d.cfg.HarborChartVer // 1.14.0
    pull.DestDir = "/tmp/helm-charts"
    pull.Untar = true

    pull.Run("harbor/harbor")
    // Downloads harbor-1.14.0.tgz → extracts to /tmp/helm-charts/harbor/

    return loader.Load("/tmp/helm-charts/harbor")
    // Returns a *chart.Chart struct with all templates and default values
}
```

```
https://helm.goharbor.io/harbor-1.14.0.tgz
        │
        ▼ download + extract
/tmp/helm-charts/harbor/
  ├── Chart.yaml
  ├── values.yaml        ← Harbor's defaults (we override these)
  ├── templates/
  │   ├── core/
  │   ├── registry/
  │   ├── jobservice/
  │   └── ...
  └── charts/
      └── postgresql/    ← sub-charts (PostgreSQL, Redis)
```

In the K8s Deployment, `/tmp/helm-charts` is an `emptyDir` volume — it resets on pod restart, but the chart is re-downloaded quickly.

---

## 7. Action Config — Connecting to Kubernetes

The Helm SDK needs to know how to talk to the Kubernetes cluster. This is handled by `actionConfig`:

```go
func (d *Deployer) actionConfig(namespace string) (*action.Configuration, error) {
    cfg := new(action.Configuration)

    // ConfigFlags auto-reads KUBECONFIG env var or in-cluster service account
    configFlags := genericclioptions.NewConfigFlags(true)
    configFlags.Namespace = &namespace

    cfg.Init(configFlags, namespace, "secret", func(format string, v ...interface{}) {
        d.logger.Debug(fmt.Sprintf(format, v...))
        //         ^ helm's own logs go to our zap logger
    })
    return cfg, nil
}
```

The `"secret"` parameter means Helm stores its release metadata in Kubernetes **Secrets** (inside the target namespace). Each Helm release creates a secret like:

```
sh.helm.release.v1.harbor-tenant-a.v1   ← first install
sh.helm.release.v1.harbor-tenant-a.v2   ← after first upgrade
```

This is how `helm status` and `helm history` work — they read these secrets.

---

## 8. Upgrade and Uninstall

```go
// Upgrade — called when tenant changes plan
func (d *Deployer) Upgrade(ctx context.Context, tenantID, namespace string, values []byte) error {
    upgrade := action.NewUpgrade(cfg)
    upgrade.ReuseValues = true    // ← keep existing values, only override what we pass
    upgrade.Wait = false
    upgrade.Timeout = 10 * time.Minute
    upgrade.RunWithContext(ctx, releaseName(tenantID), chart, vals)
}

// Uninstall — called on soft delete (keeps PVCs)
func (d *Deployer) Uninstall(ctx context.Context, tenantID, namespace string) error {
    uninstall := action.NewUninstall(cfg)
    uninstall.Wait = false
    uninstall.IgnoreNotFound = true  // ← OK if already deleted
    uninstall.Run(releaseName(tenantID))
}
```

After `Uninstall`:
- All Kubernetes objects created by Helm are deleted (Deployments, Services, etc.)
- But **PVCs are kept** because the Harbor chart has `resourcePolicy: keep`
- This means image data is preserved and can be recovered if needed

---

## 9. Helm Release States

```
helm install  →  deployed
                    │
helm upgrade  →  deployed (new revision)
                    │
helm rollback →  deployed (previous revision)
                    │
helm uninstall → (deleted — release history removed)

Failed states:
  failed      (install/upgrade failed partway)
  pending-install
  pending-upgrade
```

```go
// ReleaseStatus — used to check if Harbor is deployed
func (d *Deployer) ReleaseStatus(ctx context.Context, tenantID, namespace string) (string, error) {
    status := action.NewStatus(cfg)
    rel, _ := status.Run(releaseName(tenantID))
    return rel.Info.Status.String(), nil  // "deployed", "failed", etc.
}
```

---

## 🏋️ Exercises

### Exercise 1 — Read the Values Template
Open [backend/internal/helm/values_generator.go](../backend/internal/helm/values_generator.go).

Run the template mentally for `tenantID="my-org"`, `plan="starter"`, `baseDomain="example.com"`:
- What will the Harbor URL be?
- What registry storage size?
- What PostgreSQL storage size?

### Exercise 2 — Understand Resource Requests vs Limits
In the values template:
```yaml
core:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
```
- `200m` CPU = 0.2 CPU cores. What does "limit 1000m" mean?
- `256Mi` memory request — what happens if the container tries to use more than `1Gi`?
- Why set a request lower than a limit?

### Exercise 3 — Trace the Install Flow
Follow the code from `worker/deploy_worker.go` step 3 through `helm/deployer.go`:
1. Where does `Install()` get called in `deploy_worker.go`?
2. What is `values []byte`? How was it created?
3. Why does `writeTempValues` use `os.CreateTemp` instead of a fixed path?
4. Why is `defer cleanup()` important?

### Exercise 4 — Simulate a Values Render
Write a small Go program (or just trace mentally) that calls `GenerateValues` with:
- TenantID: "test-corp"
- Plan: enterprise
- BaseDomain: "cloud.example.com"
- AdminPass: "hunter2"
- DBPass: "password123"

What will the ingress host be? What storage size?

### Exercise 5 — Why `install.Wait = false`?
The code sets `install.Wait = false`. This means Helm returns immediately after creating objects, before pods are ready.

But the worker **does** wait — in Step 4. Why do it this way instead of `install.Wait = true`?
Hint: look at `WaitForAllReady` in `k8s/client.go` and notice the `progressCb` callback.

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Chart** | A package of K8s templates for an app |
| **Release** | One installation of a chart |
| **Values** | Configuration that fills in template placeholders |
| **Repository** | Server hosting charts (like npm registry) |
| **Action** | A Helm operation: Install, Upgrade, Uninstall, Status |
| **Release name** | Unique identifier: `harbor-{tenantID}` |
| **emptyDir** | K8s volume that resets on pod restart (used for chart cache) |
| **Memory emptyDir** | emptyDir stored in RAM — used for sensitive values files |
| **resourcePolicy: keep** | Tells Helm not to delete PVCs on uninstall |

**Next:** [06 — Go Language & REST APIs →](./06-go-language-and-rest-api.md)
