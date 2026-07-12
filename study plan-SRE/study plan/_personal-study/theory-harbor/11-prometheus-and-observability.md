# 11 — Prometheus & Observability

> **Goal:** Understand what observability is, how Prometheus metrics work, how the audit log provides accountability, and what the monitoring stack in this system looks like.

---

## 1. What Is Observability?

Observability answers: "What is my system doing right now, and what went wrong?"

The three pillars:

```
LOGS          METRICS           TRACES
─────         ───────           ──────
"What         "How many?"       "Where did time go?"
happened?"    "How fast?"
              "Any errors?"

Example:      Example:          Example:
[ERROR]       provision_total   Request → DB → K8s → Helm
tenant-a      counter{          (shows 8s was spent in Helm)
helm failed   result="failure"}
              = 3
```

This system implements **logs** (via Zap) and **metrics** (via Prometheus). Traces are not implemented.

---

## 2. Prometheus — Pull-Based Metrics

Prometheus is a **time-series database** that scrapes metrics from your app on a schedule:

```
Provisioner App                    Prometheus
───────────────                    ──────────
/metrics endpoint                  every 30s:
  EXPOSES current metrics     ◄────  GET http://provisioner/metrics
                                     stores: (metric_name, labels, value, timestamp)
```

The `/metrics` endpoint returns plain text:

```
# HELP registry_provisions_total Total number of provision attempts
# TYPE registry_provisions_total counter
registry_provisions_total{result="success"} 42
registry_provisions_total{result="failure"} 3

# HELP registry_provision_duration_seconds Time to provision Harbor
# TYPE registry_provision_duration_seconds histogram
registry_provision_duration_seconds_bucket{tenant="acme",le="60"} 0
registry_provision_duration_seconds_bucket{tenant="acme",le="300"} 1
registry_provision_duration_seconds_bucket{tenant="acme",le="600"} 1
registry_provision_duration_seconds_sum{tenant="acme"} 423.5
registry_provision_duration_seconds_count{tenant="acme"} 1
```

---

## 3. Metric Types

### Counter — only goes up

```go
// Something that accumulates over time
provisions_total counter

Uses: request count, error count, bytes sent, jobs completed
// Can be rate()-d to get "per second"
rate(provisions_total[5m])  // provisions per second over last 5 minutes
```

### Gauge — can go up or down

```go
// A current value at a point in time
active_deployments gauge

Uses: queue depth, memory usage, temperature, current connections
```

### Histogram — distribution of values

```go
// Samples values into buckets — good for durations
provision_duration_seconds histogram

// Stores:
//   _bucket{le="60"}  = how many took ≤ 60 seconds
//   _bucket{le="300"} = how many took ≤ 300 seconds
//   _sum              = total of all observed values
//   _count            = number of observations

// Query:
histogram_quantile(0.95, rate(provision_duration_seconds_bucket[1h]))
// → 95th percentile: 95% of deployments finish within X seconds
```

---

## 4. This System's Metrics

```go
// From backend/internal/metrics/prometheus.go

type Registry struct {
    // Counter: how many provisions total
    provisionsTotal *prometheus.CounterVec
    // Labels: result="success"|"failure", tenant="..."

    // Histogram: how long does provisioning take?
    provisionDuration *prometheus.HistogramVec
    // Labels: tenant="..."

    // Gauge: how many deployments are in each state?
    deploymentsActive *prometheus.GaugeVec
    // Labels: status="PENDING"|"DEPLOYING"|"READY"|"FAILED"

    // Counter: credential rotation events
    credentialRotations *prometheus.CounterVec

    // Counter: API requests
    apiRequests *prometheus.CounterVec
    // Labels: method="POST"|"GET", path="/registry", status="200"|"500"
}
```

### How the Worker Uses Metrics

```go
// Start timing a provision
func (r *Registry) StartProvisionTimer(tenantID string) *prometheus.Timer {
    return prometheus.NewTimer(
        r.provisionDuration.WithLabelValues(tenantID),
    )
}

// In the worker:
timer := w.metrics.StartProvisionTimer(d.TenantID)
defer timer.ObserveDuration()   // records duration when deploy() returns
//     ^ called automatically, whether success or failure

// Record the result:
w.metrics.ProvisionResult(tenantID, "success")
// → increments registry_provisions_total{result="success",tenant="..."}
```

---

## 5. Labels — Adding Dimensions

Labels let you slice and dice metrics:

```
registry_provisions_total{result="success"} 40
registry_provisions_total{result="failure"} 5

# Query: success rate
rate(registry_provisions_total{result="success"}[1h])
  /
rate(registry_provisions_total[1h])
→ 88.9% success rate
```

```
registry_provision_duration_seconds_count{tenant="acme"}    = 10
registry_provision_duration_seconds_count{tenant="startup"} = 3

# Query: is tenant "startup" slower than average?
histogram_quantile(0.50, rate(provision_duration_seconds_bucket{tenant="startup"}[24h]))
```

---

## 6. ServiceMonitor — K8s Prometheus Integration

In production, Prometheus doesn't need to be configured manually. A `ServiceMonitor` custom resource tells Prometheus Operator which services to scrape:

```yaml
# From monitoring/service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: registry-provisioner
  namespace: registry-controller
spec:
  selector:
    matchLabels:
      app: registry-provisioner          # ← scrape pods with this label
  endpoints:
    - port: http
      path: /metrics
      interval: 30s                      # ← every 30 seconds
```

```
Prometheus Operator sees ServiceMonitor
  → automatically adds a scrape job for registry-provisioner pods
  → no manual prometheus.yml editing needed
```

There's also a ServiceMonitor for Harbor itself (per-tenant):

```yaml
# Harbor exposes its own /metrics endpoint
# Prometheus scrapes each tenant's Harbor for:
#   - push/pull counts
#   - storage usage
#   - scan results
```

---

## 7. Prometheus Alerts

Alerts fire when a metric crosses a threshold:

```yaml
# From monitoring/prometheus-rules.yaml

groups:
  - name: registry-provisioner
    rules:

      # Alert if provisioning fails more than 3 times in 1 hour
      - alert: RegistryProvisionFailureRate
        expr: |
          rate(registry_provisions_total{result="failure"}[1h]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High provision failure rate"
          description: "More than 5% of provisions failing"

      # Alert if provisioning takes too long
      - alert: RegistryProvisionSlow
        expr: |
          histogram_quantile(0.95,
            rate(registry_provision_duration_seconds_bucket[1h])
          ) > 900
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Provisioning is slow (p95 > 15 min)"
```

```
Alert lifecycle:
  metric crosses threshold  → state: PENDING (for: 5m)
  still above after 5 min  → state: FIRING
  metric drops below        → state: RESOLVED

Alertmanager routes alerts to:
  Slack / PagerDuty / Email
```

---

## 8. Audit Log — Accountability

The audit log is different from metrics — it records **what happened** (not counts/durations), **who did it**, and **when**:

```go
// From backend/internal/audit/logger.go

type Logger struct {
    store  *db.Store
    logger *zap.Logger
}

func (l *Logger) Log(ctx context.Context, entry *db.AuditEntry) {
    // Non-blocking: write audit to DB in a goroutine
    // If it fails, log the error but don't fail the request
    go func() {
        if err := l.store.WriteAuditLog(ctx, entry); err != nil {
            l.logger.Error("failed to write audit log", zap.Error(err))
        }
    }()
}
```

### Audit Entry Structure

```go
type AuditEntry struct {
    TenantID   string                 // "acme-corp"
    Action     string                 // "registry.create"
    ActorID    string                 // JWT sub: "user-abc-123"
    ActorEmail string                 // "sandaruw@wso2.com"
    SourceIP   string                 // "203.0.113.42"
    Result     string                 // "success" or "failure"
    Details    map[string]interface{} // {"plan": "professional"}
    CreatedAt  time.Time
}
```

### Sample Audit Log Entries

```sql
SELECT actor_email, action, result, created_at, details
FROM audit_log
WHERE tenant_id = 'acme-corp'
ORDER BY created_at DESC;

 actor_email           | action                    | result  | created_at
-----------------------+---------------------------+---------+--------------------
 sandaruw@wso2.com     | registry.create           | success | 2024-01-15 10:00
 sandaruw@wso2.com     | credentials.rotate        | success | 2024-01-15 14:30
 admin@wso2.com        | registry.delete           | failure | 2024-01-15 18:00
                       |                           |         | (tenant mismatch)
```

### Why Non-Blocking?

The audit log write runs in a goroutine — it doesn't hold up the HTTP response:

```
Without non-blocking:
  Handler → save to DB → write audit → return response
  If audit DB is slow: user waits 2 extra seconds for every request!

With non-blocking goroutine:
  Handler → save to DB → return response (immediately)
            ↓ (concurrently)
            write audit to DB (happens in background)
  User doesn't wait. If audit write fails: error logged, request still succeeded.
```

---

## 9. Grafana Dashboards (Conceptual)

In production, Grafana visualises Prometheus data:

```
┌─────────────────────────────────────────────────────────────┐
│              Registry Provisioner Dashboard                 │
├───────────────────────┬─────────────────────────────────────┤
│  Provision Success    │  Provision Duration (p50/p95)       │
│  Rate: 94.2%          │  p50: 4m32s  p95: 8m15s            │
├───────────────────────┼─────────────────────────────────────┤
│  Active Deployments   │  API Request Rate                   │
│  PENDING: 2           │  200: 145/min                       │
│  DEPLOYING: 3         │  429: 2/min (rate limited)          │
│  READY: 87            │  500: 0/min ✓                       │
│  FAILED: 1            │                                     │
└───────────────────────┴─────────────────────────────────────┘
```

---

## 🏋️ Exercises

### Exercise 1 — View Live Metrics
```bash
# Hit the metrics endpoint on the running provisioner
curl -s http://localhost:8080/metrics | grep registry_

# What metrics are exposed?
# What are their current values? (most will be 0 since nothing deployed)
```

### Exercise 2 — Understand the Histogram
The code registers histogram buckets:
```go
Buckets: []float64{30, 60, 120, 300, 600, 900}
```
These represent seconds. What do they mean?
- `le="300"` = "less than or equal to 300 seconds"
- A provision that takes 7 minutes (420s) goes in which bucket?
- How would you calculate the 50th percentile from these buckets?

### Exercise 3 — Write an Audit Log Query
Using the audit log table, write SQL to answer:
1. "How many registries were created this week?"
2. "Which user deleted the most registries?"
3. "Show all failed operations"

```sql
-- Q1: Registries created this week
SELECT COUNT(*) FROM audit_log
WHERE action = ??? AND result = ??? AND created_at > ???;
```

### Exercise 4 — Design a New Metric
The system doesn't track how many credentials rotations happen per day.
Add a metric for this:
- What type? (Counter, Gauge, or Histogram)
- What labels would you add?
- Where in the code would you increment it?

### Exercise 5 — Trace a Request Through Logs + Metrics
```bash
# Make an API call
curl -s http://localhost:8080/api/v1/tenants/dev-tenant/registry

# Check the logs
docker logs implementation-of-auto-harbor-deployment-provisioner-1 --tail 10

# What does the RequestLogger middleware log?
# What metrics would this increment?
```

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **Metrics** | Numerical measurements over time |
| **Counter** | Value that only increases (`provisions_total`) |
| **Gauge** | Current value (`active_deployments`) |
| **Histogram** | Distribution of values (`provision_duration_seconds`) |
| **Labels** | Key=value pairs on metrics — enable filtering/grouping |
| **ServiceMonitor** | K8s CRD — tells Prometheus Operator what to scrape |
| **Alert** | Rule that fires when a PromQL expression is true for N minutes |
| **Audit log** | Append-only record of who did what (accountability) |
| **Non-blocking audit** | Audit writes in a goroutine — doesn't slow down requests |
| **p95 latency** | 95% of requests complete within this time |

**Next:** [12 — Network Policies & Tenant Isolation →](./12-network-policies-and-security.md)
