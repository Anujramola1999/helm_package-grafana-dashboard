# Kyverno Grafana Dashboard Installation Guide

## Prerequisites

Install kube-prometheus-stack for Prometheus and Grafana with sidecar enabled (default configuration):

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

Wait for all monitoring pods to be running before proceeding.

---

## Step 1: Install Kyverno with Monitoring (Recommended Method)

Install Kyverno using the packaged chart with monitoring configuration using the included `monitoring-values.yaml` file.

### Why use `n4k-kyverno` as the release name?

**Important:** The Grafana dashboards in this package are configured to query metrics with `job="n4k-kyverno-svc-metrics"`. The Helm release name directly creates Kubernetes service names, which become the Prometheus job labels.

Using `n4k-kyverno` as the release name:
- Creates the service `n4k-kyverno-svc-metrics` automatically
- This service name becomes the Prometheus job label `job="n4k-kyverno-svc-metrics"`
- Dashboards work without any configuration changes

**Alternative (not recommended):** You could use a different release name and manually configure ServiceMonitor `relabelings` to override the job label, but this adds unnecessary complexity.

### Understanding monitoring-values.yaml

The `monitoring-values.yaml` file in this repository contains:

```yaml
# Kyverno Monitoring Configuration
# Enable Grafana dashboards, Prometheus ServiceMonitors, and Reports Server with ETCD

crds:
  reportsServer:
    enabled: true

grafana:
  enabled: true

admissionController:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: monitoring

backgroundController:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: monitoring

cleanupController:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: monitoring

reportsController:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: monitoring

# Enable Reports Server with ETCD backend (ETCD is enabled by default with 2Gi PVC and ~1.8GiB quota)
reports-server:
  install: true
  metrics:
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: monitoring
```

**Note:** The `release: monitoring` label is required for Prometheus to discover ServiceMonitors. To find the correct release name for your Prometheus installation, run:

```bash
kubectl get prometheus -o yaml -n monitoring | grep -A 20 serviceMonitorSelector
```

Or use this simpler command to see all monitor-related configuration:

```bash
kubectl get prometheus -o yaml | grep monitor
```

Look for the `matchLabels` section under `serviceMonitorSelector`:

```yaml
serviceMonitorSelector:
  matchLabels:
    release: monitoring
```

Use the value of the `release` label in your monitoring-values.yaml file.

### Install Kyverno with monitoring enabled:

```bash
helm install n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace -f monitoring-values.yaml
```

This single command deploys Kyverno with:
- ✅ **4 Grafana Dashboards** - Auto-discovered by Grafana sidecar
- ✅ **4 Kyverno Controller ServiceMonitors** - For admission, background, cleanup, reports controllers
- ✅ **Reports Server with ETCD** - 3-node ETCD cluster with 2Gi PVC and ~1.8GiB quota
- ✅ **Reports Server ServiceMonitor** - For reports-server application metrics

The release name `n4k-kyverno` creates service names prefixed with `n4k-kyverno-`, making them easily identifiable in Prometheus.

### Apply ETCD ServiceMonitor for ETCD storage metrics:

```bash
kubectl apply -f etcd-servicemonitor.yaml
```

This enables the ETCD Storage Observability dashboard to display ETCD cluster health, database size, and performance metrics.

---

## Step 2: Install Policy Reporter

Policy Reporter collects and exposes policy compliance metrics required for the Policy Reports dashboard.

### Add Policy Reporter Helm repository:

```bash
helm repo add policy-reporter https://kyverno.github.io/policy-reporter
helm repo update
```

### Install Policy Reporter with metrics and monitoring enabled:

```bash
helm install policy-reporter policy-reporter/policy-reporter \
  -n kyverno \
  --set metrics.enabled=true \
  --set monitoring.enabled=true \
  --set monitoring.serviceMonitor.namespace=kyverno \
  --set monitoring.serviceMonitor.labels.release=monitoring \
  --set monitoring.grafana.dashboards.enable.overview=true \
  --set monitoring.grafana.dashboards.enable.policyReportDetails=false \
  --set monitoring.grafana.dashboards.enable.clusterPolicyReportDetails=false
```

This creates:
- Policy Reporter deployment in kyverno namespace to watch PolicyReports and ClusterPolicyReports
- ServiceMonitor in kyverno namespace with `release: monitoring` label for Prometheus discovery
- Metrics endpoint at `/metrics` for policy compliance data
- Only the "PolicyReports" dashboard from Policy Reporter OSS (overview dashboard showing failing policies by namespace)

**Important:** We selectively enable only the "PolicyReports" dashboard from Policy Reporter OSS (by setting `overview=true`) while disabling "PolicyReport Details" and "ClusterPolicyReport Details" dashboards to avoid duplication. This OSS dashboard complements our custom "PolicyReports Compliance Dashboard" which focuses on compliance metrics and trends.

**Note:** Installing Policy Reporter in the same namespace as Kyverno keeps all monitoring resources together.

---

## Step 3: Apply Prometheus Recording Rules

Apply recording rules for efficient policy metric aggregation:

```bash
kubectl apply -f policy-reporter-prometheus-rules.yaml
```

Restart Prometheus to load the new configuration:

```bash
kubectl rollout restart statefulset -n monitoring prometheus-monitoring-kube-prometheus-prometheus
kubectl wait --for=condition=ready pod -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 --timeout=120s
```

---

## Verification

### Check ServiceMonitor was created:

```bash
kubectl get servicemonitor -n monitoring | grep kyverno
```

### Check ConfigMap with dashboards was created:

```bash
kubectl get configmap -n kyverno | grep grafana
```

### Verify ConfigMap contains both dashboards:

```bash
kubectl get configmap kyverno-grafana-grafana -n kyverno -o yaml | grep "\.json:"
```

Expected output shows:
- `kyverno-dashboard.json`
- `kyverno-performance-metrics.json`
- `kyverno-policy-reports.json`

---

## Accessing Prometheus

Port forward Prometheus service:

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
```

Access Prometheus UI at http://localhost:9090

Navigate to **Status > Targets** and search for "kyverno" to verify metrics are being scraped. All 4 endpoints should show status **UP**.

---

## Accessing Grafana

### Get Grafana admin password:

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

### Port forward Grafana service:

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Access Grafana UI at http://localhost:3000

Login with username **admin** and the password from above command.

Navigate to **Dashboards** to see the auto-imported Kyverno dashboards.

---

## What Gets Deployed

Four Grafana dashboards are automatically available (five if reports-server with ETCD is enabled):

### 1. Kyverno Metrics Dashboard (Custom)
Shows policy execution, admission requests, policy changes, and latency metrics from Kyverno controllers.

### 2. Kyverno Performance Metrics Dashboard (Custom)
Shows CPU/memory usage, admission review latency, policy execution performance, and resource utilization of Kyverno components.

### 3. PolicyReports Compliance Dashboard (Custom)
Shows compliance status, aggregated policy metrics, failed/error counts by policy, and cluster-wide compliance trends using Policy Reporter metrics. This is our custom-built dashboard for detailed compliance visualization.

### 4. PolicyReports Dashboard (Policy Reporter OSS)
Shows failing policies by namespace with graphical views. This is the overview dashboard from Policy Reporter that provides a different perspective on policy violations.

### 5. Kyverno ETCD Storage Observability Dashboard (Custom - Optional)
Shows ETCD cluster health, database size, disk usage, operation latency, and compaction metrics for the reports-server backend.

All dashboards use metrics scraped from Kyverno controllers and Policy Reporter through ServiceMonitors.

---

## Alternative: Manual ServiceMonitor Installation

If you prefer to install Kyverno manually without using the monitoring-values.yaml file, you can follow these steps instead of Step 1 above.

**Note:** This method requires more steps and is recommended only if you need granular control over each component.

### Step 1a: Install Base Kyverno

Install Kyverno using the packaged chart with release name `n4k-kyverno`:

```bash
helm install n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace
```

This installs the base Kyverno components with metrics endpoints exposed on port 8000, and creates the PolicyReport CRDs.

### Step 1b: Create ServiceMonitor for Kyverno Metrics

Create `servicemonitor.yaml` file with following content:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kyverno-metrics
  namespace: monitoring
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/instance: n4k-kyverno
  namespaceSelector:
    matchNames:
    - kyverno
  endpoints:
  - port: metrics-port
    interval: 30s
    path: /metrics
```

Apply the ServiceMonitor:

```bash
kubectl apply -f servicemonitor.yaml
```

This single ServiceMonitor matches all 4 Kyverno metrics services using the common label `app.kubernetes.io/instance=n4k-kyverno`.

### Step 1c: Enable Grafana Dashboards

Upgrade Kyverno installation to enable Grafana dashboard deployment:

```bash
helm upgrade n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --set grafana.enabled=true
```

This creates a ConfigMap with label `grafana_dashboard=1` containing three dashboards. Grafana sidecar automatically discovers and imports them within 30-60 seconds.

### Step 1d: Enable Reports Server with ETCD (Optional)

If you want the ETCD Storage Observability dashboard, upgrade with reports-server enabled:

```bash
helm upgrade n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --set grafana.enabled=true --set reports-server.install=true
```

Then apply the ETCD ServiceMonitor:

```bash
kubectl apply -f etcd-servicemonitor.yaml
```

After completing Steps 1a-1d, continue with Step 2 (Install Policy Reporter) above.

---

## Step 4: Install Kyverno Policies 

To see compliance data in the Policy Reports dashboard, install some policies:

### Install Pod Security Baseline policies:

```bash
kubectl apply -k https://github.com/nirmata/kyverno-policies/pod-security/baseline
```

### Install Pod Security Restricted policies:

```bash
kubectl apply -k https://github.com/nirmata/kyverno-policies/pod-security/restricted
```

### Install RBAC Best Practices policies:

```bash
kubectl apply -k https://github.com/nirmata/kyverno-policies/rbac-best-practices
```

These policies will generate PolicyReports that appear in the compliance dashboard.

---

## Troubleshooting

### If dashboards do not appear in Grafana:

1. **Verify Grafana sidecar is running:**
   ```bash
   kubectl get pods -n monitoring | grep grafana
   ```

2. **Check sidecar logs for dashboard discovery:**
   ```bash
   kubectl logs -n monitoring deployment/monitoring-grafana -c grafana-sc-dashboard
   ```

3. **Verify ConfigMap has correct label:**
   ```bash
   kubectl get configmap kyverno-grafana-grafana -n kyverno -o yaml | grep grafana_dashboard
   ```

4. **Confirm Grafana sidecar searches all namespaces** (default behavior in kube-prometheus-stack).

### If metrics are not showing in dashboards:

1. **Verify Prometheus is scraping Kyverno targets** (check Prometheus UI targets page).

2. **Ensure ServiceMonitor namespace matches Prometheus serviceMonitorSelector configuration.**

3. **Check Kyverno pods are running and healthy:**
   ```bash
   kubectl get pods -n kyverno
   ```

