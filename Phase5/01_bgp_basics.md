# BGP Basics — Why Cloud Engineers Need Routing Protocols

> 🔹 **Phase:** 5 — Advanced (Senior Differentiator)  
> 🔹 **Topic:** BGP Fundamentals  
> 🔹 **Priority:** MEDIUM

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Inside your VPC, AWS manages routing automatically. But when you connect your VPC to on-premises data centers (via Direct Connect or VPN), or when you need to understand how the internet's routing works — you encounter **BGP (Border Gateway Protocol)**.

BGP is the protocol that makes the internet work. It's how autonomous systems (AS — networks owned by organizations) tell each other: *"I can reach these IP prefixes. Route traffic for them through me."*

### The Analogy: Shipping Companies

BGP = shipping companies telling each other which zip codes they deliver to.

- FedEx announces: *"I deliver to 10.0.0.0/16 and 10.1.0.0/16"*
- UPS announces: *"I deliver to 10.2.0.0/16 and can also reach 10.0.0.0/16 through FedEx"*
- Each company learns all reachable zip codes and the best path to each

---

## 2. Core Concept (Deep Dive)

### What BGP Does

```
Without BGP (static routing):
  Admin manually configures: "10.1.0.0/16 → send to router X"
  Problem: doesn't scale, doesn't adapt to failures

With BGP (dynamic routing):
  Router A announces: "I can reach 10.0.0.0/16" (AS 65001)
  Router B announces: "I can reach 10.1.0.0/16" (AS 65002)
  Both learn each other's routes automatically
  If a link fails → BGP withdraws the route → traffic reroutes
```

### BGP in AWS

| AWS Service | BGP Role |
|------------|---------|
| **Direct Connect** | BGP session between your router and AWS DX router. Exchanges VPC CIDRs and on-prem CIDRs. |
| **Site-to-Site VPN (dynamic)** | BGP over VPN tunnel. Automatically learns routes from on-prem. |
| **Transit Gateway** | Accepts BGP routes from VPN/DX attachments. Propagates to VPC route tables. |
| **Calico (K8s CNI)** | Uses BGP to advertise pod CIDRs between nodes (BGP mode). |

### iBGP vs eBGP

```
eBGP (External BGP):
  Between DIFFERENT autonomous systems
  Your on-prem (AS 65001) ←→ AWS (AS 64512)
  This is what Direct Connect uses

iBGP (Internal BGP):
  Within the SAME autonomous system
  Between routers in your data center
  Calico uses iBGP between K8s nodes
```

### Key BGP Concepts

| Concept | Explanation |
|---------|------------|
| **AS (Autonomous System)** | A network under single administrative control (your org, AWS, ISP) |
| **ASN** | AS Number — unique identifier. Private: 64512-65534. |
| **Prefix** | IP CIDR being advertised (e.g., 10.0.0.0/16) |
| **Route Advertisement** | "I can reach 10.0.0.0/16" — announcing a prefix |
| **Route Withdrawal** | "I can no longer reach 10.0.0.0/16" — removing a prefix |
| **AS Path** | The chain of ASNs the route has traversed. Shorter = preferred. |
| **BGP Peering** | A TCP session (port 179) between two BGP routers |

### Direct Connect with BGP

```
On-Premises                          AWS
┌──────────┐                    ┌──────────────┐
│ Your     │    DX Connection   │ DX Router    │
│ Router   │◄══════════════════►│ (AWS side)   │
│ AS 65001 │    BGP Session     │ AS 64512     │
└──────────┘                    └──────┬───────┘
                                       │
Announcements:                         │ Routes propagated to
Your router → AWS:                     │ VPC route tables via
  "10.10.0.0/16 (on-prem)"           │ VGW or TGW
                                       │
AWS → Your router:                     
  "10.0.0.0/16 (VPC CIDR)"           
  "10.1.0.0/16 (another VPC)"        
```

---

## 3. Failure Scenarios

### Scenario 1: BGP Route Flapping

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Connectivity to on-prem intermittently drops and recovers |
| **Root Cause** | Unstable link causes BGP to repeatedly advertise and withdraw routes |
| **Impact** | Every flap triggers route recalculation across all peers |
| **Fix** | Route dampening (penalize routes that flap too frequently). Fix unstable physical link. |

### Scenario 2: DX BGP Session Down

| Attribute | Detail |
|-----------|--------|
| **Symptom** | All traffic between VPC and on-prem stops |
| **Root Cause** | BGP session lost (TCP connection to port 179 failed) |
| **Debug** | Check DX console → BGP status. Check on-prem router BGP neighbor state. |
| **Mitigation** | Two DX connections in different locations for HA. VPN as backup with lower BGP priority. |

---

## 4. Interview Preparation

### Q1: Why does AWS use BGP for Direct Connect?

**Answer:** BGP dynamically exchanges routes between on-prem and AWS. When you add a new VPC, BGP automatically advertises its CIDR to your on-prem routers without manual route configuration. If a link fails, BGP withdraws routes and traffic shifts to backup paths. Static routing would require manual updates on every change.

### Q2: What is an AS path and why does it matter?

**Answer:** AS path is the chain of Autonomous Systems a route has traversed. BGP prefers shorter AS paths. If you can reach 10.0.0.0/16 via path [65001, 65002] (2 hops) or [65001, 65003, 65002] (3 hops), BGP picks the shorter path. You can manipulate AS path (prepending) to influence traffic routing.

---

## 5. Cheat Sheet

```
BGP: dynamic routing protocol, TCP port 179
AS: network under one admin control, identified by ASN
eBGP: between different AS (DX, VPN)
iBGP: within same AS (Calico between nodes)
Prefix: IP CIDR being advertised
AS Path: chain of ASNs → shorter = preferred
AWS DX: BGP exchanges VPC CIDRs ↔ on-prem CIDRs
Private ASN: 64512-65534 (used in AWS)
```

---

## 🔗 Connections

### Previous: Linux routing (2.2) covered static routes. BGP adds dynamic route learning and automatic failover.
### Next: Service Mesh — where routing becomes application-level (L7) between microservices.
