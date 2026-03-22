# VPC Peering, Transit Gateway, and PrivateLink

> 🔹 **Phase:** 3 — Cloud Networking / AWS  
> 🔹 **Topic:** VPC Connectivity Options  
> 🔹 **Priority:** HIGH

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

A VPC is isolated by design. But real architectures need VPCs to talk to each other: production VPC to shared services VPC, your VPC to a partner's VPC, your VPC to on-premises.

AWS provides three fundamentally different ways to connect VPCs:

| Method | Analogy | Use Case |
|--------|---------|----------|
| **VPC Peering** | A private bridge between two buildings | Simple 1:1 VPC connection |
| **Transit Gateway** | A central bus terminal connecting all buildings | Hub-and-spoke for many VPCs |
| **PrivateLink** | A private delivery chute for one specific service | Expose a service without network merging |

### Why Three Options?

Each solves different problems:
- **Peering:** Cheap, simple, but doesn't scale (N VPCs = N×(N-1)/2 peerings)
- **Transit Gateway:** Centralized, scalable, but more expensive
- **PrivateLink:** Service-level exposure without route table changes or CIDR overlap concerns

---

## 2. Core Concept (Deep Dive)

### VPC Peering

```
VPC-A (10.0.0.0/16)  ◄══ Peering Connection ══►  VPC-B (10.1.0.0/16)

Requirements:
  ✅ Non-overlapping CIDRs
  ✅ Route table entries in BOTH VPCs
  ✅ Security Groups allow peered traffic
  ✅ NACLs allow peered traffic
  ❌ NO transitive routing (A↔B and B↔C does NOT mean A↔C)
```

**Route table entries (BOTH sides):**
```
VPC-A route table:
  10.0.0.0/16 → local
  10.1.0.0/16 → pcx-xxxxx    ← Route to VPC-B via peering

VPC-B route table:
  10.1.0.0/16 → local
  10.0.0.0/16 → pcx-xxxxx    ← Route to VPC-A via peering
```

**Key facts:**
- Free (no hourly charge, only data transfer)
- Works cross-account and cross-region
- No bandwidth bottleneck (AWS backbone)
- No transitive routing — each pair needs its own peering
- CIDRs CANNOT overlap

### Transit Gateway (TGW)

```
                    ┌─────────────────┐
                    │ Transit Gateway  │
                    │   (Hub)          │
                    └───┬──┬──┬──┬────┘
                        │  │  │  │
              ┌─────────┘  │  │  └─────────┐
              │            │  │            │
        ┌─────▼────┐ ┌────▼──▼───┐ ┌──────▼─────┐
        │ VPC-Prod  │ │ VPC-Stage │ │ VPC-Shared  │
        │10.0.0/16  │ │10.1.0/16  │ │10.2.0/16   │
        └──────────┘ └──────────┘ └─────────────┘
                                         │
                                    ┌────▼──────┐
                                    │ On-Prem    │
                                    │ (VPN/DX)   │
                                    └───────────┘
```

**Key facts:**
- Hub-and-spoke model — one TGW connects ALL VPCs
- **Supports transitive routing** (unlike peering)
- Connects VPCs + VPNs + Direct Connect in one place
- Route tables on the TGW control traffic flow
- ~$0.05/hr per attachment + $0.02/GB data processing
- Can segment with separate TGW route tables (e.g., prod can't reach dev)

**TGW Route Table:**
```
10.0.0.0/16 → VPC-Prod attachment
10.1.0.0/16 → VPC-Stage attachment
10.2.0.0/16 → VPC-Shared attachment
10.10.0.0/16 → VPN attachment (on-premises)
0.0.0.0/0 → VPC-Shared attachment (centralized egress)
```

### PrivateLink / Interface VPC Endpoints

```
Consumer VPC                          Provider VPC
┌──────────────────┐                 ┌──────────────────┐
│                  │                 │                  │
│  EC2/Pod ──────► │─── Interface ──►│ ──► NLB ──────► │──► Service
│                  │    Endpoint     │                  │
│  Uses: vpce-     │    (ENI with    │                  │
│  svc.region.     │    private IP)  │                  │
│  vpce.aws        │                 │                  │
└──────────────────┘                 └──────────────────┘

No route table changes needed!
No CIDR overlap concerns!
Consumer only sees the ENI — not the provider's network.
```

**Key facts:**
- Service-level connectivity (not network-level)
- Consumer gets an ENI in their VPC with a private IP → connects to provider's NLB
- CIDRs CAN overlap (consumer and provider can both be 10.0.0.0/16)
- Provider controls who can access (IAM policies on the endpoint service)
- Used by AWS services (SQS, SNS, KMS) and custom services
- ~$0.01/hr per AZ + $0.01/GB processed

---

## 3. Decision Matrix

```
How many VPCs to connect?
│
├── 2 VPCs, simple → VPC Peering (free, fast)
│
├── 3+ VPCs, hub-spoke → Transit Gateway
│   ├── Need transitive routing? → Must use TGW
│   ├── Need centralized egress? → TGW with shared NAT VPC
│   └── Need on-prem connectivity? → TGW + VPN/Direct Connect
│
├── Need to expose ONE service? → PrivateLink
│   ├── CIDRs overlap? → only PrivateLink works
│   ├── Third-party SaaS? → PrivateLink Marketplace
│   └── Cross-account service? → PrivateLink
│
└── Want to access AWS services privately?
    ├── S3 / DynamoDB → Gateway VPC Endpoint (free)
    └── Everything else → Interface VPC Endpoint (PrivateLink)
```

---

## 4. Failure Scenarios

### Scenario 1: VPC Peering Route Missing on One Side

| Attribute | Detail |
|-----------|--------|
| **Symptom** | VPC-A can reach VPC-B but VPC-B can't reach VPC-A |
| **Root Cause** | Route table entry missing in VPC-B pointing VPC-A's CIDR to the peering connection |
| **Fix** | Add route in VPC-B: `10.0.0.0/16 → pcx-xxxxx` |

### Scenario 2: Transitive Routing Not Working

| Attribute | Detail |
|-----------|--------|
| **Symptom** | VPC-A peers with VPC-B, VPC-B peers with VPC-C. VPC-A can't reach VPC-C. |
| **Root Cause** | VPC Peering doesn't support transitive routing |
| **Fix** | Use Transit Gateway, or create direct peering A↔C |

### Scenario 3: PrivateLink DNS Not Resolving

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Interface endpoint created but `dig vpce-svc-xxx.region.vpce.amazonaws.com` returns nothing |
| **Root Cause** | Private DNS not enabled on the endpoint, or Route53 Private Hosted Zone not associated |
| **Fix** | Enable "Private DNS" on the VPC endpoint → creates Route53 records automatically |

---

## 5. Interview Preparation

### Q1: When would you use Transit Gateway over VPC Peering?

**Answer:** TGW when: 3+ VPCs (avoids N² peering), need transitive routing (A→B→C), centralized egress through a shared NAT VPC, or connecting VPC + VPN + Direct Connect. Peering when: 2 VPCs with simple 1:1 connection (cheaper, simpler).

### Q2: How does PrivateLink differ from VPC Peering?

**Answer:** Peering merges two networks — full L3 connectivity between all resources (requires non-overlapping CIDRs, route entries). PrivateLink exposes a single service via an ENI — consumer never sees provider's network (CIDRs can overlap, no route table changes, granular access control).

### Q3: You have 10 VPCs across 3 AWS accounts that all need to communicate. Design the connectivity.

**Answer:** Transit Gateway in a shared networking account. Attach all 10 VPCs. Use TGW route tables for segmentation: production TGW RT only routes to prod + shared services. Dev TGW RT only routes to dev + shared services. On-premises connected via VPN/Direct Connect to TGW. Centralized egress through a shared services VPC with NAT GW.

---

## 6. Cheat Sheet

```
VPC Peering:     Free, 1:1, no transitive, CIDRs can't overlap
Transit Gateway: $0.05/hr, hub-spoke, transitive, centralized
PrivateLink:     $0.01/hr/AZ, service-level, CIDRs can overlap
Gateway Endpoint: Free (S3, DynamoDB only), route table entry
Interface Endpoint: $0.01/hr/AZ, ENI in your subnet
```

---

## 🔗 Connections

### Previous: VPC (3.1) and Load Balancers (3.2) provide the foundation for single-VPC architecture. This topic extends to multi-VPC connectivity.

### Next: Route53 — DNS for Production Architectures — how DNS ties into multi-VPC and multi-region designs.
