# Monitoring with Prometheus and Grafana — Kubernetes Interview Guide

## Interview Question

"How do you set up and operate a production-grade monitoring stack in Kubernetes using Prometheus and Grafana? Explain ServiceMonitor, PrometheusRule, kube-state-metrics, node-exporter, and Alertmanager. How do you handle alerting, dashboards, and long-term metrics storage at scale?"

---

## Simple Explanation

Monitoring your Kubernetes cluster is like having a health dashboard for your entire system. Prometheus is the data collector — it goes around and "scrapes" (collects) numbers from your applications and the cluster itself every 15 seconds. Grafana is the display board — it shows those numbers as beautiful charts and graphs. Alertmanager is the alarm system — when something goes wrong (CPU too high, pod crashing), it sends you an alert via Slack, PagerDuty, or email.

Think of it like a hospital monitoring system: Prometheus is the sensors (heart rate, blood pressure), Grafana is the display monitors at the nurse station, and Alertmanager is the alarm that pages the doctor when something is critical.

---

## Technical Explanation

### Prometheus Architecture

Prometheus uses a **pull-based model** — it scrapes HTTP endpoints (usually `/metrics`) on a defined interval. This is different from push-based systems (like StatsD) where applications send metrics to the collector.

**Core Components:**

1. **Prometheus Server**: Time-series database + scrape engine + PromQL query engine
2. **kube-state-metrics**: Exposes Kubernetes object state (pod status, deployment replicas, node conditions) as metrics
3. **node-exporter**: Exposes host-level metrics (CPU, memory, disk, network) from each node via DaemonSet
4. **Alertmanager**: Handles alert routing, deduplication, grouping, and notification
5. **Grafana**: Visualization and dashboarding, queries Prometheus via PromQL

### Prometheus Operator

The **kube-prometheus-stack** (formerly prometheus-operator) extends Kubernetes with CRDs:
- **ServiceMonitor**: Defines how Prometheus scrapes a Service
- **PodMonitor**: Defines how Prometheus scrapes individual Pods
- **PrometheusRule**: Defines alerting and recording rules
- **Alertmanager**: CRD for Alertmanager configuration
- **Probe**: Defines blackbox monitoring endpoints

### Metrics Types

- **Counter**: Monotonically increasing value (requests_total, errors_total)
- **Gauge**: Value that goes up and down (memory_usage, replicas)
- **Histogram**: Distribution of values (request_duration_seconds_bucket)
- **Summary**: Similar to Histogram, pre-calculated quantiles

---

## Real-World Example

**Financial Services Platform — Production Monitoring Setup:**

A fintech company running 50 microservices on EKS with 10M daily transactions needed end-to-end observability.

**Implementation:**
1. **kube-prometheus-stack** installed via Helm with custom values
2. Each microservice exposes `/metrics` with business metrics (transaction_success_total, payment_latency_seconds)
3. ServiceMonitors automatically discover services by label matching
4. PrometheusRules define SLO alerts: error budget burn rate > 5% in 1 hour
5. Alertmanager routes: P1 alerts → PagerDuty (wake someone up), P2 → Slack #alerts, P3 → Jira ticket
6. Grafana dashboards: Executive SLO dashboard, service-level latency dashboards, node utilization
7. Thanos deployed for 13-month metrics retention with S3 storage

**Result**: MTTR reduced from 45 minutes to 8 minutes due to pre-built runbook links in alerts.

---

## Diagram / Flow

```
PROMETHEUS MONITORING ARCHITECTURE
====================================

  KUBERNETES CLUSTER
  ==================

  [Nodes]
  Node-1: node-exporter (DaemonSet) --> /metrics (port 9100)
  Node-2: node-exporter (DaemonSet) --> /metrics (port 9100)
  Node-3: node-exporter (DaemonSet) --> /metrics (port 9100)

  [kube-system]
  kube-state-metrics --> /metrics (port 8080)
  kube-apiserver     --> /metrics
  kubelet            --> /metrics (port 10255)
  cadvisor           --> /metrics (container metrics)

  [Applications]
  myapp-pod-1 --> /metrics (port 8080)
  myapp-pod-2 --> /metrics (port 8080)
  payment-pod --> /metrics (port 9090)

  ServiceMonitor CR:            PodMonitor CR:
  +--------------+              +-----------+
  | selector:    |              | selector: |
  |   app: myapp |              |   app: pay|
  | namespaces:  |              +-----------+
  |   production |                   |
  +--------------+                   |
         |                           |
         v                           v
  +-------------------------------------------+
  |         PROMETHEUS SERVER                 |
  |  +--------------+  +------------------+   |
  |  | Scrape Engine|  | TSDB (Time Series|   |
  |  | (every 15s)  |  |    Database)     |   |
  |  +--------------+  +------------------+   |
  |  | PromQL Engine|  | Rule Evaluator   |   |
  |  +--------------+  +------------------+   |
  +-------------------------------------------+
         |                     |
         |                     v
         |            +------------------+
         |            |   ALERTMANAGER   |
         |            |  +------------+  |
         |            |  | Dedup      |  |
         |            |  | Grouping   |  |
         |            |  | Silencing  |  |
         |            |  | Inhibition |  |
         |            +------------------+
         |                     |
         |          +----------+----------+
         |          |          |          |
         |     [PagerDuty] [Slack]   [Email]
         v
  +-------------------+
  |      GRAFANA       |
  |  +-----------+    |
  |  | Dashboard |    |
  |  | Builder   |    |
  |  +-----------+    |
  |  Data Sources:    |
  |  - Prometheus     |
  |  - Thanos Query   |
  |  - Loki (logs)    |
  +-------------------+

LONG-TERM STORAGE WITH THANOS
================================

Prometheus-1 (cluster-A)          Prometheus-2 (cluster-B)
       |                                   |
  Thanos Sidecar                    Thanos Sidecar
  (uploads to S3)                   (uploads to S3)
       |                                   |
       +----------> S3/GCS Bucket <--------+
                          |
                   Thanos Store
                          |
                   Thanos Query  <--- Grafana
                   (federated view
                    across clusters)
```

---

## Why It Is Important

**Business Value:**
- Proactive alerting prevents outages before customers notice
- SLO/SLA tracking demonstrates reliability commitments to customers
- Cost optimization through resource utilization metrics
- Regulatory compliance (financial services require audit trails of system metrics)

**Technical Value:**
- Prometheus is the CNCF gold standard for Kubernetes monitoring
- PromQL enables powerful time-series analysis beyond simple dashboards
- ServiceMonitor/PodMonitor enable declarative, GitOps-managed monitoring configuration
- Understanding this stack is required for any senior Kubernetes role

---

## Common Interview Follow-Up Questions

1. **"What is the difference between ServiceMonitor and PodMonitor?"**
   - ServiceMonitor scrapes via a Service endpoint (uses Service's label selector)
   - PodMonitor scrapes pods directly (useful when there's no Service, or for batch jobs)
   - ServiceMonitor is more common for long-running services

2. **"How does Prometheus discover targets?"**
   - Service discovery via Kubernetes API (pods, services, endpoints, nodes)
   - Controlled by `prometheus.io/scrape: "true"` annotations (legacy) or ServiceMonitor CRDs
   - Prometheus Operator watches ServiceMonitor CRDs and dynamically updates scrape configs

3. **"What is Thanos and when do you need it?"**
   - Thanos extends Prometheus with: global query across multiple Prometheus instances, unlimited long-term storage (S3/GCS), downsampling, and deduplication
   - Needed when Prometheus TSDB disk becomes too large (>100GB) or metrics older than 15 days are needed

4. **"Explain recording rules vs alerting rules"**
   - Recording rules: Pre-compute expensive PromQL expressions and store as new metrics (performance optimization)
   - Alerting rules: Fire alerts when PromQL expressions evaluate to true for a duration

5. **"How do you avoid alert fatigue?"**
   - Alertmanager grouping: combine related alerts into one notification
   - Inhibition: suppress lower-priority alerts when a higher-priority alert is firing
   - Silences: temporarily mute alerts during maintenance windows
   - Route to appropriate channel: P1 → PagerDuty, P3 → Slack

6. **"What is the difference between CPU requests and actual CPU usage in Prometheus?"**
   - `kube_pod_container_resource_requests{resource="cpu"}` = requested CPU
   - `container_cpu_usage_seconds_total` (from cAdvisor via kubelet) = actual usage
   - Comparing both reveals over-provisioned pods (optimization opportunity)

7. **"How do you monitor a custom application with Prometheus?"**
   - Add Prometheus client library (golang/java/python) to the app
   - Expose `/metrics` endpoint
   - Add ServiceMonitor pointing to the service
   - Define PrometheusRules for business-level alerting

---

## Common Mistakes Candidates Make

### Mistake 1: Confusing kube-state-metrics with metrics-server
**Wrong**: "I use metrics-server to get pod CPU metrics for Prometheus"
**Correct**: metrics-server is for HPA and `kubectl top` (short-term, no history). kube-state-metrics exposes Kubernetes object STATE (is the pod running? how many desired replicas?). cAdvisor (embedded in kubelet) provides container resource usage metrics to Prometheus.

### Mistake 2: Not knowing the scrape path for custom apps
**Wrong**: "Prometheus automatically finds all metrics"
**Correct**: Prometheus only scrapes endpoints defined in its configuration or via ServiceMonitor CRDs. Apps must expose a `/metrics` endpoint (or configurable path) and you must create a ServiceMonitor or annotate the Service with `prometheus.io/scrape: "true"`.

### Mistake 3: Not understanding Alertmanager routing
**Wrong**: "Alertmanager just sends alerts to Slack"
**Correct**: Alertmanager has a routing tree: match labels → route to receiver. It supports grouping (batch similar alerts), inhibition (suppress child alerts when parent fires), silences (time-window muting), and repeat_interval (don't spam the same alert every minute).

### Mistake 4: Using counters as gauges
**Wrong**: Creating a Prometheus metric `requests_in_flight` as a Counter (only increases)
**Correct**: `requests_in_flight` should be a Gauge (goes up and down). Counters are for monotonically increasing values like `requests_total`. Use `rate()` function with Counters to get per-second rate.

### Mistake 5: Not sizing Prometheus storage
**Wrong**: Deploying Prometheus with default 8Gi PVC in production
**Correct**: Prometheus ingests ~1-2 bytes per sample. With 1000 metrics, 15s interval, 30-day retention: 1000 * (86400/15) * 30 * 1.5 bytes ≈ 260GB. Size PVC accordingly or use Thanos for remote storage.

---

## Troubleshooting Scenario

**Scenario**: Alerts are not firing even though Prometheus shows the metric is beyond threshold. "Why is my PrometheusRule not working?"

**Step 1: Check if PrometheusRule is being picked up**
```bash
kubectl get prometheusrule -n monitoring
kubectl describe prometheusrule myapp-alerts -n monitoring
# Look for: any validation errors in Events
```

**Step 2: Check Prometheus targets for the rule namespace**
```bash
# Port-forward to Prometheus UI
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring

# In browser: http://localhost:9090/rules
# Check if your rule group appears. If not, Prometheus isn't picking it up.
```

**Step 3: Check PrometheusRule label matching**
```bash
# Check what ruleSelector the Prometheus CRD uses
kubectl get prometheus prometheus-kube -n monitoring -o yaml | grep -A 5 ruleSelector
# Expected: matchLabels: release: prometheus

# Check if your PrometheusRule has the matching label
kubectl get prometheusrule myapp-alerts -n monitoring --show-labels
# If missing label, add it:
kubectl label prometheusrule myapp-alerts release=prometheus -n monitoring
```

**Step 4: Test the PromQL expression manually**
```bash
# Port-forward Prometheus
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring

# Test the exact expression from your PrometheusRule in Prometheus UI
# http://localhost:9090/graph
# If expression returns no data, the metric may not be scraped
```

**Step 5: Check Alertmanager is receiving alerts**
```bash
kubectl port-forward svc/alertmanager-operated 9093:9093 -n monitoring
# http://localhost:9093/#/alerts
# If alert is here but no notification, check Alertmanager config
```

**Step 6: Check Alertmanager config and routing**
```bash
kubectl get secret alertmanager-prometheus-kube-alertmanager -n monitoring -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d
# Verify route matchers match your alert labels
# Verify receiver config (Slack webhook URL, PagerDuty key)
```

**Root Cause Found**: PrometheusRule was missing the `release: prometheus` label required by the Prometheus CR's `ruleSelector`. Alerts were created but Prometheus was not loading them.

---

## kubectl Commands

```bash
# ===== INSTALL KUBE-PROMETHEUS-STACK =====

# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install with custom values
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values prometheus-values.yaml \
  --version 55.0.0

# Upgrade existing installation
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml \
  --reuse-values

# ===== CHECK PROMETHEUS COMPONENTS =====

# List all monitoring resources
kubectl get all -n monitoring

# Check Prometheus pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
# Expected: prometheus-prometheus-kube-prometheus-0   2/2   Running

# Check Alertmanager
kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager

# Check Grafana
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Check node-exporter DaemonSet (should be 1 pod per node)
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus-node-exporter
# Expected: one pod per node

# Check kube-state-metrics
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics

# ===== SERVICEMONITOR AND PROMETHEUSRULE =====

# List all ServiceMonitors
kubectl get servicemonitor -n monitoring
kubectl get servicemonitor -A

# Describe ServiceMonitor
kubectl describe servicemonitor myapp -n production

# List all PrometheusRules
kubectl get prometheusrule -A

# Check rule evaluation errors
kubectl logs prometheus-prometheus-kube-prometheus-0 -n monitoring -c prometheus | grep "error"

# ===== PORT FORWARDING FOR UI ACCESS =====

# Access Prometheus UI
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring &
# Open: http://localhost:9090

# Access Alertmanager UI
kubectl port-forward svc/alertmanager-operated 9093:9093 -n monitoring &
# Open: http://localhost:9093

# Access Grafana UI
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring &
# Open: http://localhost:3000
# Default: admin/prom-operator

# ===== PROMETHEUS TARGETS =====

# Check scrape targets via API
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health, lastError: .lastError}'

# Check targets that are DOWN
curl http://localhost:9090/api/v1/targets?state=unhealthy | jq '.data.activeTargets[] | .labels.job'

# ===== ALERTS =====

# Check active alerts
curl http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name: .labels.alertname, state: .state}'

# Silence an alert in Alertmanager (maintenance window)
curl -X POST http://localhost:9093/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [{"name": "alertname", "value": "HighMemoryUsage", "isRegex": false}],
    "startsAt": "2024-01-01T00:00:00Z",
    "endsAt": "2024-01-01T02:00:00Z",
    "createdBy": "admin",
    "comment": "Maintenance window"
  }'

# ===== GRAFANA =====

# Get Grafana admin password
kubectl get secret prometheus-grafana -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 -d && echo

# Import dashboard via API
curl -X POST http://admin:password@localhost:3000/api/dashboards/import \
  -H 'Content-Type: application/json' \
  -d @dashboard.json

# ===== DEBUGGING =====

# Check Prometheus config
kubectl get secret prometheus-prometheus-kube-prometheus-prometheus -n monitoring \
  -o jsonpath='{.data.prometheus\.yaml\.gz}' | base64 -d | gunzip

# Check if metrics are being scraped
kubectl exec -it prometheus-prometheus-kube-prometheus-0 -n monitoring -c prometheus -- \
  wget -qO- http://localhost:9090/api/v1/targets | python3 -m json.tool | head -50

# Check Prometheus disk usage
kubectl exec -it prometheus-prometheus-kube-prometheus-0 -n monitoring -c prometheus -- \
  df -h /prometheus
```

---

## YAML Example

```yaml
# ============================================================
# HELM VALUES FOR KUBE-PROMETHEUS-STACK
# ============================================================
# prometheus-values.yaml

prometheus:
  prometheusSpec:
    # Retention settings
    retention: 30d                    # Keep 30 days of metrics
    retentionSize: 50GB               # Max storage size
    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3       # EKS: use gp3 for performance
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    # Resource limits
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2
        memory: 8Gi
    # Replicas for HA (requires Thanos or remote_write for deduplication)
    replicas: 2
    # Rule selector - which PrometheusRules to load
    ruleSelector:
      matchLabels:
        release: prometheus           # IMPORTANT: PrometheusRules need this label
    # Service monitor selector
    serviceMonitorSelector:
      matchLabels:
        release: prometheus
    # Allow ServiceMonitors from all namespaces
    serviceMonitorNamespaceSelector: {}
    podMonitorNamespaceSelector: {}
    # External labels (for Thanos multi-cluster)
    externalLabels:
      cluster: production-us-east-1
    # Thanos sidecar for long-term storage
    thanos:
      image: quay.io/thanos/thanos:v0.32.0
      objectStorageConfig:
        key: objstore.yml
        name: thanos-objectstorage

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    replicas: 3                       # HA Alertmanager

grafana:
  adminPassword: "CHANGE_ME_USE_SECRET"
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  # Pre-load dashboards from ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard         # Grafana looks for CMs with this label
  # Additional data sources
  additionalDataSources:
    - name: Thanos
      type: prometheus
      url: http://thanos-query.monitoring:9090
      isDefault: false

# Node exporter - expose host metrics
nodeExporter:
  enabled: true
  resources:
    requests:
      cpu: 50m
      memory: 30Mi
    limits:
      cpu: 200m
      memory: 50Mi

# kube-state-metrics
kubeStateMetrics:
  enabled: true

# Component monitoring
kubeApiServer:
  enabled: true
kubeControllerManager:
  enabled: true
kubeScheduler:
  enabled: true
kubeEtcd:
  enabled: true
---
# ============================================================
# SERVICEMONITOR - Prometheus scrapes your application
# ============================================================
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: production
  labels:
    release: prometheus               # MUST match prometheus.serviceMonitorSelector
    app: myapp
spec:
  # Which namespaces to look for Services
  namespaceSelector:
    matchNames:
      - production
  # Which Services to monitor (must match Service labels)
  selector:
    matchLabels:
      app: myapp
      monitoring: enabled
  # Define the endpoint to scrape
  endpoints:
    - port: http-metrics               # Must match Service port name
      path: /metrics                   # Default is /metrics
      interval: 15s                    # Scrape every 15 seconds
      scrapeTimeout: 10s               # Timeout per scrape
      # TLS config for HTTPS metrics
      # scheme: https
      # tlsConfig:
      #   caFile: /etc/prom-certs/ca.crt
      # Basic auth (if metrics endpoint is protected)
      # basicAuth:
      #   username:
      #     name: metrics-auth-secret
      #     key: username
      #   password:
      #     name: metrics-auth-secret
      #     key: password
      # Relabeling: add custom labels to scraped metrics
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel: node
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
      # Metric relabeling: drop unwanted metrics
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: 'go_.*'              # Drop all Go runtime metrics
          action: drop
---
# The Service that ServiceMonitor targets
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    monitoring: enabled               # Matches ServiceMonitor selector
spec:
  selector:
    app: myapp
  ports:
    - name: http                      # Application port
      port: 80
      targetPort: 8080
    - name: http-metrics              # Metrics port (matches ServiceMonitor endpoint)
      port: 9090
      targetPort: 9090
---
# ============================================================
# PROMETHEUSRULE - Alerting and Recording Rules
# ============================================================
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: production
  labels:
    release: prometheus               # MUST match prometheus.ruleSelector
    app: myapp
spec:
  groups:
    # ---- Recording Rules (pre-compute expensive queries) ----
    - name: myapp.recording
      interval: 1m                    # Evaluate every 1 minute
      rules:
        - record: job:http_request_duration_seconds:p99
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m]))
              by (le, job, namespace)
            )
        - record: job:http_requests:rate5m
          expr: |
            sum(rate(http_requests_total[5m])) by (job, namespace, status_code)

    # ---- Alerting Rules ----
    - name: myapp.alerts
      rules:
        # High error rate alert
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service, namespace)
            /
            sum(rate(http_requests_total[5m])) by (service, namespace)
            > 0.01
          for: 5m                     # Must be true for 5 minutes (avoid flapping)
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "High error rate on {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.service }} in {{ $labels.namespace }}"
            runbook_url: "https://runbooks.example.com/high-error-rate"

        # High latency alert (using recording rule)
        - alert: HighP99Latency
          expr: job:http_request_duration_seconds:p99 > 1.0
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "High P99 latency on {{ $labels.job }}"
            description: "P99 latency is {{ $value | humanizeDuration }} on {{ $labels.job }}"

        # Pod crash looping
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total[1h]) > 5
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
            description: "Pod has restarted {{ $value }} times in the last hour"

        # Deployment replicas mismatch
        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas
            != kube_deployment_status_available_replicas
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has unavailable replicas"

        # Node not ready
        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
            team: infra
          annotations:
            summary: "Node {{ $labels.node }} is not ready"

        # High memory usage
        - alert: HighMemoryUsage
          expr: |
            (container_memory_usage_bytes{container!="",container!="POD"}
            / container_spec_memory_limit_bytes{container!="",container!="POD"})
            > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Container {{ $labels.container }} in {{ $labels.namespace }}/{{ $labels.pod }} memory usage is high"
            description: "Memory usage is {{ $value | humanizePercentage }} of limit"

        # PVC storage running out
        - alert: PersistentVolumeFillingUp
          expr: |
            kubelet_volume_stats_available_bytes
            / kubelet_volume_stats_capacity_bytes
            < 0.15
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} is filling up"
            description: "Only {{ $value | humanizePercentage }} storage remaining"

    # ---- SLO Rules ----
    - name: myapp.slo
      rules:
        # SLO: 99.9% availability
        - alert: SLOBudgetBurnRateCritical
          expr: |
            (
              sum(rate(http_requests_total{status_code=~"5..",service="myapp"}[1h]))
              /
              sum(rate(http_requests_total{service="myapp"}[1h]))
            ) > (1 - 0.999) * 14.4     # 14.4x burn rate = exhaust monthly budget in 2 days
          for: 2m
          labels:
            severity: critical
            slo: availability
          annotations:
            summary: "SLO error budget burning too fast for myapp"
---
# ============================================================
# ALERTMANAGER CONFIGURATION
# ============================================================
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-prometheus-kube-alertmanager
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      # Default SMTP config (fallback)
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'alerts@company.com'
      smtp_auth_username: 'alerts@company.com'
      smtp_auth_password: 'APP_PASSWORD'
      # Resolve timeout - how long to wait before marking alert as resolved
      resolve_timeout: 5m

    # Templates for notification messages
    templates:
      - '/etc/alertmanager/templates/*.tmpl'

    # Routing tree
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s                 # Wait 30s to group alerts before first notification
      group_interval: 5m              # Resend grouped alert every 5 minutes
      repeat_interval: 4h             # Repeat alert every 4 hours if not resolved
      receiver: 'slack-default'       # Default receiver

      routes:
        # Critical alerts go to PagerDuty AND Slack
        - match:
            severity: critical
          receiver: pagerduty-critical
          continue: true              # Continue to also send to Slack
        - match:
            severity: critical
          receiver: slack-critical

        # SLO alerts go to separate channel
        - match:
            slo: availability
          receiver: slack-slo
          group_wait: 0s              # No delay for SLO alerts

        # Infrastructure alerts go to infra team
        - match:
            team: infra
          receiver: slack-infra

        # All other warnings go to default Slack
        - match:
            severity: warning
          receiver: slack-default

    # Inhibition rules - suppress lower severity when higher fires
    inhibit_rules:
      # If critical node alert fires, suppress pod alerts on same node
      - source_match:
          severity: critical
          alertname: NodeNotReady
        target_match:
          severity: warning
        equal: ['node']

    receivers:
      - name: 'slack-default'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
            channel: '#k8s-alerts'
            send_resolved: true
            title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
            text: |
              *Severity:* {{ .CommonLabels.severity }}
              *Namespace:* {{ .CommonLabels.namespace }}
              {{ range .Alerts }}
              *Alert:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              *Runbook:* {{ .Annotations.runbook_url }}
              {{ end }}

      - name: 'pagerduty-critical'
        pagerduty_configs:
          - service_key: 'PAGERDUTY_SERVICE_KEY'
            send_resolved: true
            description: '{{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}'
            severity: '{{ .CommonLabels.severity }}'
            details:
              cluster: '{{ .CommonLabels.cluster }}'
              namespace: '{{ .CommonLabels.namespace }}'

      - name: 'slack-critical'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
            channel: '#k8s-critical'
            send_resolved: true
            color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

      - name: 'slack-infra'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
            channel: '#infra-alerts'
            send_resolved: true

      - name: 'slack-slo'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
            channel: '#slo-alerts'
            send_resolved: true
---
# ============================================================
# GRAFANA DASHBOARD AS CONFIGMAP
# ============================================================
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"           # Grafana sidecar picks up CMs with this label
data:
  myapp-dashboard.json: |
    {
      "title": "MyApp Production Dashboard",
      "panels": [
        {
          "title": "Request Rate",
          "type": "graph",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{service='myapp'}[5m])) by (status_code)",
              "legendFormat": "{{ status_code }}"
            }
          ]
        },
        {
          "title": "P99 Latency",
          "type": "graph",
          "targets": [
            {
              "expr": "job:http_request_duration_seconds:p99{job='myapp'}",
              "legendFormat": "P99 Latency"
            }
          ]
        },
        {
          "title": "Error Rate",
          "type": "singlestat",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{service='myapp',status_code=~'5..'}[5m])) / sum(rate(http_requests_total{service='myapp'}[5m]))"
            }
          ]
        }
      ]
    }
---
# ============================================================
# THANOS - Long-term Storage Configuration
# ============================================================
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objectstorage
  namespace: monitoring
type: Opaque
stringData:
  objstore.yml: |
    type: S3
    config:
      bucket: my-thanos-bucket
      endpoint: s3.us-east-1.amazonaws.com
      region: us-east-1
      # Use IRSA (IAM Roles for Service Accounts) in EKS - no keys needed
      # aws_sdk_auth: true
```

---

## AWS/EKS Perspective

### EKS Monitoring with AWS Native Services

**Amazon Managed Service for Prometheus (AMP):**
```bash
# Create AMP workspace
aws amp create-workspace --alias my-eks-monitoring --region us-east-1

# Get workspace ID
WORKSPACE_ID=$(aws amp list-workspaces --query 'workspaces[0].workspaceId' --output text)

# Configure Prometheus to remote_write to AMP
# remote_write:
#   - url: https://aps-workspaces.us-east-1.amazonaws.com/workspaces/<ID>/api/v1/remote_write
#     sigv4:
#       region: us-east-1
#       role_arn: arn:aws:iam::ACCOUNT:role/prometheus-remote-write-role
```

**Amazon Managed Grafana (AMG):**
```bash
# Create AMG workspace via console or CLI
aws grafana create-workspace \
  --workspace-name my-grafana \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type SERVICE_MANAGED
```

**Container Insights (CloudWatch):**
```bash
# Install CloudWatch agent for Container Insights
ClusterName=my-cluster
RegionName=us-east-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'

kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml
```

**IRSA for Prometheus Remote Write to AMP:**
```bash
# Create IAM role for Prometheus service account
eksctl create iamserviceaccount \
  --name prometheus \
  --namespace monitoring \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess \
  --approve \
  --override-existing-serviceaccounts
```

**EKS Managed Add-on for Metrics:**
```bash
# Install AWS Distro for OpenTelemetry (ADOT) as EKS add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name adot \
  --service-account-role-arn arn:aws:iam::ACCOUNT:role/adot-role
```

---

## Interview Answer (2-Minute Version)

"I use kube-prometheus-stack installed via Helm for Kubernetes monitoring. This gives me Prometheus for metric collection, Grafana for dashboards, and Alertmanager for notifications. Prometheus uses kube-state-metrics to track Kubernetes object state — like whether a deployment has the right number of replicas — and node-exporter running as a DaemonSet to collect host-level CPU, memory, and disk metrics from every node.

For custom application metrics, I add a ServiceMonitor CRD that tells Prometheus which Services to scrape and at what interval. Alerting rules are defined as PrometheusRule CRDs. Alertmanager routes alerts to PagerDuty for critical issues and Slack for warnings. For production I also set up Thanos for long-term metric storage in S3 beyond Prometheus's local retention period."

---

## Interview Answer (Senior Engineer Version)

"Production Prometheus setup requires thinking about cardinality, storage, HA, and alert quality. I install kube-prometheus-stack via Helm with custom values for storage sizing — typically 100Gi PVC with gp3 StorageClass for performance. Prometheus retention is set to 15-30 days locally, with Thanos sidecar uploading to S3 for 13 months of compliance retention.

ServiceMonitor CRDs provide declarative, GitOps-managed scrape configurations. A common pitfall is the label selector mismatch — the ServiceMonitor must have the `release: prometheus` label to match the Prometheus CR's ruleSelector, and so must PrometheusRules.

For alerting, I use SLO-based alerting with burn rate calculations rather than raw threshold alerts. A 14.4x burn rate over 1 hour means the monthly error budget will be exhausted in 2 days — this fires as Critical. This approach reduces alert fatigue because it only fires when users are actually being impacted, not just when a metric crosses an arbitrary threshold.

In EKS, I evaluate whether Amazon Managed Service for Prometheus is cost-effective for the scale — at very high cardinality (10M+ active series), AMP becomes cheaper than running Thanos. I use IRSA for Prometheus's remote_write authentication to AMP, eliminating IAM key rotation risk.

Cardinality control is critical — one bad label (like including request IDs in metrics labels) can cause millions of series and crash Prometheus. I use `metricRelabelings` in ServiceMonitors to drop high-cardinality labels before storage."

---

## What Impresses the Interviewer

1. Knowing the label selector chain: ServiceMonitor label → Prometheus CR serviceMonitorSelector → PrometheusRule label → ruleSelector
2. Mentioning cardinality as a Prometheus scalability concern
3. Explaining SLO-based burn rate alerting instead of simple threshold alerts
4. Knowing the difference between metrics-server, kube-state-metrics, and cAdvisor
5. Discussing Thanos for multi-cluster and long-term storage
6. Mentioning IRSA for AMP in EKS (no long-lived credentials)
7. Knowing `metricRelabelings` for controlling what gets stored

---

## Red Flags

- Saying "Prometheus pushes metrics to applications" (it PULLS/scrapes)
- Not knowing that kube-state-metrics exposes object STATE not resource usage
- Thinking metrics-server data is accessible via PromQL
- Not knowing the label selector requirements for ServiceMonitor/PrometheusRule
- No understanding of alert fatigue and Alertmanager grouping/inhibition
- No mention of storage sizing and retention planning

---

## Production Best Practices

1. **Size Prometheus storage proactively**: Calculate expected series count × scrape interval × retention × bytes per sample. Default 8Gi is insufficient for any real cluster.

2. **Use recording rules for expensive queries**: Any PromQL expression used in dashboards or alerts that scans large time ranges should be pre-computed as a recording rule.

3. **Implement SLO-based alerting**: Don't alert on CPU > 80%. Alert on error budget burn rate. This reduces alert fatigue and ensures alerts mean real user impact.

4. **Set Alertmanager repeat_interval carefully**: 1-hour repeat for Critical, 4-hour for Warning. Too short causes spam; too long means forgotten alerts.

5. **Control label cardinality**: Never include unbounded values (user IDs, request IDs, URLs) as metric labels. Each unique label combination creates a new time series.

6. **Add runbook_url to every alert**: Every PrometheusRule alert annotation should include a runbook URL. Reduces MTTR by giving on-call engineers immediate context.

7. **Test alerts with amtool**: Use `amtool check-config` and Prometheus `promtool check rules` in CI pipeline to validate alert syntax before deploying.

8. **Deploy Alertmanager in HA mode**: 3 replicas with mesh clustering to avoid missed alerts during pod restarts.

---

## Key Points to Remember

- Prometheus uses PULL model — it scrapes `/metrics` endpoints
- kube-state-metrics = Kubernetes object state; node-exporter = host OS metrics; cAdvisor = container resource metrics
- ServiceMonitor CRD defines which Services Prometheus scrapes
- PrometheusRule CRD defines alerting and recording rules
- Both ServiceMonitor and PrometheusRule MUST have labels matching Prometheus CR selectors
- Alertmanager handles routing, grouping, deduplication, and silencing
- Thanos provides multi-cluster federation and long-term S3 storage
- High cardinality (too many unique label combinations) can crash Prometheus
- SLO burn rate alerting > threshold alerting for reducing alert fatigue
- In EKS: Amazon Managed Service for Prometheus + IRSA for production scale

---

## Interviewer's Expectation

The interviewer is testing:
1. **Operational depth**: Not just "install Prometheus" but sizing, HA, retention planning
2. **CRD knowledge**: ServiceMonitor, PrometheusRule label selector chain
3. **PromQL competency**: Ability to write meaningful queries for rate, histogram_quantile, recording rules
4. **Alert design**: SLO-based alerting, Alertmanager routing, reducing alert fatigue
5. **Scale thinking**: Thanos, cardinality control, multi-cluster monitoring
6. **AWS/EKS specifics**: AMP, AMG, IRSA, Container Insights

---

## Final Perfect Interview Answer

"For production Kubernetes monitoring I deploy kube-prometheus-stack via Helm, which bundles Prometheus, Alertmanager, Grafana, kube-state-metrics, and node-exporter. My first concern is storage sizing — I calculate expected series count times scrape interval times retention period, typically needing 50-100Gi for a medium cluster with 30-day retention.

Application monitoring uses ServiceMonitor CRDs with the correct release label to match the Prometheus CR's serviceMonitorSelector. A common failure point is missing this label, causing Prometheus to silently ignore the ServiceMonitor. Similarly, PrometheusRules must have matching labels to be loaded.

For alerting I use SLO burn rate calculations rather than raw thresholds. A 14.4x error budget burn rate fires as Critical — this only triggers when users are actually impacted, not just when CPU crosses 80%. Alertmanager routes Critical alerts to PagerDuty and Warning to Slack, with grouping and inhibition rules to prevent alert storms.

For scale beyond a single cluster or 15-day retention, I deploy Thanos with S3 backend. In EKS, I evaluate Amazon Managed Service for Prometheus at high cardinality because it eliminates storage management overhead, using IRSA for authentication rather than long-lived credentials."

---
*Senior Kubernetes Architect Guide — Monitoring with Prometheus and Grafana*
