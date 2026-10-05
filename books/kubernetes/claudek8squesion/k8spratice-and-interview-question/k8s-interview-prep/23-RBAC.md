# RBAC — Kubernetes Interview Guide

## Interview Question
"Explain Kubernetes RBAC. What is the difference between Role, ClusterRole, RoleBinding, and ClusterRoleBinding? How do you implement least-privilege access in a multi-team cluster?"

---

## Simple Explanation
Imagine a large office building. The building has:
- A **security policy document** (Role/ClusterRole) that lists what each type of employee can access — "Developers can enter floors 3 and 4, read files in Room 302, but cannot enter the server room."
- A **badge assignment system** (RoleBinding/ClusterRoleBinding) that assigns those policies to specific people — "Give John and Mary the Developer badge."

Without RBAC, everyone in the building has a master key to every room. That's dangerous. RBAC is the Kubernetes system that says: "Only specific users/applications can do specific things, in specific places."

- **Role** = Permission list for ONE floor (one namespace)
- **ClusterRole** = Permission list for the ENTIRE building (all namespaces or cluster-wide resources)
- **RoleBinding** = Assign a permission list to a person, but only for ONE floor
- **ClusterRoleBinding** = Assign a permission list to a person for the ENTIRE building

---

## Technical Explanation
Kubernetes RBAC (Role-Based Access Control) is an authorization mechanism that regulates access to Kubernetes API resources based on the roles assigned to users, groups, or service accounts. It answers the question: "Can this identity perform this verb on this resource in this namespace?"

### The Four Core Resources:

**Role**
- Defines permissions within a **single namespace**
- Grants access only to namespaced resources (Pods, Services, Deployments, etc.)
- Cannot grant access to cluster-scoped resources (Nodes, PersistentVolumes, Namespaces themselves)

**ClusterRole**
- Defines permissions at the **cluster level**
- Can grant access to:
  1. Cluster-scoped resources (Nodes, PersistentVolumes, Namespaces, StorageClasses)
  2. Namespaced resources across ALL namespaces
  3. Non-resource URLs (e.g., `/healthz`, `/metrics`)
- Can be reused across multiple namespaces via RoleBinding

**RoleBinding**
- Grants the permissions defined in a Role (or ClusterRole) **within a specific namespace**
- Subjects (users/groups/service accounts) receive the permissions only in that namespace
- A RoleBinding can reference a ClusterRole — this is a common pattern for reusing permission templates

**ClusterRoleBinding**
- Grants the permissions defined in a ClusterRole **across all namespaces** (or to cluster-scoped resources)
- High privilege — use sparingly

### RBAC Authorization Flow:
When an API request arrives at the API server:
1. Authentication (who are you?) — certificates, tokens, OIDC
2. Authorization (what can you do?) — RBAC checks happen here
3. Admission Control (is the request valid?) — webhooks, policies

RBAC checks: Does the subject (user/group/SA) have a RoleBinding or ClusterRoleBinding that grants the requested verb on the requested resource in the requested namespace?

### Verbs (Actions):
- `get`, `list`, `watch` — read operations
- `create`, `update`, `patch`, `delete` — write operations
- `deletecollection` — delete all resources of a type
- `bind`, `escalate` — special verbs for RBAC itself
- `use` — for PodSecurityPolicies and ResourceQuotas

---

## Real-World Example

**Production Scenario: Multi-Team SaaS Platform**

A company has these teams in a Kubernetes cluster:
- **App Team A** — owns `team-a` namespace, should control their own Deployments/Services
- **App Team B** — owns `team-b` namespace, similar needs
- **Data Engineers** — need to read logs from all namespaces but not modify anything
- **CI/CD Pipeline** — needs to deploy to team-a and team-b namespaces, but not delete resources
- **Platform SRE** — full cluster admin (but even SREs should use dedicated SA, not cluster-admin)
- **Monitoring System** — needs to read pod metrics from all namespaces

RBAC Design:
```
Role "app-developer"           → Namespace-scoped (create/update/delete Pods, Deployments, Services)
  RoleBinding in team-a        → Grants app-developer to Team A members
  RoleBinding in team-b        → Grants app-developer to Team B members

ClusterRole "log-reader"       → Cluster-wide (list/get Pods, read Pod logs)
  ClusterRoleBinding           → Grants log-reader to Data Engineers

ClusterRole "deployer"         → Custom (create/update Deployments, no delete)
  RoleBinding in team-a        → Grants deployer to CI/CD SA (only in team-a)
  RoleBinding in team-b        → Grants deployer to CI/CD SA (only in team-b)

ClusterRole "monitoring"       → Cluster-wide read of pods/metrics
  ClusterRoleBinding           → Grants monitoring to prometheus SA
```

---

## Diagram / Flow

```
RBAC RESOURCE HIERARCHY
========================

     ClusterRole                    Role
   (cluster-scoped)            (namespace-scoped)
   +--------------+             +-------------+
   | • list nodes |             | • get pods  |
   | • get pvs    |             | • list svcs |
   | • read pods  |             | • update    |
   |   (all ns)   |             |   deploys   |
   +--------------+             +-------------+
         |                           |
         | referenced by             | referenced by
         |                           |
   +-----------+  +-----------+ +-----------+
   |ClusterRole|  |RoleBinding| |RoleBinding|
   |  Binding  |  |(namespace:| |(namespace:|
   |           |  |  team-a)  |  |  team-a)  |
   +-----------+  +-----------+ +-----------+
         |              |            |
         v              v            v
    Applies to      Applies to   Applies to
    ALL namespaces  team-a only  team-a only


SCOPE MATRIX
=============

                    ROLE              CLUSTERROLE
                 +-----------+      +------------------+
RoleBinding      | Namespace |      | Namespace        |
                 | resources |      | resources (reuse |
                 | in 1 NS   |      | template in 1 NS)|
                 +-----------+      +------------------+
ClusterRoleBinding  N/A             | Cluster-scoped + |
                                    | All namespaces   |
                                    +------------------+

Key: Use RoleBinding + ClusterRole when you want to reuse
     permission templates but limit scope to one namespace.


API REQUEST AUTHORIZATION FLOW
================================

User: alice (group: developers)
Request: GET /apis/apps/v1/namespaces/team-a/deployments

API Server
    |
    v
+---+---+
| Authn |  <- Who are you? (certificate/token)
| alice |     Verified: alice in group "developers"
+---+---+
    |
    v
+---+-------+
|  RBAC     |
|  Authz    |
+---+-------+
    |
    | Check: Does alice or group "developers" have a
    |        RoleBinding/ClusterRoleBinding that grants
    |        GET on deployments in namespace team-a?
    |
    | -> Finds: RoleBinding "dev-binding" in namespace team-a
    |    -> References: ClusterRole "app-developer"
    |    -> ClusterRole has: {verbs: [get,list], resources: [deployments]}
    |    -> alice is in subject group: developers
    |
    v
+---+------+
| ALLOWED  |  -> Request proceeds to etcd
+----------+


ClusterRoleBinding (grants access to ALL namespaces)
=====================================================

prometheus SA
    |
    | ClusterRoleBinding: "prometheus-monitoring"
    v
ClusterRole: "monitoring-reader"
    |
    | Grants: get/list/watch pods, services, endpoints
    v
ALL namespaces simultaneously:
  team-a: [pods, services, endpoints] <- readable
  team-b: [pods, services, endpoints] <- readable
  kube-system: [pods, services]        <- readable

vs.

RoleBinding + ClusterRole (limits to one namespace)
=====================================================

ci-cd SA
    |
    | RoleBinding in team-a: "cicd-deployer-team-a"
    | References ClusterRole: "deployer"
    v
team-a namespace only:
  team-a: [deployments, services] <- create/update allowed
  team-b: [deployments, services] <- NO ACCESS
```

---

## Why It Is Important

**Security Value:**
- Principle of least privilege — each component only has the permissions it needs
- Blast radius reduction — compromised service account cannot delete cluster resources
- Audit compliance — RBAC is required for SOC2, PCI-DSS, HIPAA Kubernetes workloads
- Defense in depth — even if application code is compromised, RBAC limits damage

**Operational Value:**
- Multi-tenancy — multiple teams can share a cluster safely
- Self-service — teams manage their own namespaces without needing cluster-admin
- Automation safety — CI/CD pipelines can deploy without having delete permissions
- Clear ownership — RBAC makes it explicit who can do what where

---

## Common Interview Follow-Up Questions

1. **"Can a RoleBinding reference a ClusterRole?"**
   Yes. This is a powerful pattern for permission template reuse. The ClusterRole defines the permissions, but the RoleBinding limits them to a specific namespace. The ClusterRole itself grants nothing — only bindings grant access.

2. **"What is the difference between `update` and `patch` verbs?"**
   `update` replaces the entire resource (PUT semantics). `patch` applies partial changes (PATCH semantics). In practice, most tools use `patch` for updates, so you often need both.

3. **"How do you give a user cluster-admin access temporarily?"**
   Create a time-bound ClusterRoleBinding manually, or use tools like `kubectl-whoami` and `kubectl-sudo` plugins. Best practice: use `kubectl auth can-i` to verify before and after. Delete the binding immediately after use.

4. **"What is `kubectl auth can-i`?"**
   A command to check if a user/SA can perform an action: `kubectl auth can-i create pods --namespace=team-a --as=alice`. Critical for RBAC debugging and auditing.

5. **"How do RBAC and Namespaces work together for multi-tenancy?"**
   Namespaces provide resource isolation; RBAC provides permission isolation. Together they create soft multi-tenancy. For hard multi-tenancy, you need NetworkPolicies + LimitRanges + ResourceQuotas + PodSecurity + separate clusters.

6. **"What is `system:masters` group?"**
   A special group hardcoded in Kubernetes that bypasses RBAC entirely. Any user in this group has unrestricted cluster access. Used for break-glass scenarios. Never put regular users in this group.

7. **"How do you audit RBAC in a cluster?"**
   Use `kubectl auth can-i --list --as=<user>` to list all permissions. Use tools like `rbac-lookup`, `kubectl-who-can`, or `rakkess` for comprehensive auditing.

---

## Common Mistakes Candidates Make

**Mistake 1: Confusing Role scope with ClusterRole scope**
- Wrong: "A ClusterRole gives access to all namespaces."
- Correct: A ClusterRole *defines* permissions. It only *grants* access when bound via ClusterRoleBinding (all namespaces) or RoleBinding (single namespace). The Role itself grants nothing — only the binding grants access.

**Mistake 2: Not knowing the RoleBinding → ClusterRole pattern**
- Missing: Most candidates only know Role+RoleBinding or ClusterRole+ClusterRoleBinding.
- Impressive: Knowing you can use RoleBinding to reference a ClusterRole, scoping a cluster-wide permission template to a single namespace — avoiding duplicate Role objects.

**Mistake 3: Using `cluster-admin` everywhere**
- Wrong: "I just bind cluster-admin to my CI/CD service account."
- Correct: Create a minimal ClusterRole with only the verbs and resources needed. Audit monthly. Use `kubectl auth can-i --list --as=<sa>` to verify scope.

**Mistake 4: Not knowing nonResourceURLs**
- Missing: RBAC can also control access to non-resource endpoints like `/healthz`, `/metrics`, `/api`.
- Example: Prometheus needs `nonResourceURLs: ["/metrics"]` with verb `get` to scrape the API server.

**Mistake 5: Forgetting that RBAC is additive, not subtractive**
- Wrong: "I can create a Deny rule in RBAC."
- Correct: RBAC is purely additive — you can only grant permissions, never deny them. To remove access, you remove the binding, not add a deny rule.

---

## Troubleshooting Scenario

**Problem:** A CI/CD pipeline service account (`cicd-sa` in namespace `ci-cd`) is failing to deploy to the `production` namespace. Error: `"cicd-sa" cannot create resource "deployments" in API group "apps" in the namespace "production"`

**Step-by-Step Debugging:**

```bash
# Step 1: Verify what the SA can currently do
kubectl auth can-i create deployments \
  --namespace=production \
  --as=system:serviceaccount:ci-cd:cicd-sa
# OUTPUT: no
# Confirms the SA lacks the needed permission

# Step 2: List all permissions the SA has (comprehensive view)
kubectl auth can-i --list \
  --namespace=production \
  --as=system:serviceaccount:ci-cd:cicd-sa
# OUTPUT:
# Resources                Non-Resource URLs  Resource Names  Verbs
# configmaps               []                 []              [get list watch]
# ... (no deployments listed)

# Step 3: Check if there are any RoleBindings in production namespace for this SA
kubectl get rolebindings -n production -o yaml | grep -A5 "cicd-sa"
# OUTPUT: nothing — no RoleBinding references this SA in production

# Step 4: Check ClusterRoleBindings
kubectl get clusterrolebindings -o yaml | grep -A5 "cicd-sa"
# OUTPUT: nothing — no ClusterRoleBinding either

# Root Cause: No binding exists granting the SA access to production namespace

# Step 5: Identify what ClusterRole we should use or create
kubectl get clusterroles | grep deploy
# OUTPUT: deployer-role (custom), admin, edit, view

# Step 6: Check what 'edit' ClusterRole provides (built-in)
kubectl describe clusterrole edit | grep -A2 "deployments"
# OUTPUT: deployments  create update patch delete get list watch

# Decision: Use built-in 'edit' role or create minimal custom role

# Option A: Use built-in 'edit' ClusterRole with namespace-scoped RoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer
  namespace: production
subjects:
  - kind: ServiceAccount
    name: cicd-sa
    namespace: ci-cd     # SA is in ci-cd namespace, binding is in production
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 7: Verify the fix works
kubectl auth can-i create deployments \
  --namespace=production \
  --as=system:serviceaccount:ci-cd:cicd-sa
# OUTPUT: yes

# Step 8: Verify scope is limited (should NOT have production node access)
kubectl auth can-i list nodes \
  --as=system:serviceaccount:ci-cd:cicd-sa
# OUTPUT: no (correct — cluster-scoped resources not granted)

# Step 9: Audit — list ALL permissions for this SA in production
kubectl auth can-i --list \
  --namespace=production \
  --as=system:serviceaccount:ci-cd:cicd-sa
# Review output carefully — 'edit' may grant more than needed
# If too broad, create a minimal custom ClusterRole instead
```

---

## kubectl Commands

```bash
# --- Inspection Commands ---

# List all Roles in a namespace
kubectl get roles -n team-a

# List all ClusterRoles
kubectl get clusterroles

# List all RoleBindings in a namespace
kubectl get rolebindings -n team-a

# List all ClusterRoleBindings
kubectl get clusterrolebindings

# Describe a Role (shows rules in readable format)
kubectl describe role developer-role -n team-a

# Describe a ClusterRole
kubectl describe clusterrole monitoring-reader

# Get Role in YAML
kubectl get role developer-role -n team-a -o yaml

# --- Authorization Checking ---

# Check if current user can do something
kubectl auth can-i get pods -n production
# OUTPUT: yes

# Check if another user can do something (impersonation)
kubectl auth can-i get pods -n production --as=alice
# OUTPUT: no

# Check as a ServiceAccount
kubectl auth can-i create deployments -n production \
  --as=system:serviceaccount:ci-cd:cicd-sa
# OUTPUT: yes

# List ALL permissions for a user in a namespace
kubectl auth can-i --list --as=alice -n production

# List ALL permissions for a SA cluster-wide
kubectl auth can-i --list --as=system:serviceaccount:ci-cd:cicd-sa

# --- Creation Commands ---

# Create a Role imperatively
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n team-a

# Create a ClusterRole imperatively
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create a RoleBinding (bind a Role to a user)
kubectl create rolebinding alice-pod-reader \
  --role=pod-reader \
  --user=alice \
  -n team-a

# Create a RoleBinding (bind a ClusterRole to a SA)
kubectl create rolebinding cicd-deployer \
  --clusterrole=edit \
  --serviceaccount=ci-cd:cicd-sa \
  -n production

# Create a ClusterRoleBinding
kubectl create clusterrolebinding prometheus-monitoring \
  --clusterrole=monitoring-reader \
  --serviceaccount=monitoring:prometheus-sa

# --- Audit Tools ---

# Who can create deployments in production?
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.roleRef.name == "edit" or .roleRef.name == "admin")'

# View RBAC using rbac-lookup tool (install separately)
rbac-lookup alice
# OUTPUT:
# SUBJECT  SCOPE         ROLE
# alice    team-a        Role/developer-role
# alice    cluster-wide  ClusterRole/view

# Delete a RoleBinding
kubectl delete rolebinding alice-pod-reader -n team-a

# Delete a ClusterRoleBinding
kubectl delete clusterrolebinding prometheus-monitoring
```

---

## YAML Example

```yaml
# ============================================================
# 1. ROLE — Namespace-scoped permissions
# ============================================================
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                          # Scope: single namespace only
metadata:
  name: app-developer               # Name of this Role
  namespace: team-a                 # Only applies to the team-a namespace
  labels:
    team: team-a
    managed-by: platform-team
rules:                              # List of permission rules (additive)
  - apiGroups: [""]                 # "" = core API group (pods, services, configmaps, secrets)
    resources:                      # Which resource types this rule applies to
      - pods
      - services
      - configmaps
      - endpoints
    verbs:                          # Which actions are allowed on these resources
      - get                         # Read a single resource
      - list                        # List all resources of this type
      - watch                       # Stream changes (used by kubectl -w)
      - create                      # Create new resources
      - update                      # Replace entire resource
      - patch                       # Partial update
      - delete                      # Delete individual resource

  - apiGroups: ["apps"]             # apps API group contains Deployments, StatefulSets, etc.
    resources:
      - deployments
      - replicasets
      - statefulsets
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      # Note: "delete" intentionally omitted — developers cannot delete Deployments
      # This prevents accidental production outages

  - apiGroups: [""]
    resources:
      - pods/log                    # Subresource: allows reading pod logs
      - pods/exec                   # Subresource: allows kubectl exec into pods
      - pods/portforward            # Subresource: allows kubectl port-forward
    verbs:
      - get
      - create                      # exec/portforward require "create" verb

  - apiGroups: [""]
    resources:
      - secrets                     # Restrictive rule for Secrets
    verbs:
      - get                         # Developers can read secrets (for debugging)
      # create/update/delete intentionally omitted — platform team manages secrets

---
# ============================================================
# 2. ROLEBINDING — Assign a Role to a user in a namespace
# ============================================================
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                   # Grants Role permissions in ONE namespace
metadata:
  name: team-a-developers           # Name of this binding
  namespace: team-a                 # Binding only applies to team-a namespace
  labels:
    team: team-a
subjects:                           # WHO gets these permissions
  - kind: User                      # A human user (authenticated via OIDC/cert)
    name: alice@company.com         # Exact user name as seen by the API server
    apiGroup: rbac.authorization.k8s.io

  - kind: User
    name: bob@company.com
    apiGroup: rbac.authorization.k8s.io

  - kind: Group                     # A group of users (from OIDC groups claim)
    name: team-a-engineers          # All users with this group get the permissions
    apiGroup: rbac.authorization.k8s.io

  - kind: ServiceAccount            # A machine identity (Pod's identity)
    name: team-a-deploy-sa          # ServiceAccount name
    namespace: team-a               # ServiceAccount's namespace (may differ from binding's NS)

roleRef:                            # WHAT permissions are granted
  kind: Role                        # Referring to a Role (namespace-scoped)
  name: app-developer               # The Role defined above
  apiGroup: rbac.authorization.k8s.io
  # NOTE: roleRef is immutable after creation — delete and recreate to change

---
# ============================================================
# 3. CLUSTERROLE — Cluster-wide permissions
# ============================================================
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole                   # Scope: cluster-wide OR reusable template
metadata:
  name: monitoring-reader           # Cluster-unique name
  labels:
    app: monitoring
    managed-by: platform-team
rules:
  - apiGroups: [""]
    resources:
      - nodes                       # Cluster-scoped resource — only ClusterRole can grant this
      - nodes/metrics               # Node metrics subresource
      - pods                        # Pod listing across all namespaces
      - services
      - endpoints
      - namespaces                  # Cluster-scoped: cannot be in a Role
      - persistentvolumes           # Cluster-scoped: cannot be in a Role
    verbs:
      - get
      - list
      - watch                       # Read-only: never create/update/delete in monitoring role

  - apiGroups: ["apps"]
    resources:
      - deployments
      - daemonsets
      - statefulsets
      - replicasets
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["batch"]
    resources:
      - jobs
      - cronjobs
    verbs:
      - get
      - list
      - watch

  - nonResourceURLs:               # Non-resource endpoints (not K8s objects)
      - "/metrics"                  # API server metrics endpoint
      - "/healthz"                  # Health check endpoint
      - "/readyz"
    verbs:
      - get                         # Prometheus needs this to scrape API server

---
# ============================================================
# 4. CLUSTERROLEBINDING — Assign ClusterRole to ALL namespaces
# ============================================================
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding            # Grants ClusterRole across ALL namespaces
metadata:
  name: prometheus-monitoring       # Cluster-unique name
  labels:
    app: monitoring
subjects:
  - kind: ServiceAccount
    name: prometheus-sa             # Prometheus ServiceAccount
    namespace: monitoring           # Prometheus lives in monitoring namespace
    # This SA will have monitoring-reader permissions in ALL namespaces
    apiGroup: ""                    # Empty for ServiceAccount subjects

  - kind: Group
    name: platform-sre              # SRE team gets cluster-wide read access
    apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: ClusterRole
  name: monitoring-reader           # References the ClusterRole defined above
  apiGroup: rbac.authorization.k8s.io

---
# ============================================================
# 5. ROLEBINDING referencing CLUSTERROLE (reuse pattern)
# This is the most powerful and commonly missed RBAC pattern
# ============================================================
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                   # Namespace-scoped binding
metadata:
  name: cicd-deployer-production    # Unique name within production namespace
  namespace: production             # Binding only effective in production namespace
subjects:
  - kind: ServiceAccount
    name: cicd-sa                   # CI/CD pipeline service account
    namespace: ci-cd                # SA is in ci-cd namespace, but binding in production
    apiGroup: ""

roleRef:
  kind: ClusterRole                 # REFERENCES a ClusterRole (not a Role)
  name: edit                        # Built-in ClusterRole with create/update/patch/delete
  apiGroup: rbac.authorization.k8s.io
  # RESULT: cicd-sa can do anything 'edit' allows, BUT ONLY in production namespace
  # This avoids creating a duplicate Role in every namespace where CI/CD deploys
  # The same ClusterRole template is scoped to one namespace by this RoleBinding
```

---

## AWS/EKS Perspective

### EKS RBAC Integration with IAM:

**AWS IAM and Kubernetes RBAC are separate systems.** EKS bridges them via the `aws-auth` ConfigMap (legacy) or EKS Access Entries (new, recommended for K8s 1.29+).

**Legacy: `aws-auth` ConfigMap (kube-system):**
```yaml
# Maps IAM users/roles to Kubernetes usernames/groups
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # EKS Node Group IAM Role (required for nodes to join cluster)
    - rolearn: arn:aws:iam::123456:role/eks-node-group-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

    # CI/CD Pipeline IAM Role mapped to a Kubernetes username
    - rolearn: arn:aws:iam::123456:role/GitHubActionsDeployRole
      username: github-actions           # Kubernetes username
      groups:
        - deployers                      # Kubernetes group — bind RBAC here

    # Developer team IAM Role mapped to a Kubernetes group
    - rolearn: arn:aws:iam::123456:role/DeveloperRole
      username: "{{SessionName}}"        # Maps to individual developer
      groups:
        - team-a-engineers               # Kubernetes group used in RoleBinding subjects

  mapUsers: |
    # Individual IAM User (use sparingly — prefer roles)
    - userarn: arn:aws:iam::123456:user/alice
      username: alice@company.com
      groups:
        - team-a-engineers
```

**New: EKS Access Entries (K8s 1.29+, recommended):**
```bash
# Create an access entry for an IAM role
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456:role/DeveloperRole \
  --type STANDARD \
  --kubernetes-groups team-a-engineers

# Associate with a Kubernetes access policy (AWS managed)
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456:role/DeveloperRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=team-a
```

**Common EKS RBAC Pattern — Least Privilege for AWS:**
```bash
# 1. Create IAM Role for team
aws iam create-role --role-name EKS-TeamA-Developers ...

# 2. Map role to K8s group in aws-auth or Access Entry
# 3. Create K8s RoleBinding that grants permissions to the group
kubectl create rolebinding team-a-dev \
  --clusterrole=edit \
  --group=team-a-engineers \
  -n team-a

# 4. Developers assume the IAM role, get K8s permissions
aws eks get-token --cluster-name my-cluster
# Token contains the IAM role ARN, which is mapped to team-a-engineers group
# RoleBinding grants edit permissions in team-a namespace
```

**EKS-Specific RBAC Gotcha:**
The `system:masters` group is bound to `cluster-admin` in every cluster. The EKS cluster creator's IAM entity is automatically in this group via the initial `aws-auth` ConfigMap entry. If you lose access to this IAM entity, you lose cluster admin access — plan accordingly with multiple admin IAM roles.

---

## Interview Answer (2-Minute Version)

"Kubernetes RBAC controls what users and service accounts can do in a cluster using four resources. A Role defines permissions within a single namespace. A ClusterRole defines permissions cluster-wide or for cluster-scoped resources like Nodes. A RoleBinding assigns a Role (or ClusterRole) to a user or service account within a specific namespace. A ClusterRoleBinding assigns a ClusterRole across all namespaces.

A key pattern often missed: you can use a RoleBinding to reference a ClusterRole, which gives you a reusable permission template that's scoped to a single namespace. This avoids duplicating Role definitions across multiple namespaces.

In production, I design RBAC around principle of least privilege — CI/CD service accounts get create/update but not delete; monitoring systems get list/watch but not write; and I use groups instead of individual users so permissions follow team membership automatically."

---

## Interview Answer (Senior Engineer Version)

"RBAC in Kubernetes is a purely additive, allow-only authorization system — you can only grant permissions, never deny them. The four resources form two concepts: Role and ClusterRole define what can be done, while RoleBinding and ClusterRoleBinding define who can do it and where.

The nuanced pattern most engineers miss is that RoleBinding can reference a ClusterRole, creating a permission template that's namespace-scoped at assignment time. This is how built-in ClusterRoles like `edit` and `view` are designed to be reused — you write the template once in a ClusterRole and bind it per-namespace with RoleBindings.

In production multi-tenant clusters, I enforce least privilege with namespace-scoped bindings, use `kubectl auth can-i --list` regularly to audit permission creep, and implement OPA/Kyverno policies that reject ClusterRoleBindings granting `cluster-admin` except for explicitly approved subjects.

On EKS, the RBAC bridge to IAM via `aws-auth` ConfigMap or Access Entries is critical to understand. IAM handles authentication — who you are — and Kubernetes RBAC handles authorization — what you can do. I use IRSA for service account AWS permissions and IAM role-to-group mappings for human access, so Kubernetes RBAC stays consistent regardless of who assumes the IAM role."

---

## What Impresses the Interviewer

- Knowing RoleBinding can reference ClusterRole (the reuse pattern)
- Stating that RBAC is additive-only (no deny rules)
- Mentioning `kubectl auth can-i --list` for auditing
- Explaining EKS aws-auth ConfigMap and Access Entries
- Discussing `system:masters` bypass behavior
- Knowing `nonResourceURLs` for API server metrics access
- Mentioning tools like `rbac-lookup`, `kubectl-who-can`, or `rakkess`

---

## Red Flags

- "I can deny access with RBAC" — RBAC is additive only, no deny rules
- Confusing authentication with authorization — IAM is auth, RBAC is authz
- Not knowing the RoleBinding → ClusterRole pattern
- "I use cluster-admin for everything" — Shows no security awareness
- Not knowing `kubectl auth can-i` for testing permissions
- Inability to explain namespace scope vs cluster scope clearly

---

## Production Best Practices

1. **Follow least privilege principle** — Start with `view` ClusterRole and add only what's needed. Never start with `cluster-admin` and remove permissions.

2. **Use Groups, not individual Users** — Bind RBAC to groups (`system:authenticated`, `team-a-engineers`). When team members change, update group membership — not dozens of RoleBindings.

3. **Reuse ClusterRoles via RoleBinding** — Define permission templates as ClusterRoles and bind them per-namespace with RoleBindings. Avoid duplicating Role objects across namespaces.

4. **Audit RBAC monthly** — Use `kubectl auth can-i --list --as=<subject>` or tools like `rakkess`/`rbac-lookup` to review effective permissions. Permission creep is real.

5. **Prohibit `cluster-admin` for automation** — CI/CD systems should have minimal ClusterRoles. Use OPA/Kyverno admission webhooks to reject ClusterRoleBindings to `cluster-admin` for non-approved subjects.

6. **Use ServiceAccounts for all workloads** — Never use the `default` ServiceAccount. Each workload should have its own SA with minimal permissions. Set `automountServiceAccountToken: false` by default, and enable per-workload as needed.

7. **Treat RBAC as code** — Store all RBAC manifests in Git, require peer review for ClusterRoleBinding changes, and use GitOps (ArgoCD/Flux) to apply them — preventing out-of-band privilege escalation.

8. **Monitor for privilege escalation** — Use Falco or Sysdig to alert when pods access unexpected Kubernetes API paths. An application that suddenly calls `/api/v1/secrets` is a red flag.

---

## Key Points to Remember

- Role = namespace-scoped permissions; ClusterRole = cluster-wide permissions or reusable template
- RoleBinding = assigns permissions in ONE namespace; ClusterRoleBinding = assigns across ALL namespaces
- RoleBinding CAN reference a ClusterRole — this scopes a reusable template to one namespace
- RBAC is additive ONLY — no deny rules exist; remove bindings to revoke access
- `kubectl auth can-i` is the primary debugging tool for RBAC issues
- Built-in ClusterRoles: `cluster-admin`, `admin`, `edit`, `view` — use these before creating custom ones
- `system:masters` group bypasses RBAC entirely — use for emergency break-glass only
- On EKS: IAM handles authentication, Kubernetes RBAC handles authorization — two separate systems
- RBAC subjects: `User`, `Group`, `ServiceAccount` — all case-sensitive in YAML
- `roleRef` in a binding is immutable after creation — delete and recreate to change the referenced role

---

## Interviewer's Expectation

The interviewer is testing:
1. **Four-resource mastery** — Can you explain Role, ClusterRole, RoleBinding, ClusterRoleBinding clearly with scope differences?
2. **The RoleBinding → ClusterRole pattern** — This separates junior from senior candidates
3. **Security mindset** — Do you default to least privilege or do you mention cluster-admin?
4. **Debugging ability** — Can you use `kubectl auth can-i` and interpret its output?
5. **EKS awareness** — Do you understand how IAM maps to Kubernetes RBAC?
6. **Production experience** — Do you mention auditing, GitOps, and RBAC as code?

---

## Final Perfect Interview Answer

"Kubernetes RBAC uses four resources to control access. Role and ClusterRole define what can be done — verbs like get, list, create on resources like pods, deployments. The difference is scope: Role is namespace-scoped, ClusterRole is cluster-wide or handles cluster-scoped resources like Nodes and PersistentVolumes.

RoleBinding and ClusterRoleBinding define who gets those permissions. RoleBinding grants access within one namespace; ClusterRoleBinding grants access across all namespaces.

The pattern most engineers overlook: a RoleBinding can reference a ClusterRole. This means you define a permission template once as a ClusterRole, then scope it per-namespace with RoleBindings — avoiding duplicate Role definitions across every namespace.

RBAC is purely additive — you can only grant, never deny. To revoke access, you delete the binding, not add a deny rule.

In production I design with least privilege — CI/CD gets create/update but not delete, monitoring gets read-only cluster-wide via ClusterRoleBinding, and teams get edit access only in their namespace via RoleBinding. I store all RBAC manifests in Git, review ClusterRoleBinding changes with OPA policies, and audit effective permissions monthly with `kubectl auth can-i --list`. On EKS, I map IAM roles to Kubernetes groups via Access Entries, keeping authentication in IAM and authorization cleanly in Kubernetes RBAC."
