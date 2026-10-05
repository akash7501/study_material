# Kubernetes Troubleshooting — Kubernetes Interview Guide

## Interview Question
**"Walk me through your systematic approach to troubleshooting issues in a Kubernetes cluster. How do you diagnose a pod that is not starting? What are the top 10 most common Kubernetes production issues and how do you debug each one?"**

---

## Simple Explanation
Troubleshooting Kubernetes is like being a detective. When something goes wrong, you follow a methodical approach: start at the top (the cluster), then go down layer by layer until you find the culprit.

The golden rule is: **always look at Events first, then Logs, then Describe, then Resource usage.** Events tell you what happened, Logs tell you what the application said, Describe tells you the full state, and Resource usage tells you if you are running out of CPU or memory.

Think of it as checking the dashboard of a car before opening the hood. You read the warning lights first, then you dig deeper.

---

## Technical Explanation
Kubernetes troubleshooting follows a hierarchical diagnostic approach across multiple layers:

**Layer 1 — Cluster Level:** Is the control plane healthy? Are nodes ready?
**Layer 2 — Node Level:** Is the node schedulable? Does it have resources? Is the kubelet running?
**Layer 3 — Pod Level:** Is the pod scheduled? Is the container image pulling? Is the container starting?
**Layer 4 — Application Level:** Is the app crashing? Is it healthy? Is it receiving traffic?
**Layer 5 — Networking Level:** Can the pod communicate? Is DNS working? Are Services routing correctly?

**The Core Diagnostic Toolkit:**
- `kubectl describe` — full object state, events, conditions
- `kubectl logs` — container stdout/stderr
- `kubectl events` — cluster-wide event stream
- `kubectl exec` — interactive debugging inside containers
- `kubectl top` — real-time resource usage (requires metrics-server)
- `kubectl get -o yaml` — raw object spec and status
- `kubectl debug` — ephemeral debug containers (K8s 1.23+)

**Key Status Indicators:**
- Pod phases: Pending → Running → Succeeded/Failed
- Container states: Waiting (with reason) → Running → Terminated (with exit code)
- Node conditions: Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable

---

## Real-World Example
**Production incident at an e-commerce platform:**

At 2:47 AM, PagerDuty fires: "checkout-service has 0 available replicas." The on-call engineer follows this process:

1. `kubectl get pods -n ecommerce` — sees all pods in `CrashLoopBackOff`
2. `kubectl describe pod checkout-xxx` — events show `OOMKilled`
3. A recent deployment doubled the number of replicas, which consumed all available memory on the node
4. The node entered `MemoryPressure`, and the kubelet started evicting pods
5. New pods were scheduled but also OOMKilled because their memory limit was too low for the new traffic spike during a flash sale
6. Fix: increase memory limits, scale horizontally, add a HPA to handle traffic spikes

**Time to resolution: 14 minutes** because the engineer followed a systematic approach rather than guessing.

---

## Diagram / Flow

```
┌─────────────────────────────────────────────────────────────────┐
│           KUBERNETES TROUBLESHOOTING DECISION TREE              │
└─────────────────────────────────────────────────────────────────┘

START: Something is broken
         │
         ▼
┌─────────────────────┐
│  kubectl get pods   │
│  What is the status?│
└─────────────────────┘
         │
         ├── Pending ──────────────────────────────────────────────┐
         │                                                          │
         ├── CrashLoopBackOff ─────────────────────────┐          │
         │                                              │          │
         ├── ImagePullBackOff ──────────────────┐       │          │
         │                                      │       │          │
         ├── OOMKilled ────────────────────┐    │       │          │
         │                                 │    │       │          │
         └── Running but no traffic ───┐   │    │       │          │
                                       │   │    │       │          │
                                       ▼   ▼    ▼       ▼          ▼
┌──────────────────┐  ┌──────────┐  ┌──────┐  ┌──────┐  ┌──────────────┐
│ Check Service,   │  │ Check    │  │Check │  │Check │  │  Check       │
│ Endpoints,       │  │ Logs +   │  │Image │  │ App  │  │  Node        │
│ NetworkPolicy,   │  │ Liveness │  │Name, │  │Memory│  │  Resources,  │
│ DNS              │  │ Probe    │  │Auth  │  │Limits│  │  Affinity,   │
└──────────────────┘  └──────────┘  └──────┘  └──────┘  │  Taints/PVC  │
                                                          └──────────────┘

SYSTEMATIC DIAGNOSIS LAYERS:
═══════════════════════════════════════════════════════════════════

Layer 1: CLUSTER HEALTH
─────────────────────
kubectl get nodes
kubectl get componentstatuses (deprecated but still works)
kubectl get events -A --sort-by='.lastTimestamp' | tail -50

Layer 2: NODE HEALTH  
─────────────────────
kubectl describe node <node-name>
  └── Check: Conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
  └── Check: Capacity vs Allocatable
  └── Check: Pods running on the node
  └── Check: Events at the bottom

Layer 3: POD HEALTH
─────────────────────
kubectl describe pod <pod-name>
  └── Check: Events (most important — read from bottom!)
  └── Check: Container state and reason
  └── Check: Last State (previous container exit code)
  └── Check: Conditions (PodScheduled, ContainersReady, Ready)
  └── Check: Resource requests/limits
  └── Check: Volumes mounted successfully

Layer 4: APPLICATION HEALTH  
─────────────────────────────
kubectl logs <pod-name> --previous  (if currently crashing)
kubectl logs <pod-name> -c <container>  (specific container)
kubectl logs <pod-name> --since=1h  (last hour)

Layer 5: NETWORKING HEALTH
──────────────────────────
kubectl get svc, kubectl get endpoints
kubectl exec -it <debug-pod> -- nslookup <service-name>
kubectl exec -it <debug-pod> -- curl <service-name>:<port>/health
```

---

## Why It Is Important
**Business value:**
- Faster MTTR (Mean Time to Repair) directly reduces revenue impact during outages
- Systematic troubleshooting prevents "shotgun" approaches that can make problems worse
- Understanding root causes prevents recurrence, reducing operational toil

**Technical value:**
- Kubernetes has many interacting components — a methodical approach prevents missing the actual root cause
- Production systems have PodDisruptionBudgets and rolling update requirements — wrong troubleshooting steps can make outages worse
- Evidence-based debugging (logs, events, metrics) builds institutional knowledge

---

## Common Interview Follow-Up Questions

1. **"A pod is stuck in Pending state. Walk me through exactly how you diagnose it."**
2. **"How do you debug a pod that is Running but not receiving any traffic?"**
3. **"How do you investigate a node that shows NotReady?"**
4. **"What is the difference between `kubectl logs` and `kubectl describe`? When do you use each?"**
5. **"How do you debug networking issues between two pods in different namespaces?"**
6. **"A deployment rollout is stuck at 50%. What do you check?"**
7. **"How do you debug issues without the application having a shell installed?"**
8. **"What observability tools do you use alongside kubectl for production debugging?"**

---

## Common Mistakes Candidates Make

**Mistake 1: Starting with logs before checking events.**
- Correct: Always check `kubectl describe pod` first. Events tell you WHY the pod cannot start before it even reaches the logging stage. Logs are only available after the container starts.

**Mistake 2: Not using `--previous` flag when a container is crashing.**
- Correct: When a pod is in CrashLoopBackOff, `kubectl logs <pod>` shows the CURRENT (failed) container. You need `kubectl logs <pod> --previous` to see what happened in the last run before the crash.

**Mistake 3: Forgetting that Pending pods have no logs.**
- Correct: A Pending pod has never run — there are zero logs. All diagnostic information is in `kubectl describe pod` events section. Trying to read logs is a waste of time.

**Mistake 4: Not checking namespace when pod names are not found.**
- Correct: Always use `-n <namespace>` or `-A` for cluster-wide. More than 80% of "I can't find the pod" issues are namespace problems.

**Mistake 5: Not checking resource quotas and LimitRanges.**
- Correct: Pods can be stuck in Pending because the namespace ResourceQuota is exhausted, or because a LimitRange enforces minimum resource requirements that are not met.

---

## Troubleshooting Scenario
**Production scenario: "Deployment rolled out but users are getting 503 errors."**

```
Step 1: Check pod status
$ kubectl get pods -n production -l app=api-server
NAME                          READY   STATUS    RESTARTS   AGE
api-server-7d9f4b6c-xk2p9    0/1     Running   0          5m
api-server-7d9f4b6c-mnq7r    0/1     Running   0          5m

Observation: Pods are Running but READY is 0/1

Step 2: Describe a pod
$ kubectl describe pod api-server-7d9f4b6c-xk2p9 -n production

Conditions:
  Type              Status
  Initialized       True
  Ready             False   ← POD NOT READY
  ContainersReady   False
  PodScheduled      True

Events:
  Warning  Unhealthy  30s  kubelet  Readiness probe failed: 
           HTTP probe failed with statuscode: 500

Observation: Readiness probe failing — app is starting but returning 500

Step 3: Check logs for application error
$ kubectl logs api-server-7d9f4b6c-xk2p9 -n production --since=5m

ERROR: Failed to connect to database: dial tcp 10.0.5.23:5432: 
       connection refused

Observation: Cannot connect to PostgreSQL

Step 4: Check if database pod is running
$ kubectl get pods -n production -l app=postgres
NAME              READY   STATUS    RESTARTS
postgres-0        0/1     Pending   0

Step 5: Describe database pod
$ kubectl describe pod postgres-0 -n production

Events:
  Warning  FailedScheduling  2m  default-scheduler  
           0/5 nodes are available: 5 Insufficient memory.

Observation: Postgres PVC PVC grew too large + recent scale-up 
             consumed all available memory

ROOT CAUSE: Memory exhaustion caused postgres to be evicted, 
            api-server readiness probe fails → pods show 0/1 Ready 
            → Service has no healthy endpoints → users get 503

FIX:
1. Add nodes or scale down other workloads to free memory
2. Once postgres pod starts, api-server readiness probe will pass
3. Long-term: add resource quotas and VPA for postgres
```

---

## kubectl Commands

```bash
# ══════════════════════════════════════════════════════════════════
# SECTION 1: CLUSTER-LEVEL DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Check all nodes
kubectl get nodes -o wide
# NAME              STATUS   ROLES    AGE   VERSION   INTERNAL-IP
# node-1            Ready    <none>   10d   v1.29.0   10.0.1.10
# node-2            NotReady <none>   10d   v1.29.0   10.0.1.11  ← PROBLEM

# Describe a NotReady node
kubectl describe node node-2
# Look for: Conditions, Events, kubelet status

# Check recent cluster-wide events (most useful for incident triage)
kubectl get events -A --sort-by='.lastTimestamp'

# Filter only Warning events
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp'

# ══════════════════════════════════════════════════════════════════
# SECTION 2: POD DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Get all pods with detailed status
kubectl get pods -n <namespace> -o wide

# Get pods in all namespaces (incident triage)
kubectl get pods -A | grep -v Running | grep -v Completed

# Describe pod (THE most important command)
kubectl describe pod <pod-name> -n <namespace>
# READ: Events section at bottom — most critical information

# Get pod logs (current container)
kubectl logs <pod-name> -n <namespace>

# Get logs of PREVIOUS container run (for CrashLoopBackOff)
kubectl logs <pod-name> -n <namespace> --previous

# Follow logs in real time
kubectl logs <pod-name> -n <namespace> -f

# Get logs from a specific container in multi-container pod
kubectl logs <pod-name> -n <namespace> -c <container-name>

# Get logs from last N lines
kubectl logs <pod-name> -n <namespace> --tail=100

# Get logs from last 1 hour
kubectl logs <pod-name> -n <namespace> --since=1h

# Get pod YAML with full status
kubectl get pod <pod-name> -n <namespace> -o yaml

# Check pod conditions specifically
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.conditions[*]}'

# Check container exit code
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# ══════════════════════════════════════════════════════════════════
# SECTION 3: INTERACTIVE DEBUGGING
# ══════════════════════════════════════════════════════════════════

# Exec into a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Run a one-off debug command
kubectl exec <pod-name> -n <namespace> -- env | grep -i database

# Create ephemeral debug container (K8s 1.23+)
kubectl debug -it <pod-name> \
  --image=busybox \
  --target=<container-name> \
  -n <namespace>

# Create a temporary debug pod in the same namespace
kubectl run debug-pod \
  --image=nicolaka/netshoot \
  --rm -it \
  --restart=Never \
  -n <namespace> \
  -- /bin/bash

# Debug a node (creates privileged pod on the node)
kubectl debug node/<node-name> \
  -it \
  --image=ubuntu \
  -- bash

# ══════════════════════════════════════════════════════════════════
# SECTION 4: RESOURCE USAGE DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Node resource usage (requires metrics-server)
kubectl top nodes
# NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1    850m         21%    3200Mi          80%    ← HIGH MEMORY!
# node-2    200m         5%     1000Mi          25%

# Pod resource usage
kubectl top pods -n <namespace>
# NAME                      CPU(cores)   MEMORY(bytes)
# api-server-7d9f-xk2p9    450m         900Mi

# Sort by memory
kubectl top pods -A --sort-by=memory

# Sort by CPU
kubectl top pods -A --sort-by=cpu

# ══════════════════════════════════════════════════════════════════
# SECTION 5: DEPLOYMENT DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Check deployment status
kubectl rollout status deployment/<name> -n <namespace>
# Waiting for deployment "api-server" rollout to finish: 
# 2 out of 5 new replicas have been updated...

# Check deployment history
kubectl rollout history deployment/<name> -n <namespace>
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         Update image to v1.5.0
# 3         Update image to v1.6.0   ← current (broken)

# Rollback to previous version
kubectl rollout undo deployment/<name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=2 -n <namespace>

# ══════════════════════════════════════════════════════════════════
# SECTION 6: NETWORKING DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>
# NAME          ENDPOINTS           AGE
# api-service   10.0.1.5:8080       10m
# db-service    <none>              10m  ← NO ENDPOINTS = problem!

# Describe a service
kubectl describe svc <service-name> -n <namespace>

# Check NetworkPolicy
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>

# DNS debugging inside a pod
kubectl exec -it <pod-name> -n <namespace> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>.<namespace>.svc.cluster.local

# Test HTTP connectivity between pods
kubectl exec -it <pod-name> -n <namespace> -- curl -v http://<service-name>:80/health

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# ══════════════════════════════════════════════════════════════════
# SECTION 7: CONFIGMAP / SECRET DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Check if ConfigMap exists
kubectl get configmap -n <namespace>
kubectl describe configmap <name> -n <namespace>

# Check if Secret exists
kubectl get secrets -n <namespace>
kubectl describe secret <name> -n <namespace>
# Note: describe shows keys but NOT values (security)

# Decode a secret value
kubectl get secret <name> -n <namespace> \
  -o jsonpath='{.data.password}' | base64 --decode

# ══════════════════════════════════════════════════════════════════
# SECTION 8: PERSISTENT VOLUME DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Check PVCs
kubectl get pvc -n <namespace>
# NAME        STATUS    VOLUME    CAPACITY   ACCESS MODES
# data-pvc    Pending   <none>    <none>     <none>      ← PROBLEM

# Describe PVC for details
kubectl describe pvc data-pvc -n <namespace>
# Events: no persistent volumes available for this claim

# Check PVs
kubectl get pv
kubectl describe pv <pv-name>

# Check StorageClass
kubectl get storageclass
kubectl describe storageclass <name>

# ══════════════════════════════════════════════════════════════════
# SECTION 9: RBAC DIAGNOSTICS
# ══════════════════════════════════════════════════════════════════

# Check if a service account can do something
kubectl auth can-i get pods \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>
# yes/no

# Check RBAC for current user
kubectl auth can-i create deployments -n production
kubectl auth can-i "*" "*"  # Check if cluster-admin

# ══════════════════════════════════════════════════════════════════
# SECTION 10: SYSTEM COMPONENT HEALTH
# ══════════════════════════════════════════════════════════════════

# Check kube-system pods
kubectl get pods -n kube-system

# Check CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Check metrics-server
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Check controller-manager (if accessible)
kubectl logs -n kube-system -l component=kube-controller-manager
```

---

## YAML Example

```yaml
# ── LIVENESS AND READINESS PROBES (prevent false "Running" pods) ───
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: myapp:v1.5.0
          ports:
            - containerPort: 8080
          
          # Readiness probe: is this pod ready to receive traffic?
          # Pod is removed from Service endpoints if this fails
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10  # Wait 10s before first check
            periodSeconds: 5         # Check every 5 seconds
            failureThreshold: 3      # Remove from endpoints after 3 failures
            successThreshold: 1      # Add back after 1 success
          
          # Liveness probe: is this pod alive? Restart if not
          # Pod is restarted if this fails
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30  # Give app time to start (30s)
            periodSeconds: 10        # Check every 10 seconds
            failureThreshold: 3      # Restart after 3 consecutive failures
            timeoutSeconds: 5        # Probe times out after 5s
          
          # Startup probe: for slow-starting apps (prevents liveness killing it)
          startupProbe:
            httpGet:
              path: /health/started
              port: 8080
            failureThreshold: 30     # Allow up to 5 minutes (30 * 10s) to start
            periodSeconds: 10
          
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          
          # Environment from ConfigMap and Secret
          envFrom:
            - configMapRef:
                name: api-server-config
                optional: false     # Pod fails if ConfigMap missing
            - secretRef:
                name: api-server-secrets
                optional: false     # Pod fails if Secret missing

---
# ── NETWORK POLICY (for debugging networking issues) ──────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # Allow traffic from frontend pods
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
    # Allow health checks from monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  
  egress:
    # Allow DNS (ALWAYS include this or DNS breaks!)
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow database access
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

---
# ── POD DISRUPTION BUDGET (protection during node drains) ─────────
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  # At least 2 pods must be available at all times during disruptions
  minAvailable: 2
  # OR use maxUnavailable: 1
  selector:
    matchLabels:
      app: api-server

---
# ── RESOURCE QUOTA (prevent namespace resource exhaustion) ────────
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"          # Max 20 CPU cores requested
    requests.memory: 40Gi       # Max 40Gi memory requested
    limits.cpu: "40"            # Max 40 CPU cores limits
    limits.memory: 80Gi         # Max 80Gi memory limits
    pods: "100"                 # Max 100 pods
    services: "20"              # Max 20 services
    persistentvolumeclaims: "10" # Max 10 PVCs

---
# ── DEBUG POD (use for network/DNS troubleshooting) ───────────────
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: production
  labels:
    purpose: debugging
spec:
  # Auto-delete after 1 hour
  activeDeadlineSeconds: 3600
  containers:
    - name: netshoot
      image: nicolaka/netshoot:latest
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi
  restartPolicy: Never
```

---

## AWS/EKS Perspective

**EKS-specific troubleshooting tools:**

```bash
# Check EKS control plane logs in CloudWatch
aws logs describe-log-groups --log-group-name-prefix /aws/eks/<cluster-name>

# EKS log groups:
# /aws/eks/<cluster>/cluster          → API server, controller-manager, scheduler
# /aws/eks/<cluster>/authenticator    → IAM authentication issues
# /aws/eks/<cluster>/audit            → K8s audit log

# Stream authenticator logs (IAM/IRSA issues)
aws logs tail /aws/eks/prod-cluster/cluster --follow

# Check worker node bootstrap issues (userdata/cloud-init)
# SSH to node or use EC2 Systems Manager:
aws ssm start-session --target <instance-id>
# Then check: /var/log/cloud-init-output.log

# VPC CNI troubleshooting
kubectl describe daemonset aws-node -n kube-system
kubectl logs -n kube-system -l k8s-app=aws-node --tail=50

# Check EC2 ENI limits (too many pods per node)
aws ec2 describe-instance-types \
  --instance-types m5.xlarge \
  --query "InstanceTypes[0].NetworkInfo.MaximumNetworkInterfaces"

# Check Node taints from EKS (e.g., node not yet ready)
kubectl describe node <node-name> | grep Taint

# EKS Managed Node Group health
aws eks describe-nodegroup \
  --cluster-name prod-cluster \
  --nodegroup-name general \
  --query "nodegroup.health"

# Container Insights metrics (if enabled)
aws cloudwatch get-metric-statistics \
  --namespace ContainerInsights \
  --metric-name pod_memory_utilization \
  --dimensions Name=ClusterName,Value=prod-cluster \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 60 \
  --statistics Average
```

---

## Top 10 Common Kubernetes Issues

```
ISSUE 1: Pod Stuck in Pending
══════════════════════════════
Causes:
  a) No nodes have enough resources
  b) Node selector / affinity not matching any nodes
  c) Taint without matching toleration
  d) PVC not bound
  e) ResourceQuota exceeded

Diagnose:
  kubectl describe pod <name> → Events: "0/N nodes available"
  kubectl describe node → check Allocatable vs Requests
  kubectl get pvc → STATUS Pending?
  kubectl describe resourcequota -n <namespace>

Fix:
  Add nodes / free up resources / fix affinity/taint
  Provision PV or fix StorageClass
  Increase ResourceQuota

────────────────────────────────────────────────────────────────────
ISSUE 2: CrashLoopBackOff
══════════════════════════
Causes:
  a) Application crash on startup
  b) OOMKilled (exit code 137)
  c) Wrong startup command
  d) Missing env var/ConfigMap/Secret
  e) Liveness probe too aggressive

Diagnose:
  kubectl logs <pod> --previous
  kubectl describe pod <pod> → Last State exit code
  kubectl get pod <pod> -o yaml → check command/args

────────────────────────────────────────────────────────────────────
ISSUE 3: ImagePullBackOff
══════════════════════════
Causes:
  a) Wrong image name or tag
  b) Private registry without imagePullSecret
  c) Registry rate limiting
  d) Network connectivity to registry

Diagnose:
  kubectl describe pod <pod> → Events: "Failed to pull image"
  kubectl get secret -n <namespace> → check imagePullSecret exists

────────────────────────────────────────────────────────────────────
ISSUE 4: Service Not Routing Traffic
══════════════════════════════════════
Causes:
  a) Label selector mismatch (most common!)
  b) Pod readiness probe failing → not added to endpoints
  c) Wrong port number
  d) NetworkPolicy blocking

Diagnose:
  kubectl get endpoints <svc-name> → is it empty?
  kubectl get pods --show-labels → do labels match service selector?
  kubectl describe svc <name> → check Selector field
  kubectl get networkpolicy -n <namespace>

────────────────────────────────────────────────────────────────────
ISSUE 5: Node NotReady
═══════════════════════
Causes:
  a) kubelet not running on the node
  b) Node ran out of disk space (DiskPressure)
  c) Node ran out of memory (MemoryPressure)
  d) Network connectivity issue
  e) Certificate expired

Diagnose:
  kubectl describe node <name> → Conditions section
  SSH to node: systemctl status kubelet
  Check: journalctl -u kubelet -n 100

────────────────────────────────────────────────────────────────────
ISSUE 6: OOMKilled
═══════════════════
Causes:
  a) Memory limit too low
  b) Memory leak in application
  c) JVM heap not configured for container limits

Diagnose:
  kubectl describe pod <name> → OOMKilled in Last State
  kubectl top pods → current memory usage
  Check JVM -Xmx vs container memory limit

────────────────────────────────────────────────────────────────────
ISSUE 7: Deployment Rollout Stuck
══════════════════════════════════
Causes:
  a) New pods failing health checks
  b) Not enough nodes for new replicas (maxSurge)
  c) Image pull failure
  d) PDB preventing pod termination

Diagnose:
  kubectl rollout status deployment/<name>
  kubectl describe deployment <name> → Conditions
  kubectl get pods -l <deployment-selector>
  kubectl describe pdb -n <namespace>

────────────────────────────────────────────────────────────────────
ISSUE 8: DNS Resolution Failure
════════════════════════════════
Causes:
  a) CoreDNS pods not running
  b) NetworkPolicy blocking port 53
  c) ndots configuration issue
  d) Custom DNS overrides conflict

Diagnose:
  kubectl get pods -n kube-system -l k8s-app=kube-dns
  kubectl exec -it <pod> -- nslookup kubernetes.default
  kubectl logs -n kube-system -l k8s-app=kube-dns

────────────────────────────────────────────────────────────────────
ISSUE 9: ConfigMap/Secret Not Mounted
══════════════════════════════════════
Causes:
  a) ConfigMap/Secret does not exist in the namespace
  b) Pod spec references wrong name
  c) Key name mismatch

Diagnose:
  kubectl describe pod <name> → Events: "configmap not found"
  kubectl get cm,secret -n <namespace>
  Compare pod spec name vs actual CM/Secret name

────────────────────────────────────────────────────────────────────
ISSUE 10: Horizontal Pod Autoscaler Not Scaling
════════════════════════════════════════════════
Causes:
  a) metrics-server not installed
  b) Pods missing resource requests (HPA needs requests)
  c) Target metric not being emitted
  d) Min/max replicas already at limit

Diagnose:
  kubectl describe hpa <name>
  kubectl get apiservices | grep metrics
  kubectl top pods → can metrics-server read pod metrics?
  Check: resource requests defined on all pods
```

---

## Interview Answer (2-Minute Version)
"My troubleshooting approach is layered. I start with `kubectl get pods` to see the status, then `kubectl describe pod` to read the Events section — that is almost always where the answer is. Then I check `kubectl logs` with `--previous` if the container has crashed. For Pending pods, I look at node resources and scheduling constraints. For Running-but-unhealthy pods, I check readiness probe failures in Events and endpoints to see if the pod is actually receiving traffic.

The top issues I encounter are: CrashLoopBackOff from application crashes or missing config, ImagePullBackOff from wrong image names or missing imagePullSecrets, Pending pods from resource constraints or affinity mismatches, and services not routing because of label selector mismatches.

I always check events cluster-wide with `kubectl get events -A --sort-by=lastTimestamp` during incidents because it gives me a timeline of what happened."

---

## Interview Answer (Senior Engineer Version)
"I troubleshoot Kubernetes in layers. Starting at the cluster level: are nodes Ready? Are there Warning events cluster-wide? Then at the pod level: describe always comes before logs because logs do not exist until the container starts. For Pending pods, describe tells me everything — scheduling failures, resource constraints, affinity violations, PVC binding issues.

For CrashLoopBackOff, I look at the exit code in Last State: 137 means OOMKilled, 1 or 2 is an app crash, 128+N is a signal. I use `--previous` to get the logs from the crashed container.

For networking, I always verify endpoints first. An empty endpoints object means either no pods match the service selector, or all matching pods have a failing readiness probe. I use ephemeral debug containers or a netshoot pod to do live network testing — DNS lookups, curl, tcp checks.

In production on EKS, I complement kubectl with CloudWatch Container Insights for historical metrics, and I use the EKS control plane audit logs when I need to trace RBAC or authentication failures. For chronic issues, I set up Prometheus alerts on pod restart counts, OOMKilled events, and pending pod durations so I catch problems before users notice them."

---

## What Impresses the Interviewer
- Mentioning exit codes and what they mean (137 = OOMKilled, 1 = app crash)
- Knowing `--previous` flag for crashed containers
- Understanding that Pending pods have no logs (describe is the only tool)
- Checking endpoints before assuming service is broken
- Using ephemeral debug containers (shows modern K8s knowledge)
- Discussing observability tools beyond kubectl (Prometheus, CloudWatch Container Insights)
- Mentioning PodDisruptionBudgets and how they can block rollouts
- Being specific about EKS-specific tools (CloudWatch Logs Insights, SSM for nodes)

---

## Red Flags
- "I restart the pod and see if it fixes itself"
- Cannot explain the difference between `kubectl logs` and `kubectl describe`
- Tries to read logs for a Pending pod
- Doesn't know what `--previous` does
- Says "I would check the application code" without first using kubectl to diagnose
- Cannot explain what an empty Endpoints object means

---

## Production Best Practices

1. **Always use structured logging in applications** — JSON logs are parseable by CloudWatch Logs Insights, Elasticsearch, and Loki. `kubectl logs` readable but machine-searchable at scale.

2. **Instrument every service with readiness and liveness probes** — Without probes, a starting-but-not-ready pod receives traffic and appears Ready even if it is broken.

3. **Set resource requests and limits on every container** — Without requests, the Scheduler cannot make good placement decisions. Without limits, one runaway pod can starve an entire node.

4. **Use PodDisruptionBudgets for all production workloads** — Prevents node drains, cluster upgrades, and Karpenter consolidation from causing outages.

5. **Enable cluster-level logging to an external system** — kubectl logs is only available while the pod exists. Ship logs to CloudWatch, Elasticsearch, or Loki for post-mortem analysis.

6. **Set up Prometheus alerts for pod restart counts** — A pod restarting 3 times in 10 minutes is a leading indicator of an impending outage.

7. **Use `kubectl top` with metrics-server for baseline CPU/memory** — Know what normal looks like so you can spot anomalies instantly during incidents.

8. **Document runbooks for the top 5 recurring issues** — Institutional knowledge in runbooks reduces MTTR during 2 AM incidents.

---

## Key Points to Remember
- Troubleshoot in layers: Cluster → Node → Pod → Application → Network
- `kubectl describe` Events section is the most valuable information for pod problems
- `kubectl logs --previous` is required for CrashLoopBackOff diagnosis
- Pending pods have zero logs — use describe only
- Exit code 137 = OOMKilled, Exit code 1 = application crash
- Empty Endpoints = service selector mismatch OR readiness probe failing
- `kubectl get events -A --sort-by=lastTimestamp` for cluster-wide incident triage
- Ephemeral debug containers for distroless/minimal images
- NetworkPolicy port 53 rule is mandatory — forgetting it breaks DNS
- RBAC issues: use `kubectl auth can-i` to verify permissions

---

## Interviewer's Expectation
The interviewer is testing whether you:
1. Follow a **systematic approach** rather than guessing
2. Know the **right commands** for each type of problem
3. Can **interpret kubectl output** (exit codes, conditions, events)
4. Have dealt with **real production issues** not just textbook scenarios
5. Understand the **relationship between objects** (Service → Endpoints → Pods → Readiness)
6. Know **when to use what tool** (describe vs logs vs exec vs top)
7. Have **production-grade habits** (PDBs, resource limits, probes, logging)

---

## Final Perfect Interview Answer
"My troubleshooting approach starts at the cluster level and drills down systematically. First, I check node health with `kubectl get nodes`, then get a cluster-wide event summary sorted by time to understand what happened and when. For individual pod issues, I always read `kubectl describe pod` before logs because events tell me why the pod cannot start, which I cannot see in logs if the container never launched.

For CrashLoopBackOff, the exit code in Last State guides me immediately — 137 means OOMKilled, 1 is an app crash. I use `kubectl logs --previous` to see the output from the crashed container. For Pending pods, I check scheduling events for resource constraints, affinity mismatches, or unbound PVCs.

For networking issues, I verify endpoints first — an empty endpoints object tells me the service is not finding its pods. I use debug pods or ephemeral containers for live network testing. The top 10 issues I debug regularly are: CrashLoopBackOff, ImagePullBackOff, OOMKilled, Pending from resource exhaustion, service routing failures from label mismatches, DNS failures, stuck rollouts, PVC binding failures, RBAC errors, and Node NotReady.

In production on EKS, I complement kubectl with CloudWatch Container Insights for historical metrics and set up Prometheus alerts on restart counts to catch problems before users do."
