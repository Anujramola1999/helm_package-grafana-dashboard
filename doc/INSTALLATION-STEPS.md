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

## Step 1: Install Kyverno

Install Kyverno using the packaged chart with release name `n4k-kyverno`:

```bash
helm install n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace
```

### Why use `n4k-kyverno` as the release name?

**Important:** The Grafana dashboards in this package are configured to query metrics with `job="n4k-kyverno-svc-metrics"`. The Helm release name directly creates Kubernetes service names, which become the Prometheus job labels.

Using `n4k-kyverno` as the release name:
- Creates the service `n4k-kyverno-svc-metrics` automatically
- This service name becomes the Prometheus job label `job="n4k-kyverno-svc-metrics"`
- Dashboards work without any configuration changes

**Alternative (not recommended):** You could use a different release name and manually configure ServiceMonitor `relabelings` to override the job label, but this adds unnecessary complexity.

This installs the base Kyverno components with metrics endpoints exposed on port 8000, and creates the PolicyReport CRDs.

---

## Step 2: Create ServiceMonitor for Kyverno Metrics

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

---

## Step 3: Enable Grafana Dashboards

Upgrade Kyverno installation to enable Grafana dashboard deployment:

```bash
helm upgrade n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --set grafana.enabled=true
```

This creates a ConfigMap with label `grafana_dashboard=1` containing three dashboards. Grafana sidecar automatically discovers and imports them within 30-60 seconds.

---

## Step 4: Install Policy Reporter

Policy Reporter collects and exposes policy compliance metrics required for the Policy Reports dashboard.

### Add Policy Reporter Helm repository:

```bash
helm repo add policy-reporter https://kyverno.github.io/policy-reporter
helm repo update
```

### Install Policy Reporter with metrics and monitoring enabled:

```bash
helm install policy-reporter policy-reporter/policy-reporter \
  --create-namespace -n policy-reporter \
  --set metrics.enabled=true \
  --set monitoring.enabled=true \
  --set monitoring.serviceMonitor.labels.release=monitoring
```

This creates:
- Policy Reporter deployment to watch PolicyReports and ClusterPolicyReports
- ServiceMonitor with `release: monitoring` label for Prometheus discovery
- Metrics endpoint at `/metrics` for policy compliance data

---

## Step 5: Apply Prometheus Recording Rules

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

Three Grafana dashboards are automatically available (four if reports-server with ETCD is enabled):

### 1. Kyverno Metrics Dashboard
Shows policy execution, admission requests, policy changes, and latency metrics.

### 2. Kyverno Performance Metrics Dashboard
Shows CPU/memory usage, admission review latency, policy execution performance, and resource utilization.

### 3. Kyverno Policy Reports Dashboard
Shows compliance status, policy report results, failed/error counts by policy, and cluster-wide policy compliance metrics.

### 4. Kyverno ETCD Storage Observability Dashboard (Optional - requires Step 6)
Shows ETCD cluster health, database size, disk usage, operation latency, and compaction metrics for the reports-server backend.

All dashboards use metrics scraped from the Kyverno controller services through ServiceMonitors.

---

## Alternative One-Command Installation

Instead of steps 1-3, you can use a values file for single command Kyverno installation.

**Note:** You still need to install Policy Reporter (Steps 4-5) and Kyverno Policies (Step 6) separately.

Create `monitoring-values.yaml`:

```yaml
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

Then install with:

```bash
helm install n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace -f monitoring-values.yaml
```

This deploys Kyverno with both dashboards and ServiceMonitors in one command. The release name `n4k-kyverno` creates service names prefixed with `n4k-kyverno-`, making them easily identifiable in Prometheus.

---

## Step 6: Enable Reports Server with ETCD (Optional - For ETCD Storage Dashboard)

Reports Server with ETCD provides persistent storage for policy reports and enables ETCD storage observability.

### Use the complete monitoring-values.yaml with reports-server enabled:

The `monitoring-values.yaml` file in this repository already includes reports-server configuration. Use it as-is:

```bash
helm uninstall n4k-kyverno -n kyverno  # If already installed
helm install n4k-kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace -f monitoring-values.yaml
```

**Note:** Uninstall is needed to avoid APIService ownership conflicts when enabling reports-server on an existing installation.

### Apply ETCD ServiceMonitor for ETCD metrics:

```bash
kubectl apply -f etcd-servicemonitor.yaml
```

### Verify ETCD deployment:

```bash
kubectl get pods -n kyverno | grep etcd
```

Expected output: 3 ETCD pods (etcd-0, etcd-1, etcd-2) running as a StatefulSet.

### What gets deployed:

- **Reports Server**: API server for policy reports with persistent ETCD backend
- **ETCD Cluster**: 3-node StatefulSet with 2Gi PVC per pod and ~1.8GiB database quota
- **ServiceMonitors**: For reports-server and ETCD metrics (6 total)
- **ETCD Dashboard**: Kyverno ETCD Storage Observability dashboard (4th dashboard)

---

## Step 7: Install Kyverno Policies (Optional)

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

