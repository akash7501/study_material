# Labels and Selectors — Kubernetes Interview Guide

## Interview Question
"Explain Labels and Selectors in Kubernetes. How are they used, and what is the difference between equality-based and set-based selectors?"

---

## Simple Explanation
Imagine you have a huge warehouse with thousands of boxes. Each box has sticky labels on it — "color: red", "size: large", "type: fragile". When you need to find all large red boxes, you ask the warehouse system: "give me everything with color=red AND size=large." That search is a selector.

In Kubernetes:
- **Labels** are sticky tags you put on any Kubernetes object (pods, services, nodes, etc.)
- **Selectors** are the search queries that find objects with matching labels

For example, you label all your web server pods with `app: frontend`. Then when you create a Service, you say "connect to everything with `app: frontend`." Kubernetes finds all those pods automatically using the selector.

This is the foundation of how Kubernetes connects things together:
- Services find their pods using selectors
- Deployments manage their pods using selectors
- Node selectors decide which node a pod goes to
- Network Policies pick which pods to apply rules to

Labels are just metadata — they don't do anything by themselves. The power comes when you use selectors to find and act on groups of objects.

---

## Technical Explanation
**Labels** are key-value pairs attached to Kubernetes objects. They are stored in the object's `metadata.labels` field. Label keys can have an optional prefix (e.g., `app.kubernetes.io/name`) and a name. Values are strings.

**Selectors** are expressions that match labels. There are two types:

1. **Equality-based selectors**: Use `=`, `==`, or `!=` operators.
   - `app=frontend` — match objects where label `app` equals `frontend`
   - `environment!=production` — match objects where `environment` is not `production`

2. **Set-based selectors**: Use `in`, `notin`, `exists` operators. More expressive.
   - `environment in (production, staging)` — matches either value
   - `tier notin (frontend)` — matches anything not frontend
   - `release` — matches if the label key `release` exists (any value)
   - `!release` — matches if the label key `release` does NOT exist

**Internal use of labels:**

- **ReplicaSet/Deployment**: The `spec.selector.matchLabels` field tells the ReplicaSet which pods it "owns." If you change pod labels, the ReplicaSet loses track of the pod (it becomes orphaned).

- **Service**: `spec.selector` uses equality-based matching to build the Endpoints object — the list of pod IPs the Service routes to.

- **NodeSelector and NodeAffinity**: Pods use label selectors on Nodes to constrain scheduling.

- **PodAffinity/Anti-affinity**: Pods use selectors to attract or repel other pods during scheduling.

- **NetworkPolicy**: `podSelector` and `namespaceSelector` use label matching.

**Annotations** (related but different): Also key-value pairs, but designed for non-identifying metadata (tool information, checksums, descriptions). Annotations are NOT used by selectors — they are for human/tool consumption only.

---

## Real-World Example
**Company**: A large media company running 200+ microservices on a shared EKS cluster.

**Label Strategy:**
Every pod, service, and deployment is labeled with a consistent set of keys following the `app.kubernetes.io/` prefix convention:
```
app.kubernetes.io/name: "user-service"        # App name
app.kubernetes.io/version: "v2.3.1"           # Current version
app.kubernetes.io/component: "backend"        # Component role
app.kubernetes.io/part-of: "media-platform"   # Larger application it belongs to
app.kubernetes.io/managed-by: "argocd"        # What manages this resource
environment: "production"                      # Environment
team: "content-team"                           # Owning team
cost-center: "media-eng-1234"                  # For billing
```

**Real use cases:**
1. **Blue-Green deployment**: Label old pods `release: blue`, new pods `release: green`. Switch Service selector from `release: blue` to `release: green` to cut over traffic instantly.
2. **Canary**: Label 10% of pods `track: canary`, 90% `track: stable`. Use weighted Ingress to send 10% of traffic to canary pods.
3. **Node selection**: GPU nodes are labeled `accelerator: nvidia-tesla-v100`. ML training jobs use nodeSelector to only run on those nodes.
4. **Monitoring**: Prometheus uses label selectors to discover which pods to scrape for metrics.

---

## Diagram / Flow

```
LABELS ATTACHED TO KUBERNETES OBJECTS
=======================================

+---------------------------------------------+
|  Pod: web-pod-abc123                        |
|  Labels:                                    |
|    app: frontend                            |
|    version: v2.1                            |
|    environment: production                  |
|    tier: web                                |
|    team: platform                           |
+---------------------------------------------+

+---------------------------------------------+
|  Pod: web-pod-def456                        |
|  Labels:                                    |
|    app: frontend                            |
|    version: v2.1                            |
|    environment: production                  |
|    tier: web                                |
|    team: platform                           |
+---------------------------------------------+

+---------------------------------------------+
|  Pod: api-pod-ghi789                        |
|  Labels:                                    |
|    app: backend                             |
|    version: v1.5                            |
|    environment: production                  |
|    tier: api                                |
|    team: platform                           |
+---------------------------------------------+


HOW SERVICE SELECTOR WORKS
============================

Service:
  selector:
    app: frontend          ←─────────────────────────┐
    environment: production                           │
                                                      │ Matches BOTH labels
Endpoint result:                                      │
  10.244.1.5:80   ←── web-pod-abc123 (has both) ────┘
  10.244.2.7:80   ←── web-pod-def456 (has both)
  (api-pod NOT included — app: backend, not frontend)


DEPLOYMENT → REPLICASET → POD LABEL CHAIN
==========================================

Deployment
  spec:
    selector:
      matchLabels:
        app: frontend       ←──── Deployment manages ReplicaSets with this label
    template:
      metadata:
        labels:
          app: frontend     ←──── Pods created have this label
          version: v2.1
      spec:
        containers: [...]

ReplicaSet (created by Deployment)
  spec:
    selector:
      matchLabels:
        app: frontend       ←──── RS manages Pods with this label
  status:
    replicas: 3             ←──── Counts pods matching selector

Pods (owned by ReplicaSet)
  labels:
    app: frontend ✓         ←──── Matches → ReplicaSet owns this pod
    version: v2.1


EQUALITY-BASED vs SET-BASED SELECTORS
======================================

Equality-based:
  app=frontend              ─── Simple equals
  app==frontend             ─── Same as above
  environment!=staging      ─── Not equals

Set-based:
  environment in (production, staging)    ─── In list
  tier notin (frontend, backend)          ─── Not in list
  release                                 ─── Key exists (any value)
  !beta                                   ─── Key does NOT exist


LABEL SELECTOR IN KUBECTL
===========================

kubectl get pods -l "app=frontend"
kubectl get pods -l "app=frontend,environment=production"
kubectl get pods -l "environment in (production,staging)"
kubectl get pods -l "!beta"


NODE LABEL SELECTORS FOR SCHEDULING
======================================

Node Labels:
  node-type: gpu
  accelerator: nvidia-tesla-v100
  topology.kubernetes.io/zone: us-east-1a
  kubernetes.io/arch: amd64

Pod nodeSelector:
  nodeSelector:
    node-type: gpu           ←── Pod ONLY schedules on GPU nodes
```

---

## Why It Is Important
**Business Value:**
- Enables zero-downtime deployments — blue-green and canary via label switching.
- Supports cost allocation — label pods with team/cost-center, measure per-team resource usage.
- Enables self-service in multi-tenant clusters — teams manage their own labeled resources.

**Technical Value:**
- Fundamental to how Kubernetes connects all objects — without labels, Services, Deployments, and Network Policies cannot function.
- Enables flexible grouping without rigid hierarchy — a pod can have 10 different labels and participate in multiple selector groups.
- Drives automated operations — Prometheus service discovery, Helm release management, ArgoCD sync, Kyverno policy enforcement all use labels.
- Node affinity and pod affinity scheduling decisions rely entirely on labels.

---

## Common Interview Follow-Up Questions
1. What is the difference between Labels and Annotations?
2. What happens if you change the labels on a pod that is managed by a Deployment?
3. What is the difference between `matchLabels` and `matchExpressions`?
4. How do you use labels for blue-green deployments?
5. What are recommended label conventions (like `app.kubernetes.io/` prefix)?
6. How does Prometheus use label selectors for service discovery?
7. What is the difference between nodeSelector and nodeAffinity?
8. Can two Deployments select the same pods? What happens?

---

## Common Mistakes Candidates Make

**Mistake 1: Confusing Labels and Annotations**
- Wrong: "Labels and annotations are basically the same thing — both are key-value pairs."
- Correct: Labels are for identifying objects and are used by selectors. Annotations store non-identifying metadata (descriptions, tool info, checksums) and are NOT used by selectors. Functional difference — labels drive behavior, annotations carry information.

**Mistake 2: Not knowing what happens when you manually change pod labels**
- Wrong: "If I change a pod's label, nothing much happens."
- Correct: If you remove a label that the ReplicaSet selector looks for, the ReplicaSet loses track of that pod. It will create a NEW pod to replace what it thinks was lost. Now you have an extra "orphaned" pod running that the RS does not manage. This can cause unexpected extra pods in production.

**Mistake 3: Using only `matchLabels` and not knowing `matchExpressions`**
- Wrong: "I only use `matchLabels` in my YAMLs."
- Correct: `matchExpressions` is more powerful — supports operators like `In`, `NotIn`, `Exists`, `DoesNotExist`. Used for complex scheduling rules like "schedule on any node EXCEPT zone us-east-1a" which cannot be expressed with simple equality.

**Mistake 4: Overlapping selectors between Deployments**
- Wrong: "Two Deployments can have the same selector labels — the pods just get shared."
- Correct: Two ReplicaSets/Deployments should NEVER have overlapping selectors. Both would try to manage the same pods and fight over them — scaling one up scales the other down because both count the same pods. This causes unexpected behavior and is very hard to debug.

**Mistake 5: Not knowing that Service selector is always equality-based**
- Wrong: "Services can use `in` or `notin` selectors like `matchExpressions`."
- Correct: Service `spec.selector` only supports equality-based selectors (key=value). For set-based selection in Services, you would need to update pod labels or use Endpoint slices manually.

---

## Troubleshooting Scenario
**Problem**: A Service is not routing traffic to any pods, but the pods are running and healthy.

```bash
# Step 1: Check the service
kubectl get service frontend-service -n production
# NAME               TYPE        CLUSTER-IP      PORT(S)   AGE
# frontend-service   ClusterIP   10.96.123.45    80/TCP    5d

# Step 2: Check if there are endpoints (IPs behind the service)
kubectl get endpoints frontend-service -n production
# NAME               ENDPOINTS   AGE
# frontend-service   <none>      5d    <-- No endpoints! Service selector matches nothing

# Step 3: Describe the service — check selector
kubectl describe service frontend-service -n production
# Selector:  app=frontend,environment=production

# Step 4: Check what labels the pods actually have
kubectl get pods -n production --show-labels
# NAME                    READY   STATUS    LABELS
# frontend-abc-7d9f8b     1/1     Running   app=frontend,environment=prod,version=v2.1
#                                                              ^^^^
#                                                         "prod" not "production"!

# Root Cause: Service selector has "environment=production"
#             but pods have "environment=prod"

# Step 5: Fix option A — update pod labels (quick fix, not recommended long term)
kubectl label pod frontend-abc-7d9f8b environment=production --overwrite -n production

# Step 5: Fix option B — update service selector to match pod labels (better)
kubectl patch service frontend-service -n production \
  -p '{"spec":{"selector":{"app":"frontend","environment":"prod"}}}'

# Step 5: Fix option C — fix the Deployment template so new pods have correct label
kubectl edit deployment frontend -n production
# Change: environment: prod
# To:     environment: production

# Step 6: Verify endpoints are now populated
kubectl get endpoints frontend-service -n production
# NAME               ENDPOINTS                          AGE
# frontend-service   10.244.1.5:80,10.244.2.7:80       5d

# Step 7: Test connectivity
kubectl run debug --image=curlimages/curl --rm -it --restart=Never -- \
  curl http://frontend-service.production.svc.cluster.local
# <!DOCTYPE html><html>...   <-- Working!
```

---

## kubectl Commands

```bash
# Add a label to a pod
kubectl label pod my-pod app=frontend -n production

# Update an existing label (--overwrite required)
kubectl label pod my-pod environment=production --overwrite -n production

# Remove a label from a pod (use label-key followed by minus sign)
kubectl label pod my-pod beta- -n production

# Get pods with a specific label
kubectl get pods -l app=frontend -n production
# NAME                  READY   STATUS    RESTARTS   AGE
# frontend-abc-7d9f8b   1/1     Running   0          5d
# frontend-def-8e0g9c   1/1     Running   0          5d

# Get pods with multiple label selectors (AND logic)
kubectl get pods -l "app=frontend,environment=production" -n production

# Get pods using set-based selector
kubectl get pods -l "environment in (production,staging)" -n production

# Get pods where a label key exists
kubectl get pods -l "release" -n production

# Get pods where a label key does NOT exist
kubectl get pods -l "!beta" -n production

# Show all labels on pods
kubectl get pods --show-labels -n production
# NAME                  READY   STATUS    LABELS
# frontend-abc-7d9f8b   1/1     Running   app=frontend,env=production,version=v2.1

# Get all pods in a namespace and filter with label selector
kubectl get pods -n production -l app=frontend -o wide

# Label a node (useful for nodeSelector)
kubectl label node ip-10-0-1-100.us-east-1.compute.internal node-type=gpu

# Remove a label from a node
kubectl label node ip-10-0-1-100.us-east-1.compute.internal node-type-

# Show labels on nodes
kubectl get nodes --show-labels

# Get all resources (pods, services, deployments) with a label
kubectl get all -l app=frontend -n production

# Use labels to delete specific pods
kubectl delete pods -l "version=v1.0,app=frontend" -n production
```

---

## YAML Example

```yaml
# ============================================================
# Deployment with detailed label strategy
# ============================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment              # Deployment name
  namespace: production
  labels:                                # Labels on the DEPLOYMENT OBJECT itself
    app.kubernetes.io/name: frontend     # Recommended: app name (kubernetes standard)
    app.kubernetes.io/version: "v2.1.0"  # Recommended: current version
    app.kubernetes.io/component: web     # Recommended: component role
    app.kubernetes.io/part-of: web-app   # Recommended: larger system it belongs to
    app.kubernetes.io/managed-by: argocd # Recommended: tool managing this resource
    environment: production              # Custom: environment
    team: platform                       # Custom: owning team
    cost-center: "1001"                  # Custom: for billing
spec:
  replicas: 3                            # Number of pod replicas
  selector:
    matchLabels:                         # MUST match pod template labels below
      app: frontend                      # ReplicaSet uses this to track its pods
      environment: production            # Narrows selection for multi-env clusters
  # selector with matchExpressions (more powerful):
  # selector:
  #   matchExpressions:
  #   - key: app
  #     operator: In                     # In, NotIn, Exists, DoesNotExist
  #     values: ["frontend", "web"]
  #   - key: environment
  #     operator: NotIn
  #     values: ["dev"]
  template:
    metadata:
      labels:                            # Labels APPLIED TO PODS — must include selector labels
        app: frontend                    # Used by Service and ReplicaSet selector
        environment: production          # Used by selector above
        version: "v2.1.0"               # Used for canary/blue-green switching
        app.kubernetes.io/name: frontend # Follows Kubernetes recommended label convention
        track: stable                    # Custom: identifies stable vs canary pods
    spec:
      containers:
      - name: frontend
        image: myregistry/frontend:v2.1.0
        ports:
        - containerPort: 80
---
# ============================================================
# Service using equality-based selector
# ============================================================
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: production
  labels:
    app: frontend                        # Labels on the service itself
    environment: production
spec:
  selector:                              # EQUALITY-BASED ONLY (no matchExpressions)
    app: frontend                        # Selects pods with app=frontend
    environment: production              # AND environment=production
    track: stable                        # AND track=stable (excludes canary pods!)
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
# ============================================================
# Canary Service — routes to canary pods only
# ============================================================
apiVersion: v1
kind: Service
metadata:
  name: frontend-canary-service         # Separate service for canary
  namespace: production
spec:
  selector:
    app: frontend
    environment: production
    track: canary                        # Only selects pods labeled track=canary
  ports:
  - port: 80
    targetPort: 80
---
# ============================================================
# Pod with nodeSelector — schedule only on GPU nodes
# ============================================================
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-pod
  namespace: production
  labels:
    app: ml-training                     # Pod's own labels
    workload-type: gpu
spec:
  nodeSelector:                          # Equality-based node label matching
    node-type: gpu                       # Only schedule on nodes labeled node-type=gpu
    accelerator: nvidia-tesla-v100       # AND with this specific GPU type
  containers:
  - name: trainer
    image: pytorch/pytorch:2.0
---
# ============================================================
# Pod with nodeAffinity — more powerful than nodeSelector
# ============================================================
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: production
  labels:
    app: frontend
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:    # HARD requirement
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In                                  # Set-based: In
            values:
            - us-east-1a
            - us-east-1b                                  # Run in either zone
      preferredDuringSchedulingIgnoredDuringExecution:   # SOFT preference
      - weight: 1                                        # Higher weight = stronger preference
        preference:
          matchExpressions:
          - key: node-type
            operator: NotIn                               # Prefer not to use GPU nodes
            values:
            - gpu
    podAntiAffinity:                     # Anti-affinity: spread pods across nodes
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - frontend                 # Avoid scheduling near other frontend pods
          topologyKey: kubernetes.io/hostname  # One per node
  containers:
  - name: frontend
    image: myregistry/frontend:v2.1.0
```

---

## AWS/EKS Perspective

**Node Labels in EKS:**
EKS automatically adds labels to nodes that are very useful for scheduling decisions:

```
topology.kubernetes.io/zone: us-east-1a          # Availability zone
topology.kubernetes.io/region: us-east-1          # AWS region
node.kubernetes.io/instance-type: m5.xlarge       # EC2 instance type
eks.amazonaws.com/nodegroup: my-node-group         # EKS node group name
eks.amazonaws.com/capacityType: ON_DEMAND          # ON_DEMAND or SPOT
kubernetes.io/arch: amd64                          # CPU architecture
kubernetes.io/os: linux                            # Operating system
```

**Key EKS use cases:**

1. **Spot vs On-Demand scheduling**: Label pods with `node-preference: spot` and use nodeAffinity to prefer SPOT nodes for non-critical workloads, saving up to 70% on compute costs.

2. **Multi-AZ pod spread**: Use `topologySpreadConstraints` with `topology.kubernetes.io/zone` to spread pods evenly across AZs.

3. **Karpenter node provisioning**: Karpenter uses pod label selectors to decide which node type to provision — e.g., a pod labeled with specific resource requests gets a matching EC2 instance type.

```yaml
# Schedule on SPOT nodes preferably, fallback to ON_DEMAND
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: eks.amazonaws.com/capacityType
            operator: In
            values: ["SPOT"]

# Spread pods across availability zones
  topologySpreadConstraints:
  - maxSkew: 1                           # Max difference in pod count between zones
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule     # Hard constraint
    labelSelector:
      matchLabels:
        app: frontend                    # Spread pods with this label
```

---

## Interview Answer (2-Minute Version)
"Labels are key-value pairs you attach to Kubernetes objects like pods, services, and nodes. They are just metadata tags. The power comes from selectors, which are queries that find objects with matching labels.

Labels are the fundamental wiring in Kubernetes — Services use selectors to find their pods, Deployments use them to track which pods they own, NetworkPolicies use them to decide which pods to apply rules to, and Prometheus uses them for service discovery.

There are two types of selectors: equality-based — simple key equals value — and set-based, which supports operators like `in`, `not in`, and `exists` for more complex matching. In practice I use labels for blue-green deployments by switching a Service selector from `track: blue` to `track: green`, and for node selection to route GPU workloads to GPU nodes."

---

## Interview Answer (Senior Engineer Version)
"Labels are the identifier system that makes Kubernetes' loosely-coupled architecture work. Almost every controller in Kubernetes uses label selectors to dynamically discover the objects it manages — ReplicaSets track pods, Services build endpoint lists, schedulers apply affinity rules, and operators watch their owned resources, all through label matching.

The key architectural principle is that relationships are established dynamically through selectors, not by hardcoded references. This is powerful: you can add a pod to a Service's backend just by labeling it correctly, without touching the Service definition. But it's also dangerous: changing a pod's labels incorrectly can orphan it from its ReplicaSet, causing the RS to create a duplicate.

In production I follow the `app.kubernetes.io/` prefix conventions from the Kubernetes recommended labels standard. This ensures tooling like Helm, ArgoCD, and Lens can display resources correctly. I also use labels as the foundation for canary deployments — we have a `track: stable` and `track: canary` label on pod templates, with separate Services selecting each track. The Ingress Controller does weighted routing between those two Services, allowing gradual traffic shifting during releases.

For node scheduling on EKS, labels like `eks.amazonaws.com/capacityType=SPOT` combined with podAntiAffinity and topologySpreadConstraints give fine-grained control over cost and availability tradeoffs."

---

## What Impresses the Interviewer
- Explaining that labels drive the loosely-coupled architecture of Kubernetes controllers
- Knowing what happens when you manually change pod labels (ReplicaSet orphaning)
- Describing practical label strategies: blue-green deployment via label switching
- Knowing the `app.kubernetes.io/` label conventions
- Understanding `matchExpressions` vs `matchLabels` difference
- Mentioning topologySpreadConstraints for multi-AZ pod distribution

---

## Red Flags
- Confusing Labels with Annotations
- Not knowing that Services only support equality-based selectors
- Not being able to explain what happens to orphaned pods
- Saying "labels are just cosmetic — they don't affect anything"
- Not knowing how node labels are used for scheduling

---

## Production Best Practices
1. **Follow Kubernetes recommended labels** — use the `app.kubernetes.io/` prefix for standard labels (name, version, component, part-of, managed-by).
2. **Keep label keys consistent** — define a company-wide label taxonomy and enforce it with Kyverno or OPA policies.
3. **Never change selector labels** on live Deployments — immutable in ReplicaSets; requires a rolling update or recreation.
4. **Use labels for cost allocation** — label with team and cost-center, feed into Kubecost for per-team billing visibility.
5. **Label nodes for workload placement** — separate node groups for different workload types (general, GPU, spot), use nodeAffinity to match.
6. **Implement topologySpreadConstraints** for high-availability — spread pods across AZs and nodes using `topology.kubernetes.io/zone` key.
7. **Avoid over-labeling** — too many labels create confusion; settle on 5-8 standard labels per object.
8. **Use set-based selectors in NodeAffinity** — `matchExpressions` with `In`/`NotIn` operators are more maintainable than multiple `nodeSelector` entries.

---

## Key Points to Remember
- Labels are key-value metadata pairs on any Kubernetes object
- Selectors query labels to find matching objects — they power all object relationships
- Services use equality-based selectors; NodeAffinity uses set-based matchExpressions
- Deployments have immutable `.spec.selector` — you cannot change it without recreating
- Manually changing pod labels can orphan pods from their ReplicaSet
- Annotations store non-identifying info — they are NOT used by selectors
- Follow `app.kubernetes.io/` label conventions for standard metadata
- Labels drive blue-green/canary deployments, node scheduling, network policies, monitoring
- `matchLabels` is shorthand for equality; `matchExpressions` supports `In`, `NotIn`, `Exists`, `DoesNotExist`
- Node labels in EKS (zone, instance-type, capacityType) are critical for cost optimization

---

## Interviewer's Expectation
The interviewer is testing whether you understand the core mechanism that makes Kubernetes work — how objects are connected and managed dynamically through labels and selectors. They want to know if you have used labels strategically for deployments, monitoring, and scheduling, not just as decorative metadata.

---

## Final Perfect Interview Answer
"Labels are key-value pairs you attach to Kubernetes objects, and selectors are queries that find objects with matching labels. This combination is the fundamental wiring of Kubernetes — it's how Services find their pods, how ReplicaSets track which pods they manage, how NetworkPolicies decide which pods to restrict, and how Prometheus discovers what to scrape.

There are two selector types: equality-based, which is simple key equals value, and set-based, which supports operators like `in`, `not in`, and `exists` for more expressive matching. Services only support equality-based selectors, while nodeAffinity rules support set-based matchExpressions.

In production, I use labels strategically. For deployments, I label pods with `track: stable` or `track: canary`, with separate Services selecting each track. The Ingress Controller does weighted routing between them for gradual canary releases. For node scheduling on EKS, I use the built-in `eks.amazonaws.com/capacityType` label to prefer SPOT instances for non-critical workloads, significantly reducing costs.

One thing candidates often miss: the Deployment selector is immutable — you cannot change it without recreating the Deployment. And if you manually remove a label from a pod that its ReplicaSet is tracking, the RS thinks it lost a pod and creates a new one, leaving you with an orphaned extra pod in production."
