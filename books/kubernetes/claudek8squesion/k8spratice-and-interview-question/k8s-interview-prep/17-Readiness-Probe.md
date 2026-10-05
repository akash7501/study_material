# Readiness Probe — Kubernetes Interview Guide

## Interview Question
"What is a Readiness Probe in Kubernetes? How does it differ from a Liveness Probe, what happens when it fails, and when would you use it in production?"

---

## Simple Explanation
Imagine a new chef just joined a restaurant kitchen. They are physically present (not sick — not a liveness issue), but they haven't finished their mise en place yet. They haven't prepped their ingredients, their station isn't set up, and they aren't ready to take orders. You wouldn't seat customers at their table yet.

The head chef (Kubernetes) periodically checks: "Are you ready to take orders?" If the chef says "not yet," no new orders are sent their way. Once they say "ready," orders start coming in. If they suddenly say "not ready" mid-shift (maybe they ran out of a key ingredient), orders stop coming to them until they're ready again.

That is a Readiness Probe.

- It answers: **"Is this container ready to receive traffic?"**
- Failure does NOT restart the container — it just removes it from the load balancer
- The pod stays Running, the container keeps running — it just gets no traffic
- When it recovers, traffic resumes automatically

---

## Technical Explanation
A Readiness Probe determines whether a container is ready to serve requests. When a readiness probe fails, the Kubernetes endpoint controller removes the pod's IP from the Endpoints object for all matching Services. This means the kube-proxy and any service mesh stop routing traffic to that pod.

### Key Difference from Liveness Probe
| Aspect | Liveness Probe | Readiness Probe |
|--------|---------------|-----------------|
| Question | "Is this container alive?" | "Is this container ready?" |
| Failure Action | Restart container | Remove from Service Endpoints |
| Container State | Container is killed | Container keeps running |
| Pod Status | Pod restarts | Pod stays Running, READY shows 0/1 |
| Use Case | Deadlock recovery | Traffic management |
| `successThreshold` | Must be 1 | Can be > 1 |

### How It Works Internally
1. After `initialDelaySeconds`, kubelet starts running the readiness probe.
2. The probe runs every `periodSeconds`.
3. If the probe fails `failureThreshold` consecutive times, the pod's `Ready` condition becomes `False`.
4. The EndpointSlice controller (in modern Kubernetes) removes this pod's IP from all Service Endpoints.
5. kube-proxy updates iptables/IPVS rules — no more traffic to this pod.
6. When the probe succeeds `successThreshold` consecutive times, the pod is re-added to endpoints.
7. The container was never restarted during any of this.

### Three Probe Types (same as Liveness)

**1. httpGet Probe**
- Kubelet sends HTTP GET to specified path/port.
- 200-399 response = ready.
- Best for: web services, REST APIs.
- Use a deeper check here than in liveness — check database connectivity.

**2. exec Probe**
- Executes command inside container.
- Exit code 0 = ready.
- Best for: custom readiness logic, file existence checks.

**3. tcpSocket Probe**
- Opens TCP connection to specified port.
- Connection success = ready.
- Best for: databases, message brokers.

### Key Configuration Parameters
| Parameter | Description | Default |
|-----------|-------------|---------|
| `initialDelaySeconds` | Wait before first probe | 0 |
| `periodSeconds` | Probe frequency | 10 |
| `timeoutSeconds` | Probe timeout | 1 |
| `failureThreshold` | Failures before not-ready | 3 |
| `successThreshold` | Successes before ready (can be > 1) | 1 |

### Important: `successThreshold` for Readiness
Unlike liveness probes, readiness probes CAN have `successThreshold > 1`. This means you can require, for example, 3 consecutive successful health checks before a pod starts receiving traffic after recovery. This is useful for ensuring a pod is truly stable before re-introducing it to production traffic.

---

## Real-World Example
**Production Scenario: E-commerce Checkout Service with Database Dependency**

An e-commerce platform's checkout service must connect to both PostgreSQL and Redis before it can process payments. During rolling deployments, new pods start up, the JVM initializes, but the database connection pool takes an additional 15 seconds to fully establish.

Without readiness probes: Kubernetes adds the pod to the Service load balancer immediately after the container starts. Requests arrive before the connection pool is ready, resulting in payment failures during the deployment.

With readiness probes configured on `/health/ready` (which checks DB connectivity and Redis connection):
- New pod starts
- Pod is NOT added to Service endpoints yet (probe not passing)
- App initializes DB connections
- Probe starts passing
- ONLY THEN does Kubernetes add the pod to Service endpoints
- Zero payment failures during deployment

Additionally, if the database becomes unavailable at 2 AM, all pod readiness probes fail. All pods are removed from the Service. The Service returns "no endpoints available" rather than sending traffic to pods that would fail anyway. When the database recovers, pods automatically rejoin the load balancer.

---

## Diagram / Flow

```
+-------------------------------------------------------------------+
|                    READINESS PROBE FLOW                           |
+-------------------------------------------------------------------+

  POD STARTUP SEQUENCE:
  ========================

  t=0s   Container starts
  t=0s   Readiness = FALSE (pod not added to Service endpoints yet)
         Liveness probe starts after initialDelaySeconds
  t=10s  Readiness probe fires → App still initializing → FAIL
  t=20s  Readiness probe fires → DB pool not ready → FAIL
  t=30s  Readiness probe fires → All checks pass → SUCCESS (count=1)
         [If successThreshold=1]: Pod added to Service Endpoints
  t=30s  Traffic starts flowing to this pod ✓

  READINESS FAILURE DURING OPERATION:
  =====================================

  t=1h   Pod serving traffic normally
  t=1h5m Database goes down
  t=1h5m Readiness probe → DB check fails → FAIL (count=1)
  t=1h5m10s Readiness probe → DB check fails → FAIL (count=2)
  t=1h5m20s Readiness probe → DB check fails → FAIL (count=3 = failureThreshold)
         Pod REMOVED from Service Endpoints
         Traffic stops flowing to this pod
         Container is still RUNNING (not restarted!)
  t=1h10m Database recovers
  t=1h10m Readiness probe → DB check passes → SUCCESS (count=1)
         Pod RE-ADDED to Service Endpoints
         Traffic resumes to this pod ✓

KUBERNETES SERVICE ROUTING:
============================

  ┌─────────────────────────────────────────────────────┐
  │                   SERVICE                            │
  │              (ClusterIP/LoadBalancer)                │
  │                                                      │
  │    Endpoint: [pod1:8080, pod2:8080, pod3:8080]       │
  └──────────────────────┬──────────────────────────────┘
                         │ routes traffic to
                         ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Pod 1   │    │  Pod 2   │    │  Pod 3   │
  │ READY ✓  │    │ READY ✓  │    │ READY ✗  │
  │ In LB    │    │ In LB    │    │ NOT in LB│
  └──────────┘    └──────────┘    └──────────┘
                                       │
                                  DB connection
                                  lost — readiness
                                  probe failing

  After Pod 3 readiness fails:
  Endpoints: [pod1:8080, pod2:8080]  ← pod3 removed
  Traffic only goes to pod1 and pod2

LIVENESS vs READINESS COMPARISON:
====================================

           LIVENESS                    READINESS
           --------                    ---------
  Probe    Is the app alive?           Is the app ready?
  fails    ┌──────────────────┐        ┌──────────────────┐
           │ kubelet kills    │        │ pod removed from │
           │ container        │        │ Service endpoints│
           │ container        │        │ container still  │
           │ restarts         │        │ running          │
           └──────────────────┘        └──────────────────┘
  Pod      Restarts (RESTARTS++)       Stays Running
  status   0/1 CrashLoopBackOff        0/1 Running (not ready)
  Recovery Auto-restart                Auto-rejoin when healthy

3 PROBE TYPES:
===============

  httpGet                 exec                   tcpSocket
  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
  │ GET /health/ready│   │ run script or    │   │ TCP connect to   │
  │ port: 8080       │   │ command inside   │   │ port 5432        │
  │ 200-399 = ready  │   │ exit 0 = ready   │   │ connected = ready│
  │                  │   │                  │   │                  │
  │ Best for:        │   │ Best for:        │   │ Best for:        │
  │ REST APIs        │   │ DB health checks │   │ Databases,       │
  │ Web services     │   │ File existence   │   │ Message brokers  │
  └──────────────────┘   └──────────────────┘   └──────────────────┘
```

---

## Why It Is Important

**Business Value:**
- Enables **zero-downtime deployments** — new pods only get traffic when truly ready
- Prevents **cascading failures** — unhealthy pods don't receive traffic and make things worse
- Enables **graceful degradation** — overloaded pods can temporarily remove themselves from rotation
- Supports **circuit breaker patterns** at the infrastructure level

**Technical Value:**
- Critical for **rolling updates** — without readiness probes, Kubernetes can't know when it's safe to take down old pods
- Enables **traffic shaping** — pods can signal their capacity
- Supports **dependency management** — pods wait for dependencies before accepting traffic
- Works with **PodDisruptionBudgets** to maintain availability during node drains

---

## Common Interview Follow-Up Questions

1. **"What happens to a pod when its readiness probe fails?"**
   - The pod's Ready condition becomes False, it is removed from the Endpoints object for all matching Services, and no traffic is routed to it. The container is NOT restarted.

2. **"How does Readiness Probe enable zero-downtime deployments?"**
   - During rolling updates, Kubernetes waits for each new pod's readiness probe to pass before terminating an old pod. The `minReadySeconds` field adds an extra stability window.

3. **"Can `successThreshold` be > 1 for a readiness probe?"**
   - Yes, unlike liveness probes. Setting `successThreshold: 3` means 3 consecutive successes are needed before a pod re-enters the load balancer. Useful for ensuring stability after recovery.

4. **"How do readiness probes interact with rolling deployments?"**
   - `maxUnavailable` and `maxSurge` in the deployment strategy work in conjunction with readiness probes. Kubernetes uses readiness state to determine how many pods are "available" when calculating rolling update progress.

5. **"What is `minReadySeconds` in a Deployment?"**
   - It specifies how many seconds after a new pod becomes Ready should Kubernetes wait before considering it "available" for the rolling update. Adds a stability buffer beyond just the readiness probe passing once.

6. **"Should readiness probes check external dependencies?"**
   - Yes, unlike liveness probes. Readiness probes should verify the pod can actually serve requests, which includes checking database connections, cache connectivity, etc.

7. **"What is the difference between 0/1 Running (not ready) and 0/1 CrashLoopBackOff?"**
   - 0/1 Running (not ready): readiness probe failing, container is running, no restart.
   - 0/1 CrashLoopBackOff: liveness probe failing or container crashing, container is being restarted repeatedly.

8. **"How do you implement a readiness probe for an app that depends on an external service being healthy?"**
   - Check the connection to the external service in the readiness endpoint. If the external service is down, return 503 from `/health/ready`. This removes the pod from the load balancer.
   - Caveat: If ALL pods fail their readiness probes because an external service is down, all pods get removed from the load balancer. Consider whether this is the desired behavior or if you should fail gracefully.

---

## Common Mistakes Candidates Make

**Mistake 1: Saying "readiness probe restarts the container"**
- Wrong: "When the readiness probe fails, Kubernetes restarts the container."
- Correct: Readiness probe failure NEVER restarts the container. It only removes the pod from Service Endpoints. Only liveness probe failure (or container exit) causes a restart.

**Mistake 2: Making readiness probe identical to liveness probe**
- Wrong: Using the same `/ping` endpoint for both liveness and readiness.
- Correct: Liveness should be lightweight (is the process responding?). Readiness should be comprehensive (are all dependencies ready? Can I serve requests?).

**Mistake 3: Not using readiness probes in deployments**
- Wrong: "I don't need readiness probes, the container is ready once it starts."
- Correct: Without readiness probes, Kubernetes cannot know when an app has finished initializing, loaded caches, established DB connections, etc. This causes failed requests during deployments.

**Mistake 4: Forgetting about `minReadySeconds`**
- Wrong: "The rolling update moves to the next pod as soon as readiness passes."
- Correct: `minReadySeconds` adds a stability buffer — the pod must remain ready for this many seconds before Kubernetes considers the rollout step complete.

**Mistake 5: Setting `successThreshold` too high for readiness after failure**
- Wrong: Setting `successThreshold: 10` means a pod needs 10 consecutive successes before rejoining the load balancer — this causes slow recovery.
- Correct: `successThreshold: 2` or `3` is usually sufficient to confirm stability without delaying recovery.

---

## Troubleshooting Scenario

**Problem:** Rolling deployment is stuck. New pods show `0/1 Running` for 10 minutes. Old pods haven't been terminated. Deployment seems frozen.

**Step-by-Step Debugging:**

```bash
# Step 1: Check deployment rollout status
kubectl rollout status deployment/checkout-service -n production
# Waiting for deployment "checkout-service" rollout to finish:
# 1 out of 3 new replicas have been updated...
# (stuck here)

# Step 2: Check pod status
kubectl get pods -n production -l app=checkout-service
# NAME                                READY   STATUS    RESTARTS   AGE
# checkout-service-old-abc-pod1       1/1     Running   0          2d
# checkout-service-old-abc-pod2       1/1     Running   0          2d
# checkout-service-old-abc-pod3       1/1     Running   0          2d
# checkout-service-new-xyz-pod1       0/1     Running   0          10m  <-- NOT READY

# Step 3: Describe the new pod
kubectl describe pod checkout-service-new-xyz-pod1 -n production
# Events:
#   Warning  Unhealthy  30s   kubelet  Readiness probe failed:
#            HTTP probe failed with statuscode: 503

# Step 4: Check what the readiness endpoint returns
kubectl exec -it checkout-service-new-xyz-pod1 -n production -- \
  curl -v http://localhost:8080/health/ready
# HTTP/1.1 503 Service Unavailable
# {"status": "DOWN", "details": {"database": {"status": "DOWN",
#   "details": {"error": "Connection refused: postgres:5432"}}}}

# Step 5: Check if the database service exists and is reachable
kubectl get svc postgres -n production
# Error: services "postgres" not found  <-- PROBLEM FOUND!

# Step 6: Check the database
kubectl get pods -n production -l app=postgres
# No resources found.  <-- Database not deployed!

# Root cause: New deployment depends on a PostgreSQL database
# that was not deployed to this namespace yet.

# Resolution:
# 1. Deploy PostgreSQL to the production namespace
# 2. OR update readiness probe to be more forgiving during DB absence
# 3. Verify: kubectl rollout status deployment/checkout-service -n production

# Step 7: After deploying the database, watch pods become ready
kubectl get pods -n production -w
# NAME                             READY   STATUS    RESTARTS
# checkout-service-new-xyz-pod1   0/1     Running   0
# checkout-service-new-xyz-pod1   1/1     Running   0   <-- Now ready!
# checkout-service-old-abc-pod1   1/1     Terminating
```

---

## kubectl Commands

```bash
# Get pods and see READY column (X/Y where X=ready containers)
kubectl get pods -n production
# NAME                          READY   STATUS    RESTARTS   AGE
# my-app-7d4b9c-xkp2m          1/1     Running   0          2d     # Fully ready
# my-app-7d4b9c-yz9n3          0/1     Running   0          30s    # Readiness failing

# Describe pod to see readiness probe config and failures
kubectl describe pod my-app-7d4b9c-yz9n3 -n production
# Readiness:  http-get http://:8080/health/ready delay=10s timeout=5s period=5s #success=1 #failure=3
# Events:
#   Warning  Unhealthy  10s   kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503

# Check endpoints — see which pods are in the Service load balancer
kubectl get endpoints my-service -n production
# NAME         ENDPOINTS                         AGE
# my-service   10.0.0.1:8080,10.0.0.2:8080      2d
# (10.0.0.3 is missing because that pod's readiness probe is failing)

# Check EndpointSlices (modern Kubernetes)
kubectl get endpointslices -n production -l kubernetes.io/service-name=my-service
# NAME              ADDRESSTYPE   PORTS   ENDPOINTS                 AGE
# my-service-abc    IPv4          8080    10.0.0.1,10.0.0.2         2d

# Watch readiness state changes
kubectl get pods -n production -w
# NAME              READY   STATUS    RESTARTS
# my-app-pod-1      1/1     Running   0
# my-app-pod-2      0/1     Running   0       <-- readiness failing
# my-app-pod-2      1/1     Running   0       <-- readiness recovered

# Check rolling deployment progress (relies on readiness probes)
kubectl rollout status deployment/my-app -n production
# Waiting for deployment "my-app" rollout to finish: 1 of 3 updated replicas are available...
# deployment "my-app" successfully rolled out

# Rollback deployment if readiness probes keep failing
kubectl rollout undo deployment/my-app -n production
# deployment.apps/my-app rolled back

# Manually test readiness endpoint
kubectl exec -it my-app-pod -n production -- curl -s http://localhost:8080/health/ready
# {"status":"UP","components":{"db":{"status":"UP"},"redis":{"status":"UP"}}}

# Get pod YAML to inspect readiness probe config
kubectl get pod my-app-pod -n production -o yaml | grep -A 20 readinessProbe

# Force-remove pod from endpoints temporarily (for testing)
# The proper way is to fail the readiness probe, but for testing:
kubectl label pod my-app-pod -n production app=my-app-maintenance
# (this removes it from Service selector)
```

---

## YAML Example

```yaml
# Complete Deployment YAML showing all 3 readiness probe types
# and comparison with liveness probe
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo                    # Deployment name
  namespace: production                   # Target namespace
  labels:
    app: readiness-demo
spec:
  replicas: 3                             # 3 replicas for HA
  minReadySeconds: 10                     # Pod must be ready for 10s before considered available
  selector:
    matchLabels:
      app: readiness-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1                   # At most 1 pod unavailable during update
      maxSurge: 1                         # At most 1 extra pod during update
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:

      # ============================================================
      # CONTAINER 1: HTTP GET Readiness Probe (most common)
      # This checks DB + cache connectivity — more thorough than liveness
      # ============================================================
      - name: web-api
        image: my-checkout-service:2.0.0  # Application image
        ports:
        - containerPort: 8080
          name: http                       # Named port
        
        # LIVENESS PROBE: Lightweight — just check if process responds
        livenessProbe:
          httpGet:
            path: /health/live            # Lightweight endpoint: just returns 200 if process running
            port: 8080
          initialDelaySeconds: 30         # Wait for JVM startup
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1             # Must be 1 for liveness
        
        # READINESS PROBE: Thorough — check all dependencies
        readinessProbe:
          httpGet:
            path: /health/ready           # Checks DB + Redis connectivity
            port: 8080
            httpHeaders:
            - name: X-Probe-Type          # Custom header to identify probe traffic in logs
              value: readiness
          initialDelaySeconds: 10         # Start checking earlier — readiness is more conservative
          periodSeconds: 5                # Check more frequently — fast failover
          timeoutSeconds: 5               # Allow time for DB health checks
          failureThreshold: 3             # 3 failures → remove from LB
          successThreshold: 2             # 2 consecutive successes before re-adding to LB
                                          # (successThreshold can be > 1 for readiness!)
        
        env:
        - name: DB_HOST
          value: "postgres-service"
        - name: CACHE_HOST
          value: "redis-service"
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "1000m"

      # ============================================================
      # CONTAINER 2: exec Readiness Probe
      # Use for: complex readiness logic, database clients
      # ============================================================
      - name: db-client
        image: my-db-client:1.0.0
        readinessProbe:
          exec:                           # Execute command inside container
            command:
            - /bin/sh
            - -c
            - |
              mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -e "SELECT 1" && \
              test -f /tmp/app-ready      # Both DB check AND file check must pass
          initialDelaySeconds: 15         # MySQL client needs time to initialize
          periodSeconds: 10
          timeoutSeconds: 10              # Allow time for DB query
          failureThreshold: 3
          successThreshold: 1
        
        # Liveness probe is simpler — just check if process responds
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "pgrep -x my-db-client"     # Just check if the process is running
          initialDelaySeconds: 30
          periodSeconds: 30
          failureThreshold: 3

      # ============================================================
      # CONTAINER 3: tcpSocket Readiness Probe
      # Use for: databases, message queues, non-HTTP services
      # ============================================================
      - name: kafka-consumer
        image: my-kafka-consumer:1.0.0
        ports:
        - containerPort: 8081             # Management port
          name: management
        readinessProbe:
          tcpSocket:                      # TCP socket readiness probe
            port: management             # Use named port
          initialDelaySeconds: 20         # Kafka consumer takes time to join consumer group
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        
        livenessProbe:
          tcpSocket:
            port: management
          initialDelaySeconds: 60         # Give more time for liveness
          periodSeconds: 20
          failureThreshold: 3

---
# Service: only routes to pods with passing readiness probes
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo-svc
  namespace: production
spec:
  selector:
    app: readiness-demo                   # Selects pods with this label
  ports:
  - port: 80                              # Service port
    targetPort: http                      # Named port on pod
    protocol: TCP
  type: ClusterIP

---
# PodDisruptionBudget: works with readiness probes
# Ensures minimum ready pods during voluntary disruptions
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: readiness-demo-pdb
  namespace: production
spec:
  minAvailable: 2                         # At least 2 ready pods at all times
  selector:
    matchLabels:
      app: readiness-demo                 # Same selector as deployment

---
# Example: Implementing circuit-breaker behavior with readiness
# When the pod is overloaded, it returns 503 from /health/ready
# This removes it from the LB — natural circuit breaker
apiVersion: v1
kind: Pod
metadata:
  name: circuit-breaker-example
  namespace: production
spec:
  containers:
  - name: api
    image: my-overload-aware-api:1.0.0
    readinessProbe:
      httpGet:
        path: /health/ready              # Returns 503 when queue is full
        port: 8080
      periodSeconds: 5                   # Check frequently
      failureThreshold: 2                # Quick removal when overloaded
      successThreshold: 3                # Require 3 consecutive successes to re-add
                                         # (prevents flapping under load)
```

---

## AWS/EKS Perspective

**EKS-Specific Considerations:**

1. **AWS Load Balancer Controller (ALB/NLB):**
   - The ALB Ingress Controller respects Kubernetes readiness probes indirectly.
   - ALB deregisters targets when pods fail readiness (via EndpointSlice changes).
   - ALB has its own deregistration delay (default 300 seconds) — configure with annotation:
     `alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30`
   - This means even after readiness fails, ALB might still send traffic for up to 300 seconds!
   - Always tune this annotation for faster failover.

2. **EKS and Target Health Checks:**
   - NLB with IP target type: targets are deregistered when pod readiness fails.
   - NLB deregistration delay also applies — set it low for fast failover.
   - Use annotation: `service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: deregistration_delay.timeout_seconds=30`

3. **EKS Node Termination and Readiness:**
   - AWS Node Termination Handler (NTH) sends SIGTERM and waits for pods to drain.
   - Proper readiness probe failure (returning 503) during shutdown ensures graceful drain.
   - Pattern: on SIGTERM, set a flag that makes readiness probe return 503, wait for connections to drain, then exit.

4. **Container Insights Metrics:**
   - `pod_status_ready` metric in CloudWatch Container Insights tracks readiness state.
   - Create CloudWatch alarms: alert when `pod_status_ready = 0` for > 5 minutes.
   - Use AWS Managed Grafana with the CloudWatch datasource to visualize readiness trends.

5. **EKS Rolling Deployments with Readiness:**
   - EKS CodeDeploy integration (EKS Blue/Green via CodeDeploy) uses readiness probes to determine shift timing.
   - AWS recommends setting `minReadySeconds: 30` in EKS production deployments.

6. **Fargate Considerations:**
   - Cold starts on Fargate are longer — set `initialDelaySeconds` to 60+ for Java apps on Fargate.
   - Fargate pods have dedicated resources — no CPU throttling from neighbors.
   - Fargate ENI provisioning adds ~30-60 seconds to pod startup time.

---

## Interview Answer (2-Minute Version)
*For candidates with 1-2 years of experience:*

"A Readiness Probe determines whether a container is ready to receive traffic. The key difference from a Liveness Probe is what happens on failure: a liveness probe failure restarts the container, while a readiness probe failure only removes the pod from the Service's load balancer. The container keeps running — it just stops receiving traffic.

There are three probe types: httpGet, exec, and tcpSocket — same as liveness probes. For readiness, I typically use an httpGet probe on a `/health/ready` endpoint that checks database connectivity and any required external dependencies.

A classic use case is during rolling deployments: new pods should only receive traffic after they've finished connecting to the database and loading any required caches. Without a readiness probe, Kubernetes might route traffic to a pod that's not fully initialized yet, causing request failures.

When the readiness probe fails during normal operation — say the database goes down — the pod is removed from the load balancer automatically, preventing it from serving requests it can't handle."

---

## Interview Answer (Senior Engineer Version)
*For candidates with 3-7 years of experience:*

"The Readiness Probe is fundamentally about traffic management rather than container lifecycle management — that distinction is critical. When a readiness probe fails, the Endpoint controller removes the pod's IP from the Endpoints and EndpointSlice objects for all matching Services. kube-proxy then updates iptables or IPVS rules, removing that pod from the rotation. The container is untouched.

This has several important implications for production deployments. First, for rolling updates, Kubernetes relies on readiness state to determine update progress. A pod must become ready before Kubernetes terminates an old pod, maintaining availability throughout the rollout. Combined with `minReadySeconds`, you can add a stability buffer to catch transient failures after initial startup.

Second, unlike liveness probes which should be lightweight, readiness probes should be comprehensive — they should check every dependency the app needs to serve requests. If the app needs a database, check the database. If it needs a cache, check the cache.

Third, `successThreshold` can be greater than 1 for readiness probes, which is valuable for preventing flapping — if a pod keeps toggling between ready and not-ready, requiring 3 consecutive successes before re-adding it to the load balancer provides stability.

In AWS/EKS environments, there's an important interaction to be aware of: the ALB deregistration delay (default 300s) means that even after readiness fails, ALB continues sending traffic for up to 5 minutes. I always tune this annotation down to 30 seconds for applications that need fast failover.

I also use readiness probe failure as a circuit breaker pattern — when a pod detects it's overloaded (queue depth too high, response time too high), it can return 503 from its readiness endpoint, voluntarily removing itself from the load balancer until it drains."

---

## What Impresses the Interviewer

- Clearly distinguishing: readiness failure = remove from LB (not restart)
- Knowing that `successThreshold` can be > 1 for readiness (unlike liveness)
- Mentioning `minReadySeconds` in conjunction with readiness probes
- Explaining the circuit-breaker pattern using readiness probes
- Discussing ALB deregistration delay in EKS context
- Knowing about EndpointSlices vs Endpoints
- Mentioning PodDisruptionBudgets in context of readiness
- Understanding flapping prevention with `successThreshold`

---

## Red Flags

- Saying "readiness probe restarts the container" — this is completely wrong
- Making the readiness probe identical to the liveness probe
- Not knowing what happens to Service Endpoints when readiness fails
- No mention of rolling deployment behavior
- Not knowing the difference between `successThreshold` behavior for liveness vs readiness
- Saying "I just set `initialDelaySeconds: 120` so the app has time to start" — this prevents Kubernetes from knowing the pod is ready early

---

## Production Best Practices

1. **Separate liveness and readiness endpoints** — `/health/live` for liveness (lightweight), `/health/ready` for readiness (comprehensive, checks all dependencies).

2. **Readiness checks should mirror the request path** — if serving a request requires DB + cache, the readiness check must verify DB + cache are available.

3. **Tune `successThreshold: 2` or `3` for high-churn services** — prevents flapping behavior where a recovering pod oscillates in and out of the load balancer.

4. **Set `minReadySeconds` in Deployments** — adds stability buffer beyond the first readiness success. Recommended: 10-30 seconds for production services.

5. **Implement graceful shutdown via readiness** — on SIGTERM, immediately fail the readiness probe to drain in-flight connections before the process exits.

6. **Consider ALL pods failing readiness simultaneously** — if all pods check the same database and the database goes down, all pods get removed from the load balancer. Design your system to handle this (return cached data, circuit breaker, etc.).

7. **Use `failureThreshold: 1` with `periodSeconds: 5` for fast failover** — for latency-sensitive services, quick detection and removal of unhealthy pods is critical.

8. **Monitor readiness metrics** — alert on `pod_status_ready = 0` for extended periods. A pod that's persistently not ready indicates a real problem requiring investigation.

---

## Key Points to Remember

- Readiness probe answers: **"Is this container ready to receive traffic?"**
- Failure action: **removes pod from Service Endpoints** (does NOT restart container)
- Container continues running when readiness probe fails
- `successThreshold` **CAN be > 1** for readiness probes (unlike liveness where it must be 1)
- Critical for **zero-downtime rolling deployments**
- Readiness probe should be **more comprehensive** than liveness — check dependencies
- Works with **`minReadySeconds`** in Deployment spec for stability buffers
- All pods failing simultaneously means **no endpoints** in the Service
- In EKS, ALB has its own **deregistration delay** separate from readiness probe
- Supports **circuit breaker patterns** — pods can voluntarily remove themselves from LB

---

## Interviewer's Expectation

The interviewer is testing:
1. **Core understanding** — do you know it controls traffic routing, not container restarts?
2. **Deployment knowledge** — do you understand how it enables zero-downtime rolling updates?
3. **Probe design** — do you know when to check dependencies (readiness) vs. lightweight check (liveness)?
4. **Production awareness** — do you know about flapping, `minReadySeconds`, `successThreshold > 1`?
5. **AWS/EKS integration** — do you know about ALB deregistration delay?
6. **Advanced patterns** — circuit breaker using readiness, graceful shutdown pattern?

---

## Final Perfect Interview Answer

"A Readiness Probe determines whether a Kubernetes pod should receive traffic from a Service. The critical distinction from a Liveness Probe is the failure action: readiness failure removes the pod from the Service's Endpoints — the container continues running, it simply stops receiving traffic. Liveness failure, by contrast, restarts the container.

This design enables zero-downtime rolling deployments: Kubernetes waits for each new pod's readiness probe to pass before terminating an old pod. The readiness probe confirms the new pod has finished initializing, connected to its database, warmed its cache — whatever it needs to serve requests correctly.

In production, I implement distinct health endpoints: a lightweight `/health/live` for liveness that just confirms the process is responding, and a comprehensive `/health/ready` for readiness that validates all upstream dependencies. I also set `successThreshold: 2` to prevent flapping during recovery, and `minReadySeconds` in the Deployment to add a stability buffer.

A powerful production pattern is using readiness probes as a circuit breaker — when a service detects it's overloaded, it returns 503 from its readiness endpoint, voluntarily removing itself from the load balancer until it recovers. This is self-healing infrastructure at its best."
