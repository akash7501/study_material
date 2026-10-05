# Namespace — Kubernetes Interview Guide

## Interview Question
"What are Kubernetes Namespaces, why do companies use them, and how have you structured namespaces in a production environment?"

---

## Simple Explanation
Imagine a large office building shared by multiple companies. Each company has its own floor. On each floor, the company has its own people (pods), its own equipment (services), and its own rules (policies). People from Company A cannot just walk into Company B's floor without permission.

Namespaces in Kubernetes are exactly like those floors. They divide ONE Kubernetes cluster into multiple virtual clusters. Each team or application gets its own "floor" (namespace) where:
- Their pods run in isolation from other teams' pods
- They have their own services, secrets, and config maps
- You can set resource limits per floor (so one team cannot use all the memory)
- You can control who can access each floor

Without namespaces, everything runs in one big open space — a team could accidentally delete another team's database, or one buggy app could consume all cluster resources.

**Default namespaces Kubernetes creates:**
- `default` — where things go if you don't specify a namespace
- `kube-system` — Kubernetes own components (DNS, controller manager, etc.)
- `kube-public` — publicly readable data (rarely used directly)
- `kube-node-lease` — heartbeat data for nodes

---

## Technical Explanation
A **Namespace** is a Kubernetes API object that provides a mechanism for isolating groups of resources within a single cluster. It is a virtual partition of the cluster.

**How namespaces work internally:**

1. **Resource scoping**: Most Kubernetes objects are namespace-scoped (Pods, Services, Deployments, PVCs, ConfigMaps, Secrets, etc.). Some are cluster-scoped and exist outside namespaces (Nodes, PersistentVolumes, ClusterRoles, StorageClasses).

2. **DNS isolation**: Kubernetes DNS uses the format `<service>.<namespace>.svc.cluster.local`. Services in the same namespace can talk using just the service name. Cross-namespace requires the full DNS name.

3. **RBAC scoping**: Role and RoleBinding objects are namespace-scoped. This allows you to grant a user admin rights in `dev` namespace but read-only in `production`. ClusterRole and ClusterRoleBinding span all namespaces.

4. **Resource quotas**: ResourceQuota objects limit total CPU, memory, pod count, etc. within a namespace. LimitRange objects set default/min/max resource requests for individual pods in a namespace.

5. **Network isolation**: By default, pods in different namespaces CAN communicate. NetworkPolicy objects add actual network isolation rules to restrict cross-namespace traffic.

6. **Namespace lifecycle**: When a namespace is deleted, ALL resources within it are deleted — this is a dangerous operation.

---

## Real-World Example
**Company**: A mid-size e-commerce company with 8 development teams and a shared EKS cluster.

**Namespace structure:**

| Namespace | Purpose | Teams | Resource Quota |
|---|---|---|---|
| `production` | Live customer traffic | Platform team | 200 CPU, 400Gi RAM |
| `staging` | Pre-prod validation | All teams | 50 CPU, 100Gi RAM |
| `dev-team-a` | Team A development | Team A | 20 CPU, 40Gi RAM |
| `dev-team-b` | Team B development | Team B | 20 CPU, 40Gi RAM |
| `monitoring` | Prometheus, Grafana | Platform | 10 CPU, 50Gi RAM |
| `logging` | ELK Stack | Platform | 20 CPU, 80Gi RAM |
| `ingress-nginx` | Ingress Controller | Platform | 5 CPU, 10Gi RAM |
| `cert-manager` | TLS management | Platform | 2 CPU, 4Gi RAM |

- Each dev team namespace has a ResourceQuota to prevent any team from consuming all cluster resources.
- RBAC: Dev teams have full admin rights in their own namespace but read-only in `monitoring`.
- NetworkPolicy: `production` namespace only accepts traffic from `ingress-nginx` and `monitoring` namespaces.
- CI/CD: GitHub Actions pipelines deploy to the team's namespace using a dedicated ServiceAccount with limited RBAC.

---

## Diagram / Flow

```
KUBERNETES CLUSTER — NAMESPACE ISOLATION
==========================================

+------------------------------------------------------------+
|                   KUBERNETES CLUSTER                       |
|                                                            |
|  +------------------+   +------------------+              |
|  |   production     |   |   staging        |              |
|  |  +-----------+   |   |  +-----------+   |              |
|  |  | web-pod   |   |   |  | web-pod   |   |              |
|  |  | api-pod   |   |   |  | api-pod   |   |              |
|  |  | db-pod    |   |   |  | db-pod    |   |              |
|  |  +-----------+   |   |  +-----------+   |              |
|  |  Services ✓      |   |  Services ✓      |              |
|  |  Secrets  ✓      |   |  Secrets  ✓      |              |
|  |  Quota: 200CPU   |   |  Quota: 50CPU    |              |
|  +------------------+   +------------------+              |
|                                                            |
|  +------------------+   +------------------+              |
|  |   dev-team-a     |   |   monitoring     |              |
|  |  +-----------+   |   |  +-----------+   |              |
|  |  | feature-  |   |   |  | prometheus|   |              |
|  |  |   pods    |   |   |  | grafana   |   |              |
|  |  +-----------+   |   |  | alertmgr  |   |              |
|  |  Quota: 20CPU    |   |  +-----------+   |              |
|  +------------------+   +------------------+              |
|                                                            |
|  CLUSTER-SCOPED (no namespace):                           |
|  Nodes, PersistentVolumes, StorageClasses                 |
|  ClusterRoles, ClusterRoleBindings                        |
|  IngressClasses, CustomResourceDefinitions               |
+------------------------------------------------------------+


DNS RESOLUTION ACROSS NAMESPACES
===================================

From pod in "production" namespace:
  http://api-service              ─── Resolves to api-service.production.svc.cluster.local
  http://api-service.staging      ─── Resolves to api-service.staging.svc.cluster.local
  http://prometheus.monitoring    ─── Cross-namespace access to Prometheus


NAMESPACE-SCOPED vs CLUSTER-SCOPED RESOURCES
=============================================

Namespace-Scoped:              Cluster-Scoped:
  Pod                            Node
  Service                        PersistentVolume
  Deployment                     StorageClass
  StatefulSet                    ClusterRole
  ConfigMap                      ClusterRoleBinding
  Secret                         Namespace (itself)
  PVC                            CustomResourceDefinition
  Role                           IngressClass
  RoleBinding                    MutatingWebhookConfiguration
  ServiceAccount


RESOURCE QUOTA ENFORCEMENT
============================

  New Pod Request ──► Kubernetes Admission ──► Check Namespace Quota
                                                      |
                                  ┌───────────────────┴────────────────────┐
                                  |                                        |
                          Quota available?                         Quota exceeded?
                                  |                                        |
                             Pod Created                          Pod Rejected (403)
                                                            "exceeded quota: limits.cpu"
```

---

## Why It Is Important
**Business Value:**
- Multi-tenancy: Multiple teams share one cluster, reducing infrastructure costs.
- Blast radius reduction: A mistake in `dev-team-a` namespace cannot delete production resources.
- Chargeback: Resource quotas per namespace enable per-team cost allocation.
- Compliance: Separate namespaces for PCI-DSS or HIPAA workloads with strict network policies.

**Technical Value:**
- Enables RBAC at team/namespace granularity without cluster-wide permissions.
- Resource quotas prevent noisy-neighbor problems.
- Simplifies CI/CD — pipelines deploy to specific namespaces with scoped credentials.
- Clean separation of lifecycle — delete a namespace to clean up all resources at once.

---

## Common Interview Follow-Up Questions
1. What is the difference between namespace-scoped and cluster-scoped resources?
2. How does DNS work across namespaces?
3. How do you restrict resource consumption per namespace?
4. How do you control who can access a namespace using RBAC?
5. How do you implement network isolation between namespaces?
6. What happens when you delete a namespace?
7. Can a service in one namespace communicate with a service in another namespace by default?
8. What is the difference between ResourceQuota and LimitRange?

---

## Common Mistakes Candidates Make

**Mistake 1: Thinking namespaces provide network isolation by default**
- Wrong: "If I put apps in different namespaces, they cannot talk to each other."
- Correct: Namespaces do NOT provide network isolation by default. Pods in different namespaces can communicate freely. You need **NetworkPolicy** objects to enforce network isolation.

**Mistake 2: Not knowing which resources are cluster-scoped**
- Wrong: "Every Kubernetes resource is inside a namespace."
- Correct: Nodes, PersistentVolumes, StorageClasses, ClusterRoles, and Namespaces themselves are cluster-scoped. They exist outside any namespace.

**Mistake 3: Using `default` namespace in production**
- Wrong: "We deploy everything to the default namespace — it is easier."
- Correct: Using `default` in production is a bad practice. You lose isolation, resource quota control, and RBAC granularity. Always create dedicated namespaces.

**Mistake 4: Confusing ResourceQuota and LimitRange**
- Wrong: "ResourceQuota sets CPU and memory limits on individual pods."
- Correct: **ResourceQuota** sets total limits for the entire namespace (sum of all pods). **LimitRange** sets default/min/max limits for individual pods or containers within the namespace.

**Mistake 5: Not knowing cross-namespace DNS**
- Wrong: "Services in different namespaces cannot be reached by name."
- Correct: Services are reachable across namespaces using the full DNS name: `<service-name>.<namespace>.svc.cluster.local` or the short form `<service-name>.<namespace>`.

---

## Troubleshooting Scenario
**Problem**: A new developer's pod is failing to deploy to the `dev-team-a` namespace with a quota error. Also, their pod is not getting default resource limits.

```bash
# Step 1: Check the error when deploying
kubectl apply -f my-deployment.yaml -n dev-team-a
# Error: pods "my-app-xyz" is forbidden: exceeded quota: dev-team-a-quota,
# requested: requests.cpu=2, used: requests.cpu=19, limited: requests.cpu=20

# Step 2: Check current resource quota usage
kubectl get resourcequota -n dev-team-a
# NAME                AGE   REQUEST                               LIMIT
# dev-team-a-quota    30d   requests.cpu: 19/20, memory: 38/40Gi ...

kubectl describe resourcequota dev-team-a-quota -n dev-team-a
# Resource             Used   Hard
# --------             ----   ----
# limits.cpu           19     20
# limits.memory        38Gi   40Gi
# requests.cpu         19     20
# requests.memory      38Gi   40Gi
# pods                 48     50

# Step 3: Find which pods are consuming the most resources
kubectl top pods -n dev-team-a --sort-by=cpu
# NAME                     CPU(cores)   MEMORY(bytes)
# old-feature-abc-7d9f8b   800m         2Gi
# old-feature-xyz-9c8d7e   600m         1.5Gi

# Step 4: Check if those pods are still needed
kubectl get pods -n dev-team-a --sort-by=.metadata.creationTimestamp
# NAME                     READY   STATUS    RESTARTS   AGE
# old-feature-abc-...      1/1     Running   0          15d    <-- Old feature branch pod

# Fix: Delete old/unused pods or deployments
kubectl delete deployment old-feature-abc -n dev-team-a
kubectl delete deployment old-feature-xyz -n dev-team-a

# Step 5: Also check LimitRange — why pod has no default limits
kubectl get limitrange -n dev-team-a
# No resources found.   <-- No LimitRange set!

# Without LimitRange, pods without explicit resource requests bypass quota tracking
# (the quota won't count them if no request is set)

# Fix: Apply LimitRange so all pods get default limits
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev-team-a
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
EOF

# Step 6: Now try deploying again
kubectl apply -f my-deployment.yaml -n dev-team-a
# deployment.apps/my-app created

# Verify
kubectl get pods -n dev-team-a | grep my-app
# my-app-abc123   1/1   Running   0   30s
```

---

## kubectl Commands

```bash
# List all namespaces
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   60d
# dev-team-a        Active   30d
# kube-node-lease   Active   60d
# kube-public       Active   60d
# kube-system       Active   60d
# monitoring        Active   30d
# production        Active   30d

# Create a namespace
kubectl create namespace dev-team-b

# Describe a namespace (see labels, annotations, resource quotas)
kubectl describe namespace production

# Delete a namespace (WARNING: deletes ALL resources inside it)
kubectl delete namespace dev-team-b

# Set a default namespace for your current context
kubectl config set-context --current --namespace=production

# Check which namespace your context is using
kubectl config view --minify | grep namespace
# namespace: production

# Get all resources in a namespace
kubectl get all -n production

# Get all resources across ALL namespaces
kubectl get pods --all-namespaces
kubectl get pods -A   # Short form

# Get ResourceQuotas in a namespace
kubectl get resourcequota -n dev-team-a
kubectl describe resourcequota dev-team-a-quota -n dev-team-a

# Get LimitRanges in a namespace
kubectl get limitrange -n dev-team-a

# Get NetworkPolicies in a namespace
kubectl get networkpolicy -n production

# Check RBAC for a namespace — who has access
kubectl get rolebinding -n production

# List only namespace-scoped resources
kubectl api-resources --namespaced=true

# List cluster-scoped resources (not in any namespace)
kubectl api-resources --namespaced=false
```

---

## YAML Example

```yaml
# ============================================================
# Namespace Definition
# ============================================================
apiVersion: v1                          # Core API group
kind: Namespace                         # Resource type
metadata:
  name: production                      # Namespace name — must be DNS-compliant (lowercase, hyphens)
  labels:
    environment: production             # Label for identification and selection
    team: platform                      # Owning team
    cost-center: "1001"                 # For billing/chargeback purposes
  annotations:
    description: "Production workloads — handle with care"  # Human description
    contact: "platform-team@company.com"                     # Who owns this namespace
---
# ============================================================
# ResourceQuota — limits total resource consumption per namespace
# ============================================================
apiVersion: v1
kind: ResourceQuota                     # Resource type
metadata:
  name: production-quota                # Name of the quota
  namespace: production                 # Apply to this namespace
spec:
  hard:
    # Compute resources
    requests.cpu: "100"                 # Total CPU requests cannot exceed 100 cores
    requests.memory: 200Gi             # Total memory requests cannot exceed 200Gi
    limits.cpu: "200"                   # Total CPU limits cannot exceed 200 cores
    limits.memory: 400Gi               # Total memory limits cannot exceed 400Gi

    # Object count limits
    pods: "200"                         # Maximum 200 pods in this namespace
    services: "50"                      # Maximum 50 services
    persistentvolumeclaims: "30"        # Maximum 30 PVCs
    secrets: "100"                      # Maximum 100 secrets
    configmaps: "100"                   # Maximum 100 config maps

    # Storage quotas
    requests.storage: "5Ti"            # Total PVC storage requests

    # Service type restrictions
    services.loadbalancers: "2"         # Maximum 2 LoadBalancer services (cost control)
    services.nodeports: "0"             # No NodePort services allowed
---
# ============================================================
# LimitRange — sets defaults and boundaries for individual pods
# ============================================================
apiVersion: v1
kind: LimitRange                        # Resource type
metadata:
  name: production-limits               # Name of the LimitRange
  namespace: production                 # Apply to this namespace
spec:
  limits:
  # Container-level limits
  - type: Container                     # Applies to individual containers
    default:                            # Default LIMITS if container doesn't specify
      cpu: "1"                          # Default CPU limit: 1 core
      memory: "1Gi"                     # Default memory limit: 1 GiB
    defaultRequest:                     # Default REQUESTS if container doesn't specify
      cpu: "100m"                       # Default CPU request: 0.1 core
      memory: "128Mi"                   # Default memory request: 128 MiB
    max:                                # Maximum allowed limits
      cpu: "4"                          # No container can request more than 4 cores
      memory: "8Gi"                     # No container can use more than 8Gi
    min:                                # Minimum required requests
      cpu: "50m"                        # Container must request at least 50m CPU
      memory: "64Mi"                    # Container must request at least 64Mi memory

  # Pod-level limits (sum across all containers in the pod)
  - type: Pod
    max:
      cpu: "8"                          # A single pod cannot use more than 8 cores total
      memory: "16Gi"                    # A single pod cannot use more than 16Gi total
---
# ============================================================
# NetworkPolicy — restrict cross-namespace traffic
# ============================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy                     # Resource type
metadata:
  name: production-isolation            # Name of the policy
  namespace: production                 # Apply to pods in this namespace
spec:
  podSelector: {}                       # Apply to ALL pods in the namespace (empty = all)
  policyTypes:
  - Ingress                             # Control incoming traffic
  - Egress                              # Control outgoing traffic
  ingress:
  - from:
    - namespaceSelector:                # Allow traffic FROM specific namespaces
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx  # Allow from ingress-nginx namespace
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring      # Allow from monitoring namespace
  egress:
  - to:
    - namespaceSelector: {}             # Allow outgoing to any namespace (for flexibility)
  - ports:
    - port: 53                          # Allow DNS queries
      protocol: UDP
    - port: 53
      protocol: TCP
---
# ============================================================
# Role and RoleBinding — RBAC for namespace access
# ============================================================
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                              # Namespace-scoped role (not ClusterRole)
metadata:
  name: dev-team-a-role                 # Role name
  namespace: dev-team-a                 # Only grants permissions IN THIS namespace
rules:
- apiGroups: [""]                       # Core API group
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]                   # Apps API group (Deployments, etc.)
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]  # Allow kubectl logs and exec
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                       # Binds the role to a user/group/serviceaccount
metadata:
  name: dev-team-a-binding              # Binding name
  namespace: dev-team-a                 # Applies in this namespace only
subjects:
- kind: Group                           # Bind to an OIDC/SSO group
  name: "dev-team-a"                    # Group name from identity provider
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role                            # Reference the Role defined above
  name: dev-team-a-role
  apiGroup: rbac.authorization.k8s.io
```

---

## AWS/EKS Perspective

**Namespace and EKS IAM Integration:**
1. **IAM Roles for Service Accounts (IRSA)**: In EKS, ServiceAccounts in a specific namespace can be mapped to AWS IAM roles. This means you can give a pod in `production` namespace an IAM role with S3 read access, while a pod in `dev` has no AWS permissions.

2. **EKS Access Entries**: With newer EKS auth, IAM users/roles can be mapped to Kubernetes RBAC roles per namespace.

3. **AWS Cost Allocation**: Namespace labels can be used with AWS cost allocation tags when combined with tools like Kubecost — essential for multi-team chargeback in shared EKS clusters.

4. **Fargate Profiles**: AWS EKS Fargate profiles are namespace-based — you can run specific namespaces on Fargate (serverless) and others on EC2.

```bash
# Create a Fargate profile for a specific namespace
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name production-fargate \
  --pod-execution-role-arn arn:aws:iam::123456789:role/eks-fargate-role \
  --selectors namespace=production

# Annotate ServiceAccount for IRSA in a specific namespace
kubectl annotate serviceaccount -n production my-app-sa \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789:role/my-app-role

# Check what namespace a Fargate pod runs in
kubectl get pod -A -o wide | grep fargate
```

---

## Interview Answer (2-Minute Version)
"Namespaces are virtual partitions inside a Kubernetes cluster. They let you run multiple isolated environments — like dev, staging, and production — on the same cluster without them interfering with each other.

The practical benefits are: you can set resource quotas per namespace so one team can't consume all the cluster's CPU and memory, you can apply RBAC permissions at the namespace level so developers have full access to dev but read-only on production, and you can use NetworkPolicies to restrict traffic between namespaces.

In production I structure namespaces by team and environment. Each development team gets their own namespace with resource quotas. System components like monitoring and ingress controllers get their own namespaces. One important thing: namespaces don't provide network isolation by default — you still need NetworkPolicies for that."

---

## Interview Answer (Senior Engineer Version)
"Namespaces are the primary multi-tenancy primitive in Kubernetes — they partition cluster resources into virtual scopes. They serve three key purposes: resource scoping for RBAC and quota enforcement, DNS segmentation, and lifecycle grouping.

In large organizations running shared clusters, namespace design is a critical architecture decision. I follow a few patterns: environment separation (prod/staging/dev), team separation, and component separation (monitoring stack, ingress layer, application workloads). The tradeoffs are cluster sprawl vs. namespace sprawl — too many small clusters is expensive and hard to manage; too few namespaces in one cluster creates blast radius and resource contention issues.

For resource control, I always deploy both ResourceQuota and LimitRange together. Quota controls namespace-level totals; LimitRange sets per-pod defaults. Without LimitRange, developers forget to set resource requests, which means pods bypass quota accounting and can starve other pods.

On EKS specifically, I tie namespaces to IRSA (IAM Roles for Service Accounts) so that only pods in specific namespaces can assume specific AWS IAM roles. This is critical for least-privilege security. I also use namespace labels to drive Fargate profile selection — stateless workloads go to Fargate namespaces for serverless scaling, while stateful apps stay on managed node groups."

---

## What Impresses the Interviewer
- Explaining that namespaces do NOT provide network isolation by default (very common misconception)
- Knowing the difference between ResourceQuota and LimitRange and why you need both
- Understanding which resources are cluster-scoped vs namespace-scoped
- Discussing namespace design strategy (by team, by environment, by component)
- Mentioning IRSA and namespace-level IAM on EKS
- Knowing the full DNS name format for cross-namespace service discovery

---

## Red Flags
- Saying namespaces are like separate clusters or provide full isolation
- Not knowing that NetworkPolicy is required for network isolation
- Saying "we put everything in the default namespace"
- Not being able to list any cluster-scoped resources
- Confusing Role (namespace-scoped) with ClusterRole

---

## Production Best Practices
1. **Never use the `default` namespace** for production workloads — create named namespaces.
2. **Apply ResourceQuota to every namespace** to prevent resource starvation.
3. **Apply LimitRange alongside ResourceQuota** — without it, pods with no resource requests bypass quota tracking.
4. **Use NetworkPolicy** to enforce traffic isolation between namespaces, especially for production.
5. **Label namespaces consistently** — include environment, team, and cost-center labels for tooling and cost allocation.
6. **Restrict access to `kube-system`** — only platform/SRE teams should have access; never deploy application workloads there.
7. **Automate namespace creation** via GitOps (Argo CD Application Sets or Helm) so every new namespace gets standard ResourceQuota, LimitRange, and NetworkPolicy.
8. **Audit namespace deletion** — add admission webhooks or policies (OPA/Kyverno) to prevent accidental namespace deletion in production.

---

## Key Points to Remember
- Namespaces provide virtual isolation of cluster resources, not full security isolation
- Most resources are namespace-scoped; Nodes, PVs, ClusterRoles are cluster-scoped
- Default namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`
- Cross-namespace DNS: `<service>.<namespace>.svc.cluster.local`
- Namespaces do NOT block network traffic by default — NetworkPolicy is needed
- ResourceQuota limits total resources per namespace; LimitRange sets per-pod defaults
- RBAC Roles are namespace-scoped; ClusterRoles span all namespaces
- Deleting a namespace deletes ALL resources inside it
- Use namespace labels for EKS Fargate profiles and cost allocation
- Good namespace design balances isolation needs vs operational complexity

---

## Interviewer's Expectation
The interviewer is testing whether you understand Kubernetes multi-tenancy, have thought about resource management and access control at the team/namespace level, and know the practical limitations of namespaces (such as the fact they don't provide network isolation). They want to see evidence of designing namespace structures for real organizations.

---

## Final Perfect Interview Answer
"Namespaces are Kubernetes' way of creating virtual partitions within a single cluster. They let multiple teams or environments share infrastructure while maintaining logical separation. The main benefits are resource quotas to prevent any one team from consuming all cluster resources, RBAC scoping so developers have full access to their namespace but read-only elsewhere, and clean lifecycle management.

In production I always design namespaces with both ResourceQuota and LimitRange — quota controls the namespace total, while LimitRange sets defaults for individual pods. Without LimitRange, developers forget to set resource requests and those pods bypass quota accounting entirely.

One thing I always make clear to teams: namespaces do not provide network isolation by default. Pods in different namespaces can still talk to each other. You need explicit NetworkPolicy rules to block cross-namespace traffic, which is essential for separating production from development environments.

On EKS I tie namespaces to IRSA to give namespace-specific pods specific AWS IAM permissions — so a pod in the production namespace can write to S3 while a pod in dev has no AWS access at all. I also automate namespace creation via GitOps so every new namespace automatically gets standard quota, limit, and network policies applied."
