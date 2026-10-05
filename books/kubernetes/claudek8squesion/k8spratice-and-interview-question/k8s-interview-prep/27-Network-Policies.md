# Network Policies — Kubernetes Interview Guide

---

## Interview Question

**"What are Kubernetes Network Policies? How do you implement a zero-trust network model in Kubernetes using Network Policies?"**

---

## Simple Explanation

By default, every pod in a Kubernetes cluster can talk to every other pod — like an office where every desk can call every other desk with no restrictions. This is fine for development but dangerous in production.

**Network Policies are like a firewall inside your Kubernetes cluster.** They let you define rules about which pods can talk to which other pods, and which external traffic is allowed.

**Everyday analogy:**
Think of a hospital:
- The pharmacy (database pods) should ONLY receive requests from the dispensing desk (backend pods), not from the reception (frontend) or visitors.
- The ICU (sensitive services) should ONLY accept connections from authorized medical staff pods.
- Network Policies are the security guards at each door, checking credentials before allowing entry.

**Key concept:** Network Policies are **allow rules** — by default, if no policy exists, all traffic is allowed. Once you create a policy that selects a pod, all traffic not explicitly allowed by that policy is denied.

---

## Technical Explanation

### How Network Policies Work

Network Policies are implemented at the **CNI (Container Network Interface) plugin level** — NOT by Kubernetes core. The Kubernetes API server only stores the policy objects; the enforcement is done by the CNI plugin.

**CNI plugins that support Network Policies:**
- **Calico** — Most widely used, supports both Kubernetes and Calico-native policies
- **Cilium** — eBPF-based, excellent performance, also supports L7 policies
- **Weave Net** — Supports standard Kubernetes Network Policies
- **Antrea** — VMware's CNI, used in some enterprise deployments

**CNI plugins that do NOT enforce Network Policies (despite Kubernetes accepting the objects):**
- Flannel (by default)
- kubenet

**CRITICAL:** If you apply a Network Policy and your CNI doesn't support enforcement, the policy will be silently ignored. This is a major security risk.

### Policy Selection Mechanism

A Network Policy selects pods using **podSelector** (label-based). The policy then defines ingress (incoming) and egress (outgoing) rules.

**Three types of traffic sources/destinations in rules:**
1. **podSelector** — other pods (optionally combined with namespaceSelector)
2. **namespaceSelector** — all pods in selected namespaces
3. **ipBlock** — specific IP CIDR ranges (for external traffic)

### Policy Types

- **Ingress** — Controls incoming traffic TO the selected pods
- **Egress** — Controls outgoing traffic FROM the selected pods
- If `policyTypes` includes `Ingress` but no `ingress` rules are specified → all ingress is DENIED
- If `policyTypes` includes `Egress` but no `egress` rules are specified → all egress is DENIED

### Default Behavior (No Policy)

```
Pod A  <──────────────────>  Pod B     # All traffic allowed
Pod A  <──────────────────>  External  # All traffic allowed
```

### After Deny-All Policy Applied

```
Pod A  ✗──────────────────✗  Pod B     # All traffic blocked
Pod A  ✗──────────────────✗  External  # All traffic blocked
```

### After Allow-Specific Policy Added

```
Frontend ──────────────────>  Backend   # Explicitly allowed
Frontend ✗──────────────────✗ Database  # Still blocked
Backend  ──────────────────>  Database  # Explicitly allowed
```

---

## Real-World Example

### Scenario: E-Commerce Platform Microservices Isolation

**Architecture:**
- `frontend` pods (namespace: `web`) — serve the React UI
- `backend` pods (namespace: `app`) — REST API
- `database` pods (namespace: `data`) — PostgreSQL

**Security requirement:** Database should ONLY receive traffic from backend. Frontend should NOT be able to directly query the database.

**Without Network Policies:**
- A compromised frontend pod could directly connect to the database and exfiltrate all customer data.

**With Network Policies:**
- Even if frontend is compromised, it cannot reach the database.
- The database only accepts connections from specifically labeled backend pods.

---

## Diagram / Flow

```
NETWORK POLICY ARCHITECTURE
=============================

NAMESPACE: web                NAMESPACE: app              NAMESPACE: data
┌─────────────────────┐      ┌──────────────────────┐    ┌─────────────────────┐
│   frontend pods     │      │   backend pods        │    │   database pods     │
│   app: frontend     │      │   app: backend        │    │   app: postgres     │
│                     │      │                       │    │                     │
│  ┌───────────────┐  │      │  ┌────────────────┐  │    │  ┌───────────────┐  │
│  │ frontend:8080 │──┼──────┼─>│ backend:8080   │  │    │  │ postgres:5432 │  │
│  └───────────────┘  │  ✓   │  └────────────────┘  │    │  └───────────────┘  │
│                     │      │           │           │    │          ▲          │
│  ✗ Direct DB access │      │           │ ✓         │    │          │          │
│  blocked            │      │           ▼           │    │          │          │
└─────────────────────┘      │  ┌────────────────┐  │────┼──────────┘          │
                             │  │  DB connection │  │ ✓  │                     │
                             │  └────────────────┘  │    └─────────────────────┘
         ✗                   └──────────────────────┘
INTERNET ──────────────────────────────────────────────> database
(direct access blocked)


INGRESS vs EGRESS RULES
========================

          INGRESS (incoming)
               │
               ▼
  ┌────────────────────────┐
  │       POD              │
  └────────────────────────┘
               │
               ▼
          EGRESS (outgoing)


DENY-ALL THEN ALLOW PATTERN (Zero Trust)
==========================================

  Step 1: Apply deny-all to namespace
  ┌──────────────────────────────────────┐
  │  NAMESPACE: production               │
  │                                      │
  │  Policy: deny-all                    │
  │  ┌───────┐    ✗    ┌───────┐         │
  │  │ pod-A │─────────│ pod-B │         │
  │  └───────┘         └───────┘         │
  │                                      │
  │  ALL traffic blocked                 │
  └──────────────────────────────────────┘

  Step 2: Add specific allow rules
  ┌──────────────────────────────────────┐
  │  NAMESPACE: production               │
  │                                      │
  │  Policy: allow-frontend-to-backend   │
  │  ┌──────────┐    ✓    ┌──────────┐  │
  │  │ frontend │─────────│ backend  │  │
  │  └──────────┘  :8080  └──────────┘  │
  │                                      │
  │  Policy: allow-backend-to-db         │
  │  ┌──────────┐    ✓    ┌──────────┐  │
  │  │ backend  │─────────│ database │  │
  │  └──────────┘  :5432  └──────────┘  │
  └──────────────────────────────────────┘


CNI ENFORCEMENT FLOW
=====================

  kubectl apply -f network-policy.yaml
              │
              ▼
  ┌─────────────────────┐
  │   Kubernetes API    │
  │   Server            │
  │   (stores policy)   │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │   CNI Plugin        │
  │   (e.g. Calico)     │
  │   Watches for new   │
  │   NetworkPolicy     │
  │   objects           │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │   iptables / eBPF   │
  │   rules on each     │
  │   node              │
  │   (actual           │
  │   enforcement)      │
  └─────────────────────┘
```

---

## Why It Is Important

### Business Value
- **Data Protection:** Prevents lateral movement — even if one pod is compromised, the attacker can't reach sensitive data pods.
- **Compliance:** PCI-DSS, HIPAA, SOC 2 require network segmentation. Network Policies provide this in Kubernetes.
- **Breach Containment:** Limits blast radius of security incidents to single pods or namespaces.
- **Regulatory Audits:** Demonstrable network segmentation passes audit requirements.

### Technical Value
- **Zero-Trust Networking:** Implements "never trust, always verify" at the network layer.
- **Namespace Isolation:** Prevents cross-namespace communication without explicit permission.
- **Microservice Hardening:** Each service only exposes exactly what it needs to.
- **Defense in Depth:** Adds network-layer security even if application-layer auth is bypassed.

---

## Common Interview Follow-Up Questions

1. **"Does Kubernetes enforce Network Policies natively?"**
   - No. Kubernetes only stores the NetworkPolicy objects in etcd. Enforcement is done by the CNI plugin. If you have Flannel (no enforcement), your policies exist but have zero effect.

2. **"What happens if you apply a Network Policy with no ingress rules?"**
   - If `policyTypes` includes `Ingress` with no ingress rules specified, ALL ingress to selected pods is denied. This is how you create a deny-all policy.

3. **"Can Network Policies control egress traffic?"**
   - Yes, using `policyTypes: [Egress]` and defining `egress` rules. You can restrict which external IPs or which pods a pod can connect to.

4. **"How do you allow DNS resolution in a deny-all egress policy?"**
   - Always add an egress rule allowing UDP/TCP port 53 to kube-dns (namespace: kube-system, label: k8s-app=kube-dns) or to CIDR 0.0.0.0/0 on port 53.

5. **"What is the difference between podSelector and namespaceSelector?"**
   - podSelector selects pods by labels within the same (or specified) namespace. namespaceSelector selects entire namespaces by their labels. Combined in one rule with AND logic; in separate rules with OR logic.

6. **"Can Network Policies enforce Layer 7 (HTTP) rules?"**
   - Standard Kubernetes NetworkPolicy is Layer 3/4 only (IP/port). For L7 (HTTP path, headers), you need Cilium's CiliumNetworkPolicy or a service mesh like Istio.

7. **"How do you test if a Network Policy is working?"**
   - Deploy a test pod (`kubectl run test --image=busybox`) and try to curl/nc from it to the target pod. Or use `kubectl exec` into a pod and test connectivity.

8. **"What is a default-deny-all policy and why should every production namespace have one?"**
   - A policy that selects all pods with an empty podSelector and specifies ingress/egress types with no rules. Forces explicit whitelisting — nothing is allowed unless explicitly permitted.

---

## Common Mistakes Candidates Make

### Mistake 1: Thinking Kubernetes enforces policies natively
**Wrong:** "I just apply the YAML and Kubernetes enforces the network rules."
**Correct:** Kubernetes only stores the policy. The CNI plugin (Calico, Cilium, etc.) does the actual enforcement. Without a compatible CNI, policies are silently ignored.

### Mistake 2: Forgetting DNS egress in deny-all policies
**Wrong:** "I'll deny all egress and then add rules for my services."
**Correct:** If you deny all egress without allowing port 53 (DNS), pods can't resolve service names. Always add a DNS exemption: `egress: [{ports: [{port: 53, protocol: UDP}]}]`

### Mistake 3: AND vs OR in selectors
**Wrong:** "Combining podSelector and namespaceSelector in a rule means either condition matches."
**Correct:** When both are in the SAME rule block (same `from` entry), it's AND logic — both conditions must match. When they're in SEPARATE `from` entries, it's OR logic.

### Mistake 4: Network Policies are not stateful
**Wrong:** "I need separate egress rules for both directions of communication."
**Correct:** Network Policies are stateful (connection tracking). If you allow ingress on port 80, the response traffic is automatically allowed. You don't need matching egress rules for responses.

### Mistake 5: Empty podSelector means "no pods"
**Wrong:** "An empty `podSelector: {}` matches no pods — the policy applies to nothing."
**Correct:** An empty `podSelector: {}` matches ALL pods in the namespace. This is intentional for creating namespace-wide deny-all policies.

---

## Troubleshooting Scenario

### Problem: Backend Service Cannot Connect to Database After Security Hardening

**Situation:** Security team applied deny-all policies to the production namespace. Now the backend API pods are returning 500 errors because they can't connect to PostgreSQL.

**Step-by-Step Debug:**

```bash
# Step 1: Check if pods are running
kubectl get pods -n production
# All pods Running, but backend logs show connection refused to postgres

# Step 2: Test connectivity from backend pod
kubectl exec -it backend-pod-xyz -n production -- \
  nc -zv postgres-service 5432
# Output: nc: connect to postgres-service (10.96.45.12) port 5432 (tcp): Connection refused

# Step 3: List all Network Policies in the namespace
kubectl get networkpolicies -n production
# NAME              POD-SELECTOR   AGE
# deny-all          <none>         2h
# allow-monitoring  app=backend    30m

# Step 4: Describe the deny-all policy
kubectl describe networkpolicy deny-all -n production
# Shows: Spec: PodSelector: <none> (Allowing the specific traffic to all pods)
#         PolicyTypes: Ingress, Egress
#         No Ingress rules → all ingress denied
#         No Egress rules → all egress denied

# Step 5: Check what policies apply to the postgres pod
kubectl describe networkpolicy -n production | grep -A5 "Pod Selector"

# Step 6: Check if there's an allow policy for postgres ingress
kubectl get networkpolicy allow-postgres-ingress -n production
# Error: not found — this policy is MISSING

# Step 7: Check the backend pod's labels (needed for the selector in the policy)
kubectl get pod backend-pod-xyz -n production --show-labels
# Labels: app=backend, tier=api

# Step 8: Check the postgres pod's labels
kubectl get pod postgres-pod-abc -n production --show-labels
# Labels: app=postgres, tier=database

# Step 9: Create the missing allow policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
EOF

# Step 10: Verify fix
kubectl exec -it backend-pod-xyz -n production -- \
  nc -zv postgres-service 5432
# Output: Connection to postgres-service (10.96.45.12) port 5432 [tcp/postgresql] succeeded!

# Step 11: Check backend logs
kubectl logs backend-pod-xyz -n production --tail=20
# Should show successful database connections

# Bonus: Verify with a network policy testing tool
# Install netassert or use Cilium's hubble for policy visualization
```

---

## kubectl Commands

```bash
# List all network policies in a namespace
kubectl get networkpolicies -n production
# Expected:
# NAME                     POD-SELECTOR   AGE
# deny-all                 <none>         2h
# allow-frontend-backend   app=frontend   1h
# allow-backend-db         app=backend    1h

# Describe a specific network policy
kubectl describe networkpolicy allow-frontend-backend -n production
# Expected: Shows ingress/egress rules, selectors, ports

# Apply a network policy from a file
kubectl apply -f network-policy.yaml
# Expected: networkpolicy.networking.k8s.io/allow-frontend-backend created

# Delete a network policy
kubectl delete networkpolicy deny-all -n production
# Expected: networkpolicy.networking.k8s.io/deny-all deleted

# Test pod-to-pod connectivity
kubectl exec -it frontend-pod -n web -- \
  curl -v http://backend-service.app.svc.cluster.local:8080/health
# Expected: HTTP 200 if policy allows, connection refused if denied

# Run a temporary test pod for connectivity testing
kubectl run nettest \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -n production \
  -- wget -qO- http://postgres-service:5432
# Expected: Connection established (or refused based on policy)

# Check all pods in namespace with their labels (needed for policy troubleshooting)
kubectl get pods -n production --show-labels

# Get network policies across ALL namespaces
kubectl get networkpolicies --all-namespaces
# Expected: Lists policies with their namespaces

# Check if a specific pod is affected by any network policy
kubectl get networkpolicies -n production -o yaml | \
  grep -A5 "podSelector"

# View the full YAML of a network policy
kubectl get networkpolicy allow-backend-db -n production -o yaml

# Watch network policy events (useful during apply)
kubectl get events -n production --sort-by='.lastTimestamp' | grep NetworkPolicy
```

---

## YAML Example

```yaml
# =========================================================
# EXAMPLE 1: DEFAULT DENY-ALL POLICY (Zero Trust Foundation)
# Apply this FIRST to every production namespace
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
  annotations:
    description: "Denies all ingress and egress by default. Add explicit allow policies."
spec:
  # Empty podSelector matches ALL pods in the namespace
  podSelector: {}
  # Specifying both types with no rules = deny all
  policyTypes:
    - Ingress
    - Egress
  # No ingress rules → all incoming traffic denied
  # No egress rules → all outgoing traffic denied

---
# =========================================================
# EXAMPLE 2: ALLOW DNS EGRESS (Required for service discovery)
# Without this, pods cannot resolve service names
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}           # Applies to all pods in namespace
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 53
          protocol: UDP     # DNS over UDP
        - port: 53
          protocol: TCP     # DNS over TCP (fallback for large responses)

---
# =========================================================
# EXAMPLE 3: ALLOW FRONTEND TO BACKEND (Specific service access)
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  # This policy applies TO the backend pods (DESTINATION)
  podSelector:
    matchLabels:
      app: backend
      tier: api
  policyTypes:
    - Ingress
  ingress:
    # Allow traffic FROM frontend pods
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080        # Only allow on port 8080, not all ports

---
# =========================================================
# EXAMPLE 4: ALLOW BACKEND TO DATABASE (Cross-namespace)
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: data           # Policy lives in the data namespace
spec:
  # This policy applies TO postgres pods
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    # Allow traffic from backend pods in the 'app' namespace
    # BOTH conditions must be true (AND logic in same 'from' entry)
    - from:
        - podSelector:
            matchLabels:
              app: backend
          namespaceSelector:    # Combined in same entry = AND
            matchLabels:
              kubernetes.io/metadata.name: app
      ports:
        - protocol: TCP
          port: 5432            # PostgreSQL port only

---
# =========================================================
# EXAMPLE 5: DENY ALL INGRESS EXCEPT FROM MONITORING
# Allow Prometheus to scrape metrics from all pods
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scraping
  namespace: production
spec:
  podSelector: {}   # Applies to all pods in production namespace
  policyTypes:
    - Ingress
  ingress:
    # Allow traffic from prometheus pods in monitoring namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090    # Or whatever your metrics port is
        - protocol: TCP
          port: 8080    # Some apps expose metrics on app port

---
# =========================================================
# EXAMPLE 6: ALLOW EXTERNAL TRAFFIC FROM SPECIFIC CIDR
# Used when an on-premises system needs to reach a K8s service
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-onprem-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
    - Ingress
  ingress:
    - from:
        # Allow from specific on-premises IP range
        - ipBlock:
            cidr: 192.168.10.0/24       # On-prem office network
        # Exclude a specific IP within that range
        - ipBlock:
            cidr: 10.0.0.0/8            # Internal VPC range
            except:
              - 10.0.1.0/24             # Exclude this specific subnet
      ports:
        - protocol: TCP
          port: 443

---
# =========================================================
# EXAMPLE 7: COMPLETE EGRESS POLICY FOR BACKEND
# Controls all outbound connections from backend pods
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    # Allow DNS resolution
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

    # Allow connection to PostgreSQL database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Allow connection to Redis cache
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

    # Allow HTTPS to external APIs (e.g., payment gateway, AWS services)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0            # Any external IP
            except:
              - 10.0.0.0/8             # Exclude private ranges
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443                     # HTTPS only

---
# =========================================================
# EXAMPLE 8: OR vs AND SELECTOR LOGIC DEMONSTRATION
# =========================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: or-vs-and-demo
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: target
  policyTypes:
    - Ingress
  ingress:
    # ENTRY 1: Separate items in 'from' list = OR logic
    # Traffic allowed if: (from pods labeled app=service-a) OR (from monitoring namespace)
    - from:
        - podSelector:             # OR condition 1
            matchLabels:
              app: service-a
        - namespaceSelector:       # OR condition 2
            matchLabels:
              purpose: monitoring

    # ENTRY 2: Combined in same item = AND logic
    # Traffic allowed if: (from pods labeled app=service-b) AND (from app-team namespace)
    - from:
        - podSelector:             # AND condition: both must be true
            matchLabels:
              app: service-b
          namespaceSelector:       # Same indentation level = AND
            matchLabels:
              kubernetes.io/metadata.name: app-team
```

---

## AWS/EKS Perspective

### CNI Plugin on EKS

**Default EKS CNI (amazon-vpc-cni):**
- Amazon VPC CNI is the default plugin on EKS.
- By default, it does NOT enforce Kubernetes Network Policies.
- Since EKS 1.25+, Amazon VPC CNI supports Network Policy enforcement via `--enable-network-policy-controller` flag.
- Enable it by updating the `aws-node` DaemonSet configuration.

```bash
# Enable network policy support on EKS with VPC CNI (EKS 1.25+)
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "true"}'

# Verify Network Policy controller is running
kubectl get pods -n kube-system | grep network-policy
# Expected:
# aws-network-policy-agent-xxxxx   1/1   Running   0   5m
```

**Using Calico on EKS (recommended for advanced policies):**
```bash
# Install Calico as a CNI overlay (keeps AWS VPC CNI for networking, adds Calico for policies)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# OR using Helm
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm install calico projectcalico/tigera-operator --namespace tigera-operator
```

**Using Cilium on EKS:**
```bash
# Install Cilium with EKS support
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set eni.enabled=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set tunnel=disabled
```

### Security Groups vs Network Policies on EKS

| Feature | Security Groups | Network Policies |
|---|---|---|
| Level | AWS infrastructure | Kubernetes pod level |
| Granularity | Node/ENI level | Pod level |
| Management | AWS console/IaC | kubectl YAML |
| L7 support | No | Only with Cilium |
| Cross-cluster | Yes (VPC peering) | No |

**Best practice:** Use BOTH — Security Groups for node-level isolation, Network Policies for pod-level isolation.

### Security Groups for Pods (EKS Feature)

EKS supports "Security Groups for Pods" — assigning AWS Security Groups directly to individual pods for fine-grained AWS-level network control:

```yaml
# SecurityGroupPolicy CRD (EKS-specific)
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: my-security-group-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  securityGroups:
    groupIds:
      - sg-0123456789abcdef0    # Your AWS Security Group ID
```

---

## Interview Answer (2-Minute Version)

"Kubernetes Network Policies are firewall rules for pod-to-pod communication. By default, all pods can talk to all other pods. Network Policies let you restrict this.

A policy selects pods using label selectors and defines ingress rules for incoming traffic and egress rules for outgoing traffic. The key thing to know is that Kubernetes itself doesn't enforce policies — the CNI plugin does. Calico and Cilium enforce them; Flannel by default doesn't.

The recommended pattern is: apply a default deny-all policy to each namespace first, then explicitly allow only the necessary traffic. For example, allow frontend to backend on port 8080, and allow backend to database on port 5432. When creating egress deny-all policies, always remember to allow port 53 for DNS, or pods won't be able to resolve service names."

---

## Interview Answer (Senior Engineer Version)

"Network Policies implement zero-trust networking within Kubernetes clusters. The architecture is declarative — you define policies in YAML, store them in etcd via the API server, and the CNI plugin translates them into iptables rules or eBPF programs on each node.

On EKS, the default VPC CNI plugin added NetworkPolicy support in version 1.25+, but for production I still prefer Calico or Cilium. Cilium is particularly compelling because it uses eBPF rather than iptables, providing better performance at scale and supporting Layer 7 policies for HTTP path-based filtering.

The critical implementation detail most people miss is the AND vs OR logic in from/to selectors. When you combine podSelector and namespaceSelector in the same list entry, it's AND logic — both must match. When they're separate list entries, it's OR. This trips up a lot of teams who end up with unintended broad access.

For production, I implement a layered approach: namespace-level deny-all as a baseline, then service-specific allow policies using the principle of least privilege. I also maintain egress policies to control data exfiltration — even if an attacker compromises a pod, they can't use it to call out to external C2 infrastructure.

On EKS I also use Security Groups for Pods for AWS-level network controls alongside Kubernetes Network Policies for defense in depth. For L7 policy enforcement, I use Cilium's CiliumNetworkPolicy CRDs or service mesh policies via Istio."

---

## What Impresses the Interviewer

- Knowing that CNI enforcement is required — not all CNIs support it
- Explaining AND vs OR logic in selector combinations
- Mentioning the DNS egress rule requirement in deny-all policies
- Discussing Layer 7 policies (Cilium, Istio) vs Layer 3/4 policies
- Knowing EKS-specific options: VPC CNI network policy support, Security Groups for Pods
- Explaining eBPF vs iptables enforcement and performance implications
- Showing understanding of stateful connection tracking (return traffic is automatic)
- Mentioning compliance use cases (PCI-DSS, HIPAA network segmentation requirements)

---

## Red Flags

- Thinking Kubernetes core enforces Network Policies without a CNI
- Claiming deny-all with no rules blocks nothing (it actually blocks everything)
- Not knowing about the DNS port 53 egress exception requirement
- Never mentioning CNI plugin requirements
- Theory-only answers with no troubleshooting or production experience
- Not knowing the difference between ingress and egress policies
- Claiming Network Policies are stateless and requiring bidirectional rules

---

## Production Best Practices

1. **Start with deny-all, then allow explicitly:** Never work backwards from an allow-all model. Apply `default-deny-all` to every production namespace first, then add specific allow policies. This is the zero-trust approach.

2. **Always add DNS egress exemption:** A deny-all egress policy breaks DNS resolution. Always include an egress rule allowing port 53/UDP and 53/TCP to kube-dns.

3. **Use meaningful policy names:** Name policies descriptively like `allow-frontend-to-backend-8080` instead of `np-1`. Include the direction, source, destination, and port.

4. **Validate CNI enforcement:** Before relying on Network Policies for security, verify your CNI actually enforces them. Test with `kubectl exec` and connectivity tests. Flannel silently ignores policies.

5. **Use Cilium or Calico on EKS:** The default VPC CNI's network policy support is relatively new. For mature production environments, use Calico (battle-tested) or Cilium (modern eBPF, L7 support).

6. **Test policies in staging first:** A misconfigured egress policy can silently break service-to-service communication. Always test in non-production and use `kubectl exec` connectivity tests to verify before applying to production.

7. **Label namespaces for namespaceSelector:** To use `namespaceSelector`, namespaces must have identifying labels. Kubernetes 1.21+ auto-adds `kubernetes.io/metadata.name` label, but older clusters need manual labeling.

8. **Implement egress controls for data exfiltration prevention:** Don't just protect ingress. Restrict what pods can call out to — only allow known internal services and specific external HTTPS endpoints. This limits damage from compromised pods.

---

## Key Points to Remember

- Network Policies are **allow rules** — default is allow-all; once a policy selects a pod, unmatched traffic is denied
- **CNI plugin** enforces policies, not Kubernetes core — Calico, Cilium support enforcement; Flannel typically doesn't
- Empty `podSelector: {}` matches **ALL pods** in the namespace (useful for deny-all)
- **AND logic:** podSelector AND namespaceSelector in the same `from` entry
- **OR logic:** podSelector and namespaceSelector in separate `from` list entries
- **Stateful:** Return traffic is automatically allowed; you don't need bidirectional rules
- Always allow **port 53** (DNS) in egress policies or pods can't resolve service names
- Three levels of traffic selection: **podSelector**, **namespaceSelector**, **ipBlock**
- Policies are **additive** — multiple policies on the same pod combine with OR logic
- On **EKS**, enable VPC CNI network policy support or use Calico/Cilium overlay

---

## Interviewer's Expectation

The interviewer is testing whether you understand:

1. **CNI dependency** — Many candidates don't know the CNI is responsible for enforcement
2. **Policy logic** — AND vs OR in selectors, how deny-all actually works
3. **Real-world patterns** — Zero trust, deny-all + explicit allow, DNS exception
4. **Debugging skills** — How to troubleshoot pod connectivity issues caused by policies
5. **Security mindset** — Not just knowing the feature but understanding WHY it matters (compliance, lateral movement prevention)
6. **Cloud-specific knowledge** — EKS CNI options, Security Groups for Pods on EKS

At senior level, they expect L7 policy knowledge (Cilium, Istio), eBPF vs iptables trade-offs, and production-grade security architecture patterns.

---

## Final Perfect Interview Answer

"Kubernetes Network Policies are the mechanism for implementing pod-level network segmentation. By default, all pods can communicate freely with each other. Network Policies let you define explicit allow rules — once a policy selects a pod, any traffic not matched by a rule is denied.

The critical architecture point is that Kubernetes only stores policy objects; the CNI plugin does the actual enforcement. Calico and Cilium enforce policies; Flannel by default doesn't. This is a silent failure mode that many teams discover too late.

My production approach is zero-trust by default: apply a deny-all policy to every namespace first, then add specific allow policies. Always include a DNS egress exception — port 53 — otherwise pods can't resolve service names and everything breaks silently.

For selector logic, it's important to understand that combining podSelector and namespaceSelector in the same from entry creates AND logic, while separate list entries create OR logic. Getting this wrong leads to either too-permissive or too-restrictive policies.

On EKS, I use Cilium or Calico rather than relying solely on the VPC CNI's network policy support. Cilium is particularly powerful because it uses eBPF for high-performance enforcement and supports Layer 7 policies for HTTP-based filtering, which standard NetworkPolicy doesn't provide. For AWS-level controls, I combine this with Security Groups for Pods to get defense-in-depth network security."
