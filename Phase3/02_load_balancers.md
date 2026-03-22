# Load Balancers — ALB vs NLB and When to Use Each

> 🔹 **Phase:** 3 — Cloud Networking / AWS  
> 🔹 **Topic:** Elastic Load Balancing (ALB and NLB)  
> 🔹 **Priority:** HIGH

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

One server can't handle all traffic. When you have multiple servers (pods, instances), you need something to distribute incoming requests across them — and stop sending traffic to servers that are broken.

**A load balancer (LB)** sits between clients and your servers, distributing traffic while checking health.

But AWS has two very different load balancers, and choosing wrong causes real production problems.

### The Analogy

**ALB = Smart receptionist.** Reads your request, understands which department you need, routes you there, logs your visit.

**NLB = High-speed revolving door.** Pushes you through to the next available room based on your destination number. Doesn't read anything, doesn't ask questions, incredibly fast.

> 💡 **Decision rule:** If your traffic speaks HTTP → ALB. If it doesn't, or you need static IPs or extreme performance → NLB.

---

## 2. Core Concept (Deep Dive)

### ALB (Application Load Balancer) — Layer 7

```
What ALB can do:
├── Read HTTP headers, method, path, query strings
├── Route by host: api.example.com → TG-A, www.example.com → TG-B
├── Route by path: /api/* → TG-API, /static/* → TG-CDN
├── Terminate TLS (ACM certificates, SNI for multiple certs)
├── Add headers: X-Forwarded-For, X-Forwarded-Proto
├── WebSocket upgrade support
├── gRPC support (HTTP/2)
├── WAF integration (rate limiting, SQL injection blocking)
├── HTTP/2 to clients, HTTP/1.1 or gRPC to backends
├── Sticky sessions (cookie-based)
├── Redirect actions (HTTP → HTTPS)
├── Fixed response actions (maintenance page)
└── Access logs (every request logged to S3)

What ALB CANNOT do:
├── Static IP addresses (IPs change dynamically)
├── Non-HTTP protocols (TCP, UDP, database protocols)
├── Preserve client source IP at L4 (uses X-Forwarded-For header)
└── Ultra-low latency (<1ms overhead added)
```

### NLB (Network Load Balancer) — Layer 4

```
What NLB can do:
├── Route TCP/UDP/TLS traffic by IP + port
├── Static IP addresses (one Elastic IP per AZ)
├── Preserve client source IP (no SNAT by default)
├── Handle millions of connections per second
├── Ultra-low latency (~100μs added)
├── TLS termination (optional) with ACM
├── TLS passthrough (let backend handle TLS)
├── TCP health checks
├── UDP support (DNS, QUIC, gaming)
└── Cross-zone load balancing (optional)

What NLB CANNOT do:
├── Read HTTP content (no path/host-based routing)
├── Add X-Forwarded-For headers
├── WAF integration
├── Sticky sessions by cookie
├── HTTP → HTTPS redirects
└── WebSocket-aware routing
```

### Complete Comparison

| Feature | ALB | NLB |
|---------|-----|-----|
| **OSI Layer** | L7 (HTTP/HTTPS/gRPC) | L4 (TCP/UDP/TLS) |
| **Routing** | Host, path, header, query, method | IP + port only |
| **Static IPs** | ❌ (IPs change) | ✅ (Elastic IPs) |
| **Source IP** | Lost (use X-Forwarded-For) | Preserved |
| **TLS termination** | ✅ (ACM, SNI) | ✅ (ACM, optional) |
| **TLS passthrough** | ❌ | ✅ |
| **Performance** | Good (~ms added) | Excellent (~μs added) |
| **Throughput** | Millions req/s | Millions conn/s |
| **gRPC** | ✅ | Via TLS passthrough |
| **WebSocket** | ✅ | ✅ (TCP passthrough) |
| **WAF** | ✅ | ❌ |
| **Access logs** | HTTP-level (S3) | Flow logs |
| **Health checks** | HTTP/HTTPS | TCP/HTTP/HTTPS |
| **Idle timeout** | 60s (configurable) | 350s (TCP, not configurable) |
| **Cost** | ~$0.0225/hr + LCU | ~$0.0225/hr + NLCU |

### Target Groups and Health Checks

```
ALB Listener (port 443, HTTPS)
├── Rule 1: Host: api.* → Target Group API
│   ├── Pod 10.244.1.5:8080 (healthy ✅)
│   ├── Pod 10.244.1.6:8080 (healthy ✅)
│   └── Pod 10.244.1.7:8080 (unhealthy ❌ → no traffic)
│
├── Rule 2: Path: /admin/* → Target Group Admin
│   └── Pod 10.244.2.10:3000 (healthy ✅)
│
└── Default: → Target Group Web
    ├── Pod 10.244.3.1:80 (healthy ✅)
    └── Pod 10.244.3.2:80 (healthy ✅)

Health Check:
  Protocol: HTTP
  Path: /health
  Interval: 15s
  Healthy threshold: 2 (2 consecutive 200s → mark healthy)
  Unhealthy threshold: 3 (3 consecutive failures → mark unhealthy)
```

### Target Types

| Type | What It Is | When to Use |
|------|-----------|-------------|
| **Instance** | EC2 instance ID | Traditional EC2 deployments |
| **IP** | Direct IP address | EKS pods (VPC CNI), cross-VPC targets, on-premises (via Direct Connect) |
| **Lambda** | Lambda function ARN | Serverless backends |

> For EKS: use **IP target type** — ALB sends traffic directly to pod IPs. The AWS Load Balancer Controller creates target groups with pod IP registration.

---

## 3. Internal Working (Under the Hood)

### ALB Request Processing

```
1. Client DNS lookup → Route53 returns ALB IPs (multiple, per AZ)
2. Client TCP connect to ALB IP
3. ALB terminates TLS (using ACM cert, selected by SNI)
4. ALB reads HTTP request:
   a. Extract Host header → match listener rule
   b. Extract path → match listener rule
   c. First matching rule wins (by priority)
5. ALB selects target from target group:
   a. Round-robin (default) or least-outstanding-requests
   b. Only healthy targets
   c. Sticky session? → use cookie to route to same target
6. ALB opens NEW TCP connection to target:
   a. Adds X-Forwarded-For: <client-ip>
   b. Adds X-Forwarded-Proto: https
7. Forwards request → waits for response → returns to client
```

**Key detail:** ALB creates a NEW TCP connection to the backend. This is why the backend sees the ALB's IP as the source, not the client's IP. And why `keepalive_timeout` on the backend must be > ALB idle timeout (60s).

### ALB + EKS Integration (AWS Load Balancer Controller)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/group.name: shared-alb
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port: { number: 8080 }
```

The AWS LB Controller watches Ingress resources → creates/updates ALB → registers pod IPs in target groups → handles pod scaling (register/deregister).

---

## 4. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: 502 Bad Gateway — Backend Keepalive

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Intermittent HTTP 502 from ALB |
| **Root Cause** | Backend closes the TCP connection (keepalive timeout) before ALB does. ALB tries to reuse the closed connection → 502. |
| **Fix** | Backend keepalive > ALB idle timeout. ALB default: 60s → backend: 65s+. |
| **Evidence** | ALB access log: `target_status_code = -`, error_reason = `ConnectionTerminated` |

### Scenario 2: 504 Gateway Timeout

| Attribute | Detail |
|-----------|--------|
| **Symptom** | HTTP 504, `target_processing_time` near ALB timeout |
| **Root Cause** | Backend took too long to respond |
| **Fix** | Optimize backend, increase ALB idle timeout, or use async processing (return 202) |

### Scenario 3: Uneven Pod Distribution (NLB)

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Some pods receive much more traffic than others |
| **Root Cause** | NLB cross-zone load balancing is OFF by default. All traffic to an AZ goes to pods in that AZ only. |
| **Fix** | Enable cross-zone load balancing on the NLB target group. Note: this incurs cross-AZ data transfer charges. |

### Scenario 4: NLB Health Check Failing but Service Works

| Attribute | Detail |
|-----------|--------|
| **Symptom** | NLB marks targets as unhealthy. Direct curl to pod IP works. |
| **Root Cause** | NLB health checks come from the NLB node IPs (not from the VPC CIDR you'd expect). Security Group on pods must allow the VPC CIDR or NLB's IPs. |
| **Fix** | Allow health check traffic from the VPC CIDR (e.g., 10.0.0.0/16) in the pod's SG. |

---

## 5. Debugging Playbook

```
LB Issues:

ALB 502:
  1. Check ALB access logs (S3) → target_status_code, error_reason
  2. Direct curl to target IP → does it respond?
  3. Check keepalive settings → backend > ALB idle timeout?
  4. Check target group health → unhealthy targets?

ALB 503:
  1. All targets unhealthy → no healthy targets to route to
  2. Check health check path → returns 200?
  3. Check SG → health check port allowed?

ALB 504:
  1. Check target_processing_time → near idle timeout?
  2. Optimize backend or increase timeout
  
NLB targets unhealthy:
  1. Check SG allows VPC CIDR for health checks
  2. Check health check port matches what app listens on
  3. NLB preserves source IP → backend SG must allow NLB client IPs
```

---

## 6. Interview Preparation

### Q1: ALB vs NLB — when do you choose each?

**Answer:** ALB: HTTP/HTTPS traffic, path/host-based routing, gRPC, WAF, WebSockets. NLB: non-HTTP (databases, Kafka), need static IPs (partner whitelisting), extreme performance, source IP preservation at L4, TCP/UDP passthrough.

### Q2: How do you achieve zero-downtime deploys behind an ALB?

**Answer:** Rolling update with readiness probes. New pod starts → passes readiness → registered in target group (healthy threshold: 2 checks). Old pod: removed from target group → deregistration delay (default 300s) → existing connections finish → pod terminated. Key: set deregistration delay appropriately and ensure health checks are configured correctly.

### Q3: Your EKS Ingress creates an ALB but health checks fail. Debug steps?

**Answer:** 1. Check target group health in AWS console → what's the error? 2. Verify `healthcheck-path` annotation matches an endpoint that returns 200. 3. Check pod SG allows traffic from ALB SG (or VPC CIDR). 4. Check ALB target type is `ip` (not `instance`) for EKS with VPC CNI. 5. Verify AWS Load Balancer Controller logs: `kubectl logs -n kube-system deployment/aws-load-balancer-controller`.

### Q4: ALB returns 502 intermittently. Root cause?

**Answer:** Most likely backend keepalive timeout < ALB idle timeout. ALB thinks the connection is still alive, sends a request, backend already closed it → 502. Fix: nginx keepalive_timeout 65; (> ALB's 60s). Also check: backend crashing, protocol mismatch (HTTPS target but HTTP backend), target group draining.

---

## 7. Cheat Sheet

### Decision Matrix

```
HTTP routing?      → ALB
gRPC?              → ALB (HTTPS + gRPC target group)
Static IPs?        → NLB (with Elastic IPs)
Source IP needed?   → NLB (or ALB with X-Forwarded-For header)
TCP passthrough?   → NLB
WAF needed?        → ALB
WebSockets?        → ALB
Database proxy?    → NLB
Extreme perf?      → NLB
```

### Critical Settings

```
ALB idle timeout:           60s (configurable)
Backend keepalive:          MUST be > ALB idle timeout
Deregistration delay:       300s default (lower for faster deploys)
Health check interval:      15-30s
Healthy threshold:          2-3 checks
Unhealthy threshold:        2-3 checks
NLB cross-zone:             OFF by default (enable for even distribution)
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**HTTP (Topic 1.6):** Expanded deep dive into ALB HTTP routing, status codes (502/503/504), and the idle timeout trap first introduced in Phase 1.

**VPC (Topic 3.1):** Load balancers live in VPC subnets. ALBs need public subnets with IGW route. Target groups reference private subnet pods via IP or instance targets.

### What This Prepares You For Next
**Next topic: VPC Peering, Transit Gateway, and PrivateLink** — connecting multiple VPCs together, which is essential for multi-account architectures and accessing shared services.
