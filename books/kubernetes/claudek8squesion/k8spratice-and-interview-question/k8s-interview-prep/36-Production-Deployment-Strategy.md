# Production Deployment Strategy — Kubernetes Interview Guide

## Interview Question

"Explain the different deployment strategies available in Kubernetes. When would you choose Rolling Update vs Blue-Green vs Canary vs Recreate? How do you implement zero-downtime deployments in production using ArgoCD Rollouts and Flagger?"

---

## Simple Explanation

When you update an app, you have choices about HOW to replace the old version with the new one:

- **Recreate**: Kill everything, then start new. Simple but causes downtime.
- **Rolling Update**: Replace old pods one by one. No downtime, default in Kubernetes.
- **Blue-Green**: Run two full environments (old=Blue, new=Green), switch traffic instantly.
- **Canary**: Send 5% traffic to new version, test it, then slowly send more.

Think of it like updating a restaurant menu: Recreate = close restaurant, print new menus, reopen. Rolling = update one table's menu at a time. Blue-Green = open a second restaurant, move all customers at once. Canary = give 5 tables the new menu first to get feedback.

---

## Technical Explanation

Kubernetes deployment strategies control how pod updates are rolled out, balancing speed, risk, and availability:

### 1. Recreate Strategy
- Terminates ALL existing pods before creating new ones
- Results in downtime equal to pod startup time
- Uses `strategy.type: Recreate`
- Suitable for stateful apps where two versions cannot coexist

### 2. Rolling Update Strategy (Default)
- Incrementally replaces old pods with new pods
- Controlled by `maxSurge` (extra pods during update) and `maxUnavailable` (pods that can be down)
- Zero-downtime if `maxUnavailable: 0`
- Kubernetes health checks (readinessProbe) gate the rollout

### 3. Blue-Green Deployment
- Two identical environments run in parallel (Blue=current, Green=new)
- Traffic switches via Service selector change or Ingress rule update
- Instant rollback by switching back to Blue
- Doubles infrastructure cost during deployment
- Not natively supported — implemented via label switching or Ingress controllers

### 4. Canary Deployment
- Small percentage of traffic routed to new version
- Gradual traffic shift based on metrics (error rate, latency)
- Automated promotion/rollback via Flagger or ArgoCD Rollouts
- Requires traffic splitting at Ingress or Service Mesh level (Istio, Linkerd)

### 5. A/B Testing
- Route traffic based on user attributes (headers, cookies, region)
- Requires Ingress controller support (NGINX, Istio VirtualService)
- Used for feature flags and user experience testing

---

## Real-World Example

**E-commerce Platform — Black Friday Deployment Strategy:**

A major e-commerce company with 2M daily users needs to deploy a new checkout service (v2.0) on Black Friday eve.

**Chosen Strategy: Canary with Automatic Promotion**

1. Deploy v2.0 to 5% of users first
2. Monitor error rate, latency, and conversion rate for 30 minutes
3. If metrics are healthy, promote to 25% -> 50% -> 100%
4. If error rate exceeds 1%, auto-rollback to v1.9
5. Used Flagger with Prometheus metrics for automated decision-making

**Result**: Zero downtime, caught a database connection pool bug at 5% traffic (only 100K users affected vs 2M), rolled back in 30 seconds.

---

## Diagram / Flow

```
ROLLING UPDATE STRATEGY
========================

Time -->
Pod-1: [v1.0] --> [v2.0 Starting] --> [v2.0 Ready]    DELETED v1.0
Pod-2: [v1.0] --> [v1.0]          --> [v2.0 Starting]  --> [v2.0 Ready]
Pod-3: [v1.0] --> [v1.0]          --> [v1.0]            --> [v2.0 Starting] --> [v2.0 Ready]

maxSurge=1, maxUnavailable=0: Always has 3 pods running, adds 1 extra during rollout


BLUE-GREEN STRATEGY
===================

                    +------------------+
Users --> Ingress-->| Service (Blue)   |-->  Pods v1.0 (Blue)
                    +------------------+
                                            Pods v2.0 (Green) [Warm, ready]

SWITCH: Update Service selector from blue to green

                    +------------------+
Users --> Ingress-->| Service (Green)  |-->  Pods v2.0 (Green)
                    +------------------+
                                            Pods v1.0 (Blue) [Keep for rollback]


CANARY STRATEGY (with Ingress)
================================

                           5% traffic
Users --> Ingress -------> [Canary Pod v2.0]
               |
               |  95% traffic
               +---------> [Stable Pods v1.0 x 10]

        Phase 1: 5%  -> Monitor for 30min -> metrics OK?
        Phase 2: 25% -> Monitor for 30min -> metrics OK?
        Phase 3: 50% -> Monitor for 30min -> metrics OK?
        Phase 4: 100% -> Decommission v1.0

        Auto-rollback if:
          - Error rate > 1%
          - P99 latency > 500ms
          - Success rate < 99%


ARGOCD ROLLOUT FLOW
====================

Git Push v2.0 Image Tag
        |
        v
ArgoCD detects diff in Git repo
        |
        v
ArgoCD Rollout Controller
        |
        +--[Canary Step 1]--> Set Weight 20%
        |                         |
        |                    [Analysis Run]
        |                         |
        |                   Prometheus Query
        |                         |
        |                   Pass? --> Next Step
        |                   Fail? --> Rollback
        |
        +--[Canary Step 2]--> Set Weight 50%
        |                         ...
        |
        +--[Canary Step 3]--> Set Weight 100%
        |
        v
Rollout Complete - v2.0 is Stable
```

---

## Why It Is Important

**Business Value:**
- Zero-downtime deployments protect revenue (1 minute of Amazon downtime = $220K loss)
- Canary deployments reduce blast radius of bad releases
- Blue-Green enables instant rollbacks, reducing MTTR (Mean Time To Recovery)
- Automated deployment strategies reduce human error in production releases

**Technical Value:**
- Kubernetes rolling updates are the foundation of CD pipelines
- Understanding deployment strategies is essential for designing CI/CD pipelines
- Proper strategy selection impacts infrastructure costs, deployment speed, and risk
- Flagger/ArgoCD Rollouts implement GitOps-native progressive delivery

---

## Common Interview Follow-Up Questions

1. **"What is the difference between maxSurge and maxUnavailable?"**
   - `maxSurge`: How many extra pods can exist above desired count during rollout
   - `maxUnavailable`: How many pods can be unavailable during rollout
   - Setting `maxUnavailable: 0` ensures zero downtime but slower rollout

2. **"How do you implement Blue-Green in Kubernetes without a service mesh?"**
   - Use two Deployments with different labels (app: myapp, version: blue/green)
   - Single Service with selector pointing to current active version
   - Update Service selector to switch traffic

3. **"How does Flagger work with Canary deployments?"**
   - Flagger watches Rollout/Deployment objects
   - Creates primary and canary Services automatically
   - Queries Prometheus metrics to make promotion/rollback decisions
   - Integrates with Ingress controllers (NGINX, Istio) for traffic splitting

4. **"What is ArgoCD Rollouts and how does it differ from standard Deployments?"**
   - ArgoCD Rollouts is a CRD that extends Deployment with advanced rollout strategies
   - Supports BlueGreen and Canary strategies natively
   - Integrates with analysis providers (Prometheus, Datadog, CloudWatch)
   - Provides `kubectl argo rollouts` CLI for management

5. **"How do you handle database migrations in Rolling Update deployments?"**
   - Use expand/contract (backward-compatible schema changes)
   - Run migrations as Kubernetes Jobs before deployment
   - Ensure new and old app versions work with the same DB schema

6. **"How would you implement canary without a service mesh?"**
   - NGINX Ingress: `nginx.ingress.kubernetes.io/canary: "true"` annotations
   - Two Services + Ingress weight annotations
   - Limited to percentage-based splitting

7. **"What readiness probe configuration is critical for rolling updates?"**
   - `failureThreshold` and `periodSeconds` determine how quickly bad pods are detected
   - Without proper readiness probes, rolling update can serve errors
   - `initialDelaySeconds` must account for app startup time

---

## Common Mistakes Candidates Make

### Mistake 1: Confusing maxSurge and maxUnavailable defaults
**Wrong**: "Default is maxSurge=1, maxUnavailable=1 means 1 pod is always down"
**Correct**: Default is 25% for both. With 4 replicas, during rollout you can have up to 5 pods (1 surge) and at minimum 3 pods available. Setting `maxUnavailable: 0` ensures no downtime.

### Mistake 2: Saying Blue-Green is "built into Kubernetes"
**Wrong**: "I'll set strategy: BlueGreen in the Deployment spec"
**Correct**: Kubernetes Deployments only support `Recreate` and `RollingUpdate` natively. Blue-Green requires external tools: ArgoCD Rollouts (which has a `BlueGreen` strategy type), Flagger, or manual Service selector switching.

### Mistake 3: Ignoring PodDisruptionBudgets in rolling updates
**Wrong**: Only setting maxUnavailable in the Deployment
**Correct**: PodDisruptionBudget (PDB) is essential — it prevents voluntary disruptions (node drains, rolling updates) from taking too many pods offline simultaneously. Without PDB, a node drain during a rolling update can cause outage.

### Mistake 4: Not considering stateful applications
**Wrong**: "I'll use Rolling Update for my Kafka cluster"
**Correct**: StatefulSets with Recreate or careful Rolling Update considering Kafka's ISR (In-Sync Replicas). Each Kafka broker needs special handling — update one at a time, wait for ISR recovery.

### Mistake 5: Ignoring traffic draining during pod termination
**Wrong**: "Kubernetes stops the pod immediately on rollout"
**Correct**: Configure `terminationGracePeriodSeconds` appropriately. Use `preStop` lifecycle hook with a sleep to allow load balancer to drain connections before the pod terminates.

---

## Troubleshooting Scenario

**Scenario**: Rolling update is stuck — pods are not progressing. The deployment has been in `Progressing` state for 20 minutes.

**Step 1: Check deployment status**
```bash
kubectl rollout status deployment/myapp -n production
# Output: Waiting for deployment "myapp" rollout to finish: 1 out of 3 new replicas have been updated...
```

**Step 2: Describe the deployment**
```bash
kubectl describe deployment myapp -n production
# Look for: Conditions section - check for ReplicaFailure or Progressing=False
```

**Step 3: Check new pod status**
```bash
kubectl get pods -n production -l app=myapp
# Output:
# myapp-7d9f8b-xkj2p   0/1   CrashLoopBackOff   5   10m
# myapp-6c8d7a-abc12   1/1   Running            0   2h
# myapp-6c8d7a-def34   1/1   Running            0   2h
```

**Step 4: Check pod logs and events**
```bash
kubectl logs myapp-7d9f8b-xkj2p -n production --previous
# Output: Error: DATABASE_URL environment variable not set
kubectl describe pod myapp-7d9f8b-xkj2p -n production
# Events: Back-off restarting failed container
```

**Step 5: Root Cause**
New deployment is missing the DATABASE_URL environment variable (new ConfigMap reference is wrong).

**Step 6: Fix and rollback**
```bash
# Option 1: Rollback immediately
kubectl rollout undo deployment/myapp -n production

# Option 2: Fix the issue and reapply
kubectl set env deployment/myapp DATABASE_URL=postgresql://... -n production

# Verify rollback
kubectl rollout status deployment/myapp -n production
kubectl get pods -n production -l app=myapp
```

**Step 7: Set a deployment deadline to prevent infinite hangs**
```yaml
spec:
  progressDeadlineSeconds: 300  # Fail after 5 minutes if not progressing
```

---

## kubectl Commands

```bash
# ===== ROLLING UPDATE COMMANDS =====

# Update image and trigger rolling update
kubectl set image deployment/myapp container=myrepo/myapp:v2.0 -n production

# Check rollout status (live streaming)
kubectl rollout status deployment/myapp -n production
# Expected: deployment "myapp" successfully rolled out

# View rollout history
kubectl rollout history deployment/myapp -n production
# Expected:
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --record
# 2         kubectl set image deployment/myapp...

# View specific revision
kubectl rollout history deployment/myapp --revision=2 -n production

# Rollback to previous version
kubectl rollout undo deployment/myapp -n production

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=1 -n production

# Pause a rolling update (e.g., canary hold)
kubectl rollout pause deployment/myapp -n production

# Resume a paused rollout
kubectl rollout resume deployment/myapp -n production

# ===== BLUE-GREEN COMMANDS =====

# Check current service selector
kubectl get service myapp-service -n production -o jsonpath='{.spec.selector}'
# Expected: {"app":"myapp","version":"blue"}

# Switch from blue to green
kubectl patch service myapp-service -n production \
  -p '{"spec":{"selector":{"app":"myapp","version":"green"}}}'

# Verify switch
kubectl get endpoints myapp-service -n production

# ===== ARGOCD ROLLOUTS COMMANDS =====

# Get rollout status
kubectl argo rollouts get rollout myapp -n production --watch

# Promote canary to next step manually
kubectl argo rollouts promote myapp -n production

# Abort and rollback rollout
kubectl argo rollouts abort myapp -n production

# Retry a failed rollout
kubectl argo rollouts retry rollout myapp -n production

# Set canary weight manually
kubectl argo rollouts set image myapp container=myrepo/myapp:v2.0 -n production

# ===== SCALING AND INSPECTION =====

# Check replica set status during rolling update
kubectl get replicasets -n production -l app=myapp
# Expected:
# NAME              DESIRED   CURRENT   READY   AGE
# myapp-7d9f8b      3         3         3       5m    <- new
# myapp-6c8d7a      0         0         0       2h    <- old

# Watch pods during rolling update
kubectl get pods -n production -l app=myapp -w

# Check PodDisruptionBudget
kubectl get pdb -n production
kubectl describe pdb myapp-pdb -n production

# Annotate deployment for change-cause tracking
kubectl annotate deployment/myapp kubernetes.io/change-cause="v2.0 - added payment gateway" -n production
```

---

## YAML Example

```yaml
# ============================================================
# ROLLING UPDATE STRATEGY (Zero-Downtime)
# ============================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  annotations:
    kubernetes.io/change-cause: "v2.0 - new payment gateway"  # For rollout history
spec:
  replicas: 4
  progressDeadlineSeconds: 300    # Fail if not progressing after 5 min
  revisionHistoryLimit: 5         # Keep last 5 ReplicaSets for rollback
  selector:
    matchLabels:
      app: myapp
      version: v2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                 # Allow 1 extra pod (5 total during rollout)
      maxUnavailable: 0           # Never go below 4 running pods (zero-downtime)
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      terminationGracePeriodSeconds: 60  # Allow 60s to drain connections
      containers:
        - name: myapp
          image: myrepo/myapp:v2.0
          ports:
            - containerPort: 8080
          # CRITICAL: Readiness probe gates rolling update progression
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 15    # Wait for app to start
            periodSeconds: 5           # Check every 5 seconds
            failureThreshold: 3        # 3 failures = not ready
            successThreshold: 1        # 1 success = ready
          # Liveness probe restarts crashed pods
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          # Graceful shutdown hook
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]  # Allow LB to drain
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
# ============================================================
# PODDISRUPTIONBUDGET - Protect availability during updates
# ============================================================
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 3          # At least 3 pods must always be available
  # OR: maxUnavailable: 1  # At most 1 pod can be unavailable
  selector:
    matchLabels:
      app: myapp
---
# ============================================================
# BLUE-GREEN STRATEGY (Manual Implementation)
# ============================================================
# Blue Deployment (current stable)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: production
  labels:
    app: myapp
    slot: blue
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      slot: blue
  template:
    metadata:
      labels:
        app: myapp
        slot: blue
        version: v1.0
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:v1.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
# Green Deployment (new version, warm and ready)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: production
  labels:
    app: myapp
    slot: green
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      slot: green
  template:
    metadata:
      labels:
        app: myapp
        slot: green
        version: v2.0
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:v2.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
# Service - switch between blue and green by changing selector
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp
    slot: blue          # Change to 'green' to switch traffic
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# ============================================================
# CANARY WITH NGINX INGRESS (No Service Mesh Required)
# ============================================================
# Stable Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-stable
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-stable-svc
                port:
                  number: 80
---
# Canary Ingress (5% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"          # Mark as canary
    nginx.ingress.kubernetes.io/canary-weight: "5"      # 5% of traffic
    # Header-based canary (for testing specific users)
    # nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    # nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-canary-svc
                port:
                  number: 80
---
# ============================================================
# ARGOCD ROLLOUT - CANARY STRATEGY
# ============================================================
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:v2.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
  strategy:
    canary:
      canaryService: myapp-canary-svc      # Service for canary pods
      stableService: myapp-stable-svc      # Service for stable pods
      trafficRouting:
        nginx:
          stableIngress: myapp-stable      # Reference to stable Ingress
      # Analysis template for automated promotion/rollback
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2    # Start analysis at step 2
        args:
          - name: service-name
            value: myapp-canary-svc
      steps:
        - setWeight: 5         # Step 1: 5% canary traffic
        - pause: {duration: 5m}  # Wait 5 minutes
        - setWeight: 20        # Step 2: 20% traffic
        - pause: {duration: 10m}
        - setWeight: 50        # Step 3: 50% traffic
        - pause: {duration: 10m}
        - setWeight: 100       # Step 4: Full promotion
---
# Analysis Template for ArgoCD Rollouts
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      # Minimum success rate: 99%
      successCondition: result[0] >= 0.99
      failureLimit: 3     # Allow 3 failures before rollback
      provider:
        prometheus:
          address: http://prometheus-server.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
    - name: avg-latency
      interval: 1m
      successCondition: result[0] <= 0.5   # P99 latency under 500ms
      provider:
        prometheus:
          address: http://prometheus-server.monitoring:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[5m]))
              by (le)
            )
---
# ============================================================
# ARGOCD ROLLOUT - BLUE-GREEN STRATEGY
# ============================================================
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-bluegreen
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp-bluegreen
  template:
    metadata:
      labels:
        app: myapp-bluegreen
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:v2.0
          ports:
            - containerPort: 8080
  strategy:
    blueGreen:
      activeService: myapp-active-svc      # Points to current live version
      previewService: myapp-preview-svc    # Points to new version (for testing)
      autoPromotionEnabled: false          # Require manual promotion
      # autoPromotionSeconds: 300          # Auto-promote after 5 minutes
      scaleDownDelaySeconds: 30            # Keep old version 30s after promotion
      prePromotionAnalysis:                # Run analysis before promoting
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: myapp-preview-svc
---
# ============================================================
# RECREATE STRATEGY (for stateful apps)
# ============================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-monolith
  namespace: production
spec:
  replicas: 1                    # Often used with single-replica stateful apps
  selector:
    matchLabels:
      app: legacy-monolith
  strategy:
    type: Recreate               # Kill all pods, then create new ones
  template:
    metadata:
      labels:
        app: legacy-monolith
    spec:
      containers:
        - name: monolith
          image: myrepo/monolith:v2.0
          # No readiness probe needed - pod must be fully down before new starts
```

---

## AWS/EKS Perspective

### EKS-Specific Deployment Tools

**AWS CodeDeploy with EKS:**
```bash
# EKS integrates with AWS CodeDeploy for Blue/Green via CodePipeline
# Uses Load Balancer Controller for ALB-based traffic shifting
```

**AWS Load Balancer Controller for Canary:**
```yaml
# ALB Ingress with weighted target groups
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/actions.weighted-routing: |
      {"Type":"forward","ForwardConfig":{"TargetGroups":[
        {"ServiceName":"myapp-stable","ServicePort":"80","Weight":95},
        {"ServiceName":"myapp-canary","ServicePort":"80","Weight":5}
      ]}}
```

**EKS Managed Node Groups Rolling Update:**
```bash
# AWS Console or CLI for node group updates
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --release-version 1.29.x-xxxxx

# Check update status
aws eks describe-update \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --update-id <update-id>
```

**Karpenter for Deployment Capacity:**
```yaml
# Karpenter NodePool ensures capacity for blue-green deployments
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: production
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    # Respect PDBs during node consolidation
```

**AWS App Mesh (Service Mesh for Canary):**
- Integrates with EKS for Istio-style traffic splitting
- Virtual Router with weighted targets for canary deployments
- Works with AWS X-Ray for distributed tracing during canary

---

## Interview Answer (2-Minute Version)

"Kubernetes supports four main deployment strategies. The default Rolling Update incrementally replaces pods, controlled by maxSurge and maxUnavailable settings — setting maxUnavailable to zero ensures zero-downtime. Recreate terminates all pods first, causing brief downtime but useful when two versions can't coexist. Blue-Green runs two identical environments in parallel and switches traffic via Service selector update — it's fast to rollback but doubles resource cost. Canary routes a small percentage of traffic to the new version, monitors metrics, then gradually shifts more traffic — ideal for catching bugs before they affect all users.

For zero-downtime in production, I always set maxUnavailable to 0, configure proper readiness probes, and add a preStop sleep hook to drain connections. For higher-risk deployments, I implement canary using NGINX Ingress annotations or ArgoCD Rollouts with Prometheus-based automatic promotion and rollback."

---

## Interview Answer (Senior Engineer Version)

"I choose deployment strategies based on three factors: risk tolerance, rollback speed requirement, and infrastructure cost. For most microservices, Rolling Update with maxUnavailable=0 and maxSurge=1 achieves zero-downtime. But the critical detail is readiness probes — the rolling update uses probe success to gate progression, so a misconfigured probe can either block the rollout or allow bad pods to serve traffic.

For high-risk releases or payment services, I use Canary with ArgoCD Rollouts and automated analysis. The Rollout CRD integrates with Prometheus through AnalysisTemplates — it queries error rate and P99 latency at each traffic weight step, auto-promoting on healthy metrics or rolling back on failures. This removes human judgment from the critical path.

Blue-Green is reserved for major version changes that are difficult to test in production gradually, or where schema changes require an all-or-nothing switch. The key operational concern is PodDisruptionBudgets — without PDBs, a node drain during a rolling update can combine with maxUnavailable to cause an outage.

In EKS, I use the AWS Load Balancer Controller for ALB-based traffic splitting in canary deployments, which gives more precise control than NGINX weight annotations. For stateful workloads like Kafka or databases, I use ordered rolling updates on StatefulSets with partition updates to control exactly which pods are updated."

---

## What Impresses the Interviewer

1. Mentioning PodDisruptionBudgets as a required companion to deployment strategies
2. Explaining that Blue-Green is NOT natively supported and requires ArgoCD Rollouts or manual implementation
3. Discussing preStop lifecycle hooks and terminationGracePeriodSeconds for connection draining
4. Mentioning AnalysisTemplates in ArgoCD Rollouts for metrics-gated promotion
5. Knowing about `progressDeadlineSeconds` to prevent infinite rollout hangs
6. Explaining database migration strategies (expand/contract pattern) with rolling updates
7. Mentioning EKS ALB Controller for AWS-native canary deployments

---

## Red Flags

- Saying "just change the image tag and Kubernetes handles the rest" without explaining the mechanism
- Not knowing that Kubernetes only has Recreate and RollingUpdate natively
- Confusing Deployment strategy with ArgoCD Rollouts BlueGreen strategy
- Not mentioning readiness probes as the gating mechanism for rolling updates
- Not knowing what happens when a rolling update is stuck (how to debug and recover)
- Theory-only answer without mentioning specific kubectl commands or YAML configuration

---

## Production Best Practices

1. **Always set progressDeadlineSeconds**: Without it, a stuck deployment never fails — set it to 2-5x your expected deployment time.

2. **PDB is mandatory for production**: Set minAvailable to N-1 or maxUnavailable to 1 for all production workloads to prevent outages during node maintenance.

3. **Canary with automated rollback**: Never manually monitor canary deployments in production — use Flagger or ArgoCD Rollouts with Prometheus analysis to auto-rollback within seconds.

4. **Connection draining via preStop**: Always add `preStop: exec: sleep 15` to give the load balancer time to remove the pod from rotation before it stops accepting connections.

5. **Test rollback before deployment day**: Execute a rollback drill in staging. A rollback under pressure takes 10x longer than practiced.

6. **Label convention for Blue-Green**: Use consistent label schemes (slot: blue/green) and never use the `version` label in Service selectors — it creates confusion during emergency rollbacks.

7. **Canary traffic minimum**: Start canary at no less than 1% for meaningful signal but no more than 5% to limit blast radius.

8. **Revision history limit**: Set `revisionHistoryLimit: 5` to keep 5 previous ReplicaSets for rollback but not accumulate unlimited old ReplicaSets consuming etcd space.

---

## Key Points to Remember

- Kubernetes natively supports only `Recreate` and `RollingUpdate` strategies
- `maxUnavailable: 0` + `maxSurge: 1` = zero-downtime rolling update
- Blue-Green requires two full deployments + Service selector switching; doubles cost
- Canary requires traffic splitting at Ingress or Service Mesh layer
- ArgoCD Rollouts adds `BlueGreen` and `Canary` as first-class strategies via CRD
- PodDisruptionBudget prevents voluntary disruptions from causing outages
- Readiness probes are the gating mechanism for rolling update progression
- `preStop` hook + `terminationGracePeriodSeconds` enable graceful connection draining
- AnalysisTemplate in ArgoCD Rollouts enables automated metrics-gated promotion
- `kubectl rollout undo` provides instant rollback to previous ReplicaSet

---

## Interviewer's Expectation

The interviewer is testing:
1. **Depth of operational knowledge**: Not just "what" but "how" and "why" with specific configuration details
2. **Risk management mindset**: Understanding blast radius, rollback speed, and automated safety mechanisms
3. **Tooling ecosystem knowledge**: ArgoCD Rollouts, Flagger, NGINX canary annotations, service mesh
4. **Production incident experience**: Debugging stuck rollouts, understanding PDB impact, connection draining
5. **AWS/EKS specifics**: ALB Controller, CodeDeploy integration, Karpenter interaction

Expected level: Senior candidates should know specific flags, YAML fields, and have real debugging stories.

---

## Final Perfect Interview Answer

"In production, I use three deployment strategies based on risk. For standard microservices, Rolling Update with maxUnavailable set to zero and maxSurge of one gives zero-downtime deployments. The key is properly configured readiness probes — Kubernetes uses probe success to gate each pod replacement. I always add a preStop sleep hook for connection draining and set progressDeadlineSeconds to detect stuck rollouts early.

For high-risk changes like payment service updates or major API versions, I use ArgoCD Rollouts with Canary strategy. The Rollout CRD lets me define traffic weight steps — say 5%, 20%, 50%, then 100% — with Prometheus-based AnalysisTemplates that automatically promote or rollback based on error rate and P99 latency thresholds. This removes human judgment during production deployments.

Blue-Green I reserve for cases where two versions cannot coexist — for example when a database schema change is not backward compatible. I run two full deployments and switch the Service selector. ArgoCD Rollouts supports BlueGreen natively with preview services and pre-promotion analysis.

Critical companion to all strategies: PodDisruptionBudget. Without PDB, a node drain during a rolling update can violate availability even when maxUnavailable is zero."

---
*Senior Kubernetes Architect Guide — Production Deployment Strategy*
