# HPA (Horizontal Pod Autoscaler) — Kubernetes Interview Guide

---

## Interview Question

**"What is the Horizontal Pod Autoscaler in Kubernetes? How does it work, and how does it differ from Vertical Pod Autoscaler? What is the behavior field?"**

---

## Simple Explanation

Imagine you run a pizza restaurant. On a normal Tuesday, 3 staff members are enough. But on Super Bowl Sunday, you suddenly have 200 customers — you need 20 staff. After the game ends, you send 17 staff home.

**HPA does exactly this for your Kubernetes pods:**
- When traffic increases, it automatically creates more pods (scales out).
- When traffic decreases, it removes extra pods (scales in).
- It monitors metrics like CPU usage, memory, or custom metrics (request count, queue depth) to make these decisions.

**Simple analogy:**
- HPA = Restaurant manager who watches how busy it is and calls more staff in (or sends them home) based on demand.
- The metrics-server = The order tracking system that tells the manager how many orders are coming in.

**HPA vs VPA (quick distinction):**
- **HPA** = More copies of the same size container (horizontal scaling — add more)
- **VPA** = Same number of containers but each gets more CPU/memory (vertical scaling — make bigger)

---

## Technical Explanation

### Architecture and Components

**HPA Controller** runs as part of the kube-controller-manager. It periodically (default every 15 seconds) checks metrics and adjusts the number of replicas in a Deployment, ReplicaSet, or StatefulSet.

**Metrics Sources (three tiers):**

| Source | Type | Examples |
|---|---|---|
| Resource metrics | CPU, Memory utilization | `cpu: targetAverageUtilization: 70` |
| Custom metrics | Application-level metrics | HTTP requests/sec, queue depth |
| External metrics | Metrics outside the cluster | AWS SQS queue length, external API rate |

**Metrics Pipeline:**

```
Metrics Sources:
  kubelet (cAdvisor) → metrics-server → HPA controller (resource metrics)
  Application → Prometheus → prometheus-adapter → HPA controller (custom metrics)
  External service → custom adapter → HPA controller (external metrics)
```

### Scaling Formula

```
desiredReplicas = ceil[ currentReplicas × (currentMetricValue / desiredMetricValue) ]
```

**Example:**
- Current replicas: 4
- Current CPU: 80% (average across all 4 pods)
- Target CPU: 50%
- desiredReplicas = ceil[ 4 × (80 / 50) ] = ceil[ 6.4 ] = 7

HPA rounds UP to ensure metric targets aren't exceeded.

### Stabilization Windows

To prevent "flapping" (rapid scale-up then scale-down oscillations), HPA uses stabilization windows:
- **Scale-down stabilization:** Default 300 seconds (5 minutes). HPA waits until the low metric persists for 5 minutes before scaling down. Prevents removing pods during momentary traffic drops.
- **Scale-up stabilization:** Default 0 seconds. HPA scales up immediately to meet demand.

### The Behavior Field (v2 API)

The `behavior` field gives fine-grained control over scaling speed and policies:

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0     # Scale up immediately
    policies:
      - type: Pods                    # Add at most 4 pods per 60 seconds
        value: 4
        periodSeconds: 60
      - type: Percent                 # OR add at most 100% of current pods per 60s
        value: 100
        periodSeconds: 60
    selectPolicy: Max                 # Use whichever policy allows more scaling
  scaleDown:
    stabilizationWindowSeconds: 300   # Wait 5 minutes before scaling down
    policies:
      - type: Pods                    # Remove at most 2 pods per 60 seconds
        value: 2
        periodSeconds: 60
```

**selectPolicy options:**
- `Max` — Use the policy that allows the most scaling (most aggressive)
- `Min` — Use the policy that allows the least scaling (most conservative)
- `Disabled` — Completely disable scaling in that direction

---

## Real-World Example

### Scenario: E-Commerce Flash Sale Traffic Management

**Company:** Online retail platform with normal traffic of 500 req/sec and flash sale peaks of 10,000 req/sec.

**Problem Without HPA:**
- Team must manually scale deployments before each sale.
- Sales are unpredictable — a viral social media post can cause sudden spikes.
- Over-provisioning 24/7 is expensive ($50k/month in wasted compute).

**With HPA:**

```
Normal operation: 3 frontend pods, CPU at 30%
Flash sale starts: CPU jumps to 90% across all pods
HPA detects: 90% > 70% target (after 30s evaluation period)
HPA scales: 3 → 6 → 12 → 20 pods over 3 minutes
Traffic handled: All requests served
Sale ends: CPU drops back to 15%
HPA scales down: 20 → 10 → 5 → 3 pods over 15 minutes (stabilization window)
```

**Cost impact:** Pay for 20 pods only during the 2-hour flash sale vs paying for 20 pods 24/7.

**Custom metrics example:** HPA scales based on HTTP requests per second (from Prometheus), not CPU — more accurate for stateless web services where CPU doesn't always correlate with load.

---

## Diagram / Flow

```
HPA ARCHITECTURE
=================

  ┌─────────────────────────────────────────────────────────┐
  │                  KUBERNETES CLUSTER                     │
  │                                                         │
  │  ┌─────────────┐    ┌──────────────┐    ┌───────────┐  │
  │  │  Application│    │  metrics-    │    │  HPA      │  │
  │  │  Pods       │───>│  server      │───>│  Controller│ │
  │  │  (cAdvisor) │    │  (resource   │    │  (every   │  │
  │  └─────────────┘    │   metrics)   │    │  15 sec)  │  │
  │                     └──────────────┘    └─────┬─────┘  │
  │  ┌─────────────┐    ┌──────────────┐          │        │
  │  │  Prometheus │    │  Prometheus  │          │        │
  │  │  (custom    │───>│  Adapter     │──────────┤        │
  │  │   metrics)  │    │  (custom     │          │        │
  │  └─────────────┘    │   metrics    │          │        │
  │                     │   API)       │          │        │
  │  ┌─────────────┐    └──────────────┘          │        │
  │  │  External   │    ┌──────────────┐          │        │
  │  │  Metrics    │───>│  External    │──────────┘        │
  │  │  (SQS, etc) │    │  Metrics     │                   │
  │  └─────────────┘    │  Adapter     │    ┌───────────┐  │
  │                     └──────────────┘    │ Deployment│  │
  │                                         │ replicas  │  │
  │                                         │ adjusted  │  │
  │                                         └───────────┘  │
  └─────────────────────────────────────────────────────────┘


HPA SCALING DECISION FLOW
==========================

  Every 15 seconds (default sync period):
  ┌────────────────────────────┐
  │  Fetch current metrics     │
  │  from metrics API          │
  └─────────────┬──────────────┘
                │
                ▼
  ┌────────────────────────────┐
  │  Apply formula:            │
  │  desired = ceil(current ×  │
  │  (actual/target))          │
  └─────────────┬──────────────┘
                │
                ▼
  ┌────────────────────────────┐
  │  Check stabilization       │
  │  window (scale-down only)  │
  └─────────────┬──────────────┘
                │
  ┌─────────────┴──────────────────────────────┐
  │                                            │
  ▼ Need more pods?                            ▼ Need fewer pods?
  Scale UP                                     Check: has low metric
  Immediately                                  persisted for
  (stabilization = 0)                          300 seconds?
                                               If yes → Scale DOWN
                                               If no → Wait


BEHAVIOR FIELD EFFECT ON SCALING SPEED
========================================

  Without behavior (default):          With behavior (controlled):
  ──────────────────────────           ─────────────────────────────
  t=0:  3 pods, CPU=80%                t=0:  3 pods, CPU=80%
  t=15: 7 pods (immediate jump)        t=1m: 5 pods  (max 4 pods/60s)
  t=30: 12 pods                        t=2m: 9 pods
  t=60: 20 pods                        t=3m: 13 pods (max 100%/60s)
                                       t=4m: 20 pods  (under control)


HPA vs VPA COMPARISON
======================

  HPA (Horizontal)             VPA (Vertical)
  ────────────────             ──────────────
  Adds MORE pods               Makes pods BIGGER
  Same resource limits         Adjusts resource limits
  Works with stateless apps    Better for stateful apps
  Near-zero downtime           May require pod restart
  Fast response                Slow response
  ┌───────┐ ┌───────┐          ┌───────────────┐
  │  pod  │ │  pod  │          │      pod      │
  │ 500m  │ │ 500m  │          │     1000m     │
  └───────┘ └───────┘          │  (resized)    │
  ┌───────┐ ┌───────┐          └───────────────┘
  │  pod  │ │  pod  │
  │ 500m  │ │ 500m  │          Single pod, more CPU
  └───────┘ └───────┘
  Multiple pods, same size
```

---

## Why It Is Important

### Business Value
- **Cost Optimization:** Pay only for compute you need. Scale down during off-peak hours. Companies report 40-60% cost reduction vs fixed capacity.
- **Availability:** Automatically handles traffic spikes without manual intervention. Prevents outages during unexpected load surges.
- **Developer Productivity:** Removes the need for manual scaling operations. On-call engineers don't need to wake up to scale deployments.
- **SLA Compliance:** Maintains response time SLAs during traffic spikes by automatically adding capacity.

### Technical Value
- **Reactive Scaling:** Responds to actual resource usage rather than predictions.
- **Custom Metrics:** Scale on business metrics (orders/sec, queue depth) not just infrastructure metrics.
- **Integration with Cluster Autoscaler:** HPA scales pods; Cluster Autoscaler scales nodes. Together they form a complete autoscaling solution.
- **Flexible Behavior:** The `behavior` field prevents both scale thrashing and slow scale-up during critical spikes.

---

## Common Interview Follow-Up Questions

1. **"What is metrics-server and why is it required for HPA?"**
   - metrics-server is a cluster-wide aggregator of resource usage data. It collects CPU and memory metrics from kubelets (cAdvisor) and exposes them via the Kubernetes Metrics API. HPA reads this API to get current pod resource usage. Without metrics-server, HPA cannot function for CPU/memory-based scaling.

2. **"What is the difference between HPA and VPA?"**
   - HPA scales horizontally (more pod instances, same size). VPA scales vertically (same number of pods, but adjusts CPU/memory requests). HPA works best for stateless services; VPA is better for stateful applications or when you have a fixed number of replicas. They can be used together but careful coordination is needed.

3. **"Can HPA scale based on custom metrics?"**
   - Yes. Using the `custom.metrics.k8s.io` API (via an adapter like prometheus-adapter). You can scale on HTTP requests/sec, queue depth, active connections, or any application-level metric exposed via Prometheus.

4. **"What happens to HPA when metrics-server is unavailable?"**
   - HPA stops receiving metrics. It will not make scaling decisions during the outage. The current replica count stays fixed. Once metrics-server recovers, HPA resumes normal operation.

5. **"What is the stabilization window and why does it exist?"**
   - The stabilization window prevents rapid scaling oscillations (thrashing). For scale-down, HPA waits for the low-resource condition to persist for `stabilizationWindowSeconds` (default 300s) before removing pods. Without it, a brief traffic dip would remove pods, traffic would spike again, and HPA would add them back in a rapid cycle.

6. **"Can HPA and manual scaling coexist?"**
   - You can manually change replicas, but HPA will override it on the next sync cycle (15 seconds). The effective range is always bounded by HPA's minReplicas and maxReplicas. To temporarily disable HPA scaling, set `scaleTargetRef` to a non-existent resource or delete the HPA.

7. **"What is KEDA and how does it relate to HPA?"**
   - KEDA (Kubernetes Event-Driven Autoscaling) is an open-source component that extends HPA with event-based scaling from 50+ sources (Kafka, SQS, RabbitMQ, etc.). It scales on actual event counts, including scaling to zero (which standard HPA cannot do). KEDA creates and manages HPA objects internally.

---

## Common Mistakes Candidates Make

### Mistake 1: Forgetting metrics-server is not installed by default
**Wrong:** "HPA works out of the box in any Kubernetes cluster."
**Correct:** metrics-server must be separately installed (it's not included in standard Kubernetes). On EKS, you must install it via the add-on. Without it, `kubectl top` doesn't work and HPA cannot get CPU/memory metrics.

### Mistake 2: Setting resource requests without limits
**Wrong:** "I set CPU limit to 500m so HPA can calculate utilization."
**Correct:** HPA calculates CPU utilization as `actual_cpu_usage / request` (not limit). If pods have no CPU requests set, HPA cannot calculate utilization percentage and will fail. Always set resource requests for HPA to work correctly.

### Mistake 3: Not understanding the behavior field limitations
**Wrong:** "The behavior field makes HPA ignore maxReplicas."
**Correct:** The behavior field controls the RATE of scaling, not the limits. maxReplicas always caps the maximum. The behavior field only controls how fast you get there.

### Mistake 4: Expecting immediate scale-down
**Wrong:** "Traffic dropped, why hasn't HPA removed the extra pods yet?"
**Correct:** Scale-down stabilization window defaults to 300 seconds (5 minutes). HPA intentionally waits to ensure the drop is sustained, not just a brief fluctuation. This is protective behavior.

### Mistake 5: Using HPA with StatefulSets carelessly
**Wrong:** "HPA works the same way with StatefulSets as with Deployments."
**Correct:** HPA with StatefulSets can work but requires careful consideration. StatefulSets have ordered deployment/scaling, persistent volume claims, and stable network identities. Rapid HPA scaling of stateful workloads can cause data inconsistency or volume attachment issues.

---

## Troubleshooting Scenario

### Problem: HPA Not Scaling Despite High CPU Usage

**Situation:** Production deployment shows 95% CPU but HPA is stuck at minimum replicas.

```bash
# Step 1: Check HPA status
kubectl get hpa -n production
# NAME       REFERENCE              TARGETS     MINPODS  MAXPODS  REPLICAS  AGE
# web-hpa    Deployment/web-app    <unknown>/70%   2       20        2      15m
# <unknown> indicates HPA cannot get metrics

# Step 2: Describe HPA for detailed status
kubectl describe hpa web-hpa -n production
# Look for Conditions section:
# Conditions:
#   Type           Status  Reason              Message
#   AbleToScale    True    ReadyForNewScale    recent recommendations stable
#   ScalingActive  False   FailedGetMetrics    the HPA was unable to compute the replica count:
#                                              unable to get metrics for resource cpu:
#                                              unable to fetch metrics from resource metrics API:
#                                              no metrics returned from resource metrics API

# Step 3: Check if metrics-server is running
kubectl get pods -n kube-system | grep metrics-server
# If no output → metrics-server is not installed

# Step 4: Check metrics-server availability
kubectl top nodes
# Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes.metrics.k8s.io)
# This confirms metrics-server is unavailable

# Step 5: Check metrics API
kubectl get apiservices | grep metrics
# v1beta1.metrics.k8s.io     kube-system/metrics-server   False   KubeletHasNoNodeCertificate

# Step 6: Install metrics-server (if missing on EKS)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# OR via Helm
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system

# Step 7: For EKS — check if API server can reach metrics-server
# Sometimes TLS issues prevent connectivity
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# If still failing, check for TLS cert validation issues:
kubectl edit deployment metrics-server -n kube-system
# Add to args: --kubelet-insecure-tls (for testing only, use proper certs in prod)

# Step 8: Verify metrics-server is working
kubectl top pods -n production
# Expected:
# NAME                     CPU(cores)   MEMORY(bytes)
# web-app-pod-xxx          485m         128Mi

# Step 9: Check if pods have resource requests set (required for HPA)
kubectl get pod web-app-pod-xxx -n production -o yaml | grep -A5 resources
# If resources.requests.cpu is not set → HPA can't calculate utilization

# Step 10: Verify HPA is now working
kubectl get hpa -n production -w
# NAME       REFERENCE              TARGETS    MINPODS  MAXPODS  REPLICAS  AGE
# web-hpa    Deployment/web-app    95%/70%        2       20        2      20m
# → HPA should now scale up
```

---

## kubectl Commands

```bash
# Create HPA imperatively (quick for testing)
kubectl autoscale deployment web-app \
  --cpu-percent=70 \
  --min=2 \
  --max=20 \
  -n production
# Expected: horizontalpodautoscaler.autoscaling/web-app autoscaled

# List all HPAs
kubectl get hpa -n production
# Expected:
# NAME      REFERENCE              TARGETS   MINPODS  MAXPODS  REPLICAS  AGE
# web-hpa   Deployment/web-app    35%/70%      2        20        4      1h

# Describe HPA (shows current metrics and scaling events)
kubectl describe hpa web-hpa -n production
# Expected: Shows conditions, current/desired replicas, last scale time

# Watch HPA in real-time (useful during load testing)
kubectl get hpa web-hpa -n production -w
# Updates every few seconds, shows metric changes and replica adjustments

# Get HPA in wide format (more detail)
kubectl get hpa -n production -o wide

# Get HPA YAML
kubectl get hpa web-hpa -n production -o yaml

# Check pod resource usage (requires metrics-server)
kubectl top pods -n production
# Expected:
# NAME                    CPU(cores)   MEMORY(bytes)
# web-app-xxxx-abc        250m         128Mi
# web-app-xxxx-def        310m         135Mi

# Check node resource usage
kubectl top nodes
# Expected:
# NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1         1250m        31%    3042Mi          48%

# Manually trigger scale to test (then let HPA take over)
kubectl scale deployment web-app --replicas=5 -n production
# Note: HPA will override this on next sync cycle

# Delete an HPA (reverts to manual scaling)
kubectl delete hpa web-hpa -n production

# Check events related to HPA
kubectl get events -n production --sort-by='.lastTimestamp' | grep HPA
# Expected:
# Normal  SuccessfulRescale  HPA  New size: 8; reason: cpu resource utilization above target

# Check metrics API availability
kubectl get apiservices v1beta1.metrics.k8s.io
# Expected: v1beta1.metrics.k8s.io   kube-system/metrics-server   True   15m
```

---

## YAML Example

```yaml
# =========================================================
# EXAMPLE 1: BASIC HPA — CPU-BASED SCALING (v2 API)
# =========================================================
apiVersion: autoscaling/v2          # Use v2, not deprecated v1
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app                   # Must match deployment name exactly

  minReplicas: 2                    # Never go below 2 (for HA)
  maxReplicas: 20                   # Never exceed 20 (cost control)

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Target 70% CPU across all pods

---
# =========================================================
# EXAMPLE 2: MULTI-METRIC HPA — CPU AND MEMORY
# =========================================================
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api

  minReplicas: 3
  maxReplicas: 50

  metrics:
    # Scale on CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # Scale when avg CPU > 60%

    # Scale on Memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80    # Scale when avg memory > 80%

    # Scale on custom metric: HTTP requests per second
    # Requires prometheus-adapter or similar custom metrics adapter
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second   # Metric exposed via custom metrics API
        target:
          type: AverageValue
          averageValue: "100"              # Scale when avg > 100 req/sec per pod

  # HPA uses the metric that requires the MOST replicas (whichever is highest)
  # This ensures capacity for all resource dimensions

---
# =========================================================
# EXAMPLE 3: HPA WITH BEHAVIOR FIELD (Production-Grade)
# Controls scale-up and scale-down rates
# =========================================================
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-frontend-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-frontend

  minReplicas: 5
  maxReplicas: 100

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

  # -------------------------------------------------------
  # BEHAVIOR FIELD: Fine-grained scaling rate control
  # -------------------------------------------------------
  behavior:
    scaleUp:
      # How long to look back for scale-up stabilization
      # 0 means scale up as fast as needed (don't delay)
      stabilizationWindowSeconds: 0

      # Scaling policies (the more permissive one is selected)
      policies:
        # Policy 1: Add at most 4 pods per 60 seconds
        - type: Pods
          value: 4
          periodSeconds: 60
        # Policy 2: Add at most 100% of current pods per 60 seconds
        - type: Percent
          value: 100
          periodSeconds: 60
      # selectPolicy: Max means use whichever policy allows MORE scaling
      # During a spike from 5→50 pods:
      # - Pods policy: add 4 per minute
      # - Percent policy: add 100% (5 pods) per minute
      # Max selects 5 (percent wins for small deployments)
      selectPolicy: Max

    scaleDown:
      # Wait 5 minutes of sustained low metrics before scaling down
      # Prevents removing pods during brief traffic dips
      stabilizationWindowSeconds: 300

      policies:
        # Remove at most 2 pods per 60 seconds (slow and safe scale-down)
        - type: Pods
          value: 2
          periodSeconds: 60
        # OR remove at most 10% of current pods per 60 seconds
        - type: Percent
          value: 10
          periodSeconds: 60
      # Min means use whichever policy removes FEWER pods (more conservative)
      selectPolicy: Min

---
# =========================================================
# EXAMPLE 4: EXTERNAL METRICS HPA (AWS SQS Queue)
# Scale based on SQS queue depth — process messages faster
# when queue is long, scale down when empty
# =========================================================
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sqs-consumer-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sqs-message-consumer

  minReplicas: 1
  maxReplicas: 50

  metrics:
    # External metric from AWS SQS via external metrics adapter (e.g., kube-metrics-adapter)
    - type: External
      external:
        metric:
          name: sqs_messages_visible           # Metric name from adapter
          selector:
            matchLabels:
              queue-name: order-processing-queue
        target:
          type: AverageValue
          averageValue: "30"                   # 1 pod per 30 messages in queue

---
# =========================================================
# EXAMPLE 5: DEPLOYMENT WITH PROPER RESOURCE REQUESTS
# (Required for HPA to work correctly)
# =========================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3          # HPA will override this after first sync
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "250m"          # REQUIRED for CPU-based HPA
              memory: "256Mi"      # REQUIRED for memory-based HPA
            limits:
              cpu: "500m"
              memory: "512Mi"
          # Readiness probe: HPA only counts READY pods in its calculations
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 5
```

---

## AWS/EKS Perspective

### Installing metrics-server on EKS

metrics-server is NOT installed by default on EKS. It must be added:

```bash
# Method 1: kubectl (official release)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Method 2: Helm (recommended for production — more configurable)
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set replicas=2 \                           # HA deployment
  --set args[0]="--kubelet-preferred-address-types=InternalIP"

# Method 3: EKS Add-on (AWS-managed)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name metrics-server

# Verify installation
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods -A
```

### KEDA on EKS (Event-Driven HPA)

KEDA extends HPA to support 50+ event sources including AWS SQS, DynamoDB Streams, Kinesis, MSK (Kafka):

```bash
# Install KEDA via Helm
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace

# KEDA ScaledObject example for SQS
kubectl apply -f - <<EOF
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: sqs-consumer
  minReplicaCount: 0           # Can scale to ZERO (HPA cannot)
  maxReplicaCount: 50
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
        queueLength: "10"      # 1 pod per 10 messages
        awsRegion: us-east-1
        identityOwner: pod     # Use pod's IRSA role
EOF
```

### Prometheus Adapter for Custom Metrics on EKS

```bash
# Install Prometheus Adapter (after Prometheus is already running)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install prometheus-adapter \
  prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://prometheus-operated.monitoring.svc \
  --set prometheus.port=9090
```

### EKS + HPA Best Practices

- Use **IRSA (IAM Roles for Service Accounts)** when HPA scales pods that access AWS services
- Combine HPA with **Cluster Autoscaler** or **Karpenter** — HPA scales pods, CA/Karpenter scales nodes
- Set **PodDisruptionBudgets** to ensure HPA scale-down doesn't violate availability requirements
- Use **EKS Managed Node Groups** with auto-scaling for the node layer that supports HPA pod scaling

---

## Interview Answer (2-Minute Version)

"The Horizontal Pod Autoscaler automatically adjusts the number of pod replicas based on observed metrics. It runs as a controller in kube-controller-manager, checking metrics every 15 seconds and adjusting replicas to meet a target.

The most important prerequisite is metrics-server — it collects CPU and memory data from kubelets and exposes it via the Kubernetes Metrics API. Without it, HPA can't get CPU or memory metrics.

HPA supports three metric types: resource metrics like CPU and memory, custom metrics like HTTP requests per second via Prometheus adapter, and external metrics for things like SQS queue depth.

The v2 API includes a behavior field that controls scaling rates — you can limit scale-up to 4 pods per minute and scale-down to 2 pods per minute, which prevents thrashing.

The key difference from VPA is HPA adds more pods of the same size; VPA resizes existing pods. HPA is best for stateless services; VPA for stateful workloads or services with a fixed replica count."

---

## Interview Answer (Senior Engineer Version)

"HPA is the horizontal scaling arm of Kubernetes's autoscaling stack. It works by comparing the current metric value to the desired target using the formula: desiredReplicas = ceil(currentReplicas × currentMetric / desiredMetric).

In production I work with all three metric tiers: resource metrics via metrics-server for CPU and memory, custom metrics via prometheus-adapter for application-level signals like HTTP request rate or queue depth, and external metrics via KEDA for AWS SQS, Kinesis, and DynamoDB Streams.

The behavior field is where production tuning happens. I typically set scaleUp stabilizationWindowSeconds to 0 for immediate response to traffic spikes, with a Percent policy to allow exponential growth during load surges. For scaleDown I use a 300-second stabilization window with a conservative Pods policy to prevent thrashing.

One subtle issue teams hit: HPA calculates CPU utilization as actual usage divided by the pod's CPU request, not limit. If pods don't have resource requests set, HPA returns 'unknown' metrics and won't scale. This is a common production footgun.

On EKS I install metrics-server as a Helm chart with 2 replicas for HA, use KEDA for event-driven scaling including scale-to-zero for batch workloads, and always combine HPA with Cluster Autoscaler or Karpenter so the node layer grows alongside the pod layer. I also set PodDisruptionBudgets to ensure HPA scale-down respects minimum availability constraints during rolling updates."

---

## What Impresses the Interviewer

- Explaining the scaling formula (desiredReplicas = ceil[...])
- Knowing that HPA requires CPU requests (not limits) to be set on pods
- Explaining the behavior field with specific policies (Pods, Percent, selectPolicy)
- Mentioning KEDA for event-driven and scale-to-zero scenarios
- Knowing metrics-server is not installed by default on EKS
- Understanding the interaction between HPA and Cluster Autoscaler
- Discussing stabilization windows and why they prevent thrashing
- Mentioning PodDisruptionBudgets as a companion to HPA

---

## Red Flags

- Not knowing metrics-server is required (and not installed by default)
- Confusing HPA with VPA (scaling direction and use cases)
- Not knowing the behavior field exists (v2 API)
- Thinking HPA can work without resource requests set on pods
- Not knowing that scale-down has a 5-minute stabilization window by default
- Theory-only answers without practical troubleshooting experience
- Not mentioning KEDA or custom metrics as scaling sources

---

## Production Best Practices

1. **Always use autoscaling/v2 API:** The v1 API is deprecated and lacks the behavior field. Always use v2 for new deployments.

2. **Set resource requests on all containers:** HPA cannot calculate CPU/memory utilization without pod resource requests. Missing requests is the most common HPA failure cause.

3. **Use the behavior field to prevent thrashing:** Configure scale-down stabilization window (300+ seconds) and add policies to limit scale-down rate. Rapid scale-down followed by scale-up wastes resources and can cause brief capacity drops.

4. **Test HPA with load testing before production:** Use tools like k6, Locust, or Apache Bench to simulate traffic spikes in staging. Verify HPA scales correctly and the cluster has enough capacity.

5. **Combine HPA with PodDisruptionBudgets:** Ensure HPA scale-down can't remove too many pods at once. A PDB with `minAvailable: 2` prevents scaling below 2 replicas even if HPA wants to.

6. **Use custom metrics for accurate scaling:** CPU is a lagging indicator. For web services, scale on HTTP requests per second or active connections. For queue consumers, scale on queue depth. These metrics better reflect actual load.

7. **Monitor HPA events:** Set up alerts for HPA scaling events. Frequent scale-ups indicate under-provisioned baseline capacity. Frequent scale-downs followed by scale-ups indicate thrashing.

8. **Consider KEDA for event-driven workloads:** Standard HPA can't scale to zero (minReplicas must be ≥1). KEDA enables scale-to-zero for batch processors, making idle workloads completely free.

---

## Key Points to Remember

- HPA **requires metrics-server** (CPU/memory) — not installed by default on EKS
- HPA formula: `desiredReplicas = ceil[ currentReplicas × (actual / target) ]`
- Three metric types: **Resource** (CPU/mem), **Custom** (app metrics), **External** (SQS, etc.)
- Resource **requests** (not limits) must be set for CPU/memory HPA to work
- **behavior field** controls scaling rate (policies, stabilization windows)
- Scale-down stabilization default: **300 seconds** (5 minutes)
- Scale-up stabilization default: **0 seconds** (immediate)
- **HPA vs VPA**: HPA = more pods, VPA = bigger pods
- **KEDA** extends HPA for event-driven scaling and scale-to-zero
- Use `autoscaling/v2` API — v1 is deprecated

---

## Interviewer's Expectation

The interviewer is testing whether you understand:

1. **Prerequisites** — metrics-server, resource requests on pods
2. **Scaling formula** — how desired replicas are calculated
3. **Metric types** — resource, custom, external (shows depth of experience)
4. **Behavior field** — production-grade tuning to prevent thrashing
5. **HPA vs VPA** — knowing when to use each
6. **Integration** — how HPA + Cluster Autoscaler work together
7. **Cloud knowledge** — EKS-specific setup, KEDA, prometheus-adapter

Senior candidates should discuss custom metrics, KEDA, stabilization tuning, and the interaction with node autoscaling.

---

## Final Perfect Interview Answer

"The Horizontal Pod Autoscaler automatically scales pod replicas based on observed metrics. It runs as a controller checking every 15 seconds and uses the formula: desired replicas equals ceiling of current replicas multiplied by actual metric over target metric.

The critical prerequisite is metrics-server — it aggregates CPU and memory from kubelets and exposes them via the Metrics API. Without it, HPA returns unknown metrics and won't scale. On EKS it must be explicitly installed.

HPA supports three metric tiers: resource metrics for CPU and memory via metrics-server, custom metrics for application signals like HTTP request rate via Prometheus adapter, and external metrics for AWS SQS or Kinesis via KEDA.

For production use I rely on the behavior field to control scaling rates. I set scaleUp stabilizationWindowSeconds to zero for immediate response to spikes, with a Max policy combining Pods and Percent limits. For scaleDown I use a 300-second stabilization window with a conservative Min policy to prevent thrashing during brief traffic dips.

The key difference from VPA is directional: HPA adds more identically-sized pods; VPA resizes existing pods. I use HPA for stateless services and VPA for stateful workloads with fixed replica counts.

For event-driven workloads I use KEDA, which extends HPA to support scale-to-zero and 50+ event sources including AWS SQS and Kafka topics — something standard HPA cannot do with its minReplicas constraint."
