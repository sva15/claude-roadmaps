# Ingress and Ingress Controllers

> 🔹 **Phase:** 4 — Kubernetes Networking  
> 🔹 **Topic:** Ingress Controllers  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

LoadBalancer Services create one load balancer per service — expensive and unmanageable at scale. **Ingress** consolidates multiple services behind a single load balancer with HTTP routing rules (host-based, path-based).

```
Without Ingress (3 LoadBalancer Services = 3 NLBs):
  api.example.com → NLB-1 → api service
  web.example.com → NLB-2 → web service
  admin.example.com → NLB-3 → admin service
  Cost: $32/month × 3 = $96/month minimum

With Ingress (1 ALB):
  api.example.com → ALB → api service
  web.example.com → ALB → web service
  admin.example.com → ALB → admin service
  Cost: $32/month × 1 = $32/month
```

---

## 2. Core Concept (Deep Dive)

### Ingress Resource vs Ingress Controller

```
Ingress (K8s resource):
  → Declarative routing rules (YAML)
  → "I want api.example.com/v1 to route to service-a"
  → Does NOTHING by itself

Ingress Controller (actual software):
  → Watches Ingress resources
  → Configures a real load balancer/proxy
  → Makes the rules work

Without a controller, Ingress resources are ignored!
```

### Popular Ingress Controllers

| Controller | Proxy | Best For |
|-----------|-------|----------|
| **AWS Load Balancer Controller** | ALB | EKS (creates real AWS ALBs) |
| **NGINX Ingress Controller** | NGINX | Self-managed, flexible, widely used |
| **Traefik** | Traefik | Auto-discovery, Let's Encrypt |
| **Istio Gateway** | Envoy | Service mesh integrated |
| **Kong** | Kong | API gateway features |

### AWS Load Balancer Controller — EKS Standard

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123:certificate/abc
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/group.name: shared-alb    # Share ALB across Ingresses
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port: { number: 8080 }
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port: { number: 8080 }
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port: { number: 3000 }
```

**Key annotations:**
| Annotation | Purpose |
|-----------|---------|
| `target-type: ip` | Route directly to pod IPs (no NodePort hop) |
| `scheme: internet-facing` | Public ALB (vs `internal`) |
| `group.name` | Share one ALB across multiple Ingress resources |
| `certificate-arn` | ACM TLS certificate |
| `ssl-redirect: "443"` | Redirect HTTP → HTTPS |
| `healthcheck-path` | ALB health check endpoint |

### Ingress vs Gateway API

```
Ingress (current, stable):
  → Simple host/path routing
  → Limited by annotation-based config
  → Each controller has different annotations

Gateway API (future, graduating):
  → Richer routing (header matching, traffic splitting)
  → Standardized across controllers (no annotation mess)
  → Role-based: infra team manages Gateway, devs manage Routes
  → Will eventually replace Ingress
```

### AWS Load Balancer Controller — IRSA Setup (Critical)

The AWS LB Controller needs IAM permissions to create ALBs, Target Groups, and register targets. Without IRSA (IAM Roles for Service Accounts), ALBs silently fail to create.

**Step 1: Create the IAM Policy**
```bash
# Download the official IAM policy
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.0/docs/install/iam_policy.json

# Create the IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

**Step 2: Create IRSA (IAM Role + K8s Service Account)**
```bash
# Using eksctl (simplest)
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# This creates:
# 1. IAM Role with trust policy for the EKS OIDC provider
# 2. K8s ServiceAccount annotated with the IAM role ARN
```

**Step 3: Install the Controller**
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**Step 4: Verify It's Working**
```bash
# Check controller logs for AWS API calls
kubectl logs -n kube-system deployment/aws-load-balancer-controller -f

# Look for:
# ✅ "successfully built model" → controller is processing Ingress
# ❌ "AccessDenied" → IRSA misconfigured
# ❌ "failed to build model" → annotation or spec error

# Verify service account has the IAM role annotation
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
# Should show: eks.amazonaws.com/role-arn: arn:aws:iam::...:role/...
```

---

## 3. Failure Scenarios

### Scenario 1: ALB Created But Returns 404

| Attribute | Detail |
|-----------|--------|
| **Symptom** | ALB responds to all paths with 404 |
| **Root Cause** | Ingress path rules don't match request path, or no default backend |
| **Fix** | Verify path rules. Use `pathType: Prefix`. Add a default backend. |

### Scenario 2: ALB Not Created At All

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Ingress resource exists but no ALB appears in AWS |
| **Root Cause** | AWS LB Controller not installed, missing IAM permissions, or wrong ingress class |
| **Debug** | `kubectl logs -n kube-system deployment/aws-load-balancer-controller`. Check IRSA (IAM Role for Service Account). |

### Scenario 3: Multiple ALBs Instead of Shared

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Each Ingress resource creates a separate ALB |
| **Root Cause** | Missing `group.name` annotation. Each Ingress gets its own ALB. |
| **Fix** | Add same `alb.ingress.kubernetes.io/group.name` to all related Ingresses. |

---

## 4. Interview Preparation

### Q1: Why use Ingress instead of LoadBalancer Services?

**Answer:** Cost and management. Each LoadBalancer Service creates a cloud LB ($30+/month). With 20 services, that's $600/month. Ingress consolidates behind one ALB with path/host routing. Also: single point for TLS termination, WAF, access logs.

### Q2: How does the AWS Load Balancer Controller work?

**Answer:** It watches K8s Ingress resources and creates/configures ALBs. With target-type `ip`, it registers pod IPs directly as ALB targets (no NodePort). It handles scaling (registers/deregisters pods as they scale), health checks, TLS via ACM, and can share a single ALB across multiple Ingresses via `group.name`.

---

## 5. Cheat Sheet

```
Ingress = rules (YAML). Controller = implementation (software).
AWS LB Controller: creates ALBs from Ingress resources
target-type: ip → direct to pods (skip NodePort)
group.name → share ALB across Ingresses (cost saving)
Gateway API → future replacement for Ingress
```

---

## 🔗 Connections

### Previous: Services (4.3) create internal routing. Ingress adds external L7 routing.
### Next: Topic 4.5 — NetworkPolicy defines what traffic is allowed between pods.
