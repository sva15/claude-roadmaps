# Multi-Region and Hybrid Connectivity Patterns

> 🔹 **Phase:** 5 — Advanced (Senior Differentiator)  
> 🔹 **Topic:** Multi-Region and Hybrid Architecture  
> 🔹 **Priority:** MEDIUM

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Real-world production doesn't run in a single VPC. It spans multiple regions (for latency and disaster recovery), multiple accounts (for isolation), and often connects to on-premises data centers. You need patterns for:
- **Multi-region:** Active-active or active-passive across AWS regions
- **Multi-account:** Networking between isolated AWS accounts
- **Hybrid:** Connecting AWS to on-premises data centers
- **Multi-cluster K8s:** Pods across multiple EKS clusters communicating

---

## 2. Core Patterns

### Pattern 1: Multi-Region Active-Active

```
                     Route53 (Latency-based routing)
                    ┌──────────┴──────────┐
                    ▼                      ▼
           ┌── us-east-1 ──┐     ┌── eu-west-1 ──┐
           │  ALB → EKS    │     │  ALB → EKS    │
           │  RDS Primary  │     │  RDS Replica  │
           │  ElastiCache  │     │  ElastiCache  │
           └───────────────┘     └───────────────┘

Components:
  → Route53 latency routing: users hit nearest region
  → Health checks: unhealthy region removed from DNS
  → RDS cross-region read replicas (or Aurora Global Database)
  → ElastiCache per region (cache locality)
  → Data replication: async (eventual consistency)

Challenge: data consistency between regions
  → Reads: serve from local replica (fast, eventually consistent)
  → Writes: route to primary region (or use conflict resolution)
```

### Pattern 2: Multi-Region Active-Passive (DR)

```
           ┌── us-east-1 (PRIMARY) ──┐
           │  Route53 → ALB → EKS   │ ← All traffic
           │  RDS Primary            │
           └────────┬────────────────┘
                    │ Cross-region replication
           ┌────────▼────────────────┐
           │  us-west-2 (DR)         │
           │  ALB → EKS (scaled to 0)│ ← No traffic (standby)
           │  RDS Read Replica       │
           └─────────────────────────┘

Failover trigger:
  → Route53 health check fails on primary
  → DNS shifts to DR region
  → Promote RDS replica to primary
  → Scale up DR EKS cluster
  → RTO: 15-60 minutes depending on automation
```

### Pattern 3: Hybrid Connectivity

```
On-Premises Data Center              AWS
┌─────────────────────┐         ┌──────────────────────┐
│  App servers         │         │  VPC (10.0.0.0/16)   │
│  Databases          │         │  EKS cluster          │
│  Legacy systems     │         │  RDS, ElastiCache     │
│  10.10.0.0/16       │         │                       │
└─────────┬───────────┘         └──────────┬────────────┘
          │                                │
          ▼                                ▼
    ┌─────────────────────────────────────────┐
    │         Connection Options:             │
    │                                         │
    │  1. Site-to-Site VPN                    │
    │     → Quick setup (hours)              │
    │     → Over internet (encrypted)        │
    │     → ~1.25 Gbps per tunnel            │
    │     → ~$0.05/hr                        │
    │     → Higher latency, variable         │
    │                                         │
    │  2. AWS Direct Connect                  │
    │     → Dedicated fiber (weeks to setup) │
    │     → Private connection (not internet)│
    │     → 1 Gbps or 10 Gbps              │
    │     → ~$0.30/hr (port) + data transfer│
    │     → Consistent latency              │
    │                                         │
    │  3. VPN over Direct Connect             │
    │     → DX for bandwidth + VPN for       │
    │       encryption on top                │
    │     → Best of both worlds              │
    │                                         │
    │  Recommendation:                        │
    │     Production: DX primary + VPN backup│
    │     Dev/test: VPN only                 │
    └─────────────────────────────────────────┘
```

### Pattern 4: Multi-Account with Transit Gateway

```
                    ┌────────────────────┐
                    │  Transit Gateway    │
                    │  (Networking Acct) │
                    └──┬──┬──┬──┬──┬────┘
                       │  │  │  │  │
         ┌─────────────┘  │  │  │  └─────────────┐
         ▼                │  │  │                ▼
  ┌─── Shared ────┐      │  │  │      ┌─── Security ──┐
  │ Services VPC  │      │  │  │      │ Inspection VPC│
  │ CI/CD, Tools  │      │  │  │      │ Firewall, IDS │
  │ 10.0.0.0/16   │      │  │  │      │ 10.4.0.0/16  │
  └───────────────┘      │  │  │      └──────────────┘
                         │  │  │
              ┌──────────┘  │  └──────────┐
              ▼             ▼             ▼
       ┌─── Prod ───┐ ┌── Stage ──┐ ┌── Dev ────┐
       │ Prod VPC   │ │ Stage VPC │ │ Dev VPC   │
       │ 10.1.0/16  │ │ 10.2.0/16│ │ 10.3.0/16│
       └────────────┘ └──────────┘ └──────────┘

TGW Route Table Segmentation:
  Prod RT: routes to Shared + Security only (NOT to Dev/Stage)
  Dev RT: routes to Shared only (NOT to Prod)
  → Enforce isolation between environments
```

### Pattern 5: Multi-Cluster Kubernetes

```
Options for cross-cluster pod communication:

1. Shared VPC (same region):
   → Two EKS clusters in same VPC
   → Pods use VPC IPs (VPC CNI) → natural routing
   → Simplest but couples clusters

2. VPC Peering / TGW (same or cross-region):
   → Each cluster in its own VPC
   → Connect via peering or TGW
   → Pod CIDRs must not overlap
   → Use ExternalName Services or direct IPs

3. Cilium Cluster Mesh:
   → eBPF-based cross-cluster connectivity
   → Transparent pod-to-pod across clusters
   → Shared Service discovery
   → Each cluster keeps its identity

4. Istio Multi-Cluster:
   → Service mesh spans multiple clusters
   → mTLS between clusters
   → Unified traffic management
   → Complex to operate
```

---

## 3. Failure Scenarios

### Scenario 1: Cross-Region Replication Lag

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Users in DR region see stale data |
| **Root Cause** | Async replication has inherent lag (seconds to minutes) |
| **Mitigation** | Aurora Global Database (< 1s lag). Accept eventual consistency for reads. Route writes to primary region. |

### Scenario 2: DX + VPN Failover Not Working

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Direct Connect goes down but traffic doesn't shift to VPN backup |
| **Root Cause** | VPN BGP routes have same preference as DX routes. Or VPN tunnel is down. |
| **Fix** | Set DX BGP routes with shorter AS path (higher preference). Test VPN failover regularly. Monitor both connections with CloudWatch. |

### Scenario 3: CIDR Overlap Prevents New Peering

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Can't peer new VPC because CIDR overlaps with existing peer |
| **Root Cause** | Poor initial CIDR planning |
| **Fix** | Add secondary CIDR to new VPC with non-overlapping range. Plan ALL CIDRs in advance (IPAM). Use PrivateLink for service-level connectivity (works with overlapping CIDRs). |

---

## 4. Interview Preparation

### Q1: Design a multi-region active-active architecture.

**Answer:** Route53 latency-based routing → ALB per region → EKS clusters. Data: Aurora Global Database for < 1s replication. Cache: ElastiCache per region (no cross-region cache — locality matters). DNS failover: health checks on each region's ALB. Challenge: write consistency — route writes to primary, serve reads locally. Use DynamoDB Global Tables if NoSQL acceptable.

### Q2: Direct Connect vs VPN — when do you use each?

**Answer:** DX for production (dedicated bandwidth, consistent latency, higher cost, weeks to set up). VPN for dev/test or as DX backup (quick setup, encrypted over internet, variable performance). Best practice: DX primary + VPN backup with BGP failover. Test failover quarterly.

### Q3: How do you handle non-overlapping CIDRs across 50 VPCs?

**Answer:** Central IP Address Management (IPAM). AWS VPC IPAM service. Allocate from a master range (e.g., 10.0.0.0/8). Assign /16 per VPC. Document in a registry. Enforce via Service Control Policies (prevent VPC creation with unregistered CIDRs). Never use 172.16.0.0/12 or 192.168.0.0/16 (conflict with on-prem defaults).

---

## 5. Cheat Sheet

```
Multi-Region:
  Active-Active: Route53 latency + health checks + per-region infra
  Active-Passive: Route53 failover + DR region on standby

Hybrid:
  VPN: quick, cheap, variable. DX: dedicated, consistent, expensive.
  Best: DX primary + VPN backup with BGP failover.

Multi-Account:
  TGW hub-and-spoke. Route table segmentation for isolation.
  Non-overlapping CIDRs (use IPAM).

Multi-Cluster K8s:
  Same VPC (simplest) → Peering/TGW → Cilium Mesh → Istio Multi-cluster

Golden Rules:
  → Plan CIDRs upfront (no overlaps)
  → 1 NAT GW per AZ (HA)
  → Test failover regularly
  → Use VPC Endpoints for AWS services
```

---

## 🔗 Connections

### This topic synthesizes EVERYTHING from the roadmap:
- **Phase 1:** Protocols (TCP, DNS, TLS, HTTP) underpin all connectivity
- **Phase 2:** Linux networking (routing, iptables, namespaces) runs on every node
- **Phase 3:** VPC architecture, load balancers, peering/TGW, Route53
- **Phase 4:** K8s networking model, CNI, Services, Ingress, policies
- **Phase 5:** BGP (DX/VPN), service mesh, eBPF (Cilium)

You now have the complete picture: from a single packet on a wire to a globally distributed, multi-region, hybrid Kubernetes architecture. 🎉
