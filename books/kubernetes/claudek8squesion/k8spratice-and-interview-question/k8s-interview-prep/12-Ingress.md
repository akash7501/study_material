# Ingress — Kubernetes Interview Guide

## Interview Question
"Explain what Kubernetes Ingress is, how it differs from a LoadBalancer Service, and describe how you have used it in a production environment."

---

## Simple Explanation
Imagine you run a big apartment building (your Kubernetes cluster). Inside the building, you have many apartments: one for your website, one for your API, one for your admin panel. You have ONE main door (Ingress) at the entrance.

A security guard (Ingress Controller) stands at that door. When someone arrives, the guard looks at which apartment they want:
- "I want to visit the website" → guard directs them to Apartment 1 (the website service)
- "I want to use the API" → guard directs them to Apartment 2 (the API service)
- "I want the admin panel" → guard checks their credentials, then directs to Apartment 3

Without Ingress, every apartment would need its own separate entrance door — that is expensive and messy. With Ingress, ONE door handles all traffic intelligently.

In simple terms: **Ingress = smart traffic routing into your cluster based on URL path or hostname**.

---

## Technical Explanation
**Ingress** is a Kubernetes API object that defines rules for routing external HTTP/HTTPS traffic to internal Services within a cluster. It is NOT a service type — it is a configuration object.

**Two components work together:**

1. **Ingress Resource**: The YAML configuration object that defines routing rules — "if hostname is api.example.com, route to api-service on port 8080."

2. **Ingress Controller**: The actual implementation that reads Ingress resources and configures a reverse proxy (NGINX, HAProxy, Traefik, AWS ALB, etc.). Without an Ingress Controller, Ingress resources do nothing.

**How it works internally:**

1. You deploy an Ingress Controller (e.g., ingress-nginx) — this runs as a Deployment and exposes itself via a LoadBalancer Service.
2. You create an Ingress resource with routing rules.
3. The Ingress Controller watches the Kubernetes API for Ingress objects using an informer/watch mechanism.
4. When an Ingress is created/updated, the controller reconciles and updates its internal proxy configuration (e.g., generates nginx.conf).
5. External traffic hits the Ingress Controller's LoadBalancer IP.
6. The controller proxy inspects the Host header and URL path, then forwards to the matching backend Service.
7. The Service routes to one of its Pods.

**Key Features:**
- Host-based routing (virtual hosting): `api.example.com` → api-service, `www.example.com` → frontend-service
- Path-based routing: `/api` → api-service, `/static` → cdn-service
- TLS termination: HTTPS handled at the Ingress, HTTP internally
- Annotations for advanced features: rate limiting, CORS, auth, rewrites

---

## Real-World Example
**Company**: A SaaS startup with 20 engineers running a multi-tenant application on EKS.

**Architecture:**
- `app.company.com` → React frontend (port 3000)
- `api.company.com` → Node.js REST API (port 8080)
- `api.company.com/v2` → New Python API microservice (port 5000)
- `admin.company.com` → Internal admin dashboard (port 4000, requires IP allowlisting)

**Solution**: Single AWS ALB Ingress (using AWS Load Balancer Controller) routes all traffic. This costs 1 ALB instead of 4 separate LoadBalancer Services (saving ~$70/month per extra ALB). SSL certificates are managed by AWS ACM and attached to the ALB at the Ingress level.

The team uses NGINX Ingress for their staging environment (where cost matters less but features like rate limiting and custom auth are important) and AWS ALB Ingress for production (native AWS integration, WAF support, better scalability).

---

## Diagram / Flow

```
INGRESS TRAFFIC FLOW
=====================

Internet
    |
    | HTTPS Request: api.company.com/users
    v
+----------------------------------+
|   DNS: api.company.com           |
|   Points to ALB/NGINX IP         |
+----------------------------------+
    |
    v
+----------------------------------+
|   Ingress Controller             |
|   (NGINX / AWS ALB / Traefik)    |
|   - Running as Pod in cluster    |
|   - Exposed via LoadBalancer SVC |
|   - Reads Ingress rules          |
+----------------------------------+
    |
    | Matches rule: host=api.company.com, path=/users
    |
    v
+----------------------------------+
|   Ingress Resource               |
|   Rules:                         |
|   api.company.com →              |
|     /v2/* → python-api:5000      |
|     /*    → node-api:8080        |
|   www.company.com →              |
|     /*    → frontend:3000        |
|   admin.company.com →            |
|     /*    → admin-svc:4000       |
+----------------------------------+
    |
    | Routes to: node-api Service (port 8080)
    v
+----------------------------------+
|   Service: node-api              |
|   ClusterIP: 10.96.45.23:8080    |
|   Selector: app=node-api         |
+----------------------------------+
    |
    | kube-proxy routes to healthy Pod
    v
+-------------------------------+   +-------------------------------+
|  Pod: node-api-7d9f8b-abc     |   |  Pod: node-api-7d9f8b-xyz     |
|  10.244.1.5:8080              |   |  10.244.2.7:8080              |
+-------------------------------+   +-------------------------------+


HOST-BASED vs PATH-BASED ROUTING
==================================

HOST-BASED:                          PATH-BASED:
  api.company.com ──► api-service      company.com/api  ──► api-service
  www.company.com ──► web-service      company.com/shop ──► shop-service
  auth.company.com ──► auth-service    company.com/     ──► web-service


TLS TERMINATION FLOW
=====================

Client ──HTTPS──► Ingress Controller (TLS terminates here)
                        │
                        └──HTTP──► Backend Services (internal, no SSL overhead)


INGRESS CONTROLLER TYPES
==========================

ingress-nginx     ─── Most popular, feature-rich, community
AWS ALB           ─── Native AWS, WAF/Shield/ACM integration
Traefik           ─── Cloud-native, automatic SSL via Let's Encrypt
HAProxy           ─── High performance, enterprise
Istio Gateway     ─── Service mesh, advanced traffic management
```

---

## Why It Is Important
**Business Value:**
- Single public endpoint reduces cloud costs (one LoadBalancer vs many).
- Centralized SSL/TLS management — one place to renew certificates.
- Enables blue-green and canary deployments at the traffic routing layer.
- Supports WAF integration (AWS WAF on ALB Ingress) for security compliance.

**Technical Value:**
- Decouples external routing from internal Service definitions.
- Supports multiple virtual hosts on a single IP address.
- Enables cross-namespace routing (with some controllers).
- Gateway API (the successor to Ingress) builds on these concepts with more expressive routing.

---

## Common Interview Follow-Up Questions
1. What is the difference between Ingress and a LoadBalancer Service?
2. Can you have multiple Ingress Controllers in the same cluster?
3. How do you handle TLS/HTTPS with Ingress?
4. What is cert-manager and how does it work with Ingress?
5. How do you implement rate limiting or authentication with NGINX Ingress?
6. What is the Kubernetes Gateway API and how is it different from Ingress?
7. How do you do canary deployments using Ingress annotations?
8. What happens if the Ingress Controller pod crashes?

---

## Common Mistakes Candidates Make

**Mistake 1: Thinking Ingress works without an Ingress Controller**
- Wrong: "You just apply the Ingress YAML and traffic starts routing."
- Correct: An Ingress resource is just configuration. You MUST deploy an Ingress Controller (like ingress-nginx) first. Without it, Ingress resources are ignored.

**Mistake 2: Confusing Ingress with a Service type**
- Wrong: "Ingress is a type of Service like ClusterIP or NodePort."
- Correct: Ingress is a completely separate API resource, not a Service type. Services route pod-to-pod traffic; Ingress routes external HTTP/HTTPS traffic to Services.

**Mistake 3: Not knowing about TLS configuration**
- Wrong: "You configure HTTPS in the Deployment environment variables."
- Correct: TLS is configured in the Ingress resource's `tls` section using a Kubernetes Secret containing the certificate and key. The Ingress Controller handles the TLS handshake.

**Mistake 4: Assuming one Ingress Controller per cluster**
- Wrong: "You can only have one Ingress Controller."
- Correct: You can run multiple Ingress Controllers in the same cluster using `IngressClass` to specify which controller handles which Ingress resources. Common pattern: NGINX for internal, ALB for external.

**Mistake 5: Not knowing path type differences**
- Wrong: "Path matching in Ingress is always exact."
- Correct: Kubernetes Ingress has three `pathType` values: `Exact` (exact URL match), `Prefix` (prefix match), and `ImplementationSpecific` (controller decides). Getting this wrong causes 404s in production.

---

## Troubleshooting Scenario
**Problem**: Application returns 404 for `api.company.com/users` but the service is running fine.

```bash
# Step 1: Check Ingress resource exists and has correct rules
kubectl get ingress -n production
# NAME              CLASS   HOSTS               ADDRESS          PORTS   AGE
# api-ingress       nginx   api.company.com     203.0.113.10     80,443  5d

# Step 2: Describe the Ingress — look for backend service config
kubectl describe ingress api-ingress -n production
# Rules:
#   Host             Path  Backends
#   ----             ----  --------
#   api.company.com
#                    /users   api-service:8080 (10.244.1.5:8080)
#
# Events: <none>

# Step 3: Verify the backend service exists and has the right port
kubectl get service api-service -n production
# NAME          TYPE        CLUSTER-IP     PORT(S)    AGE
# api-service   ClusterIP   10.96.45.23   8080/TCP   5d

# Step 4: Check the service has healthy endpoints
kubectl get endpoints api-service -n production
# NAME          ENDPOINTS        AGE
# api-service   <none>           5d    <-- PROBLEM: No endpoints!

# Step 5: Check pods are running and labels match
kubectl get pods -n production -l app=api
# No resources found.    <-- Pods are not running or label mismatch

# Step 6: Check deployment
kubectl get deployment api -n production
# NAME   READY   UP-TO-DATE   AVAILABLE   AGE
# api    0/2     2            0           5d

kubectl describe deployment api -n production
# Events: 0/3 nodes available: 3 Insufficient memory

# Root cause: Pods cannot schedule due to resource constraints
# Fix: Adjust resource requests or scale the cluster

# Alternative: Check if label selector in service matches pod labels
kubectl get pods -n production --show-labels
# NAME              READY   STATUS    LABELS
# api-pod-abc123    1/1     Running   app=api-v2   <-- Label is api-v2, not api!

# Fix: Update service selector to match actual pod labels
kubectl edit service api-service -n production
# Change selector: app: api  to  app: api-v2

# Verify endpoints are now populated
kubectl get endpoints api-service -n production
# NAME          ENDPOINTS                    AGE
# api-service   10.244.1.5:8080,10.244.2.7:8080   5d

# Test the ingress
curl -H "Host: api.company.com" http://203.0.113.10/users
# {"users": [...]}   <-- Working!
```

---

## kubectl Commands

```bash
# List all Ingress resources in a namespace
kubectl get ingress -n production
# NAME            CLASS   HOSTS                   ADDRESS        PORTS     AGE
# api-ingress     nginx   api.company.com         203.0.113.10   80, 443   5d

# Describe Ingress — see rules, backend services, events
kubectl describe ingress api-ingress -n production

# List all IngressClasses in the cluster
kubectl get ingressclass
# NAME    CONTROLLER                             PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx                  <none>       30d
# alb     ingress.k8s.aws/alb                   <none>       30d

# Get Ingress in YAML format
kubectl get ingress api-ingress -n production -o yaml

# Check Ingress Controller pods are running
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-7d9c7b8f9c-kp2mv   1/1     Running   0          30d

# Check Ingress Controller logs for routing errors
kubectl logs -n ingress-nginx ingress-nginx-controller-7d9c7b8f9c-kp2mv --tail=50

# Check NGINX config generated by Ingress Controller
kubectl exec -n ingress-nginx ingress-nginx-controller-7d9c7b8f9c-kp2mv -- nginx -T | grep -A 10 "api.company.com"

# Check TLS secret exists
kubectl get secret api-tls-cert -n production
# NAME           TYPE                DATA   AGE
# api-tls-cert   kubernetes.io/tls   2      30d

# Check cert-manager certificate status
kubectl get certificate -n production
# NAME           READY   SECRET         AGE
# api-cert       True    api-tls-cert   30d

# Apply an Ingress
kubectl apply -f ingress.yaml

# Delete an Ingress
kubectl delete ingress api-ingress -n production
```

---

## YAML Example

```yaml
# ============================================================
# IngressClass — tells which controller handles this Ingress
# ============================================================
apiVersion: networking.k8s.io/v1        # Ingress API group
kind: IngressClass                       # Resource type
metadata:
  name: nginx                            # Name referenced by Ingress resources
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # Makes this the default class
spec:
  controller: k8s.io/ingress-nginx       # The Ingress Controller that owns this class
---
# ============================================================
# TLS Secret — holds the SSL certificate and private key
# ============================================================
apiVersion: v1
kind: Secret
metadata:
  name: api-tls-cert                     # Name referenced in Ingress tls section
  namespace: production                  # Must be in SAME namespace as Ingress
type: kubernetes.io/tls                  # Special type for TLS certificates
data:
  tls.crt: <base64-encoded-certificate>  # The certificate (base64 encoded)
  tls.key: <base64-encoded-private-key>  # The private key (base64 encoded)
---
# ============================================================
# Ingress Resource — the routing rules
# ============================================================
apiVersion: networking.k8s.io/v1        # Stable API since Kubernetes 1.19
kind: Ingress                            # Resource type
metadata:
  name: api-ingress                      # Name of this Ingress resource
  namespace: production                  # Namespace scope
  labels:
    app: api                             # Labels for organization
    environment: production
  annotations:
    # Specify which Ingress Controller handles this (alternative to spec.ingressClassName)
    kubernetes.io/ingress.class: "nginx"

    # Force HTTP to HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate limiting — 100 requests per second per IP
    nginx.ingress.kubernetes.io/limit-rps: "100"

    # Enable CORS for API
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.company.com"

    # Increase proxy timeout for long-running requests
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"

    # Rewrite target (strips path prefix before forwarding)
    # nginx.ingress.kubernetes.io/rewrite-target: /$2

spec:
  ingressClassName: nginx                # Which IngressClass to use (preferred over annotation)

  tls:                                   # TLS configuration for HTTPS
  - hosts:
    - api.company.com                    # Hostname covered by this certificate
    - www.company.com
    secretName: api-tls-cert             # Secret containing the TLS certificate

  rules:                                 # List of routing rules
  # ── Rule 1: API subdomain routing ──
  - host: api.company.com                # Only match requests for this hostname
    http:
      paths:
      - path: /v2                        # Match requests starting with /v2
        pathType: Prefix                 # Prefix match: /v2, /v2/users, /v2/orders
        backend:
          service:
            name: python-api-service     # Forward to this Service
            port:
              number: 5000              # Service port to forward to
      - path: /                          # Default: match everything else
        pathType: Prefix
        backend:
          service:
            name: node-api-service       # Forward to this Service
            port:
              number: 8080

  # ── Rule 2: Web frontend routing ──
  - host: www.company.com               # Match requests for www subdomain
    http:
      paths:
      - path: /                          # Match all paths
        pathType: Prefix
        backend:
          service:
            name: frontend-service       # Forward to frontend service
            port:
              number: 3000

  # ── Rule 3: Admin panel (same ingress, different host) ──
  - host: admin.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 4000
---
# ============================================================
# Backend Services that Ingress routes to
# ============================================================
apiVersion: v1
kind: Service
metadata:
  name: node-api-service                 # Must match Ingress backend service name
  namespace: production                  # Must be in same namespace as Ingress
spec:
  selector:
    app: node-api                        # Selects pods with this label
  ports:
  - port: 8080                           # Port the Service listens on (matches Ingress)
    targetPort: 8080                     # Port on the Pod container
  type: ClusterIP                        # ClusterIP is correct — Ingress handles external access
```

---

## AWS/EKS Perspective

**Two main options for Ingress on EKS:**

**Option 1: AWS Load Balancer Controller (ALB Ingress)**
- Provisions an Application Load Balancer per Ingress (or shared).
- Native AWS integration: ACM certificates, WAF, Shield, Target Groups.
- Better for production — supports AWS-native features.
- Annotation: `kubernetes.io/ingress.class: alb`

**Option 2: NGINX Ingress Controller on EKS**
- NGINX runs inside the cluster, exposed via AWS NLB.
- More feature-rich for HTTP routing logic.
- Better for complex routing rules, rate limiting, custom auth.

**EKS-specific considerations:**
1. **ALB IP mode vs Instance mode**: IP mode routes directly to Pod IPs (requires VPC CNI), bypassing kube-proxy. Instance mode routes to node ports. IP mode is preferred for lower latency.
2. **IngressGroup**: ALB Controller supports grouping multiple Ingress resources onto a single ALB using `alb.ingress.kubernetes.io/group.name` annotation — critical for cost optimization.
3. **ACM Certificate**: Use `alb.ingress.kubernetes.io/certificate-arn` annotation to attach ACM-managed certificates.
4. **WAF**: Attach AWS WAF to ALB via `alb.ingress.kubernetes.io/waf-acl-id` for DDoS and OWASP protection.

```yaml
# EKS ALB Ingress Example
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb              # Use AWS ALB Controller
    alb.ingress.kubernetes.io/scheme: internet-facing  # Public-facing ALB
    alb.ingress.kubernetes.io/target-type: ip     # Route to pod IPs directly
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789:certificate/abc123
    alb.ingress.kubernetes.io/ssl-redirect: "443"  # Redirect HTTP to HTTPS
    alb.ingress.kubernetes.io/group.name: production  # Share ALB across Ingresses
    alb.ingress.kubernetes.io/waf-acl-id: web-acl-id  # Attach AWS WAF
spec:
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

## Interview Answer (2-Minute Version)
"Ingress is how you expose HTTP and HTTPS applications to the outside world in Kubernetes. It sits in front of your Services and does intelligent routing — so one external IP can route to many different backend services based on the URL hostname or path.

There are two pieces: the Ingress resource, which is just the YAML config with routing rules, and the Ingress Controller, which is the actual reverse proxy that implements those rules. Without a controller, Ingress does nothing.

The main benefit over using LoadBalancer Services directly is cost and simplicity — instead of creating a separate cloud load balancer for every service, you have one entry point. On EKS, I've used both NGINX Ingress and the AWS Load Balancer Controller, choosing ALB Ingress for production because it integrates natively with ACM for certificates and AWS WAF for security."

---

## Interview Answer (Senior Engineer Version)
"Ingress is the Layer 7 HTTP routing abstraction in Kubernetes. The resource itself is just declarative intent — the actual enforcement is done by the Ingress Controller, which is a separately deployed component running a control loop that watches Ingress resources and translates them into proxy configuration.

The critical insight is that the Ingress API is intentionally thin and controller-agnostic. The real power comes from controller-specific annotations. For example, NGINX Ingress uses annotations for rate limiting, custom auth (via `auth-url`), canary deployments (via `canary-weight`), and request rewrites. This also means Ingress configs are often not fully portable between controllers.

In production on EKS I use the AWS Load Balancer Controller in IngressGroup mode — grouping multiple Ingress resources onto a single ALB with weighted target groups for canary releases. We attach AWS WAF for OWASP protection and ACM for cert management. For internal services, I deploy NGINX Ingress backed by an internal NLB to route traffic between namespaces with mTLS passthrough.

I'm also watching the Gateway API closely — it solves the limitations of Ingress by providing proper role-based objects (GatewayClass, Gateway, HTTPRoute) and better multi-tenancy support. I expect it to replace Ingress as the standard in the next 2-3 years."

---

## What Impresses the Interviewer
- Knowing that Ingress Controller must be deployed separately — it is not built into Kubernetes
- Explaining the difference between NGINX Ingress and AWS ALB Ingress and when to use each
- Mentioning IngressClass and running multiple controllers simultaneously
- Knowing about canary deployments via NGINX annotations
- Bringing up the Gateway API as the future replacement
- Mentioning cert-manager for automated TLS certificate management

---

## Red Flags
- Saying Ingress is built into Kubernetes and works without any additional components
- Confusing Ingress with a Service type
- Not being able to explain TLS termination
- Saying you need one Load Balancer per application (not knowing the cost advantage of Ingress)
- Not knowing what an IngressClass is

---

## Production Best Practices
1. **Always use IngressClass** to explicitly specify which controller handles each Ingress — avoids conflicts in multi-controller clusters.
2. **Use cert-manager** with Let's Encrypt for automatic TLS certificate provisioning and renewal.
3. **Set resource requests/limits** on your Ingress Controller pods — it is a critical path component.
4. **Run Ingress Controller with multiple replicas** and use PodDisruptionBudget — single point of failure in production is unacceptable.
5. **Use IngressGroup** on AWS ALB Ingress to consolidate multiple Ingresses onto one ALB (cost optimization).
6. **Implement rate limiting** via NGINX annotations to protect APIs from abuse.
7. **Enable access logging** on the Ingress Controller and ship logs to your observability platform (DataDog, ELK, etc.).
8. **Use `WaitForFirstConsumer`** and health checks on backends to prevent traffic routing to not-ready pods.

---

## Key Points to Remember
- Ingress = routing rules; Ingress Controller = the component that enforces them
- Ingress Controller must be installed separately — it is not included in Kubernetes
- Ingress routes HTTP/HTTPS traffic; it does NOT handle TCP/UDP (use Service for that)
- IngressClass specifies which controller handles a given Ingress resource
- TLS is terminated at the Ingress Controller; internal traffic is typically HTTP
- Path types: `Exact`, `Prefix`, and `ImplementationSpecific`
- NGINX Ingress is most feature-rich; AWS ALB Ingress has best AWS integration
- One Ingress Controller can be the entry point for dozens of services
- Annotations are controller-specific — check docs for your specific controller
- Gateway API is the next-generation replacement for Ingress (more expressive, role-based)

---

## Interviewer's Expectation
The interviewer is testing whether you understand how external traffic enters a Kubernetes cluster, the separation between routing configuration (Ingress resource) and implementation (Ingress Controller), and your practical experience with real-world routing problems. They want to hear about TLS, multi-service routing, and cloud-specific options.

---

## Final Perfect Interview Answer
"Ingress is Kubernetes' way of managing HTTP and HTTPS routing from outside the cluster to internal services. It gives you host-based and path-based routing, TLS termination, and a single external entry point for multiple applications. This is much more cost-effective than creating a separate cloud load balancer for every service.

There are two parts: the Ingress resource, which is the YAML config that defines your routing rules, and the Ingress Controller, which is the actual reverse proxy that reads those rules and enforces them. The controller is not built into Kubernetes — you deploy it separately. Popular choices are NGINX, Traefik, and on AWS, the Load Balancer Controller that provisions ALBs natively.

In production on EKS I've used the AWS Load Balancer Controller with IngressGroup mode, consolidating all our services onto a shared ALB with ACM for certificate management and AWS WAF for DDoS protection. For our staging environment we run NGINX Ingress because we need features like canary deployments via annotation weights and custom OAuth2 proxy authentication. I also use cert-manager to automate TLS certificate lifecycle — you never want a manual certificate renewal causing production downtime."
