# EKS — Amazon Elastic Kubernetes Service — Kubernetes Interview Guide

## Interview Question
**"What is Amazon EKS? How does it differ from self-managed Kubernetes? Walk me through managed node groups, Fargate, IAM integration, VPC CNI, and how you would upgrade an EKS cluster in production."**

---

## Simple Explanation
Amazon EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes service. Think of it this way: if you set up Kubernetes yourself, you have to install it, patch it, back it up, and keep the control plane running — that is a lot of work. EKS takes away all that pain. AWS runs the control plane (the brain of Kubernetes) for you. You just worry about your worker nodes and applications.

It is like renting a fully managed apartment vs building your own house. You still decide how to furnish it (your apps, your configs), but someone else handles the foundation, plumbing, and roof (control plane, etcd, API server).

---

## Technical Explanation
Amazon EKS is a fully managed Kubernetes control plane service. AWS manages the Kubernetes API server, etcd cluster, and controller manager across multiple Availability Zones with automatic scaling and patching.

**Key architectural components:**

**Control Plane (AWS-Managed):**
- Kubernetes API Server — runs across 3 AZs
- etcd — distributed key-value store, multi-AZ with automatic backups
- Controller Manager — manages control loops
- Scheduler — assigns pods to nodes
- Cloud Controller Manager — integrates with AWS services

**Data Plane (Customer-Managed):**
- Worker nodes (EC2, Fargate, or Outposts)
- kubelet, kube-proxy on each node
- Container runtime (containerd)
- VPC CNI plugin for pod networking

**Three node deployment models:**
1. **Managed Node Groups** — AWS provisions, manages, and updates EC2 instances. You define the launch template, instance type, and scaling config. AWS handles cordoning/draining during upgrades.
2. **Self-Managed Node Groups** — You manage the EC2 instances yourself using Auto Scaling Groups. Full control but more operational overhead.
3. **AWS Fargate** — Serverless pods. No nodes to manage. Each pod runs in its own isolated micro-VM. Pay per pod CPU/memory.

---

## Real-World Example
**Production scenario at a fintech company:**

A payment processing platform runs on EKS with this architecture:
- **Control plane:** EKS 1.29, multi-AZ (us-east-1a, 1b, 1c)
- **Managed Node Groups:**
  - `general-workloads`: m5.xlarge, 10-50 nodes, cluster autoscaler enabled
  - `compute-intensive`: c5.2xlarge, 0-20 nodes, for batch processing
  - `memory-optimized`: r5.2xlarge, 2-10 nodes, for caching pods
- **Fargate Profile:** for isolated compliance workloads (PCI-DSS pods run on Fargate for stronger isolation)
- **IAM integration:** IRSA (IAM Roles for Service Accounts) — each microservice has its own IAM role with least privilege
- **Networking:** AWS VPC CNI, custom networking for pods in dedicated subnets
- **Add-ons:** CoreDNS, kube-proxy, VPC CNI, EBS CSI Driver, EFS CSI Driver all managed as EKS Add-ons

During a recent upgrade from EKS 1.28 to 1.29:
1. Upgraded control plane first via eksctl
2. Updated EKS Add-ons to compatible versions
3. Upgraded managed node groups one at a time using rolling update
4. Validated all workloads after each node group upgrade

---

## Diagram / Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD                                      │
│                                                                       │
│  ┌─────────────────── EKS CONTROL PLANE (AWS Managed) ─────────────┐ │
│  │                                                                   │ │
│  │   AZ-1a              AZ-1b              AZ-1c                    │ │
│  │  ┌──────────┐      ┌──────────┐      ┌──────────┐               │ │
│  │  │API Server│      │API Server│      │API Server│               │ │
│  │  └────┬─────┘      └────┬─────┘      └────┬─────┘               │ │
│  │       │                 │                  │                      │ │
│  │  ┌────▼─────┐      ┌────▼─────┐      ┌────▼─────┐               │ │
│  │  │  etcd    │◄────►│  etcd    │◄────►│  etcd    │               │ │
│  │  └──────────┘      └──────────┘      └──────────┘               │ │
│  │                                                                   │ │
│  │   Controller Manager │ Scheduler │ Cloud Controller Manager      │ │
│  └───────────────────────┬───────────────────────────────────────── ┘ │
│                           │ kubectl / API calls                        │
│  ┌──────── DATA PLANE ────▼─────────────────────────────────────────┐ │
│  │                                                                   │ │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────┐  │ │
│  │  │ Managed Node    │  │ Self-Managed     │  │    Fargate     │  │ │
│  │  │    Group        │  │   Node Group     │  │   Profile      │  │ │
│  │  │                 │  │                  │  │                │  │ │
│  │  │ ┌─────────────┐ │  │ ┌──────────────┐ │  │ ┌────────────┐ │  │ │
│  │  │ │  EC2: Node1 │ │  │ │  EC2: Node1  │ │  │ │  Pod (VM)  │ │  │ │
│  │  │ │  kubelet    │ │  │ │  kubelet     │ │  │ │  isolated  │ │  │ │
│  │  │ │  containerd │ │  │ │  containerd  │ │  │ └────────────┘ │  │ │
│  │  │ │  VPC CNI   │ │  │ │  VPC CNI    │ │  │ ┌────────────┐ │  │ │
│  │  │ └─────────────┘ │  │ └──────────────┘ │  │ │  Pod (VM)  │ │  │ │
│  │  │ ┌─────────────┐ │  │ ┌──────────────┐ │  │ │  isolated  │ │  │ │
│  │  │ │  EC2: Node2 │ │  │ │  EC2: Node2  │ │  │ └────────────┘ │  │ │
│  │  │ └─────────────┘ │  │ └──────────────┘ │  └────────────────┘  │ │
│  │  └─────────────────┘  └──────────────────┘                       │ │
│  │                                                                   │ │
│  │  ┌──────── AWS VPC ──────────────────────────────────────────┐   │ │
│  │  │  Pod CIDR: 10.0.0.0/8  (VPC CNI — real VPC IPs for pods) │   │ │
│  │  │  Node Subnet: 10.0.1.0/24  10.0.2.0/24  10.0.3.0/24      │   │ │
│  │  └───────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────── ┘ │
│                                                                       │
│  ┌─── AWS Services Integration ─────────────────────────────────────┐ │
│  │  IAM (IRSA)  │  ALB/NLB  │  EBS/EFS  │  ECR  │  CloudWatch     │ │
│  └───────────────────────────────────────────────────────────────── ┘ │
└─────────────────────────────────────────────────────────────────────┘

IRSA (IAM Roles for Service Accounts) Flow:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Pod (with Service Account)                              │
│       │                                                  │
│       ▼                                                  │
│  Projected Token (OIDC JWT)                              │
│       │                                                  │
│       ▼                                                  │
│  AWS STS AssumeRoleWithWebIdentity                       │
│       │                                                  │
│       ▼                                                  │
│  IAM Role (with OIDC trust policy)                       │
│       │                                                  │
│       ▼                                                  │
│  Temporary AWS Credentials (env vars in pod)             │
│       │                                                  │
│       ▼                                                  │
│  S3 / DynamoDB / SQS / etc.                              │
└──────────────────────────────────────────────────────────┘
```

---

## Why It Is Important
**Business value:**
- Reduces Kubernetes operational overhead by 60-70%
- AWS SLA covers control plane availability (99.95% uptime)
- Automatic security patches for control plane
- Native integration with 200+ AWS services

**Technical value:**
- Multi-AZ control plane without manual setup
- Automatic etcd backups
- VPC-native pod networking (no overlay network performance penalty)
- Integration with AWS Load Balancer Controller for ALB/NLB provisioning
- IRSA eliminates the need for long-lived AWS credentials in pods
- EKS Add-ons simplify lifecycle management of critical cluster components

---

## Common Interview Follow-Up Questions

1. **"What is the difference between managed node groups and Fargate, and when would you choose each?"**
2. **"How does IRSA work, and why is it better than using node instance profiles?"**
3. **"What is the VPC CNI plugin, and what problem does it solve? What is custom networking in EKS?"**
4. **"How do you upgrade an EKS cluster with zero downtime in production?"**
5. **"What are EKS Add-ons, and how do you manage their lifecycle?"**
6. **"How does the AWS Load Balancer Controller work, and how is it different from the classic in-tree controller?"**
7. **"What is EKS Anywhere, and when would you use it?"**
8. **"How do you implement autoscaling on EKS — Cluster Autoscaler vs Karpenter?"**

---

## Common Mistakes Candidates Make

**Mistake 1: "EKS manages everything for me, so I don't need to worry about node patches."**
- Correct answer: EKS manages the control plane. You are still responsible for worker node updates, OS patching, and AMI updates. Managed Node Groups automate the node upgrade process, but you must initiate it.

**Mistake 2: "I'll use node instance profiles to give pods access to AWS services."**
- Correct answer: Node instance profiles give ALL pods on a node the same permissions — a major security risk. IRSA (IAM Roles for Service Accounts) grants per-pod, least-privilege IAM permissions using OIDC federation.

**Mistake 3: "Fargate is always better because I don't manage nodes."**
- Correct answer: Fargate has limitations: no DaemonSets, no privileged containers, no GPU support, higher per-pod cost for sustained workloads, cold start latency. It is best for batch jobs, isolated workloads, or variable burst traffic.

**Mistake 4: "I can upgrade the control plane and nodes at the same time."**
- Correct answer: You must always upgrade the control plane first, then Add-ons, then node groups. Kubernetes supports n-2 skew between control plane and kubelet, but best practice is to keep them aligned.

**Mistake 5: Confusing VPC CNI with Flannel/Calico overlay networks.**
- Correct answer: VPC CNI assigns real VPC IP addresses to pods (no overlay). This means pods are directly routable within the VPC, enabling better performance and direct integration with AWS security groups and NACLs.

---

## Troubleshooting Scenario
**Problem:** Pods cannot access S3 — getting "Access Denied" errors after migration to EKS.

**Root Cause Investigation:**

```bash
# Step 1: Check if IRSA is configured
kubectl describe pod <pod-name> -n <namespace> | grep -A5 "AWS_"
# Look for: AWS_ROLE_ARN and AWS_WEB_IDENTITY_TOKEN_FILE

# Step 2: Check the service account annotation
kubectl describe sa <service-account-name> -n <namespace>
# Should show: eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/ROLE-NAME

# Step 3: Verify OIDC provider is set up
aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer"
aws iam list-open-id-connect-providers

# Step 4: Check IAM role trust policy
aws iam get-role --role-name <role-name> --query "Role.AssumeRolePolicyDocument"

# Step 5: Check the token file exists in the pod
kubectl exec -it <pod-name> -- ls /var/run/secrets/eks.amazonaws.com/serviceaccount/

# Step 6: Test AWS credentials inside pod
kubectl exec -it <pod-name> -- aws sts get-caller-identity
```

**Resolution:**
1. If no `AWS_ROLE_ARN` env var: annotate the service account with the IAM role ARN
2. If trust policy missing: add the OIDC condition to the IAM role trust policy
3. If OIDC provider missing: create it with `eksctl utils associate-iam-oidc-provider`
4. Restart the pod to pick up the new service account token

---

## kubectl Commands

```bash
# ── CLUSTER MANAGEMENT ────────────────────────────────────────────

# Create EKS cluster
eksctl create cluster \
  --name prod-cluster \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name general \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed

# Expected output:
# [✓]  EKS cluster "prod-cluster" in "us-east-1" region is ready

# List clusters
eksctl get cluster --region us-east-1

# Get cluster details
aws eks describe-cluster --name prod-cluster --region us-east-1

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name prod-cluster

# ── NODE GROUPS ───────────────────────────────────────────────────

# List node groups
eksctl get nodegroup --cluster prod-cluster

# Scale a node group
eksctl scale nodegroup \
  --cluster prod-cluster \
  --name general \
  --nodes 5 \
  --nodes-min 3 \
  --nodes-max 15

# Upgrade node group
eksctl upgrade nodegroup \
  --name general \
  --cluster prod-cluster \
  --kubernetes-version 1.29

# ── FARGATE ──────────────────────────────────────────────────────

# Create Fargate profile
eksctl create fargateprofile \
  --cluster prod-cluster \
  --name compliance-workloads \
  --namespace compliance \
  --labels env=pci

# List Fargate profiles
aws eks list-fargate-profiles --cluster-name prod-cluster

# ── IRSA ─────────────────────────────────────────────────────────

# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster prod-cluster \
  --approve

# Create IAM service account with S3 read access
eksctl create iamserviceaccount \
  --name s3-reader \
  --namespace production \
  --cluster prod-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve \
  --override-existing-serviceaccounts

# ── ADD-ONS ──────────────────────────────────────────────────────

# List add-ons
aws eks list-addons --cluster-name prod-cluster

# Expected output:
# {
#     "addons": [
#         "coredns",
#         "kube-proxy",
#         "vpc-cni",
#         "aws-ebs-csi-driver"
#     ]
# }

# Get add-on versions
aws eks describe-addon-versions --addon-name vpc-cni

# Update add-on
aws eks update-addon \
  --cluster-name prod-cluster \
  --addon-name vpc-cni \
  --addon-version v1.16.0-eksbuild.1

# ── CLUSTER UPGRADE ──────────────────────────────────────────────

# Step 1: Upgrade control plane
aws eks update-cluster-version \
  --name prod-cluster \
  --kubernetes-version 1.29

# Watch upgrade status
aws eks describe-cluster \
  --name prod-cluster \
  --query "cluster.status"
# Output: "UPDATING" → "ACTIVE"

# Step 2: Upgrade add-ons (after control plane)
eksctl utils update-coredns --cluster prod-cluster --approve
eksctl utils update-kube-proxy --cluster prod-cluster --approve
eksctl utils update-aws-node --cluster prod-cluster --approve

# Step 3: Upgrade node groups
eksctl upgrade nodegroup \
  --name general \
  --cluster prod-cluster \
  --kubernetes-version 1.29 \
  --force-upgrade

# ── NETWORKING ───────────────────────────────────────────────────

# Check VPC CNI version
kubectl describe daemonset aws-node -n kube-system | grep Image

# Check pod IPs (all from VPC CIDR — no overlay)
kubectl get pods -A -o wide

# Check ENIs attached to a node
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=<node-instance-id>"

# ── AUTOSCALING ──────────────────────────────────────────────────

# Check Cluster Autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50

# Check Karpenter provisioner
kubectl get provisioners
kubectl get nodeclaims
kubectl get nodepools   # Karpenter v0.33+

# ── LOAD BALANCER CONTROLLER ─────────────────────────────────────

# Check AWS Load Balancer Controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# View ALB ingress events
kubectl describe ingress <ingress-name> -n <namespace>

# Check controller logs
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller \
  --tail=100
```

---

## YAML Example

```yaml
# ── EKS CLUSTER CONFIG (eksctl) ─────────────────────────────────────
# File: cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-cluster
  region: us-east-1
  version: "1.29"
  tags:
    Environment: production
    Team: platform

# VPC configuration
vpc:
  cidr: 10.0.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true  # Enable for private node communication
  # Custom networking: pods in separate subnets
  subnets:
    private:
      us-east-1a:
        id: subnet-xxxxxx
        cidr: 10.0.1.0/24
      us-east-1b:
        id: subnet-yyyyyy
        cidr: 10.0.2.0/24
      us-east-1c:
        id: subnet-zzzzzz
        cidr: 10.0.3.0/24

# IAM OIDC provider
iam:
  withOIDC: true
  serviceAccounts:
    # Service account for AWS Load Balancer Controller
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
    # Service account for Cluster Autoscaler
    - metadata:
        name: cluster-autoscaler
        namespace: kube-system
      wellKnownPolicies:
        autoScaler: true
    # Service account for EBS CSI Driver
    - metadata:
        name: ebs-csi-controller-sa
        namespace: kube-system
      wellKnownPolicies:
        ebsCSIController: true
    # Custom service account for application
    - metadata:
        name: payment-service
        namespace: production
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonSQSFullAccess
        - arn:aws:iam::123456789012:policy/PaymentServiceCustomPolicy

# Managed node groups
managedNodeGroups:
  # General workloads node group
  - name: general
    instanceType: m5.xlarge
    minSize: 3
    desiredCapacity: 5
    maxSize: 20
    availabilityZones:
      - us-east-1a
      - us-east-1b
      - us-east-1c
    volumeSize: 100
    volumeType: gp3
    privateNetworking: true  # Nodes in private subnets
    labels:
      workload-type: general
      node-lifecycle: on-demand
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/prod-cluster: "owned"
    iam:
      withAddonPolicies:
        imageBuilder: true
        cloudWatch: true
        albIngress: true
    updateConfig:
      maxUnavailable: 1  # Only 1 node unavailable during rolling update

  # Compute-intensive node group
  - name: compute
    instanceType: c5.2xlarge
    minSize: 0
    desiredCapacity: 2
    maxSize: 15
    privateNetworking: true
    labels:
      workload-type: compute
    taints:
      - key: workload-type
        value: compute
        effect: NoSchedule

  # Spot instance node group for cost savings
  - name: spot-workers
    instanceTypes:
      - m5.xlarge
      - m5a.xlarge
      - m4.xlarge
    spot: true
    minSize: 0
    desiredCapacity: 3
    maxSize: 30
    privateNetworking: true
    labels:
      workload-type: spot
      node-lifecycle: spot
    taints:
      - key: node-lifecycle
        value: spot
        effect: NoSchedule

# Fargate profiles
fargateProfiles:
  - name: compliance-workloads
    selectors:
      - namespace: compliance
      - namespace: pci-dss
        labels:
          fargate: "true"

# EKS Add-ons
addons:
  - name: vpc-cni
    version: latest
    attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    configurationValues: |
      {
        "enableNetworkPolicy": "true"
      }
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true

# CloudWatch logging
cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler

---
# ── IRSA-ENABLED DEPLOYMENT ─────────────────────────────────────────
# File: payment-service-deployment.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service
  namespace: production
  annotations:
    # IRSA annotation — links service account to IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/payment-service-role
    # Optional: set token expiry (default 86400 = 24h)
    eks.amazonaws.com/token-expiration: "86400"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      # Use the IRSA-enabled service account
      serviceAccountName: payment-service
      
      # Tolerate spot nodes (for non-critical pods)
      tolerations:
        - key: node-lifecycle
          value: spot
          effect: NoSchedule
      
      # Prefer on-demand for payment service (critical)
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node-lifecycle
                    operator: In
                    values:
                      - on-demand
      
      containers:
        - name: payment-service
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payment-service:v1.5.0
          ports:
            - containerPort: 8080
          env:
            # AWS region needed by SDK
            - name: AWS_REGION
              value: us-east-1
            # These are automatically injected by EKS when IRSA is set up:
            # AWS_ROLE_ARN
            # AWS_WEB_IDENTITY_TOKEN_FILE
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi

---
# ── AWS LOAD BALANCER CONTROLLER INGRESS ─────────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-api-ingress
  namespace: production
  annotations:
    # Use AWS Load Balancer Controller (not nginx)
    kubernetes.io/ingress.class: alb
    # Create internet-facing ALB
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Route to pods directly (IP mode, better performance)
    alb.ingress.kubernetes.io/target-type: ip
    # SSL certificate
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc123
    # SSL policy
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    # Force HTTPS
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: |
      {"Type":"redirect","RedirectConfig":{"Protocol":"HTTPS","StatusCode":"HTTP_301"}}
    # Health check
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    # Access logs
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=my-alb-logs,
      access_logs.s3.prefix=payment-api
spec:
  rules:
    - host: api.payment.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80

---
# ── KARPENTER NODEPOOL (replaces Cluster Autoscaler) ─────────────────
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general
spec:
  template:
    metadata:
      labels:
        workload-type: general
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: general
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - on-demand
            - spot
        - key: kubernetes.io/arch
          operator: In
          values:
            - amd64
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m5.xlarge
            - m5.2xlarge
            - m5a.xlarge
            - m5a.2xlarge
      expireAfter: 720h   # Recycle nodes every 30 days
  limits:
    cpu: 1000             # Max 1000 CPU cores total
    memory: 1000Gi        # Max 1000Gi memory total
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s

---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: general
spec:
  amiFamily: AL2
  role: KarpenterNodeRole-prod-cluster
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: prod-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: prod-cluster
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
```

---

## AWS/EKS Perspective

**EKS-Specific Tools:**

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `eksctl` | CLI for creating/managing EKS clusters | Day-to-day cluster management |
| `aws eks` | AWS CLI for EKS operations | Automation, CI/CD |
| AWS Console | Visual cluster management | Debugging, visibility |
| AWS CDK / Terraform | Infrastructure as Code | Production cluster provisioning |
| Karpenter | Next-gen autoscaler built for EKS | New clusters (recommended over CA) |

**EKS Add-ons Lifecycle:**
```
EKS Add-on Versions: AWS tests and certifies specific versions
  │
  ├── vpc-cni:          v1.16.x  (mandatory, networking)
  ├── coredns:          v1.10.x  (mandatory, DNS)
  ├── kube-proxy:       v1.29.x  (mandatory, networking rules)
  ├── aws-ebs-csi:      v2.26.x  (storage)
  ├── aws-efs-csi:      v1.7.x   (shared storage)
  └── aws-guardduty:   v1.4.x   (security monitoring)
```

**EKS Upgrade Process (Zero Downtime):**
```
Phase 1: Control Plane
  aws eks update-cluster-version → Status: UPDATING → ACTIVE
  Duration: ~15-20 minutes
  
Phase 2: Add-Ons (must be compatible with new K8s version)
  Update vpc-cni → Update coredns → Update kube-proxy
  
Phase 3: Node Groups (rolling)
  For each node group:
    eksctl upgrade nodegroup (maxUnavailable: 1)
    Cordon node → Drain node → Terminate → Launch new node (new AMI)
    
Phase 4: Validate
  kubectl get nodes (all Ready, correct version)
  Run smoke tests
  Monitor error rates in CloudWatch
```

**Fargate vs Managed Nodes Decision Matrix:**

| Criteria | Managed Nodes | Fargate |
|----------|---------------|---------|
| DaemonSets | Yes | No |
| GPU workloads | Yes | No |
| Privileged containers | Yes | No |
| Cost (sustained) | Lower | Higher |
| Cost (burst/variable) | Higher | Lower |
| Startup time | Faster | Slower (cold start) |
| Isolation level | Shared VM | Dedicated micro-VM |
| Compliance (strong isolation) | Requires extra config | Built-in |
| Node management overhead | Low (managed) | Zero |

---

## Interview Answer (2-Minute Version)
"Amazon EKS is AWS's managed Kubernetes service. AWS runs the control plane — the API server, etcd, and schedulers — across multiple availability zones with automatic patching. I only manage the worker nodes and applications.

For worker nodes, EKS gives me three options: managed node groups where AWS handles node provisioning and updates; self-managed node groups for full control; and Fargate for serverless pods where there are no nodes to manage at all.

For security, I use IRSA — IAM Roles for Service Accounts — instead of giving the entire node IAM permissions. Each pod gets its own IAM role through OIDC federation, which follows least privilege.

For networking, EKS uses the VPC CNI plugin, which assigns real VPC IP addresses to pods — no overlay network, so better performance and direct VPC routing.

I manage cluster upgrades by first upgrading the control plane with `eksctl`, then the EKS Add-ons, then the node groups one at a time with rolling updates, so there is no downtime."

---

## Interview Answer (Senior Engineer Version)
"EKS is a managed Kubernetes control plane service. At the architecture level, AWS runs multi-AZ API server replicas backed by an etcd quorum — you get 99.95% SLA on the control plane without any operational burden. The cloud controller manager integrates natively with AWS APIs for load balancer provisioning and volume management.

For the data plane, I architect different node groups based on workload characteristics: on-demand instances for stateful and latency-sensitive workloads, spot instances for stateless batch jobs with tolerations and affinities to control placement, and Fargate for regulatory workloads requiring hypervisor-level isolation like PCI-DSS.

On security, IRSA is non-negotiable for production. It works through the EKS OIDC issuer endpoint — the kubelet injects a projected service account token into each pod, the AWS SDK exchanges this via STS AssumeRoleWithWebIdentity, and the pod gets temporary credentials scoped to exactly the IAM role it needs. No instance profiles, no shared credentials.

For autoscaling, I prefer Karpenter over Cluster Autoscaler in new clusters. Karpenter provisions nodes in under 60 seconds, supports diverse instance type consolidation, and handles bin packing more efficiently. It also supports automatic node drift detection and scheduled node recycling.

For upgrades, I follow the Kubernetes skew policy: control plane first, then add-ons, then nodes. I always run `kubectl get nodes --watch` and monitor CloudWatch Container Insights error rates throughout. For critical clusters, I test the upgrade path on staging first and use PodDisruptionBudgets to protect critical workloads during node drains."

---

## What Impresses the Interviewer
- Mentioning IRSA vs instance profiles and explaining WHY IRSA is superior from a security perspective
- Knowing the EKS upgrade sequence (control plane → add-ons → nodes) and the n-2 skew policy
- Discussing Karpenter vs Cluster Autoscaler with specific differences
- Understanding VPC CNI custom networking for when you are running out of IP addresses
- Mentioning PodDisruptionBudgets during node drain operations
- Knowing specific eksctl and aws cli commands from memory
- Discussing EKS Add-ons lifecycle and version compatibility
- Bringing up CloudWatch Container Insights for observability

---

## Red Flags
- "EKS manages everything, nodes included" — shows lack of understanding of the shared responsibility model
- Cannot explain IRSA and defaults to "I use environment variables for AWS credentials"
- Does not know the upgrade sequence
- Says "I would just upgrade the cluster in the console" without mentioning rolling strategies
- Cannot name any EKS-specific tools (eksctl, aws-load-balancer-controller, Karpenter)
- Confuses EKS with ECS (very common!)
- Does not know what Fargate limitations are

---

## Production Best Practices

1. **Use private endpoint access for EKS API server** — disable public endpoint or restrict to VPN/bastion CIDR ranges to prevent internet-exposed control plane.

2. **Enable EKS control plane logging** — enable API, audit, authenticator, controllerManager, and scheduler logs to CloudWatch for security audit trails and debugging.

3. **Use IRSA for all AWS service access** — never use node instance profiles for application-level AWS access. Create per-service-account IAM roles with minimum required permissions.

4. **Use managed node groups with custom AMIs (EKS-optimized)** — stay on EKS-optimized AMIs and use launch templates to customize only what is needed (disk size, userdata). This ensures compatibility with managed upgrades.

5. **Implement PodDisruptionBudgets for all production workloads** — ensures that node draining during upgrades or Karpenter consolidation never takes down more pods than your redundancy allows.

6. **Use Karpenter with instance type diversification** — configure multiple instance types and families. This reduces spot interruption risk and improves cost optimization.

7. **Separate node groups by workload type** — use different node groups for system workloads, application workloads, and batch workloads with appropriate taints/tolerations and instance types.

8. **Enable AWS GuardDuty EKS protection** — monitors EKS audit logs and detects threats like crypto mining, privilege escalation attempts, and suspicious API calls.

---

## Key Points to Remember
- EKS manages the control plane (API server, etcd, scheduler) across 3 AZs; you manage worker nodes
- Three node types: Managed Node Groups, Self-Managed, Fargate — each with different tradeoffs
- IRSA = per-pod IAM via OIDC — always preferred over node instance profiles
- VPC CNI = real VPC IPs for pods — no overlay, better performance, but uses VPC IP space
- EKS Add-ons manage vpc-cni, coredns, kube-proxy, ebs-csi lifecycle
- Upgrade order: control plane → add-ons → node groups (never the reverse)
- Karpenter is the modern autoscaler for EKS (Cluster Autoscaler is legacy)
- AWS Load Balancer Controller replaces the in-tree cloud provider for ALB/NLB
- eksctl is the community CLI for EKS cluster management
- Fargate: no DaemonSets, no GPU, no privileged containers — serverless pods only

---

## Interviewer's Expectation
The interviewer is testing whether you:
1. Understand the **shared responsibility model** — what AWS manages vs what you manage
2. Can explain **IRSA** and why it matters for production security
3. Know the **upgrade process** and can perform it without downtime
4. Understand **networking** at the VPC CNI level
5. Can make **architectural decisions** — when to use managed nodes vs Fargate vs spot
6. Are familiar with **EKS-specific tooling** (eksctl, Karpenter, ALB Controller)
7. Have **production experience** — mentioning PDBs, monitoring, rollback strategies

---

## Final Perfect Interview Answer
"Amazon EKS is AWS's managed Kubernetes service where AWS takes full responsibility for the control plane — running the API server, etcd, and scheduler across three availability zones with automatic patching and a 99.95% SLA. My team focuses entirely on workloads rather than control plane operations.

For worker nodes, I architect multiple managed node groups based on workload needs: on-demand for stateful and critical services, spot instances for stateless batch jobs, and Fargate for compliance workloads requiring hypervisor-level isolation. I use Karpenter for autoscaling because it provisions nodes in under a minute, supports instance type diversification, and automatically consolidates underutilized nodes.

For security, I use IRSA — IAM Roles for Service Accounts — exclusively. It leverages the EKS OIDC endpoint to give each pod its own scoped IAM role via STS token exchange, eliminating shared node-level credentials.

For upgrades in production, I follow the strict sequence: control plane first, then EKS Add-ons, then node groups one at a time with maxUnavailable set to one. PodDisruptionBudgets protect critical workloads during node drains, and I monitor CloudWatch Container Insights throughout for any error rate spikes."
