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

Install Kyverno using the packaged chart:

```bash
helm install kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace
```

This installs the base Kyverno components with metrics endpoints exposed on port 8000.

---

## Step 2: Create ServiceMonitor for Metrics Scraping

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
      app.kubernetes.io/instance: kyverno
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

This single ServiceMonitor matches all 4 Kyverno metrics services using the common label `app.kubernetes.io/instance=kyverno`.

---

## Step 3: Enable Grafana Dashboards

Upgrade Kyverno installation to enable Grafana dashboard deployment:

```bash
helm upgrade kyverno ./kyverno-3.5.5.tgz -n kyverno --set grafana.enabled=true
```

This creates a ConfigMap with label `grafana_dashboard=1` containing two dashboards. Grafana sidecar automatically discovers and imports them within 30-60 seconds.

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

Two Grafana dashboards are automatically available:

### 1. Kyverno Metrics Dashboard (original)
Shows policy execution, admission requests, policy changes, and latency metrics.

### 2. Kyverno Performance Metrics Dashboard (new)
Shows CPU/memory usage, admission review latency, policy execution performance, and resource utilization.

Both dashboards use metrics scraped from the four Kyverno controller services through the single ServiceMonitor.

---

## Alternative One-Command Installation

Instead of steps 1-3, you can use a values file for single command installation.

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
helm install kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace -f monitoring-values.yaml
```

This deploys Kyverno with both dashboards and ServiceMonitors in one command.

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

