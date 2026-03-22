# VPC Architecture — The Complete Mental Model

> 🔹 **Phase:** 3 — Cloud Networking / AWS  
> 🔹 **Topic:** VPC Architecture  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

On-premises, you have a physical data center with routers, switches, firewalls, and cables. Building this takes months and millions of dollars. In the cloud, you need the same isolation and control — but instantly, programmatically, and at any scale.

**VPC (Virtual Private Cloud)** is your own private, isolated section of the AWS cloud. It's a software-defined data center. Everything you know about Linux networking — interfaces, routing, iptables, IP addressing — AWS has wrapped into managed services.

### The Analogy: The Software-Defined Data Center

```
Physical Data Center          →    AWS VPC
──────────────────────             ─────────
Physical router               →    VPC Router (implicit, always exists)
Physical switch per rack       →    Subnet (one per AZ)
Firewall appliance             →    Security Group (stateful) + NACL (stateless)
NAT gateway appliance          →    NAT Gateway (managed)
Internet uplink                →    Internet Gateway (IGW)
Cable from NIC to switch       →    ENI (Elastic Network Interface)
VLAN segmentation              →    Subnet isolation + route tables
Physical server                →    EC2 instance
Server NIC                     →    ENI with private IP from subnet CIDR
```

### Why Does It Exist?

Before VPCs, all AWS instances lived on a shared flat network (EC2 Classic). Any instance could potentially communicate with any other. This was a security nightmare for enterprises.

VPC was introduced in 2009 to give each customer an isolated virtual network with complete control over IP addressing, subnetting, routing, and access control — exactly like a physical data center, but created in seconds.

> 💡 **Key Insight:** A VPC is literally managed Linux networking. The VPC router uses longest-prefix-match. Security Groups are stateful conntrack rules. NACLs are stateless iptables filter rules. If you understand Phase 2, you already understand VPC — just with a different management interface.

---

## 2. Why This Matters in DevOps

| Scenario | VPC Concept |
|----------|------------|
| Designing infrastructure from scratch | VPC CIDR, subnet design, AZ distribution |
| "Can't connect between services" | Route tables, SG rules, NACLs |
| "Pods can't reach the internet" | NAT Gateway, route table (0.0.0.0/0 → NAT GW) |
| Multi-account architecture | VPC Peering, Transit Gateway, PrivateLink |
| Compliance isolation | Separate VPCs per environment, private subnets |
| Cost optimization | VPC Endpoints (avoid NAT GW charges for AWS services) |
| Production incident: random packet drops | VPC Flow Logs → identify REJECT actions |

---

## 3. Core Concept (Deep Dive)

### VPC Components — The Complete Picture

```
┌───────────────────── VPC (10.0.0.0/16) ─────────────────────────┐
│                                                                   │
│   Internet Gateway (IGW)                                         │
│          │                                                        │
│   ┌──────┴────── Route Table (Public) ───────────────────┐       │
│   │  10.0.0.0/16 → local                                 │       │
│   │  0.0.0.0/0   → igw-xxxxx                             │       │
│   └──────┬────────────────────────────────────────────────┘       │
│          │                                                        │
│   ┌──────┴──── Public Subnet (AZ-a) ─── 10.0.0.0/24 ───────┐   │
│   │  [ALB]  [NAT GW]  [Bastion]                              │   │
│   │  NACL: Allow 80,443 in. Allow all out.                    │   │
│   └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│   ┌──────────── Route Table (Private) ───────────────────┐       │
│   │  10.0.0.0/16 → local                                 │       │
│   │  0.0.0.0/0   → nat-xxxxx                             │       │
│   └──────┬────────────────────────────────────────────────┘       │
│          │                                                        │
│   ┌──────┴──── Private Subnet (AZ-a) ── 10.0.10.0/24 ──────┐   │
│   │  [EKS Nodes]  [ECS Tasks]  [Lambda]                      │   │
│   │  SG: Allow 8080 from ALB-SG. Allow 443 out.              │   │
│   └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│   ┌──────┴──── Private Subnet (AZ-a) ── 10.0.20.0/24 ──────┐   │
│   │  [RDS Primary]  [ElastiCache]                             │   │
│   │  SG: Allow 5432 from App-SG only. No internet access.    │   │
│   └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Every VPC Component Explained

#### VPC
- Your isolated virtual network
- You choose the CIDR (e.g., 10.0.0.0/16)
- Cannot change the primary CIDR after creation (can add secondary CIDRs)
- Spans all AZs in a region

#### Subnets
- A slice of the VPC CIDR assigned to ONE Availability Zone
- Public vs. Private is determined by the **route table**, not any inherent property
- Public subnet: route table has `0.0.0.0/0 → IGW`
- Private subnet: route table has `0.0.0.0/0 → NAT GW` (or no default route)

#### Internet Gateway (IGW)
- Allows VPC resources with public IPs to reach the internet
- Performs 1:1 NAT between private and public IPs
- Highly available by design (no single point of failure)
- **Free** — no hourly charge
- Must be attached to VPC AND have a route table entry

#### NAT Gateway
- Allows private subnet resources to initiate outbound internet connections
- Performs SNAT (Source NAT) — replaces private IP with NAT GW's Elastic IP
- **Costs money** — ~$0.045/hr + $0.045/GB processed
- Lives in a public subnet (needs IGW route)
- One per AZ for high availability
- 55,000 concurrent connection limit per destination

#### Route Tables
- Every subnet is associated with exactly one route table
- "Local" route (VPC CIDR → local) is auto-created and cannot be deleted
- You add routes for IGW, NAT GW, peering, TGW, VPC endpoints
- Longest prefix match determines which route wins

#### Security Groups (Stateful Firewall)
- Attached to ENIs (not subnets)
- **Default deny** for inbound, **default allow** for outbound
- **Stateful** — if you allow inbound, response is automatically allowed
- Can reference other SGs as sources (e.g., "allow from ALB's SG")
- Up to 5 SGs per ENI
- No explicit DENY rules — only ALLOW

#### NACLs (Stateless Firewall)
- Attached to subnets (not instances)
- **Stateless** — must explicitly allow both request AND response
- Evaluated in rule number order (lowest first, first match wins)
- Has both ALLOW and DENY rules
- Default NACL allows all traffic
- **Must allow ephemeral ports (1024-65535) for responses**

### Security Groups vs NACLs — The Critical Comparison

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | Instance (ENI) | Subnet |
| **State** | Stateful (auto-allows response) | Stateless (must allow response) |
| **Rules** | Allow only | Allow AND Deny |
| **Evaluation** | All rules evaluated | Rules in order (first match) |
| **Default** | Deny all inbound | Allow all |
| **Reference SGs** | Yes (SG-to-SG rules) | No (CIDR only) |
| **Ephemeral ports** | Handled automatically | Must explicitly allow |

> 🚨 **The #1 NACL mistake:** Adding an inbound allow rule but forgetting the outbound allow for ephemeral ports. The request arrives, the server processes it, but the response is blocked → client sees timeout.

### VPC Endpoints — Avoid NAT GW Costs

VPC Endpoints let private subnet resources access AWS services without going through NAT GW:

```
Without VPC Endpoint:
  EC2 → NAT GW → IGW → S3 (public internet path)
  Cost: $0.045/hr NAT GW + $0.045/GB data processing

With VPC Endpoint (Gateway type):
  EC2 → VPC Router → S3 (private, stays within AWS)
  Cost: FREE (for S3 and DynamoDB gateway endpoints)

With VPC Endpoint (Interface type):
  EC2 → ENI → SQS/SNS/KMS/etc. (private, stays within AWS)
  Cost: ~$0.01/hr per AZ + $0.01/GB processed (cheaper than NAT)
```

**Types:**
- **Gateway Endpoint:** S3 and DynamoDB only. Free. Adds a prefix-list route to the route table.
- **Interface Endpoint (PrivateLink):** Most other AWS services. Creates an ENI in your subnet with a private IP.

---

## 4. Internal Working (Under the Hood)

### How VPC Networking Actually Works

```
EC2 Instance sends a packet:
│
│ 1. Packet leaves ENI
│
│ 2. Security Group check (stateful)
│    → Is there an outbound rule allowing this? 
│    → If allowed → continue. If not → DROP silently.
│
│ 3. Subnet NACL check (stateless, outbound rules)
│    → Rules evaluated in order. First match wins.
│    → ALLOW → continue. DENY → DROP.
│
│ 4. VPC Router — routing decision
│    → Route table lookup (longest prefix match)
│    → 10.0.0.0/16 → local (same VPC) → deliver to target subnet
│    → 0.0.0.0/0 → igw/nat-gw (internet-bound)
│
│ 5. Target subnet NACL check (inbound rules)
│    → Rules evaluated in order.
│
│ 6. Target Security Group check (inbound rules)
│    → Is there an inbound rule allowing this source?
│
│ 7. Packet delivered to target ENI
```

**Key detail:** The VPC router is implicit — you never see it as a resource, but it's always there at x.x.x.1 (first IP in every subnet). It performs routing, NAT (for IGW), and is the backbone of VPC networking.

### ENI — The Elastic Network Interface

Every EC2 instance, Lambda function (VPC-attached), ECS task, RDS instance, and EKS pod (with VPC CNI) has at least one ENI:

```
ENI Properties:
├── Primary private IP (10.0.1.50)
├── Secondary private IPs (10.0.1.51, 10.0.1.52 — for K8s pods)
├── Elastic IP (optional, for public access)
├── MAC address
├── Security Groups (up to 5)
├── Source/Dest check (disable for NAT instances, K8s nodes)
└── Subnet association (determines AZ)
```

---

## 5. Mental Models

### Mental Model 1: The Three-Tier Building

```
           INTERNET
              │
    ┌─────────┴─────────┐
    │   LOBBY (Public)   │ ← IGW = front door
    │   ALB, NAT GW      │   Anyone with the address can enter
    │   Bastion Host      │   But must pass security (SG)
    ├─────────────────────┤
    │   OFFICE (Private)  │ ← App servers, K8s nodes
    │   Can reach lobby   │   Can't be reached from street
    │   Can exit via NAT  │   Goes through lobby to reach internet
    ├─────────────────────┤
    │   VAULT (Isolated)  │ ← Databases, secrets
    │   Can only be       │   No internet access at all
    │   reached from      │   Only app servers can connect
    │   office floor       │
    └─────────────────────┘
```

### Mental Model 2: VPC = Managed Linux Networking

```
VPC Concept              → Linux Equivalent
────────────────────────────────────────────
VPC Router               → ip route (routing table)
Subnet                   → Interface + connected route
Security Group           → iptables -m conntrack (stateful)
NACL                     → iptables (stateless, per-subnet)
NAT Gateway              → iptables MASQUERADE
IGW                      → Default gateway to internet
Route Table              → ip route show
ENI                      → eth0 (network interface)
VPC Flow Logs            → tcpdump / iptables LOG
```

---

## 6. Real-World Mapping

### Essential CLI Commands

```bash
# VPC info
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' --output table

# Subnets
aws ec2 describe-subnets --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' --output table

# Route tables
aws ec2 describe-route-tables --query 'RouteTables[*].[RouteTableId,Routes]' --output json

# Security Groups
aws ec2 describe-security-groups --query 'SecurityGroups[*].[GroupId,GroupName,IpPermissions]' --output json

# VPC Flow Logs
aws ec2 describe-flow-logs --query 'FlowLogs[*].[FlowLogId,ResourceId,TrafficType,LogDestination]' --output table

# VPC Endpoints
aws ec2 describe-vpc-endpoints --query 'VpcEndpoints[*].[VpcEndpointId,ServiceName,VpcEndpointType]' --output table
```

### VPC Architecture for EKS

```
VPC: 10.0.0.0/16

Public Subnets (tagged for ALB discovery):
  10.0.0.0/24 (AZ-a) — ALB, NAT GW-a
  10.0.1.0/24 (AZ-b) — ALB, NAT GW-b
  10.0.2.0/24 (AZ-c) — ALB, NAT GW-c
  Tags: kubernetes.io/role/elb = 1

Private Subnets (tagged for internal LB):
  10.0.10.0/20 (AZ-a) — EKS nodes + pods (4091 usable IPs)
  10.0.16.0/20 (AZ-b) — EKS nodes + pods
  10.0.22.0/20 (AZ-c) — EKS nodes + pods
  Tags: kubernetes.io/role/internal-elb = 1

DB Subnets:
  10.0.100.0/24 (AZ-a) — RDS, ElastiCache
  10.0.101.0/24 (AZ-b)
  10.0.102.0/24 (AZ-c)

VPC Endpoints: S3 (gateway), ECR (interface), STS, CloudWatch Logs
```

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: EC2/Pod Can't Reach the Internet

| Attribute | Detail |
|-----------|--------|
| **Symptom** | `curl https://example.com` times out from a private subnet |
| **Root Cause** | Missing or misconfigured NAT Gateway |
| **Debug Steps** | 1. Check subnet's route table: `0.0.0.0/0 → nat-gw?` 2. Is the NAT GW in a public subnet? 3. Does the NAT GW's subnet have `0.0.0.0/0 → igw`? 4. Does the NAT GW have an Elastic IP? 5. Check NACL rules on both subnets. |

### Scenario 2: NACL Blocking Responses

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Inbound HTTP to EC2 works (server receives request) but client never gets a response. Client sees timeout. |
| **Root Cause** | NACL outbound rules don't allow ephemeral ports (1024-65535) |
| **Debug Steps** | 1. VPC Flow Logs → look for REJECT on the response. 2. Check NACL outbound rules — is there an allow for ephemeral ports? 3. Check NACL inbound on the client's subnet for ephemeral ports. |
| **Fix** | Add outbound NACL rule: `Allow TCP 1024-65535 to 0.0.0.0/0`. |

### Scenario 3: Security Group Reference Breaks After VPC Peering

| Attribute | Detail |
|-----------|--------|
| **Symptom** | After peering VPCs, SG rule referencing a security group ID from the peered VPC doesn't work |
| **Root Cause** | Cross-VPC SG references require specific configuration. Format: `<account>/<vpc-id>/sg-xxxxx` |
| **Debug Steps** | 1. Verify peering is active. 2. Check routes exist in both VPCs. 3. Verify SG rule format for cross-VPC references. |

### Scenario 4: NAT Gateway Port Exhaustion

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Intermittent connection failures from private subnets to a single external API |
| **Root Cause** | NAT GW has 55,000 concurrent connections per destination (IP+port). Many pods/Lambdas connecting to the same endpoint exhaust this. |
| **Debug Steps** | CloudWatch → NAT GW → `ErrorPortAllocation` metric. If > 0, ports are exhausting. |
| **Fix** | 1. Multiple EIPs on NAT GW. 2. Multiple NAT GWs (one per AZ). 3. Connection pooling in apps. 4. VPC Endpoints for AWS services. |

### Scenario 5: VPC Flow Logs Show REJECT but SG Looks Correct

| Attribute | Detail |
|-----------|--------|
| **Symptom** | VPC Flow Logs show REJECT. SG rules appear correct. |
| **Root Cause** | 1. NACL is rejecting (not SG). 2. SG outbound rule on the SOURCE is missing. 3. The wrong SG is attached to the ENI. |
| **Debug Steps** | 1. Flow Logs show source IP, dest IP, port, action (ACCEPT/REJECT). 2. Check NACL rules on BOTH source and destination subnets. 3. Verify the correct SG is on the correct ENI: `aws ec2 describe-network-interfaces`. |

---

## 8. Debugging Playbook

```
VPC Connectivity Issues:

Step 1: VPC Flow Logs
──────────────────────
→ Enable if not already
→ Filter for source/destination IP of the failing connection
→ ACCEPT → packet went through (problem is at higher layer)
→ REJECT → blocked by SG or NACL

Step 2: Security Groups
────────────────────────
→ Check INBOUND rules on the DESTINATION
→ Check OUTBOUND rules on the SOURCE
→ Remember: SGs are stateful — response is auto-allowed

Step 3: NACLs
──────────────
→ Check BOTH inbound AND outbound on source subnet
→ Check BOTH inbound AND outbound on destination subnet
→ Remember: stateless — MUST allow ephemeral ports for responses

Step 4: Route Tables
─────────────────────
→ Source subnet: does a route exist for the destination?
→ Destination subnet: does a route exist for the source?
→ For internet: 0.0.0.0/0 → igw (public) or nat-gw (private)

Step 5: ENI / Instance
───────────────────────
→ Is the instance running?
→ Is the ENI attached and has an IP?
→ Source/Dest check disabled (if forwarding traffic, e.g., K8s node)?
```

---

## 9. Hands-On Practice

### Lab 1: Map a VPC

```bash
# Get all VPCs
aws ec2 describe-vpcs --output table

# For each VPC, list subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxxx" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' --output table

# For each subnet, check route table
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-xxxxx" \
  --query 'RouteTables[*].Routes' --output json

# Determine: which subnets are public (IGW route) vs private (NAT GW route)?
```

### Lab 2: Trace a Packet Path

```
Exercise: Trace the path of an HTTP request from the internet to a pod.

Request: https://api.example.com → ALB → EKS Pod

Path:
1. DNS: Route53 resolves api.example.com → ALB IP
2. Client → ALB (public subnet): passes through NACL inbound, SG inbound
3. ALB terminates TLS, reads HTTP, selects target group
4. ALB → Pod IP (private subnet): new connection, passes through private subnet NACL, pod SG
5. Pod processes request, responds
6. Response: Pod → ALB → Client (reverse path, SGs auto-allow, NACLs need ephemeral ports)
```

---

## 10. Interview Preparation

### Q1: (Must-know) Design a VPC for a production EKS cluster.

**Answer:** VPC: 10.0.0.0/16. 3 AZs. Public subnets (3× /24) for ALB, NAT GW — route table with 0.0.0.0/0 → IGW. Private subnets (3× /20) for EKS nodes — route table with 0.0.0.0/0 → NAT GW (per AZ for HA). DB subnets (3× /24) — no internet route. VPC Endpoints for S3 (gateway, free), ECR (interface, avoid pulling images through NAT). Security: app SG allows from ALB SG, DB SG allows from app SG only.

### Q2: What makes a subnet "public" vs "private"?

**Answer:** Purely the route table. A public subnet has `0.0.0.0/0 → IGW`. A private subnet has `0.0.0.0/0 → NAT GW` or no default route. There is no inherent "public" flag — it's all routing.

### Q3: Explain the difference between Security Groups and NACLs.

**Answer:** SGs are stateful (attached to ENIs, auto-allow responses, allow-only rules, evaluate all rules). NACLs are stateless (attached to subnets, must explicitly allow responses via ephemeral ports, have both allow/deny, evaluate in rule-number order). Use SGs as the primary firewall. Use NACLs for subnet-level deny rules (e.g., block a known bad CIDR).

### Q4: NAT Gateway vs IGW — what's the difference?

**Answer:** IGW enables bidirectional internet access for resources with public IPs (1:1 NAT). NAT GW enables outbound-only internet access for private resources (many-to-1 SNAT). IGW is free; NAT GW costs ~$0.045/hr + data processing. IGW is attached to VPC; NAT GW lives in a public subnet and needs an EIP.

### Q5: How would you reduce NAT Gateway costs?

**Answer:** 1. VPC Gateway Endpoints for S3 and DynamoDB (free, traffic stays in AWS). 2. VPC Interface Endpoints for other AWS services (ECR, SQS, CloudWatch — cheaper than NAT). 3. Use VPC Endpoints for ECR to avoid NAT for container image pulls. 4. Review what outbound traffic actually needs internet — often 80%+ is AWS API calls that can use endpoints.

---

## 11. Advanced Insights (Senior Level)

### Multi-Account VPC Strategy

```
Organization:
├── Shared Services Account
│   └── VPC: 10.0.0.0/16 (CI/CD, monitoring, shared services)
│
├── Production Account
│   └── VPC: 10.1.0.0/16 (production workloads)
│
├── Staging Account
│   └── VPC: 10.2.0.0/16 (staging)
│
├── Development Account
│   └── VPC: 10.3.0.0/16 (development)
│
└── Transit Gateway connects all VPCs
    → Centralized routing and inspection
    → Non-overlapping CIDRs enforced
```

### VPC Flow Logs for Security

```
Flow Log format:
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

Example:
2 123456789 eni-abc123 10.0.1.50 10.0.2.100 54321 443 6 10 840 1620000000 1620000060 ACCEPT OK

Key uses:
- Detect unauthorized access attempts (REJECT from unexpected sources)
- Identify data exfiltration (large outbound bytes to unexpected IPs)
- Audit compliance (prove no traffic between isolated tiers)
- Debug connectivity (find the REJECT that's causing timeout)
```

---

## 12. Common Mistakes

### Mistake 1: VPC CIDR too small
Using /20 or /24 for a VPC that will run EKS. With VPC CNI, each pod needs a VPC IP. Always use /16 for production.

### Mistake 2: Single NAT Gateway for multi-AZ
If the NAT GW's AZ goes down, all private subnets lose internet access. Deploy one NAT GW per AZ.

### Mistake 3: Ignoring VPC Endpoints
Running all AWS API calls through NAT GW is expensive. ECR image pulls alone can cost significant data processing fees. Gateway Endpoints for S3 are free.

### Mistake 4: Not enabling VPC Flow Logs
Flow Logs are essential for security auditing and debugging. Enable them on every VPC in production. Send to S3 for cost-effective storage.

### Mistake 5: Overlapping CIDRs across VPCs
Prevents peering and Transit Gateway connectivity. Plan ALL VPC CIDRs before creating any.

---

## 13. Cheat Sheet

### VPC Design Checklist

```
☐ VPC CIDR: /16 (non-overlapping with other VPCs)
☐ 3 AZs minimum
☐ Public subnets: /24 per AZ (ALB, NAT GW)
☐ Private subnets: /20 per AZ (App/EKS — room for pod IPs)
☐ DB subnets: /24 per AZ (no internet route)
☐ NAT Gateway per AZ (HA)
☐ VPC Endpoints: S3 (gateway), ECR, CloudWatch Logs
☐ VPC Flow Logs enabled
☐ Tags: kubernetes.io/role/elb, kubernetes.io/role/internal-elb
```

### Quick Reference

```
Public subnet  = route 0.0.0.0/0 → IGW
Private subnet = route 0.0.0.0/0 → NAT GW
Isolated subnet = no default route
SG = stateful, ENI-level, allow-only
NACL = stateless, subnet-level, allow+deny, ORDER MATTERS
IGW = free, bidirectional, needs public IP
NAT GW = ~$32/month/AZ, outbound only, needs EIP
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**IP Addressing/CIDR (Topic 1.3):** VPC design IS CIDR math. Every subnet, peering connection, and endpoint uses the CIDR concepts from Phase 1.

**Routing (Topic 2.2):** VPC route tables use the exact same longest-prefix-match algorithm as Linux routing tables. "Local" route = connected route.

**iptables (Topic 2.3):** Security Groups = stateful iptables (conntrack). NACLs = stateless iptables (filter, rule-order evaluation).

### What This Prepares You For Next
**Next topic: Load Balancers — ALB vs NLB and When to Use Each**

Now that you understand VPC architecture (subnets, routing, SG/NACL), the next topic covers the load balancers that sit in your public subnets and distribute traffic to your private subnets — ALB for HTTP routing and NLB for TCP passthrough.
