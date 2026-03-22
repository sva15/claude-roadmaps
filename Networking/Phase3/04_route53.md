# Route53 — DNS for Production Architectures

> 🔹 **Phase:** 3 — Cloud Networking / AWS  
> 🔹 **Topic:** Route53 Production DNS  
> 🔹 **Priority:** HIGH

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

You learned DNS fundamentals in Phase 1 (Topic 1.4). Route53 is AWS's managed DNS service that brings DNS into your infrastructure toolbox — not just name resolution, but health-checking, failover, traffic routing, and multi-region architectures.

Route53 does three things:
1. **Domain registration** — buy and manage domain names
2. **DNS hosting** — host authoritative DNS zones (hosted zones)
3. **Health checking** — monitor endpoints and route around failures

> 💡 **Key Insight:** Route53 is not just DNS. It's your first line of defense for high availability. Before a single packet reaches your ALB, Route53's routing policies and health checks decide WHERE to send traffic.

---

## 2. Core Concept (Deep Dive)

### Hosted Zones

```
Public Hosted Zone:
  → Resolves names on the PUBLIC internet
  → example.com → hosted zone with NS records at AWS name servers
  → Anyone on the internet can query it

Private Hosted Zone:
  → Resolves names ONLY within associated VPCs
  → internal.example.com → only resolvable from within VPC
  → Used for internal service discovery
  → Associate with one or more VPCs (even cross-account)
```

### Record Types for DevOps

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Name → IPv4 | api.example.com → 1.2.3.4 |
| **AAAA** | Name → IPv6 | api.example.com → 2001:db8::1 |
| **CNAME** | Name → another name | www.example.com → d123.cloudfront.net |
| **ALIAS** | Name → AWS resource (Route53 special) | example.com → ALB DNS name |
| **MX** | Mail server | example.com MX → mail.example.com |
| **TXT** | Text data | SPF, DKIM, domain verification |
| **NS** | Name server delegation | example.com NS → ns-xxx.awsdns-xx.com |
| **SRV** | Service locator | _sip._tcp.example.com → server:port |

### ALIAS vs CNAME — Critical Difference

```
CNAME:
  ✗ Cannot be used at zone apex (example.com)
  ✓ Can point to any DNS name
  ✓ Standard DNS

ALIAS (Route53 extension):
  ✓ CAN be used at zone apex (example.com → ALB)
  ✓ Free (no query charges for ALIAS to AWS resources)
  ✓ Returns the actual IP (faster — no extra DNS hop)
  ✓ Works with: ALB, NLB, CloudFront, S3, Elastic Beanstalk, API Gateway

ALWAYS use ALIAS for AWS resources. Use CNAME only for non-AWS targets.
```

### Routing Policies

| Policy | How It Works | Use Case |
|--------|-------------|----------|
| **Simple** | Returns all values; client picks one | Basic single-resource |
| **Weighted** | Distribute by weight (70/30 split) | Canary deploys, A/B testing |
| **Latency** | Route to region with lowest latency to user | Multi-region active-active |
| **Failover** | Primary → if unhealthy → Secondary | Active-passive DR |
| **Geolocation** | Route by user's continent/country | Compliance (EU users → EU region) |
| **Geoproximity** | Route by proximity with bias | Shift traffic between regions |
| **Multivalue** | Returns up to 8 healthy records | Simple load balancing with health checks |

### Health Checks

```
Route53 Health Check:
├── Endpoint checks (HTTP/HTTPS/TCP)
│   → Monitors: IP + port + optional path
│   → Healthy: HTTP 2xx/3xx within timeout
│   → Check interval: 10 or 30 seconds
│   → Must pass from multiple AWS regions
│
├── Calculated checks
│   → Combines multiple health checks
│   → "Healthy if 2 out of 3 children are healthy"
│
└── CloudWatch alarm checks
    → Healthy if CloudWatch alarm is OK
    → Monitor ANY metric (CPU, errors, custom)
```

**Failover example:**
```
api.example.com (Failover policy):
  Primary: ALB in us-east-1 (health check: /health → 200)
  Secondary: Static S3 page "We're experiencing issues"

If primary health check FAILS:
  → Route53 stops returning primary IP
  → Returns secondary (S3 maintenance page)
  → Failover happens in ~60-90 seconds
```

---

## 3. Real-World Patterns

### Pattern 1: Multi-Region Active-Active

```
api.example.com (Latency routing):
  us-east-1: ALB → EKS cluster (health check on)
  eu-west-1: ALB → EKS cluster (health check on)
  ap-sout-1: ALB → EKS cluster (health check on)

User in Europe → Route53 returns eu-west-1 ALB (lowest latency)
If eu-west-1 unhealthy → Route53 removes it, routes to next best
```

### Pattern 2: Blue/Green with Weighted Routing

```
api.example.com (Weighted):
  Blue (v1): ALB-blue  weight: 90
  Green (v2): ALB-green weight: 10

Gradually shift: 90/10 → 70/30 → 50/50 → 0/100
If issues with Green → set weight to 0 → instant rollback
```

### Pattern 3: Private DNS for Internal Services

```
Private Hosted Zone: internal.example.com
  Associated with: VPC-Prod, VPC-Stage

Records:
  db.internal.example.com    → RDS endpoint
  cache.internal.example.com → ElastiCache endpoint
  api.internal.example.com   → Internal ALB

Apps use DNS names, not hardcoded IPs.
Infrastructure changes → update DNS record → zero app changes.
```

---

## 4. Failure Scenarios

### Scenario 1: Health Check Fails Due to Security Group

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Route53 health check shows "unhealthy" but the endpoint works fine |
| **Root Cause** | Route53 health checkers come from public IPs. SG must allow inbound from Route53 IP ranges. |
| **Fix** | Allow Route53 health checker IPs (published AWS IP ranges) in SG. Or use a CloudWatch alarm-based health check (no inbound access needed). |

### Scenario 2: CNAME at Zone Apex

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Can't create CNAME for example.com (bare domain) |
| **Root Cause** | DNS spec prohibits CNAME at zone apex |
| **Fix** | Use ALIAS record instead (Route53 extension) |

### Scenario 3: TTL Too High During Migration

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Changed DNS record but some users still reach old IP hours later |
| **Root Cause** | Previous TTL was 86400 (24 hours). Resolvers cached the old value. |
| **Fix** | Before migration: lower TTL to 60s. Wait for old TTL to expire. Make the change. After stable: raise TTL back. |

---

## 5. Interview Preparation

### Q1: How would you implement multi-region failover with Route53?

**Answer:** Failover routing policy. Primary record → ALB in primary region with health check (HTTP check to /health, threshold 3, interval 10s). Secondary record → ALB in DR region (or S3 static site). If primary fails health check, Route53 returns secondary in ~60-90s. For active-active: latency-based routing with health checks on all regions.

### Q2: ALIAS vs CNAME — when do you use each?

**Answer:** ALIAS for AWS resources always — free queries, works at zone apex, faster (returns IP directly). CNAME only for non-AWS resources. Critical: CANNOT use CNAME at `example.com` (zone apex), must use ALIAS.

### Q3: How do you do zero-downtime DNS migration?

**Answer:** 1. Lower TTL to 60s (wait for old TTL to expire). 2. Create new records. 3. Switch the record to new target. 4. Monitor for 24-48h. 5. Raise TTL back to 3600s. The low TTL ensures resolvers refresh quickly.

---

## 6. Cheat Sheet

```
ALIAS (not CNAME) for AWS resources — free, works at apex
Failover: primary + secondary with health check
Latency: multi-region active-active
Weighted: canary deploys (90/10 → 0/100)
Health check: HTTP/HTTPS/TCP from multiple regions
TTL: lower before changes, raise after stable
Private Hosted Zone: internal DNS within VPCs (cross-account OK)
```

---

## 🔗 Connections

### Previous: DNS fundamentals (Topic 1.4) covered resolution mechanics. Route53 is the operational implementation for production architectures.

### Next: Phase 4 — Kubernetes Networking — where CoreDNS, K8s Services, and ingress controllers tie back into DNS and load balancing.
