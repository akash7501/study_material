# ConfigMap — Kubernetes Interview Guide

## Interview Question
"What is a Kubernetes ConfigMap? Why do we need it, and what are the different ways to use it inside a Pod? What are its limitations?"

---

## Simple Explanation
Imagine you have a mobile app, and your app needs to know which server to connect to, what log level to use, and the name of the database. You could hardcode these values inside your app — but then every time you need to change the server address, you'd have to rebuild and redeploy the app. That's wasteful and slow.

A ConfigMap is like a settings file that lives outside your app. You put all your configuration values there. Your app reads them when it starts. When you need to change a setting, you only update the ConfigMap — you don't have to rebuild your Docker image. This separates your application code from its configuration.

Think of it like the settings menu on your phone — you change settings without reinstalling the app.

---

## Technical Explanation
A ConfigMap is a Kubernetes API object used to store non-sensitive configuration data as key-value pairs. It decouples configuration from container images, following the 12-Factor App methodology (Factor III: Config).

**Storage Format:**
- Stores data as a flat map of string keys to string values.
- Can store individual key-value pairs or entire file contents (multi-line strings).
- Maximum size: 1 MiB (etcd has a 1.5 MiB limit per object; 1 MiB is the practical limit).
- Stored in etcd as part of the cluster state.

**How Pods consume ConfigMaps — 3 methods:**

**Method 1: Environment Variables**
- Each key in the ConfigMap becomes an environment variable in the container.
- Values are injected at Pod startup — changes to ConfigMap do NOT update running container env vars.
- Container must restart to pick up changes.

**Method 2: Volume Mount (as files)**
- The ConfigMap is mounted as a directory. Each key becomes a filename, and the value becomes the file content.
- Kubernetes automatically updates the mounted files when the ConfigMap changes (after a short delay — kubelet sync period, typically 1–2 minutes).
- Application must support config file reloading (like nginx or Spring Cloud Config).

**Method 3: Command-line arguments**
- Environment variables from ConfigMap can be used in the `command` or `args` fields of the container spec.

**Immutable ConfigMaps (Kubernetes 1.21+):**
- Setting `immutable: true` prevents changes to the ConfigMap.
- Improves performance: the kubelet doesn't need to watch the ConfigMap for changes.
- Useful for configuration that should never change (like feature flags locked for a release).

**How updates work (Volume Mount method):**
1. You update the ConfigMap via `kubectl apply`.
2. The API server stores the new ConfigMap in etcd.
3. The kubelet's sync loop (default every 60 seconds) detects the change.
4. The kubelet updates the symbolic links in the mounted volume directory.
5. The application reads the new file content (if it supports hot reload).

---

## Real-World Example
**Company**: A SaaS company running a Node.js application in 3 environments: development, staging, and production.

**Problem**: The application needs different database URLs, API rate limits, log levels, and feature flags per environment. Without ConfigMaps, they'd need separate Docker images for each environment — 3x build pipeline complexity.

**Solution**:
- They create a ConfigMap per environment (or use Helm values to generate them).
- The ConfigMap contains: `DATABASE_URL`, `LOG_LEVEL`, `MAX_REQUESTS_PER_MINUTE`, `FEATURE_FLAG_NEW_UI`, `API_TIMEOUT_MS`.
- The application Deployment references the ConfigMap as environment variables.
- In production, `LOG_LEVEL=error`. In development, `LOG_LEVEL=debug`.
- When the team needs to enable a feature flag, they update the ConfigMap and do a rolling restart — no Docker rebuild, no new image tag, no CI pipeline run needed.
- They also mount the nginx config as a ConfigMap volume, allowing ops team to update nginx configuration without rebuilding the nginx image.

---

## Diagram / Flow

```
DEVELOPER WORKFLOW:
  
  Developer writes config        Developer writes app code
  kubectl apply configmap.yaml   docker build + push image
          |                               |
          v                               v
  +------------------+           +------------------+
  |   etcd Storage   |           |  Container       |
  |                  |           |  Registry        |
  | ConfigMap:       |           |  (ECR/DockerHub) |
  |  app-config      |           +------------------+
  |  LOG_LEVEL=info  |                   |
  |  DB_URL=postgres |                   |
  |  WORKERS=4       |                   |
  +------------------+                   |
          |                              |
          +----------+-------------------+
                     |
                     v
            +------------------+
            |    POD SPEC      |
            |                  |
            | Container:       |
            |  image: my-app   |
            |  envFrom:        |
            |   configMapRef:  |
            |    name:app-cfg  |
            +------------------+
                     |
                     v
            +------------------+
            |  RUNNING POD     |
            | ENV VARS:        |
            |  LOG_LEVEL=info  |
            |  DB_URL=postgres |
            |  WORKERS=4       |
            +------------------+

VOLUME MOUNT METHOD (Auto-update):

ConfigMap Update Flow:
  kubectl edit configmap app-config
          |
          v
  [etcd updated with new values]
          |
          v (kubelet sync, ~60s)
  [Volume mount files updated]
  /etc/config/
    ├── LOG_LEVEL      (contains: "info")
    ├── DB_URL         (contains: "postgres://...")
    └── WORKERS        (contains: "4")
          |
          v (if app supports hot-reload)
  [App reads new config without restart]

ENV VAR METHOD (No auto-update):
  
  ConfigMap changes --> kubelet does NOT update env vars
  --> Pod must be restarted (rolling restart) to pick up changes
  kubectl rollout restart deployment/my-app
```

---

## Why It Is Important
**Business Value:**
- Faster deployments: change config without rebuilding Docker images.
- Same Docker image works in dev, staging, and production — reduces "works on my machine" issues.
- Operators can change configuration without touching application code.
- Supports GitOps workflows — config stored in Git, applied to cluster automatically.

**Technical Value:**
- Implements 12-Factor App methodology (separation of config from code).
- Reduces Docker image count (one image for all environments).
- Enables configuration drift detection — all config is in version control.
- Supports configuration hot-reloading via volume mounts.
- Works with Helm, Kustomize, and other templating tools for environment-specific config.
- Simplifies rollback — roll back the ConfigMap to restore previous configuration.

---

## Common Interview Follow-Up Questions
1. "What is the difference between ConfigMap and Secret?"
2. "How do you update a ConfigMap and ensure Pods pick up the new values?"
3. "What is the 1 MiB size limit on ConfigMaps and what do you do if you need more?"
4. "What is an immutable ConfigMap and why would you use it?"
5. "How would you manage ConfigMaps across multiple environments (dev, staging, prod)?"
6. "Can you mount only specific keys from a ConfigMap, not all of them?"
7. "What happens if a Pod references a ConfigMap that doesn't exist?"
8. "How is a ConfigMap different from a hardcoded environment variable in the Pod spec?"

---

## Common Mistakes Candidates Make

**Mistake 1: Thinking environment variable ConfigMaps auto-update**
- Wrong: "When I change a ConfigMap, the running Pods automatically see the new values."
- Correct: This is true ONLY for volume-mounted ConfigMaps (and with a delay). Environment variables are injected once at Pod startup and are static. To update env vars, you must restart the Pod. Best practice is to use `kubectl rollout restart deployment/<name>`.

**Mistake 2: Storing sensitive data in ConfigMaps**
- Wrong: "I put the database password in the ConfigMap because it's just a string."
- Correct: ConfigMaps are NOT encrypted at rest by default. Sensitive data (passwords, tokens, certificates) must go in Secrets. ConfigMaps are for non-sensitive configuration only.

**Mistake 3: Not knowing the size limit**
- Wrong: "ConfigMaps can hold any amount of configuration data."
- Correct: ConfigMaps are limited to 1 MiB. If you need to store large configuration files, consider storing them in a ConfigMap and referencing a file path, or use an external configuration management system like AWS AppConfig or HashiCorp Vault.

**Mistake 4: Hardcoding ConfigMap names in application code**
- Wrong: Building applications that directly call the Kubernetes API to read ConfigMaps.
- Correct: Applications should read environment variables or files — they should not know they're running in Kubernetes. This keeps applications portable. The Kubernetes-specific injection mechanism is transparent to the app.

**Mistake 5: Not understanding the volume mount symlink behavior**
- Wrong: "Files in a volume-mounted ConfigMap are regular files."
- Correct: Kubernetes uses symbolic links for ConfigMap volume mounts. The actual files are in a directory named after the ConfigMap's resource version (e.g., `..2024_01_15_10_30_00.12345678`). A symlink `..data` points to this directory, and individual file symlinks point through `..data`. This is why atomic updates work — the symlink swap is atomic.

---

## Troubleshooting Scenario

**Problem**: A deployment was updated with a new ConfigMap, but the application is still using old configuration values and showing incorrect behavior.

**Step-by-Step Debugging:**

```bash
# Step 1: Verify the ConfigMap exists and has correct values
kubectl get configmap app-config -n production -o yaml
# Look at the data: section for current values

# Step 2: Check how the ConfigMap is consumed in the Deployment
kubectl describe deployment my-app -n production
# Look at: Environment Variables (envFrom or env) and Volumes sections

# Step 3: If using env vars, check if the Pod was restarted after ConfigMap update
kubectl get pods -n production
# Look at the AGE column — old Pods still have old env vars

# Step 4: Check what env vars are actually set in a running Pod
kubectl exec -it <pod-name> -n production -- env | grep APP_
# Shows current environment variables in the container
# If they're old values, Pod hasn't restarted yet

# Step 5: If using volume mount, check if files were updated
kubectl exec -it <pod-name> -n production -- cat /etc/config/LOG_LEVEL
# Should show the new value if volume was updated

# Step 6: Check if ConfigMap is immutable (can't be changed)
kubectl get configmap app-config -n production -o jsonpath='{.immutable}'
# If 'true', ConfigMap cannot be changed — must delete and recreate

# Step 7: For env var ConfigMaps — trigger rolling restart
kubectl rollout restart deployment/my-app -n production
# This creates new Pods with fresh env vars from current ConfigMap

# Step 8: Verify rollout completed
kubectl rollout status deployment/my-app -n production
# Waiting for deployment "my-app" rollout to finish: 2 of 3 updated replicas are available...
# deployment "my-app" successfully rolled out

# Step 9: Verify new Pods have correct env vars
kubectl exec -it <new-pod-name> -n production -- env | grep LOG_LEVEL
# LOG_LEVEL=error   <-- should be the new value

# Step 10: Check if ConfigMap reference is correct (typo in name?)
kubectl describe pod <pod-name> -n production | grep -A5 "Warning"
# Look for: "configmap not found" or "key not found" errors
```

---

## kubectl Commands

```bash
# Create a ConfigMap from literal values
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_POOL_SIZE=10 \
  --from-literal=TIMEOUT_MS=5000
# configmap/app-config created

# Create a ConfigMap from a file
kubectl create configmap nginx-config --from-file=nginx.conf
# The key is the filename, value is the file content

# Create a ConfigMap from multiple files in a directory
kubectl create configmap app-configs --from-file=./config-dir/
# Each file in the directory becomes a key

# Create a ConfigMap from a file with a specific key name
kubectl create configmap app-config --from-file=config=./app.properties
# Key will be "config", not "app.properties"

# List all ConfigMaps
kubectl get configmaps
# NAME               DATA   AGE
# app-config         3      5d
# nginx-config       1      2d
# kube-root-ca.crt   1      30d

# Describe a ConfigMap (shows all key-value pairs)
kubectl describe configmap app-config
# Name:         app-config
# Namespace:    default
# Data
# ====
# DB_POOL_SIZE:  10
# LOG_LEVEL:     info
# TIMEOUT_MS:    5000

# View ConfigMap as YAML
kubectl get configmap app-config -o yaml

# Edit a ConfigMap directly
kubectl edit configmap app-config
# Opens in editor — save to apply changes

# Update a specific value using patch
kubectl patch configmap app-config \
  --type=merge \
  -p '{"data":{"LOG_LEVEL":"debug"}}'
# configmap/app-config patched

# After updating ConfigMap, restart Deployment to pick up new env var values
kubectl rollout restart deployment/my-app
# deployment.apps/my-app restarted

# Check rollout status
kubectl rollout status deployment/my-app
# deployment "my-app" successfully rolled out

# Delete a ConfigMap
kubectl delete configmap app-config
# configmap "app-config" deleted

# View all data in a ConfigMap (just the data section)
kubectl get configmap app-config -o jsonpath='{.data}'
# {"DB_POOL_SIZE":"10","LOG_LEVEL":"info","TIMEOUT_MS":"5000"}
```

---

## YAML Example

```yaml
# ============================================================
# CONFIGMAP YAML EXAMPLES
# ============================================================

# ---- EXAMPLE 1: Simple Key-Value ConfigMap ----
apiVersion: v1                    # API version for core Kubernetes objects
kind: ConfigMap                   # Resource type
metadata:
  name: app-config                # ConfigMap name — referenced in Pod specs
  namespace: production           # Namespace where this ConfigMap lives
  labels:
    app: my-application           # Labels for organizing/filtering ConfigMaps
    environment: production       # Environment label for identification
data:                             # The actual configuration data (key-value pairs)
  LOG_LEVEL: "info"               # Logging level for the application
  DB_POOL_SIZE: "10"              # Database connection pool size
  TIMEOUT_MS: "5000"              # Request timeout in milliseconds
  ENABLE_METRICS: "true"          # Feature flag for metrics collection
  API_BASE_URL: "https://api.company.com"  # Base URL for external API calls
  MAX_RETRIES: "3"                # Number of retry attempts for failed requests

---
# ---- EXAMPLE 2: ConfigMap with Multi-line File Content ----
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config              # nginx configuration stored as a ConfigMap
  namespace: production
data:
  nginx.conf: |                   # The pipe | preserves multi-line string formatting
    worker_processes 1;
    events {
      worker_connections 1024;
    }
    http {
      server {
        listen 80;
        server_name localhost;
        location / {
          proxy_pass http://app-service:8080;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
        }
        location /health {
          return 200 'OK';
        }
      }
    }
  
  upstream.conf: |                # Second file in the same ConfigMap
    upstream backend {
      server app-service:8080;
    }

---
# ---- EXAMPLE 3: Pod using ConfigMap as Environment Variables ----
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application            # Deployment name
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-application
  template:
    metadata:
      labels:
        app: my-application
    spec:
      containers:
        - name: app-container
          image: my-app:v2.1.0
          
          # Method 1: Load ALL keys from ConfigMap as env vars
          envFrom:
            - configMapRef:
                name: app-config  # Name of the ConfigMap to load from
                # ALL keys in app-config become environment variables
          
          # Method 2: Load SPECIFIC keys from ConfigMap as env vars
          env:
            - name: LOG_LEVEL               # Env var name in the container
              valueFrom:
                configMapKeyRef:
                  name: app-config          # ConfigMap name
                  key: LOG_LEVEL            # Specific key to load
            - name: MY_TIMEOUT              # Different name than the key
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: TIMEOUT_MS           # Map TIMEOUT_MS -> MY_TIMEOUT
                  optional: false           # Pod fails to start if key is missing

---
# ---- EXAMPLE 4: Pod using ConfigMap as Volume Mount ----
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment          # nginx with config from ConfigMap
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: nginx-conf-volume  # Must match volume name below
              mountPath: /etc/nginx/   # Directory where files are mounted
              readOnly: true           # Good practice: mount config as read-only
      volumes:
        - name: nginx-conf-volume      # Volume name (referenced in volumeMounts)
          configMap:
            name: nginx-config         # Name of the ConfigMap to mount
            # All keys in ConfigMap become files in /etc/nginx/
            # /etc/nginx/nginx.conf
            # /etc/nginx/upstream.conf
            
            # Optional: mount only specific keys as files
            items:
              - key: nginx.conf        # Which ConfigMap key to mount
                path: nginx.conf       # What filename to use (can differ from key)

---
# ---- EXAMPLE 5: Immutable ConfigMap ----
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-release-flags         # Immutable config for this release
  namespace: production
immutable: true                   # Once set to true, data CANNOT be changed
                                  # Must delete and recreate to change values
                                  # Improves performance: kubelet doesn't watch this
data:
  RELEASE_VERSION: "v2.1.0"       # Locked to this release version
  FEATURE_FLAG_NEW_CHECKOUT: "true"  # Feature flags locked for this release
  MIN_TLS_VERSION: "1.2"          # Security setting that should not change
```

---

## AWS/EKS Perspective

**ConfigMaps in EKS:**

1. **AWS AppConfig Integration**: For large-scale applications, AWS AppConfig provides a managed configuration service with deployment strategies, rollback, and validation. It integrates with EKS via the AppConfig Agent sidecar container or AWS SDK calls. Use this when ConfigMaps become too complex to manage.

2. **EKS + Helm Charts**: Most EKS production deployments use Helm, which generates ConfigMaps from `values.yaml` files. Different `values-prod.yaml` and `values-staging.yaml` files generate different ConfigMaps per environment.

3. **External Secrets Operator + AWS SSM Parameter Store**: For config that needs audit trails or central management, teams use External Secrets Operator to sync AWS SSM Parameter Store values into Kubernetes ConfigMaps automatically. This allows non-Kubernetes tools (like the AWS Console) to manage configuration.

4. **ConfigMap Size and etcd Performance**: In EKS, etcd is managed by AWS. Large ConfigMaps (close to 1 MiB) can impact API server performance. Monitor etcd size metrics in CloudWatch.

5. **GitOps with ArgoCD**: In EKS environments using ArgoCD, ConfigMaps are stored in Git and automatically synced to the cluster. ConfigMap changes trigger ArgoCD to apply the new values and (if configured) restart affected Deployments.

6. **AWS Secrets Manager vs ConfigMap**: A common question: use ConfigMap for non-sensitive application configuration (URLs, timeouts, feature flags) and use Kubernetes Secrets (backed by AWS Secrets Manager via External Secrets Operator) for sensitive data.

7. **Kustomize overlays**: EKS teams often use Kustomize to manage environment-specific ConfigMap patches. A base ConfigMap is overridden per environment using `configMapGenerator` in `kustomization.yaml`.

---

## Interview Answer (2-Minute Version)
"A ConfigMap is a Kubernetes object that stores configuration data as key-value pairs, separate from the application container image. This follows the 12-Factor App principle of separating config from code.

Without ConfigMaps, you'd need different Docker images for each environment, or you'd hardcode values — both of which are bad practices.

You can use ConfigMaps in three ways: as environment variables, as volume-mounted files, or in command arguments. The key difference is that environment variables are static — they're set at Pod startup and don't change until the Pod restarts. Volume-mounted ConfigMaps are dynamic — Kubernetes automatically updates the files when the ConfigMap changes, which is great for applications that support hot reloading.

The main limitations are: the 1 MiB size limit, and ConfigMaps are NOT encrypted, so they're not suitable for passwords or tokens — that's what Secrets are for."

---

## Interview Answer (Senior Engineer Version)
"ConfigMaps implement the configuration externalisation pattern that's fundamental to cloud-native application design. But the details matter in production.

The most important operational distinction I always emphasize: environment variable injection and volume mounting have completely different update semantics. Env vars are immutable after Pod creation — you must do a rolling restart. Volume mounts use a kubelet sync loop with atomic symlink swaps — the files update automatically but with up to a 2-minute delay by default (configurable via `--sync-frequency`). The atomic swap guarantees the application never sees a partially updated config directory.

In production we've had incidents where engineers changed a volume-mounted ConfigMap expecting immediate application behavior change, but the application didn't support inotify-based file reload — it only read config at startup. The fix was either implementing file watchers in the application or adding configuration reload endpoints.

For immutable ConfigMaps, I use them for release-specific configuration. The benefit isn't just semantic clarity — it actually improves kubelet performance because immutable ConfigMaps are not watched, reducing API server load in large clusters.

At scale, we moved away from raw ConfigMaps toward External Secrets Operator with AWS SSM Parameter Store for a unified configuration management experience. This gave us audit trails, versioning, and the ability for infrastructure teams to update configuration without kubectl access. The ESO syncs SSM parameters to ConfigMaps automatically with a configurable refresh interval.

One subtle issue I've encountered: ConfigMap keys must be valid DNS subdomain names (letters, numbers, dashes, dots, underscores). If you're importing configs from legacy systems with special characters in key names, you'll need to rename them."

---

## What Impresses the Interviewer
- Knowing the difference in update behavior between env var injection and volume mounts — this is where most candidates trip up.
- Mentioning immutable ConfigMaps and their performance benefits.
- Discussing the kubelet sync loop and how volume mount updates work via symlink swaps.
- Knowing the 1 MiB size limit and what to do when you exceed it.
- Mentioning External Secrets Operator or AWS SSM integration for centralized config management.
- Talking about Helm, Kustomize, or GitOps workflows for managing ConfigMaps across environments.
- Understanding that ConfigMaps and Secrets should be treated differently in terms of RBAC access.

---

## Red Flags
- Saying ConfigMaps can store passwords or sensitive data.
- Not knowing about the env var vs volume mount update difference.
- Claiming ConfigMaps auto-update container env vars in real-time.
- Not knowing the size limit.
- Never having actually used ConfigMaps in a multi-environment setup.
- Confusing ConfigMaps with Secrets or thinking they're encrypted.

---

## Production Best Practices
1. **Never store sensitive data in ConfigMaps**: Passwords, API keys, tokens, certificates must go in Secrets. ConfigMaps are stored in plaintext in etcd.
2. **Use immutable ConfigMaps for release configs**: Lock configuration for each release using `immutable: true`. Create a new ConfigMap for each release instead of modifying the existing one.
3. **Use volume mounts for file-based config**: When your application reads a config file (nginx.conf, application.yaml), use volume mounts. For simple key-value env vars, envFrom is cleaner.
4. **Automate rolling restarts**: When updating env-var ConfigMaps, use `kubectl rollout restart deployment/<name>` or configure ArgoCD/Flux to handle it automatically.
5. **Keep ConfigMaps small and focused**: Don't create one giant ConfigMap for everything. Create per-service ConfigMaps. This reduces blast radius when something goes wrong.
6. **Version your ConfigMaps**: Use names like `app-config-v1`, `app-config-v2`, or add a hash suffix. This makes rollbacks explicit and auditable.
7. **Use RBAC to control ConfigMap access**: Developers shouldn't be able to modify production ConfigMaps directly. Use RBAC to limit write access to production namespace ConfigMaps.
8. **Validate ConfigMap values**: Use admission webhooks (OPA/Gatekeeper) to validate that required keys exist and values match expected patterns before ConfigMaps are applied.

---

## Key Points to Remember
- ConfigMap stores non-sensitive configuration as key-value pairs in Kubernetes.
- Maximum size is 1 MiB (stored in etcd).
- Three injection methods: env vars (envFrom/env), volume mounts, command args.
- Env var injection is STATIC — requires Pod restart to update.
- Volume mount files update AUTOMATICALLY (with ~1 min delay via kubelet sync).
- Immutable ConfigMaps (`immutable: true`) cannot be changed and reduce kubelet overhead.
- ConfigMaps are NOT encrypted — never store passwords, tokens, or secrets in them.
- Keys must follow DNS subdomain naming rules (letters, numbers, dashes, dots, underscores).
- ConfigMaps are namespace-scoped — a Pod can only reference ConfigMaps in its own namespace.
- Use Helm, Kustomize, or External Secrets Operator for managing ConfigMaps at scale.

---

## Interviewer's Expectation
The interviewer wants to know if you understand the separation of config from code principle and why it matters. They expect you to know all three ways to use ConfigMaps in Pods. The differentiating question is always about update behavior — most candidates say "yes it auto-updates" without knowing it only applies to volume mounts, not env vars. For senior candidates, they expect knowledge of production patterns: immutability, GitOps integration, External Secrets Operator, and the operational implications of different injection methods.

---

## Final Perfect Interview Answer
"A ConfigMap in Kubernetes is a way to store configuration data separately from your application container image. The main motivation is the 12-Factor App principle: your code should be the same across environments, but the configuration changes. Without ConfigMaps, you'd need a different Docker image for dev, staging, and prod — which is wasteful and error-prone.

ConfigMaps store data as key-value pairs. You can use them in three ways inside a Pod. First, as environment variables using `envFrom` or `env.valueFrom.configMapKeyRef`. Second, as volume-mounted files where each key becomes a file. Third, in command-line arguments.

The critical thing to understand about updates: environment variable injection is static — set once at Pod startup. If you update the ConfigMap, Pods don't see the change until they restart. Volume-mounted files, however, are automatically updated by the kubelet with an approximately one-minute delay, through an atomic symlink swap.

For production, I follow a few key practices: never put sensitive data in ConfigMaps (use Secrets), use immutable ConfigMaps for release-locked configuration, and in EKS we use the External Secrets Operator to sync AWS SSM Parameter Store values into ConfigMaps for centralized config management with audit trails.

The main limitation is the 1 MiB size limit and no encryption — which is why the ConfigMap versus Secret distinction matters."
