# Kubernetes Architecture — Kubernetes Interview Guide

## Interview Question
"Walk me through the architecture of a Kubernetes cluster. What are the main components, what does each one do, and how do they communicate with each other?"

---

## Simple Explanation
Think of Kubernetes like a big company with two departments: the Head Office and the Factory Floor.

The Head Office (Control Plane) is where all the decisions are made. There's a Reception Desk (API Server) that takes all requests. A Filing Cabinet (etcd) stores every important decision ever made. A Hiring Manager (Scheduler) decides which factory worker handles which job. A Supervisor Team (Controller Manager) constantly walks the floor making sure everything matches the plan.

The Factory Floor (Worker Nodes) is where the actual work happens. Each factory has a Floor Manager (kubelet) who gets instructions from Head Office and makes sure the right workers (containers inside Pods) are doing the right jobs. There's also a Switchboard Operator (kube-proxy) who makes sure calls get routed to the right worker.

When you tell Kubernetes "run 3 copies of my app," it goes like this:
1. You hand the request to Reception (API Server)
2. Reception files it in the Filing Cabinet (etcd)
3. The Hiring Manager (Scheduler) reads the filing cabinet and decides which factory floors to use
4. The Floor Managers (kubelets) on those factory floors get the job and start the containers
5. The Supervisors (Controllers) keep watching to make sure your 3 copies stay running forever

---

## Technical Explanation
Kubernetes architecture is divided into two planes: the **Control Plane** and the **Data Plane** (worker nodes).

### Control Plane Components

**1. kube-apiserver**
The API server is the only entry point into the cluster. Every operation — from kubectl commands to internal component communication — goes through the API server. It is stateless and horizontally scalable. It authenticates requests (using certificates, tokens, or OIDC), authorizes them (RBAC/ABAC), validates them via admission controllers (webhooks), and then persists state to etcd. It serves the Kubernetes REST API and also provides a watch mechanism so components can subscribe to resource changes.

**2. etcd**
A distributed key-value store based on the Raft consensus algorithm. It stores the entire cluster state: all resource definitions, configurations, secrets, and service discovery data. Every change to the cluster is first committed to etcd before any action is taken. In production, etcd runs as a 3 or 5 node cluster for high availability. It must be backed up regularly — losing etcd without a backup means total cluster state loss.

**3. kube-scheduler**
Watches for newly created Pods that have no node assigned. Uses a two-phase algorithm:
- **Filtering:** Eliminates nodes that cannot run the Pod (insufficient CPU/memory, taints without tolerations, failed affinity rules, Pod anti-affinity conflicts)
- **Scoring:** Ranks remaining nodes using priority functions (LeastRequestedPriority, BalancedResourceAllocation, ImageLocalityPriority, etc.)
The node with the highest score gets the Pod. The scheduler writes the node assignment back to the API server; it does NOT start the Pod itself.

**4. kube-controller-manager**
Runs multiple controllers in a single process. Each controller is a separate goroutine watching specific resource types. Key controllers:
- **Node Controller:** Monitors node health, marks nodes as NotReady, and evicts Pods from failed nodes
- **ReplicaSet Controller:** Ensures the correct number of Pod replicas are running
- **Deployment Controller:** Manages rolling updates and rollbacks
- **Service Account Controller:** Creates default service accounts in new namespaces
- **Endpoints Controller:** Populates the Endpoints object linking Services to Pods
- **Job Controller:** Manages batch Jobs, tracks completions
- **Namespace Controller:** Handles namespace deletion cleanup

**5. cloud-controller-manager**
Separates cloud-specific logic from core Kubernetes. Manages:
- Node lifecycle (detecting when cloud instances are deleted)
- Route configuration in the cloud network
- Load balancer provisioning (creates ELB/ALB when you create a LoadBalancer Service)

### Worker Node Components

**6. kubelet**
The primary node agent. Runs on every worker node. Receives PodSpecs (via the API server watch mechanism) and ensures the containers described in those specs are running and healthy. Communicates with the container runtime via the Container Runtime Interface (CRI). Runs liveness and readiness probes. Reports node and Pod status back to the API server. The kubelet does NOT manage containers it didn't create via Kubernetes.

**7. kube-proxy**
A network proxy running on each node. Implements the Kubernetes Service concept by maintaining network rules. Watches the API server for Service and Endpoint changes. Historically used iptables (one rule per endpoint), now supports IPVS mode (hash-based, scales to thousands of services). Does NOT proxy actual traffic — it programs the kernel's networking rules so the kernel routes packets directly.

**8. Container Runtime**
The low-level software that actually runs containers. Kubernetes communicates with it via the CRI (Container Runtime Interface) using gRPC. Supported runtimes: containerd (most common), CRI-O, Docker (deprecated). The runtime pulls images, creates and destroys containers, and manages their lifecycle.

### Add-ons (Critical but often overlooked)

**CoreDNS:** Cluster DNS. Every Service gets a DNS record (`service-name.namespace.svc.cluster.local`). Pods can resolve service names automatically.

**CNI Plugin:** Container Network Interface. Provides Pod-to-Pod networking. Examples: Calico (network policy + routing), Flannel (simple overlay), AWS VPC CNI (native VPC IPs for EKS), Cilium (eBPF-based).

---

## Real-World Example
**Company:** A FinTech startup processing real-time payment data, 40 engineers, multi-region AWS deployment.

**Architecture setup:**
- 3-node control plane in us-east-1, spread across 3 Availability Zones (AZs) for HA
- Control plane managed by EKS (AWS manages the HA and upgrades)
- 2 worker node groups: `on-demand-general` (always-on workloads like databases, auth) and `spot-compute` (batch processing, data pipelines — 60% cost savings)
- CoreDNS scaled to 4 replicas because DNS was a bottleneck (200k requests/min)
- Calico for network policies — payment service can only talk to the database, not to the public-facing API

**Incident they learned from:** The etcd cluster filled its disk at 3am because audit logs were being written to etcd instead of a file. API server requests started failing. Fix: configure audit logging to write to files, and set `--quota-backend-bytes` on etcd with proper monitoring alerts.

---

## Diagram / Flow

```
                    KUBERNETES CLUSTER ARCHITECTURE
=======================================================================

  CONTROL PLANE (Brain of the cluster)
+---------------------------------------------------------------------+
|                                                                     |
|  +-------------------+         +---------------------------+        |
|  |   kube-apiserver  |<------->|          etcd             |        |
|  |                   |         |  (Distributed KV Store)   |        |
|  | - Auth/AuthZ      |         |  - All cluster state      |        |
|  | - Admission ctrl  |         |  - 3 or 5 nodes for HA    |        |
|  | - REST API        |         |  - Raft consensus         |        |
|  | - Watch/notify    |         +---------------------------+        |
|  +--------^----------+                                              |
|           |  ^  ^                                                   |
|           |  |  |                                                   |
|  +--------v--+  +-------------------+                              |
|  |  kube-scheduler  |  | kube-controller-manager |                 |
|  |                  |  |                         |                 |
|  | - Watch unbound  |  | - Node Controller       |                 |
|  |   Pods           |  | - ReplicaSet Controller |                 |
|  | - Filter nodes   |  | - Deployment Controller |                 |
|  | - Score nodes    |  | - Endpoint Controller   |                 |
|  | - Bind Pod->Node |  | - Job Controller        |                 |
|  +------------------+  +-------------------------+                 |
|                                                                     |
|  +---------------------------+                                      |
|  | cloud-controller-manager  |                                      |
|  | - Node lifecycle          |                                      |
|  | - Load balancer mgmt      |                                      |
|  | - Cloud route config      |                                      |
|  +---------------------------+                                      |
+---------------------------------------------------------------------+
                          |  (HTTPS / gRPC)
                          |
     +--------------------+--------------------+
     |                                         |
     v                                         v
+------------------+                  +------------------+
|   WORKER NODE 1  |                  |   WORKER NODE 2  |
|                  |                  |                  |
|  +-----------+   |                  |  +-----------+   |
|  |  kubelet  |   |                  |  |  kubelet  |   |
|  | - Watch   |   |                  |  | - Watch   |   |
|  |   API svr |   |                  |  |   API svr |   |
|  | - Run     |   |                  |  | - Run     |   |
|  |   Pods    |   |                  |  |   Pods    |   |
|  | - Health  |   |                  |  | - Health  |   |
|  |   probes  |   |                  |  |   probes  |   |
|  +-----------+   |                  |  +-----------+   |
|                  |                  |                  |
|  +-----------+   |                  |  +-----------+   |
|  |kube-proxy |   |                  |  |kube-proxy |   |
|  | iptables/ |   |                  |  | iptables/ |   |
|  | IPVS rules|   |                  |  | IPVS rules|   |
|  +-----------+   |                  |  +-----------+   |
|                  |                  |                  |
|  +-----------+   |                  |  +-----------+   |
|  |containerd |   |                  |  |containerd |   |
|  +-----------+   |                  |  +-----------+   |
|                  |                  |                  |
|  +---+ +---+     |                  |  +---+ +---+     |
|  |Pod| |Pod|     |                  |  |Pod| |Pod|     |
|  +---+ +---+     |                  |  +---+ +---+     |
+------------------+                  +------------------+

FLOW: kubectl apply -f deployment.yaml
=============================================
User
 └─► API Server (validate + store in etcd)
      └─► Scheduler (watches for unbound Pods)
           └─► Picks best node, writes binding to API Server
                └─► kubelet on that node (watches API Server)
                     └─► Calls containerd via CRI
                          └─► Container starts running
                               └─► kubelet reports status to API Server
                                    └─► etcd updated with actual state
```

---

## Why It Is Important
Understanding architecture is fundamental to:
- **Debugging** — Knowing which component failed narrows root cause immediately
- **High Availability** — You can't design HA without understanding what breaks if each component fails
- **Security** — Every attack surface (API server, etcd, kubelet) requires specific hardening
- **Performance** — Bottlenecks at the scheduler, etcd, or API server affect the entire cluster
- **Capacity Planning** — Control plane components need their own resource reservations
- **Interviews** — This is the #1 most commonly tested topic for mid to senior Kubernetes roles

---

## Common Interview Follow-Up Questions
1. "What happens if etcd goes down?"
2. "Can the API server be made highly available? How?"
3. "What is the difference between kubelet and kube-proxy?"
4. "How does a Pod end up on a specific node — walk me through the scheduling process?"
5. "What is the Container Runtime Interface (CRI) and why does it matter?"
6. "What happens to running Pods if the control plane goes down completely?"
7. "How does kube-proxy implement Services — iptables vs IPVS?"
8. "What is the role of admission controllers and can you name a few?"

---

## Common Mistakes Candidates Make

**Mistake 1: Saying kubelet runs on the control plane**
Kubelet is a worker node component. The control plane has the API server, etcd, scheduler, and controller manager. Some setups run kubelet on control plane nodes too (to manage static Pods), but this is not the core answer.

**Mistake 2: Confusing the scheduler's job — thinking it starts containers**
The scheduler ONLY decides which node a Pod goes to. It writes a binding (node assignment) to the API server. The kubelet on that node actually starts the container. Two separate components, two separate jobs.

**Mistake 3: Not knowing etcd is a separate system**
Many candidates say "Kubernetes stores state in its database." etcd is a separate, distributed key-value store. It is not a SQL database. It uses the Raft consensus algorithm and runs as its own cluster. This distinction matters for backup, performance, and failure modes.

**Mistake 4: Thinking kube-proxy proxies traffic**
Despite the name, kube-proxy does NOT sit in the data path of network packets. It programs iptables or IPVS rules in the kernel. The kernel then routes packets directly. kube-proxy just manages those rules. Actual traffic never passes through the kube-proxy process.

**Mistake 5: Not knowing what happens when the control plane goes down**
Running Pods continue to run — the nodes keep containers alive. But: no new Pods can be scheduled, crashed Pods won't be replaced, Services still work (iptables rules already in place), but no new Services can be created. The cluster is frozen in its last state.

---

## Troubleshooting Scenario
**Problem:** After a planned maintenance window, the cluster seems unhealthy. Some Pods are stuck in Pending and kubectl commands are very slow.

```bash
# Step 1: Check if API server is responsive
kubectl get nodes
# If this hangs: API server or etcd problem

# Step 2: Check control plane Pod status (if using kubeadm-style setup)
kubectl get pods -n kube-system
# Look for: kube-apiserver, kube-scheduler, kube-controller-manager, etcd

# Step 3: If API server is slow, check etcd health
kubectl exec -it etcd-control-plane -n kube-system -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
# Expected: {"endpoint":"https://127.0.0.1:2379","health":true}

# Step 4: Check etcd disk usage (common problem)
kubectl exec -it etcd-control-plane -n kube-system -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
# Look at: DB SIZE column — should be under 8GB

# Step 5: Check scheduler logs
kubectl logs kube-scheduler-control-plane -n kube-system --tail=50
# Look for: errors about failed to schedule, binding errors

# Step 6: Check controller manager logs
kubectl logs kube-controller-manager-control-plane -n kube-system --tail=50
# Look for: errors in specific controllers

# Step 7: Check kubelet status on a worker node
# SSH into the node, then:
systemctl status kubelet
journalctl -u kubelet -n 100
# Look for: certificate errors, API server connection refused, disk pressure

# Step 8: Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=30
# Look for: iptables errors, sync failures
```

---

## kubectl Commands

```bash
# View all control plane components (kubeadm setup)
kubectl get pods -n kube-system
# Output:
# NAME                                    READY   STATUS    RESTARTS   AGE
# coredns-5d78c9869d-5mhxv               1/1     Running   0          30d
# etcd-control-plane                      1/1     Running   0          30d
# kube-apiserver-control-plane            1/1     Running   0          30d
# kube-controller-manager-control-plane   1/1     Running   0          30d
# kube-proxy-4bxzd                        1/1     Running   0          30d
# kube-scheduler-control-plane            1/1     Running   0          30d

# Check node details including conditions and allocatable resources
kubectl describe node worker-node-1
# Shows: Conditions (Ready, MemoryPressure, DiskPressure), 
#        Capacity vs Allocatable, System Info, Non-terminated Pods

# Get node resource usage
kubectl top nodes
# Output:
# NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# worker-node-1   435m         10%    2847Mi          36%
# worker-node-2   892m         22%    4102Mi          52%

# Check component status
kubectl get componentstatuses
# Output:
# NAME                 STATUS    MESSAGE   ERROR
# controller-manager   Healthy   ok
# scheduler            Healthy   ok
# etcd-0               Healthy   {"health":"true","reason":""}

# View cluster events sorted by time (great for debugging)
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Check what version each node is running
kubectl get nodes -o custom-columns='NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion'
# Output:
# NAME            VERSION
# worker-node-1   v1.29.0
# worker-node-2   v1.29.0

# View API server flags/config (if accessible)
kubectl get pods kube-apiserver-control-plane -n kube-system -o yaml | grep -A 20 "command:"

# Label a node (used for node selectors and affinity)
kubectl label node worker-node-1 node-type=high-memory

# Taint a node (prevents Pods without toleration from landing here)
kubectl taint node worker-node-1 dedicated=gpu:NoSchedule

# Cordon a node (mark unschedulable — no new Pods land here)
kubectl cordon worker-node-2
# Output: node/worker-node-2 cordoned

# Drain a node (evict all Pods, used for maintenance)
kubectl drain worker-node-2 --ignore-daemonsets --delete-emptydir-data
# Output: node/worker-node-2 drained

# Uncordon (make schedulable again after maintenance)
kubectl uncordon worker-node-2
```

---

## YAML Example

```yaml
# Static Pod definition for kube-apiserver (simplified for study)
# Static Pods are managed by kubelet directly, not by the API server
# They live in /etc/kubernetes/manifests/ on the control plane node
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver                      # Static Pod name
  namespace: kube-system                    # System namespace for k8s components
  labels:
    component: kube-apiserver               # Label identifying this as an API server
    tier: control-plane                     # Label identifying control-plane tier
spec:
  hostNetwork: true                         # Uses the host's network namespace (port 6443 on host)
  priorityClassName: system-cluster-critical  # Highest priority - never evicted
  containers:
  - name: kube-apiserver
    image: registry.k8s.io/kube-apiserver:v1.29.0  # Official k8s image
    command:
    - kube-apiserver
    - --advertise-address=10.0.0.10         # IP the API server advertises to cluster
    - --allow-privileged=true               # Allow privileged containers (needed for system Pods)
    - --authorization-mode=Node,RBAC        # Authorization methods: Node (for kubelets) + RBAC (for users)
    - --client-ca-file=/etc/kubernetes/pki/ca.crt    # CA cert for verifying client certs
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt  # CA for etcd connection
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt  # Cert for etcd auth
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key   # Key for etcd auth
    - --etcd-servers=https://127.0.0.1:2379  # Where etcd is running
    - --service-cluster-ip-range=10.96.0.0/12  # CIDR for Service ClusterIPs
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub  # For validating ServiceAccount tokens
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt  # API server's own TLS certificate
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key  # API server's TLS private key
    resources:
      requests:
        cpu: 250m                           # Reserve 0.25 CPU cores for scheduling
        memory: 512Mi                       # Reserve 512MB memory
    livenessProbe:
      httpGet:
        path: /livez                        # Health check endpoint
        port: 6443                          # HTTPS port
        scheme: HTTPS
      initialDelaySeconds: 10              # Wait 10s before first probe
      periodSeconds: 10                    # Check every 10 seconds
    volumeMounts:
    - name: ca-certs                       # Mount TLS certificates
      mountPath: /etc/ssl/certs
      readOnly: true                       # Read-only — security best practice
    - name: k8s-certs
      mountPath: /etc/kubernetes/pki
      readOnly: true
  volumes:
  - name: ca-certs
    hostPath:
      path: /etc/ssl/certs               # Use certificates from the host machine
      type: DirectoryOrCreate
  - name: k8s-certs
    hostPath:
      path: /etc/kubernetes/pki          # Kubernetes PKI directory on the host
      type: DirectoryOrCreate
```

---

## AWS/EKS Perspective
In EKS, the control plane architecture is significantly different from self-managed Kubernetes:

**Control Plane is Fully Managed:**
AWS runs the control plane in its own AWS-managed VPC. You never see the API server, etcd, scheduler, or controller manager. They run in a multi-AZ setup automatically. AWS patches, upgrades, and scales them for you.

**Communication Model:**
Your worker nodes in your VPC communicate with the API server through an ENI (Elastic Network Interface) that AWS places in your VPC. Traffic between workers and the control plane crosses VPCs using VPC peering (managed by AWS).

**etcd in EKS:**
AWS runs etcd in the control plane and backs it up automatically. You cannot access etcd directly. This is a significant difference from self-managed — you have no etcd backup responsibility, but also no access.

**Worker Node Options and Their kubelets:**
- Managed Node Groups: AWS handles kubelet configuration, OS updates, and node replacement
- Self-managed nodes: You manage all of this manually
- Fargate: There is no node, no kubelet you manage. AWS provisions a micro-VM per Pod

**EKS-Specific Control Plane Logging:**
You can enable control plane logs (API server, audit, authenticator, controller manager, scheduler) to CloudWatch Logs. This is disabled by default and has a cost, but is essential for security and debugging.

```bash
# Enable control plane logging in EKS
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Check EKS cluster status
aws eks describe-cluster --name my-cluster --query 'cluster.status'
# Output: "ACTIVE"

# Get kubeconfig for EKS cluster
aws eks update-kubeconfig --region us-east-1 --name my-cluster
```

---

## Interview Answer (2-Minute Version)
"A Kubernetes cluster has two main parts: the control plane and the worker nodes.

The control plane is the brain. It has four key components: the API server, which is the front door for everything — every kubectl command you run hits the API server. Then there's etcd, which is a distributed key-value store that holds the entire cluster state. The scheduler decides which worker node each Pod goes to. And the controller manager runs loops that constantly check if the actual cluster state matches what you asked for, and fixes any differences.

The worker nodes are where your actual applications run. Each node has a kubelet, which is an agent that communicates with the API server and starts/stops containers. There's kube-proxy, which manages networking rules so traffic reaches your Pods. And a container runtime, usually containerd, which actually runs the containers.

The flow is: you run kubectl apply, it hits the API server, the API server stores the desired state in etcd, the scheduler assigns Pods to nodes, the kubelet on each node starts the containers, and controllers continuously watch and maintain the desired state."

---

## Interview Answer (Senior Engineer Version)
"Kubernetes architecture is built around a declarative reconciliation model. The control plane components all work by watching etcd and reacting to state changes.

The API server is the only component that reads and writes to etcd. All other components talk to the API server using watches — they register interest in specific resource types, and the API server streams changes to them. This watch mechanism is central to how Kubernetes achieves eventual consistency without polling.

The scheduler uses a two-phase algorithm. In the filtering phase, it runs Predicates — functions that eliminate nodes that can't satisfy the Pod's requirements, including resource requests, node affinity, pod affinity/anti-affinity, taints and tolerations, and topology spread constraints. In the scoring phase, it runs Priority functions to rank remaining nodes. The score weights are configurable through the KubeSchedulerConfiguration API.

The kubelet implements the CRI via gRPC, which abstracts the container runtime. This is why you can swap Docker for containerd or CRI-O without changing any Kubernetes code. The kubelet also manages cgroups to enforce resource limits, runs liveness and readiness probes, and reports resource usage to the Metrics Server.

For production, I pay particular attention to control plane sizing. etcd is very latency-sensitive to disk I/O — we run etcd on SSDs with dedicated IOPs in AWS. The API server can become a bottleneck at scale — at 500+ nodes, you need to tune `--max-requests-inflight` and `--max-mutating-requests-inflight`. And controller manager has per-controller rate limits worth tuning for large clusters."

---

## What Impresses the Interviewer
- Explaining the watch mechanism in the API server — not just that components "talk to" the API server
- Knowing that etcd is Raft-based and understanding quorum (why 3 or 5 nodes, never 2 or 4)
- Describing the scheduler's filter+score phases with concrete examples of each
- Mentioning that kube-proxy programs kernel rules, NOT proxying packets itself
- Discussing what happens when the control plane goes down (workloads keep running, no new scheduling)
- Mentioning admission controllers (MutatingAdmissionWebhook, ValidatingAdmissionWebhook) and tools like OPA Gatekeeper
- Bringing up control plane HA design: odd-number etcd nodes, load balancer in front of multiple API servers
- Mentioning static Pods and how they bootstrap the control plane itself

---

## Red Flags
- Cannot name all four control plane components
- Thinks Docker is a required part of Kubernetes architecture (it was deprecated as a runtime in 1.24)
- Says "etcd stores configuration" — etcd stores ALL cluster state, not just configuration
- Cannot explain what happens when a node fails (controller manager detects, reschedules Pods)
- Never heard of admission controllers or thinks authorization and admission are the same thing
- Cannot distinguish between kubelet and kube-proxy
- Has never looked at kube-system namespace Pods in a real cluster

---

## Production Best Practices
1. **Run etcd on dedicated SSDs (NVMe preferred).** etcd is extremely sensitive to disk write latency. Slow disk causes leader elections and API server timeouts. In AWS, use io2 or gp3 volumes with high IOPS for etcd nodes.
2. **Run an odd number of etcd nodes (3 or 5) for quorum.** With 3 nodes, you can tolerate 1 failure. With 5, you tolerate 2. Never run 2 or 4 — you gain no additional fault tolerance.
3. **Separate etcd traffic from API server traffic.** Use dedicated NICs or a separate network for etcd peer communication. In large clusters, etcd replication traffic can saturate shared interfaces.
4. **Schedule workloads off control plane nodes.** Taint control plane nodes. Noisy application workloads can starve the scheduler, controller manager, or API server of CPU/memory.
5. **Enable etcd compaction and defragmentation.** etcd grows unboundedly without compaction. Configure `--auto-compaction-retention` and run periodic defragmentation to reclaim disk space.
6. **Enable API server audit logging.** Send audit logs to a SIEM. This is your primary security telemetry — who did what, when, from which IP. Required for SOC 2 and PCI-DSS.
7. **Set resource reservations on nodes.** Use `--kube-reserved` and `--system-reserved` on the kubelet to prevent application Pods from consuming all node resources and starving system components.
8. **Monitor control plane component metrics.** Expose scheduler, controller manager, and API server metrics to Prometheus. Alert on: etcd commit duration > 25ms, API server request latency > 500ms, scheduler binding rate drops.

---

## Key Points to Remember
- Control plane = API Server + etcd + Scheduler + Controller Manager + Cloud Controller Manager
- Worker node = kubelet + kube-proxy + container runtime (containerd)
- API server is the ONLY component that reads/writes etcd
- All components communicate through the API server (watch mechanism), not directly with each other
- Scheduler assigns Pods to nodes but does NOT start them — kubelet does that
- etcd uses Raft consensus — needs a quorum (majority) of nodes to function
- When control plane fails: running Pods continue, but NO new scheduling or healing occurs
- kube-proxy programs iptables/IPVS rules — it does NOT sit in the network traffic path
- CRI (Container Runtime Interface) decouples Kubernetes from specific container runtimes
- CoreDNS and CNI plugin are critical add-ons without which the cluster doesn't function

---

## Interviewer's Expectation
When asking about architecture, interviewers are testing:
1. **Depth of knowledge** — Can you go beyond surface-level component names?
2. **Mental model clarity** — Do you understand HOW components interact, not just WHAT they are?
3. **Operational maturity** — Have you dealt with real architectural failures (etcd disk full, API server slow)?
4. **Security awareness** — Do you know where the security boundaries are (etcd encryption, API server auth)?
5. **Readiness for senior work** — Can you design HA setups, capacity plan control planes, troubleshoot at the component level?

A junior candidate lists component names. A mid-level candidate explains what each does. A senior candidate explains how they interact, what breaks when each fails, and how to design for HA.

---

## Final Perfect Interview Answer
"A Kubernetes cluster is divided into the control plane and the worker nodes.

The control plane is the decision-making layer. The API server is the front door — every operation, whether from kubectl, CI/CD pipelines, or internal components, goes through it. It authenticates and authorizes requests, then persists state to etcd. etcd is a distributed key-value store — it's the single source of truth for everything in the cluster. The scheduler watches for unassigned Pods and uses a filter-then-score algorithm to pick the best node. The controller manager runs reconciliation loops — constantly comparing desired state to actual state and making corrections.

The worker nodes run the actual workloads. The kubelet is an agent on each node that watches the API server and starts or stops containers based on Pod specifications. kube-proxy manages iptables rules so traffic routes correctly to Pods. And containerd actually runs the containers.

What I find most important about this architecture in practice is the watch mechanism — components don't poll the API server, they register watches and get streamed updates. This makes the system scalable and responsive.

In production, the area I've seen cause the most incidents is etcd — it's extremely sensitive to disk I/O. We learned to run etcd on dedicated NVMe volumes with monitoring on commit latency. Anything above 25ms starts degrading API server performance noticeably."
