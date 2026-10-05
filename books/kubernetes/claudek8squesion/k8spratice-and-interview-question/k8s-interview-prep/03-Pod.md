# Pod — Kubernetes Interview Guide

## Interview Question
"What is a Kubernetes Pod? How is it different from a container? Walk me through the Pod lifecycle and explain init containers, sidecar containers, and liveness vs readiness probes."

---

## Simple Explanation
A container is like a single worker in their own office — isolated, doing their job. But sometimes, workers need to share a workspace. They share the same desk (memory), the same phone (network), and the same filing cabinet (some storage). In Kubernetes, that shared workspace is called a Pod.

A Pod is the smallest thing Kubernetes manages. You never tell Kubernetes "run this container." You always say "run this Pod," and the Pod contains one or more containers.

Most of the time, a Pod has just one container — your app. But sometimes you need a helper container alongside it. For example:
- Your app writes logs to a file. A helper container ships those logs to Elasticsearch. They need to share that file, so they live in the same Pod.
- Your app needs a secret config decrypted before it starts. An init container runs first, decrypts the config, and then your app starts. That init container is also in the same Pod.

Think of a Pod as a small apartment:
- All containers in the Pod are roommates sharing the same WiFi (network) and some shared drawers (volumes)
- They can talk to each other using localhost
- If the Pod dies, all roommates leave — the whole apartment is gone

Kubernetes doesn't move individual containers — it always moves the whole Pod together.

---

## Technical Explanation
A Pod is the smallest deployable unit in Kubernetes. It represents one or more containers that are scheduled together on the same node and share:
- **Network namespace:** Same IP address and port space. Containers within a Pod communicate via `localhost`. A port used by one container is not available to another.
- **IPC namespace (optionally):** Shared inter-process communication (message queues, semaphores)
- **Volumes:** Shared mounted directories, enabling file-sharing between containers

**Pod Networking:**
Every Pod gets a unique IP within the cluster. This IP is assigned by the CNI plugin. The Pod's network namespace is created by a special container called the **pause container** (also called infra container or sandbox container). This pause container runs before any user containers and holds the network namespace open. Even if user containers crash and restart, the network namespace (and thus the Pod's IP) remains stable.

**Pod Lifecycle Phases:**
1. **Pending:** Pod is accepted by the cluster but containers haven't started. Either waiting for scheduling (no suitable node) or pulling container images.
2. **Running:** Pod has been bound to a node, all containers have been created, and at least one container is running.
3. **Succeeded:** All containers exited with code 0. Common for Jobs.
4. **Failed:** All containers have exited, and at least one exited with non-zero status or was terminated by the system.
5. **Unknown:** Pod state cannot be determined, usually because of node communication failure.

**Container Types within a Pod:**

**Init Containers:**
Run sequentially before any app containers start. Each must complete successfully (exit 0) before the next one starts. Use cases: database migration, secret injection, waiting for a dependency to be ready, setting up file permissions. They have a separate image from app containers and do not stay running.

**Sidecar Containers:**
Run alongside the main container for the full Pod lifetime. Use cases: log shipping (Fluentd), proxy (Envoy in Istio), metrics exporter, secret syncing (Vault agent), mTLS enforcement. As of Kubernetes 1.29, there is a native `sidecar` container type that starts before app containers and stays running while init containers run — solving previous ordering challenges.

**Ephemeral Containers:**
Added to a running Pod for debugging. Cannot be removed once added. Have no resource guarantees. Introduced to allow `kubectl debug` to work on distroless or minimal images that lack debugging tools.

**Health Probes:**

**Liveness Probe:** Answers "Is this container alive?" If it fails, kubelet kills the container and restarts it (based on restartPolicy). Use for: detecting deadlocks, memory leaks that don't crash the process but make it unresponsive.

**Readiness Probe:** Answers "Is this container ready to serve traffic?" If it fails, the Pod's IP is removed from Service Endpoints. Traffic stops being sent to it. The container is NOT restarted. Use for: warming up caches, waiting for DB connections, temporary overload.

**Startup Probe:** Answers "Has this container finished starting?" Only available while the container is starting. Once it succeeds, liveness and readiness take over. Use for: slow-starting legacy apps that would fail liveness probes before they're ready.

**Probe mechanisms:** httpGet (HTTP endpoint), tcpSocket (TCP connection check), exec (run a command), grpc (gRPC health check).

**restartPolicy:**
- `Always` (default): Always restart containers when they exit (regardless of exit code). Use for web servers, background workers.
- `OnFailure`: Only restart if exit code is non-zero. Use for batch Jobs.
- `Never`: Never restart. Use for one-time tasks.

**Pod Resource Management:**
- `requests`: What the scheduler uses to find a node with enough capacity. The container is GUARANTEED this much resource.
- `limits`: The hard ceiling. For CPU: container is throttled. For memory: container is OOMKilled if it exceeds this.
- QoS Classes: Guaranteed (requests == limits), Burstable (requests < limits), BestEffort (no requests/limits). BestEffort Pods are evicted first under node pressure.

---

## Real-World Example
**Company:** Media streaming company, 80 engineers, serving 5 million daily active users.

**Use case 1 — Sidecar for log shipping:**
Their video encoding service writes detailed encoding logs to `/var/log/encoder.log`. A Fluentd sidecar container in the same Pod reads that file (via a shared volume) and ships logs to Elasticsearch. When a video encoding job fails, the team searches Kibana instead of SSH-ing into nodes. Without the sidecar, logs would be lost when the Pod is rescheduled.

**Use case 2 — Init containers for DB migration:**
Their user-service Pod has two init containers:
1. `wait-for-db`: Uses `nc -z postgres 5432` in a loop until PostgreSQL is reachable
2. `run-migrations`: Runs `alembic upgrade head` to apply database migrations

Only after both succeed does the main Flask app container start. This ensures no code runs against a stale schema. Before this pattern, they had race conditions where the app started before migrations completed.

**Use case 3 — Readiness probe for graceful traffic management:**
Their search API takes 15 seconds to load its ML model into memory. Without a readiness probe, the load balancer sent traffic to it immediately on start, causing 503 errors. They added a readiness probe hitting `/health/ready` which returns 200 only after the model is loaded. Now, traffic is only sent once the container is truly ready.

---

## Diagram / Flow

```
                         POD STRUCTURE
+------------------------------------------------------------------+
|                         POD                                      |
|  IP: 10.0.0.45                                                   |
|                                                                  |
|  +------------------+  Shared Network Namespace                  |
|  |  pause container |  (holds the Pod's IP address)              |
|  |  (infra/sandbox) |  All containers share this network         |
|  +------------------+                                            |
|                                                                  |
|  +-----------------------+    +------------------------+         |
|  |  Init Container 1     |    |  Init Container 2      |         |
|  |  (wait-for-db)        |--->|  (run-migrations)      |         |
|  |  Runs first, exits 0  |    |  Runs second, exits 0  |         |
|  +-----------------------+    +------------------------+         |
|                  Both must succeed before app starts             |
|                                |                                 |
|                                v                                 |
|  +------------------------+    +------------------------+        |
|  |  MAIN CONTAINER        |    |  SIDECAR CONTAINER     |        |
|  |  (my-app)              |    |  (fluentd)             |        |
|  |                        |    |                        |        |
|  |  - Liveness probe      |    |  - Ships logs to ELK   |        |
|  |  - Readiness probe     |    |  - Runs for Pod life   |        |
|  |  - Resource limits     |    |                        |        |
|  +------------------------+    +------------------------+        |
|             |                              |                     |
|             +------------------------------+                     |
|                          |                                       |
|              SHARED VOLUME: /var/log                             |
|  (Both containers can read/write to this directory)              |
|                                                                  |
|  localhost:8080 = main app    localhost:24224 = fluentd          |
+------------------------------------------------------------------+

POD LIFECYCLE FLOW:
============================================================
[Pending] --> [Init Containers run] --> [Running] --> [Succeeded/Failed]

                    PROBE DECISION TREE
                    ====================
         Container Running?
              |
              v
    Startup Probe (if configured)
         PASS --> switch to Liveness + Readiness
         FAIL --> restart container

    Liveness Probe (ongoing)
         PASS --> container stays running
         FAIL --> container KILLED and RESTARTED

    Readiness Probe (ongoing)
         PASS --> Pod IP added to Service Endpoints (traffic flows in)
         FAIL --> Pod IP REMOVED from Endpoints (no traffic, not restarted)

RESTART POLICY:
================
restartPolicy: Always   --> restart on any exit (0 or non-0)
restartPolicy: OnFailure --> restart only on non-zero exit
restartPolicy: Never    --> never restart
```

---

## Why It Is Important
- Pod is the FUNDAMENTAL unit of Kubernetes — everything else (Deployment, StatefulSet, Job) ultimately manages Pods
- Understanding Pods is required to understand why apps crash, why traffic fails, and how to debug
- Probes directly affect user experience — misconfigured probes cause outages or traffic to unhealthy instances
- Resource requests/limits affect cost, stability, and performance — incorrect settings cause OOMKills and scheduling failures
- Init containers are the standard pattern for safe application startup ordering
- Sidecar containers are the foundation of service mesh (Istio), log shipping, and secrets management

---

## Common Interview Follow-Up Questions
1. "What happens when a liveness probe fails repeatedly?"
2. "What is the difference between liveness and readiness probes?"
3. "Why would you use an init container instead of just putting that logic in your main container's startup script?"
4. "What is CrashLoopBackOff and how do you fix it?"
5. "How do containers within the same Pod communicate?"
6. "What is the pause container and why does it exist?"
7. "What is the difference between CPU requests, CPU limits, and how does the kernel enforce them?"
8. "What QoS class gets evicted first when a node is under memory pressure?"

---

## Common Mistakes Candidates Make

**Mistake 1: "Containers in a Pod communicate over the network"**
They communicate via localhost — they share the same network namespace. Saying "over the network" implies a separate IP, which is wrong. This is the whole point of a Pod: containers that are tightly coupled share the same localhost.

**Mistake 2: "Liveness probe failure causes traffic to stop"**
No. Liveness probe failure causes the container to RESTART. Readiness probe failure causes traffic to stop (Pod removed from Endpoints). This distinction is critical for application reliability. Confusing them leads to cascading failures in production.

**Mistake 3: Not knowing about CrashLoopBackOff backoff timing**
CrashLoopBackOff means a container keeps crashing and Kubernetes keeps restarting it with exponential backoff (10s, 20s, 40s, 80s, up to 5 minutes). Many candidates say "the Pod is broken, restart it." The right answer is to investigate root cause with `kubectl logs --previous` and `kubectl describe pod`.

**Mistake 4: Thinking init containers run in parallel**
Init containers run SEQUENTIALLY. Each must exit successfully before the next starts. If you need parallel initialization, use a single init container that spawns parallel processes internally.

**Mistake 5: Confusing resource requests and limits**
Requests are for SCHEDULING — the scheduler won't place a Pod on a node with less available capacity. Limits are ENFORCED at runtime — CPU is throttled, memory causes OOMKill. A Pod with high limits and low requests can be scheduled on a node but then consume far more than what was accounted for during scheduling (this causes node overcommit problems).

---

## Troubleshooting Scenario
**Problem:** Application in production keeps going down. Users report errors every few minutes. The Pod shows high RESTARTS count.

```bash
# Step 1: Check Pod status and restart count
kubectl get pods -n production
# Output:
# NAME                    READY   STATUS             RESTARTS   AGE
# myapp-7d9f8b-xk2p9     0/1     CrashLoopBackOff   14         1h

# Step 2: Describe the Pod — look at Events and Last State
kubectl describe pod myapp-7d9f8b-xk2p9 -n production
# Look for:
# - Last State: exit code (137 = OOMKill, 1 = app crash, 143 = SIGTERM)
# - Events: OOMKilling, BackOff, Liveness probe failed
# - Container resource limits vs usage

# Step 3: Get logs from the PREVIOUS container (before latest restart)
kubectl logs myapp-7d9f8b-xk2p9 -n production --previous
# This shows what the container printed before it died
# Look for: panic, exception, connection refused, out of memory

# Step 4: Get current logs (if it's still running briefly)
kubectl logs myapp-7d9f8b-xk2p9 -n production -f
# -f follows logs in real time

# Step 5: If exit code was 137 (OOMKilled)
# The container hit its memory limit
kubectl describe pod myapp-7d9f8b-xk2p9 -n production | grep -A 5 "Limits"
# Increase memory limit or fix memory leak in application

# Step 6: If liveness probe is failing (not OOM)
kubectl describe pod myapp-7d9f8b-xk2p9 -n production | grep -A 10 "Liveness"
# Check: what endpoint, what timeout, what failure threshold
# Maybe the app just needs more time to start — add startupProbe

# Step 7: Exec into a running container to debug directly
kubectl exec -it myapp-7d9f8b-xk2p9 -n production -- /bin/sh
# Inside: check processes, check disk space, check connectivity
# df -h (disk), free -m (memory), ps aux (processes)

# Step 8: Check if it's a startup ordering issue
kubectl logs myapp-7d9f8b-xk2p9 -n production -c init-wait-for-db
# Check init container logs for connection failures

# Step 9: Check events in namespace for broader context
kubectl get events -n production --field-selector involvedObject.name=myapp-7d9f8b-xk2p9
# Shows: image pull errors, OOMKill events, scheduling events
```

---

## kubectl Commands

```bash
# List all Pods in current namespace
kubectl get pods
# Output:
# NAME                    READY   STATUS    RESTARTS   AGE
# myapp-7d9f8b-xk2p9     1/1     Running   0          2d

# List Pods in all namespaces
kubectl get pods --all-namespaces
# or: kubectl get pods -A

# List Pods with node info (which node each Pod is on)
kubectl get pods -o wide
# Output adds: NODE, NOMINATED NODE, READINESS GATES

# Describe a Pod (most useful debugging command)
kubectl describe pod myapp-7d9f8b-xk2p9
# Shows: events, conditions, volumes, container details, probe config

# Get Pod logs
kubectl logs myapp-7d9f8b-xk2p9
kubectl logs myapp-7d9f8b-xk2p9 --previous          # Logs from crashed container
kubectl logs myapp-7d9f8b-xk2p9 -f                  # Follow live logs
kubectl logs myapp-7d9f8b-xk2p9 --tail=100          # Last 100 lines
kubectl logs myapp-7d9f8b-xk2p9 -c sidecar-name     # Logs from specific container in Pod

# Execute command inside a running container
kubectl exec -it myapp-7d9f8b-xk2p9 -- /bin/bash
kubectl exec myapp-7d9f8b-xk2p9 -- env              # Run single command, show env vars

# Execute in specific container (multi-container Pod)
kubectl exec -it myapp-7d9f8b-xk2p9 -c fluentd-sidecar -- /bin/sh

# Create a Pod directly (rarely done in production, good for testing)
kubectl run test-nginx --image=nginx:alpine --port=80
# Output: pod/test-nginx created

# Create a temporary debug Pod
kubectl run debug-pod --image=busybox:latest --rm -it --restart=Never -- /bin/sh
# --rm deletes it when you exit, --restart=Never prevents restart

# Delete a Pod (it will be recreated if managed by a Deployment)
kubectl delete pod myapp-7d9f8b-xk2p9
# Output: pod "myapp-7d9f8b-xk2p9" deleted

# Port-forward to a Pod (access without exposing a Service)
kubectl port-forward pod/myapp-7d9f8b-xk2p9 8080:80
# Access at localhost:8080 → forwards to container port 80

# Copy files to/from a Pod
kubectl cp myapp-7d9f8b-xk2p9:/app/logs/error.log ./error.log
kubectl cp ./config.json myapp-7d9f8b-xk2p9:/app/config.json

# Add an ephemeral debug container to a running Pod
kubectl debug -it myapp-7d9f8b-xk2p9 --image=busybox --target=myapp

# Watch Pod status changes in real time
kubectl get pods -w
# Shows real-time status updates as Pods change states

# Get Pod in JSON/YAML format
kubectl get pod myapp-7d9f8b-xk2p9 -o yaml
kubectl get pod myapp-7d9f8b-xk2p9 -o json

# Get all containers in a Pod
kubectl get pod myapp-7d9f8b-xk2p9 -o jsonpath='{.spec.containers[*].name}'
# Output: myapp fluentd-sidecar
```

---

## YAML Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app                    # Unique name for this Pod within the namespace
  namespace: production                   # The namespace this Pod belongs to
  labels:
    app: payment-service                  # Label used by Services to select this Pod
    version: "2.1.0"                      # Version label for canary deployments
    environment: production               # Environment label for filtering
  annotations:
    prometheus.io/scrape: "true"          # Tells Prometheus to scrape metrics from this Pod
    prometheus.io/port: "9090"            # Port Prometheus should scrape
    kubectl.kubernetes.io/last-applied-configuration: "" # Added by kubectl apply

spec:
  restartPolicy: Always                   # Always restart containers if they exit (default for Deployments)

  serviceAccountName: payment-sa          # The service account this Pod runs as (controls RBAC permissions)

  securityContext:                        # Pod-level security settings (applies to all containers)
    runAsNonRoot: true                    # Prevent running as root - security best practice
    runAsUser: 1000                       # Run all containers as UID 1000
    runAsGroup: 3000                      # Run all containers as GID 3000
    fsGroup: 2000                         # File system group - shared volume files owned by this GID

  # Init containers run SEQUENTIALLY before app containers start
  # Each must exit 0 before the next begins
  initContainers:
  - name: wait-for-database               # Step 1: Wait until PostgreSQL is reachable
    image: busybox:1.36                   # Small image with networking tools
    command:
    - /bin/sh
    - -c
    - |
      until nc -z postgres-service 5432; do
        echo "Waiting for database..."
        sleep 2
      done
      echo "Database is ready!"
    resources:
      requests:
        cpu: 50m                          # Very small resource request for init container
        memory: 32Mi

  - name: run-migrations                  # Step 2: Apply database schema migrations
    image: mycompany/payment-service:2.1.0  # Same image as main app (has migration tools)
    command: ["python", "manage.py", "migrate"]  # Django migration command
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:                     # Pull DB URL from a Kubernetes Secret
          name: database-credentials
          key: url
    resources:
      requests:
        cpu: 100m
        memory: 128Mi

  # Main application containers (run in parallel after all init containers succeed)
  containers:
  - name: payment-service                 # Main application container
    image: mycompany/payment-service:2.1.0  # Container image (use a specific tag, never 'latest' in prod)
    imagePullPolicy: IfNotPresent         # Pull image only if not already on the node

    ports:
    - name: http                          # Named port (can reference by name in Services)
      containerPort: 8080                 # Port the app listens on INSIDE the container
      protocol: TCP
    - name: metrics
      containerPort: 9090                 # Prometheus metrics port
      protocol: TCP

    # Environment variables
    env:
    - name: ENVIRONMENT                   # Static value
      value: "production"
    - name: POD_NAME                      # Downward API: inject Pod's own name into env var
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: NODE_NAME                     # Downward API: inject node name
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: DATABASE_PASSWORD             # From Secret (never hardcode passwords!)
      valueFrom:
        secretKeyRef:
          name: database-credentials      # Secret name
          key: password                   # Key within the Secret
    - name: FEATURE_FLAGS                 # From ConfigMap
      valueFrom:
        configMapKeyRef:
          name: payment-config            # ConfigMap name
          key: feature-flags              # Key within the ConfigMap

    # Resource requests and limits — ALWAYS SET BOTH in production
    resources:
      requests:
        cpu: 250m                         # 0.25 CPU cores — used by scheduler to find a node
        memory: 512Mi                     # 512MB memory — guaranteed to this container
      limits:
        cpu: 1000m                        # 1 CPU core max — container throttled if exceeded
        memory: 1Gi                       # 1GB max — container OOMKilled if exceeded

    # Startup probe — gives slow-starting apps time to initialize
    # Active ONLY during startup. Once it passes, liveness/readiness take over.
    startupProbe:
      httpGet:
        path: /health/startup             # HTTP endpoint that returns 200 when app is ready to start
        port: 8080
      failureThreshold: 30               # Allow up to 30 failures before killing (30 * 10s = 5 min startup time)
      periodSeconds: 10                  # Check every 10 seconds

    # Liveness probe — detects if the container is stuck/deadlocked
    # Failure = container RESTARTED
    livenessProbe:
      httpGet:
        path: /health/live               # Endpoint that returns 200 if app is alive (no deadlocks)
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Kubernetes
      initialDelaySeconds: 0            # No delay needed (startup probe already handled slow start)
      periodSeconds: 15                 # Check every 15 seconds
      timeoutSeconds: 5                 # Fail if no response within 5 seconds
      failureThreshold: 3              # Restart after 3 consecutive failures (45 seconds total)
      successThreshold: 1              # One success is enough to be considered alive

    # Readiness probe — determines if the container should receive traffic
    # Failure = Pod REMOVED from Service Endpoints (no traffic). Container NOT restarted.
    readinessProbe:
      httpGet:
        path: /health/ready              # Endpoint that returns 200 only when fully ready for traffic
        port: 8080
      initialDelaySeconds: 5            # Wait 5s before first check
      periodSeconds: 10                 # Check every 10 seconds
      timeoutSeconds: 3                 # Fail if no response in 3 seconds
      failureThreshold: 3              # Remove from Endpoints after 3 failures
      successThreshold: 1              # Add back to Endpoints after 1 success

    # Volume mounts — where volumes appear inside the container
    volumeMounts:
    - name: app-logs                    # Mount the shared log volume
      mountPath: /app/logs              # Path inside the container
    - name: config-volume               # Mount ConfigMap as files
      mountPath: /app/config
      readOnly: true                    # Config is read-only - security best practice
    - name: secrets-volume              # Mount Secret as files
      mountPath: /app/secrets
      readOnly: true

    # Security context for this specific container
    securityContext:
      allowPrivilegeEscalation: false   # Cannot gain more privileges than parent process
      readOnlyRootFilesystem: true      # Root filesystem is read-only - security best practice
      capabilities:
        drop:
        - ALL                           # Drop all Linux capabilities (least privilege)

  # Sidecar container — runs alongside main app for full Pod lifetime
  - name: log-shipper                  # Fluentd log shipping sidecar
    image: fluent/fluentd:v1.16-1
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 256Mi
    volumeMounts:
    - name: app-logs                    # Reads from the SAME volume the main container writes to
      mountPath: /var/log/app           # Different path, same underlying volume
      readOnly: true                    # Only reading logs, not writing

  # Define volumes available to all containers in this Pod
  volumes:
  - name: app-logs                      # Shared empty directory for log files
    emptyDir: {}                        # Exists only while Pod is running; deleted when Pod is deleted
  - name: config-volume                 # ConfigMap mounted as a directory of files
    configMap:
      name: payment-config              # Name of the ConfigMap resource
      defaultMode: 0444                 # Files are readable (octal permissions)
  - name: secrets-volume               # Secret mounted as files (more secure than env vars for certs)
    secret:
      secretName: payment-tls-cert      # Name of the Secret resource
      defaultMode: 0400                 # Files readable only by owner (maximum security)

  # Scheduling controls
  nodeSelector:
    node-type: compute-optimized        # Only schedule on nodes with this label

  affinity:
    podAntiAffinity:                    # Spread Pods across nodes for HA
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - payment-service
          topologyKey: kubernetes.io/hostname  # One Pod per node preferred

  tolerations:                          # Allow this Pod to run on tainted nodes
  - key: "dedicated"
    operator: "Equal"
    value: "payment"
    effect: "NoSchedule"               # Tolerate this taint

  terminationGracePeriodSeconds: 30    # Give container 30s to gracefully shut down before SIGKILL
```

---

## AWS/EKS Perspective

**Pod Networking in EKS (VPC CNI):**
Unlike most CNI plugins that use overlay networks, the AWS VPC CNI assigns real VPC IP addresses to Pods. Each Pod gets an IP from your VPC subnet. This has major implications:
- Pods are directly routable from other AWS services (RDS, Lambda, etc.)
- You must plan subnet sizes carefully — a /24 subnet gives only 251 IPs for Pods
- Each EC2 node has a limit on how many IPs it can hold (based on instance type and ENIs)
- Large instance types (m5.4xlarge) can support more Pods than small ones

**Fargate Pods:**
On EKS Fargate, each Pod runs in its own micro-VM. You cannot run multiple containers on the "same node" in the traditional sense. Sidecar containers still work — they're in the same micro-VM. Init containers work. But DaemonSets don't work on Fargate.

**Pod Identity (IRSA):**
Instead of storing AWS credentials in the container, EKS supports IAM Roles for Service Accounts (IRSA). The Pod's service account is annotated with an IAM role ARN. When the container uses the AWS SDK, it automatically gets temporary credentials for that IAM role. No secrets to rotate, no credentials in YAML.

```yaml
# ServiceAccount with IRSA annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader-role
    # This Pod will get temporary AWS credentials for the s3-reader-role
```

**Pod Disruption in EKS during Node Upgrades:**
AWS managed node group upgrades drain nodes using `kubectl drain`. Without PodDisruptionBudgets, all replicas could be terminated simultaneously. Always set PDBs:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-pdb
spec:
  minAvailable: 1              # At least 1 replica must stay running during disruption
  selector:
    matchLabels:
      app: payment-service
```

---

## Interview Answer (2-Minute Version)
"A Pod is the smallest unit in Kubernetes — it's not a single container, it's a wrapper around one or more containers that share the same network and can share storage volumes.

The reason for this is that some applications have tightly coupled helper processes. For example, a main app container and a log-shipping sidecar need to share a log file. By putting them in the same Pod, they share a filesystem and can communicate over localhost.

The Pod lifecycle goes: Pending while waiting for a node or image pull, then Running, then Succeeded or Failed. Each container can have three kinds of health probes: liveness detects if the app is stuck and restarts it; readiness controls whether traffic is sent to it; and startup probe handles slow-starting apps.

Init containers are a separate category — they run sequentially before the main containers start. I've used them to run database migrations and wait for dependencies to be healthy before the app boots.

In terms of resources, every container should have requests — so the scheduler can place it correctly — and limits — so it doesn't consume an entire node's memory and get OOMKilled."

---

## Interview Answer (Senior Engineer Version)
"A Pod is the atomic unit of scheduling and execution in Kubernetes. It represents a group of containers that need to be co-located and co-scheduled — they share a Linux network namespace (created by the pause container) and optionally share IPC namespaces and volumes.

The pause container — also called the infra or sandbox container — is crucial and often overlooked. It's a tiny container whose only job is to hold open the Pod's network namespace. Even when application containers crash and restart, the network namespace persists, so the Pod's IP remains stable and established TCP connections can potentially recover.

On probes: I treat liveness and readiness probes as having completely different failure semantics that require separate endpoints in the application. The liveness endpoint should check only for deadlocks and unrecoverable states — it should NOT check database connectivity, because a database outage shouldn't cause your Pods to restart in a loop, creating a thundering herd when the DB comes back. The readiness endpoint should check all dependencies, because if the DB is down, the Pod genuinely shouldn't receive traffic.

For resource management, the QoS class matters operationally. BestEffort Pods (no requests/limits) are first evicted under memory pressure. Burstable Pods (requests less than limits) are next. Guaranteed Pods (requests equal limits) are last. In production, all critical workloads should be Guaranteed class for predictable behavior.

On init containers — beyond dependency ordering, they're also useful for security patterns: an init container can fetch a secret from Vault and write it to an emptyDir volume, then the main container reads it from disk. The secret never appears in Kubernetes Secrets or environment variables."

---

## What Impresses the Interviewer
- Explaining the pause container and why the Pod's IP stays stable when containers restart
- Knowing the difference between liveness and readiness failure semantics (restart vs traffic removal)
- Warning that liveness probes should NOT check external dependencies (creates cascading restarts)
- Understanding QoS classes and how they affect eviction order
- Mentioning PodDisruptionBudgets in context of Pod termination
- Discussing the preStop lifecycle hook for graceful shutdown
- Knowing that `terminationGracePeriodSeconds` is the window between SIGTERM and SIGKILL
- Experience debugging with `kubectl logs --previous` and ephemeral containers

---

## Red Flags
- Says "Pod is just a container" — shows no understanding of the multi-container model or shared namespace
- Cannot explain the difference between liveness and readiness probes
- Has never seen CrashLoopBackOff in real life — cannot explain the backoff mechanism
- Doesn't know what `--previous` flag on `kubectl logs` does (critical for debugging crashes)
- Cannot explain resource requests vs limits, or thinks they're the same thing
- Says "I always use latest tag for images" — this breaks reproducibility and causes unexpected upgrades

---

## Production Best Practices
1. **Never use the `latest` image tag in production.** Use a specific version tag (SHA digest is best). `latest` means your Pod could pull a broken image on restart without any change on your part.
2. **Always set both resource requests AND limits.** Without requests, the scheduler places Pods blindly. Without limits, a Pod can consume all node memory and trigger OOMKill storms across the cluster.
3. **Separate liveness and readiness endpoints in your application code.** Liveness should only check internal health. Readiness should check all dependencies. This prevents cascading restarts during external service outages.
4. **Use a startup probe for any app that takes more than 20 seconds to start.** Without it, the liveness probe fails during startup and Kubernetes kills the container before it's ready — causing an infinite restart loop.
5. **Set `terminationGracePeriodSeconds` based on your actual shutdown time.** Default is 30 seconds. If your app needs 60 seconds to drain connections and finish in-flight requests, set it to 90 (with buffer). SIGKILL at 30s causes in-flight request failures.
6. **Use the preStop lifecycle hook to delay SIGTERM.** In Kubernetes, the Pod's IP is removed from Endpoints asynchronously while SIGTERM is sent. Traffic may still be routed to the Pod for a few seconds after it starts shutting down. A `preStop` sleep of 5-10 seconds prevents dropped connections.
7. **Set runAsNonRoot: true and readOnlyRootFilesystem: true.** Running as root in a container is a security risk. A read-only filesystem prevents malware from writing new executables.
8. **Use PodAntiAffinity for production services.** Ensure replicas spread across nodes and availability zones. Without this, multiple replicas can land on the same node, and a node failure takes down all of them.

---

## Key Points to Remember
- Pod = smallest deployable unit; wraps one or more containers sharing network + optionally storage
- The pause/infra container holds the Pod's network namespace stable across container restarts
- Init containers run SEQUENTIALLY, must exit 0 before app containers start
- Sidecar containers run in PARALLEL with the main container for its full lifetime
- Liveness probe failure = container RESTARTED; Readiness probe failure = traffic REMOVED (no restart)
- Startup probe = initial liveness check for slow-starting apps; switches to liveness/readiness after success
- restartPolicy: Always (default), OnFailure, Never
- requests = scheduling guarantee; limits = hard enforcement (CPU throttle, memory OOMKill)
- QoS classes: Guaranteed > Burstable > BestEffort (eviction order is reversed: BestEffort first)
- CrashLoopBackOff = container keeps crashing, K8s retries with exponential backoff (10s → 5min)

---

## Interviewer's Expectation
When asking about Pods, interviewers are testing:
1. **Foundation understanding** — Pods are the base. If you don't understand Pods, you can't understand Deployments, Services, or anything else.
2. **Operational experience** — Have you debugged a crashing Pod? Have you tuned probes? Have you dealt with OOMKills?
3. **Application design thinking** — Do you know how to design apps that work well in Pods (graceful shutdown, health endpoints)?
4. **Security awareness** — Do you know about running as non-root, read-only filesystems, security contexts?
5. **Resource management maturity** — Do you set resource requests and limits? Do you know why they matter?

Beginner: "A Pod runs containers." Intermediate: "A Pod shares network between containers, probes determine health." Senior: "I design separate liveness/readiness endpoints, I use PodDisruptionBudgets, I've debugged OOMKill patterns and tuned terminationGracePeriodSeconds for zero-downtime deploys."

---

## Final Perfect Interview Answer
"A Pod is Kubernetes's smallest deployable unit — it wraps one or more containers that share the same network namespace. This means all containers in a Pod communicate over localhost, and they can share storage volumes.

The reason a Pod can hold multiple containers is for the sidecar pattern — you might have your main app container alongside a log-shipping container or a metrics exporter. They're tightly coupled and need to share files or local network communication.

Pods go through lifecycle phases: Pending while being scheduled or pulling images, Running when containers are active, and then Succeeded or Failed.

Health probes are critical for production reliability. Liveness probes detect if an app is stuck or deadlocked — failure causes a restart. Readiness probes detect if an app is ready for traffic — failure removes the Pod from the load balancer without restarting it. This distinction matters a lot: I never put database connectivity checks in liveness probes, only in readiness, because if a database is temporarily down, you don't want your entire fleet of Pods restarting simultaneously.

Init containers handle startup ordering — I've used them to run database migrations before the app starts, and to wait for dependent services to be healthy.

For resources, I always set both requests and limits. Requests are what the scheduler uses to place the Pod; limits enforce runtime caps to prevent OOMKills from cascading."
