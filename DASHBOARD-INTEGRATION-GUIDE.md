# Integrating Grafana Dashboard into Kyverno Helm Chart

## Step 1: Prepare Your Dashboard JSON File

Ensure your dashboard JSON file has these requirements:

1. No `__inputs` section (remove it if present)
2. No `__elements` section (remove it if present)
3. `"id": null` (not a number)
4. Templating section with datasource variable

### Example templating section to add:

```json
"templating": {
  "list": [
    {
      "current": null,
      "hide": 0,
      "includeAll": false,
      "label": "datasource",
      "multi": false,
      "name": "DS_PROMETHEUS",
      "options": [],
      "query": "prometheus",
      "refresh": 1,
      "regex": "",
      "skipUrlSync": false,
      "type": "datasource"
    }
  ]
}
```

---

## Step 2: Add Dashboard to Helm Chart

Copy your dashboard JSON file to the correct location:

```bash
cp your-dashboard.json kyverno/charts/grafana/dashboard/
```

### The directory structure should look like:

```
kyverno/
  charts/
    grafana/
      dashboard/
        kyverno-dashboard.json (existing)
        your-new-dashboard.json (your file)
```

---

## Step 3: Verify the Template

Check that the template will include your dashboard:

```bash
cat kyverno/charts/grafana/templates/dashboard.yaml
```

Look for this line that reads all dashboard files:

```yaml
{{ (.Files.Glob "dashboard/*").AsConfig | indent 2 }}
```

This automatically includes **ALL** JSON files from the dashboard directory.

---

## Step 4: Package the Helm Chart

Package the chart into a `.tgz` file:

```bash
helm package kyverno/
```

This creates: `kyverno-3.5.5.tgz` (or whatever version is in Chart.yaml)

---

## Step 5: Test the Packaged Chart

Verify the dashboard is included in the package:

```bash
tar -tzf kyverno-3.5.5.tgz | grep "dashboard.*\.json"
```

You should see all dashboard JSON files listed.

---

## Step 5B: Test Helm Template Output

### Preview what Kubernetes resources will be created without actually installing:

```bash
helm template kyverno ./kyverno-3.5.5.tgz --set grafana.enabled=true --show-only charts/grafana/templates/dashboard.yaml
```

This shows the ConfigMap that will be created with your dashboards.

### To see just the dashboard filenames in the ConfigMap:

```bash
helm template kyverno ./kyverno-3.5.5.tgz --set grafana.enabled=true --show-only charts/grafana/templates/dashboard.yaml | grep "\.json:"
```

**Expected output:**
```
kyverno-dashboard.json: |
kyverno-performance-metrics.json: |-
kyverno-policy-reports.json: |
```

### To verify ConfigMap has the correct discovery label:

```bash
helm template kyverno ./kyverno-3.5.5.tgz --set grafana.enabled=true --show-only charts/grafana/templates/dashboard.yaml | grep -A 3 "labels:"
```

**Expected output should include:**
```yaml
labels:
  grafana_dashboard: "1"
```

---

## Step 6: Install with Monitoring Enabled

Create a `monitoring-values.yaml` file:

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

---

## Step 7: Install the Packaged Chart

Install using the `.tgz` file:

```bash
helm install kyverno ./kyverno-3.5.5.tgz -n kyverno --create-namespace -f monitoring-values.yaml
```

This single command deploys:
- Kyverno components
- ServiceMonitors with correct labels for Prometheus
- ConfigMap with ALL dashboards for Grafana auto-discovery

---

## Step 8: Verify Installation

### Check ConfigMap was created with dashboards:

```bash
kubectl get configmap kyverno-grafana-grafana -n kyverno -o yaml | grep "\.json:"
```

### Check ServiceMonitors were created:

```bash
kubectl get servicemonitor -n kyverno
```

### Verify ConfigMap has the discovery label:

```bash
kubectl get configmap kyverno-grafana-grafana -n kyverno -o jsonpath='{.metadata.labels}'
```

---

## What Happens Automatically

When you install with `grafana.enabled=true`:

1. Helm creates ConfigMap in kyverno namespace
2. ConfigMap contains ALL JSON files from `charts/grafana/dashboard/`
3. ConfigMap gets label `grafana_dashboard="1"`
4. Grafana sidecar scans all namespaces for this label
5. Sidecar auto-imports all dashboards within 30-60 seconds
6. Dashboards appear in Grafana UI ready to use

---

## Key Files in Your Package

`kyverno-3.5.5.tgz` contains:
- `charts/grafana/dashboard/kyverno-dashboard.json`
- `charts/grafana/dashboard/kyverno-performance-metrics.json`
- `charts/grafana/dashboard/kyverno-policy-reports.json`
- `charts/grafana/templates/dashboard.yaml` (reads all JSON files)
- `values.yaml` (with `grafana.enabled=false` by default)

---

## Upgrading Existing Installation

If Kyverno is already installed, upgrade to add dashboards:

```bash
helm upgrade kyverno ./kyverno-3.5.5.tgz -n kyverno -f monitoring-values.yaml
```

---

## Removing Dashboards

To remove dashboards but keep Kyverno running:

```bash
helm upgrade kyverno ./kyverno-3.5.5.tgz -n kyverno --set grafana.enabled=false
```

This removes the ConfigMap and Grafana will remove the dashboards.

---

## Important Notes

1. Dashboard JSON files are included at **PACKAGE TIME** (`helm package`)
2. Any changes to JSON files require repackaging the chart
3. The template automatically includes all `.json` files in dashboard directory
4. No code changes needed to add new dashboards, just drop files in the folder
5. Grafana sidecar must be enabled in your Grafana installation (default in kube-prometheus-stack)
6. ServiceMonitors need `release: monitoring` label to match Prometheus selector

