# IP Addressing, Subnets, and CIDR

> 🔹 **Phase:** 1 — Network Foundations  
> 🔹 **Topic:** IP Addressing, Subnets, and CIDR  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

If you want to send a letter, you need an address. If you want to send a packet, you need an **IP address**.

But here's the real problem: the internet has **billions** of devices. How do you organize all their addresses so that routers can efficiently decide where to send packets?

The answer is **hierarchical addressing** — exactly like how postal codes work:
- Country → State → City → Street → House number
- Network → Subnet → Host

**IP addressing** gives every device a unique address.  
**Subnetting** divides large networks into smaller, manageable pieces.  
**CIDR** is the notation that tells you how many addresses a network has.

### The Analogy: Apartment Buildings

Think of IP addressing like an apartment building:

- The **building address** (street + number) = the **network portion** of the IP
- The **apartment number** = the **host portion** of the IP
- A **big building** has many apartments (large subnet, like /16 = 65,536 addresses)
- A **small building** has few apartments (small subnet, like /28 = 16 addresses)

CIDR notation tells you "how big is this building?"

### Why Does It Exist?

The original internet used **classful addressing** (Class A, B, C), which wasted huge blocks of addresses. A company that needed 300 IPs had to get a Class B (65,536 IPs) — wasting 65,236 addresses.

CIDR (Classless Inter-Domain Routing) was invented in 1993 to allow flexible network sizes. Instead of fixed classes, you can carve out exactly the block you need: /24 for 256 IPs, /25 for 128, /27 for 32.

> 💡 **Key Insight:** VPC design, Kubernetes pod CIDRs, Security Group rules, NACL rules — every single one requires you to think in CIDR. Getting comfortable with the math means you can design networks and debug connectivity issues on a whiteboard.

---

## 2. Why This Matters in DevOps

### Where This Appears in Real Systems

| Scenario | IP/CIDR Concept |
|----------|----------------|
| Designing a VPC | Choose a CIDR block (/16 typically), carve into subnets |
| Security Group rules | Allow `10.0.0.0/8` or specific `/32` IPs |
| Kubernetes cluster setup | Pod CIDR, Service CIDR must not overlap VPC CIDR |
| VPC Peering | CIDRs must not overlap between peered VPCs |
| NACL rules | Allow/deny specific CIDR ranges |
| NAT Gateway | Private subnets need NAT; public subnets need IGW |
| Debugging "can't connect" | Is the destination IP in a reachable subnet? |
| Multi-region architecture | Plan non-overlapping CIDRs for all VPCs |

### Why DevOps Engineers Must Know This

1. **VPC Design is a Day 1 Decision** — You choose the VPC CIDR *once*. If it's too small, you can't add subnets later. If it overlaps with another VPC, you can't peer them. Getting this wrong is expensive to fix.

2. **Kubernetes requires CIDR planning** — The pod CIDR (e.g., 10.244.0.0/16) and service CIDR (10.96.0.0/12) must not overlap with the VPC CIDR. You choose these at cluster creation and **cannot change them after**.

3. **Interview favorite** — "Given a VPC 10.0.0.0/16, design subnets for a 3-tier architecture" is one of the most common whiteboard questions. You must do this mental math quickly.

---

## 3. Core Concept (Deep Dive)

### IPv4 Address Structure

An IPv4 address is a **32-bit number**, written as 4 groups of 8 bits (octets), separated by dots:

```
    192     .    168     .     1      .     25
 11000000   . 10101000   . 00000001   . 00011001
 ──────────   ──────────   ──────────   ──────────
  8 bits       8 bits       8 bits       8 bits     = 32 bits total
```

Total possible IPv4 addresses: 2^32 = 4,294,967,296 (~4.3 billion) — not enough for the modern internet, which is why IPv6 exists and NAT is used extensively.

### Public vs. Private IP Addresses

**Private ranges** (RFC 1918) — these addresses are NEVER routable on the public internet. NAT translates them to public IPs for internet access:

| Range | CIDR | Size | Common Use |
|-------|------|------|-----------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | 16,777,216 IPs | AWS VPCs, large enterprise networks |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | 1,048,576 IPs | Docker default (172.17.0.0/16) |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | 65,536 IPs | Home networks, small labs |

**Special addresses:**

| Address | Meaning |
|---------|---------|
| 127.0.0.1 | Localhost (loopback) — traffic never leaves the machine |
| 0.0.0.0 | "All interfaces" when listening; "default route" in route tables |
| 255.255.255.255 | Broadcast address |
| 169.254.0.0/16 | Link-local — assigned when DHCP fails |

### CIDR Notation

CIDR notation: `IP/prefix-length`

The prefix length tells you **how many bits are the network part** (fixed) and how many are the **host part** (variable).

```
10.0.1.0/24
         └── 24 bits are the network part
             8 bits are the host part (32 - 24 = 8)
             2^8 = 256 total addresses
```

### The CIDR Math (Memorize This)

| CIDR | Network Bits | Host Bits | Total IPs | Usable IPs* | Notes |
|------|-------------|-----------|-----------|-------------|-------|
| /8 | 8 | 24 | 16,777,216 | 16,777,211 | Huge — like 10.0.0.0/8 |
| /16 | 16 | 16 | 65,536 | 65,531 | Typical VPC size |
| /20 | 20 | 12 | 4,096 | 4,091 | Large subnet |
| /24 | 24 | 8 | 256 | 251 | Standard subnet |
| /25 | 25 | 7 | 128 | 123 | Half of /24 |
| /26 | 26 | 6 | 64 | 59 | Quarter of /24 |
| /27 | 27 | 5 | 32 | 27 | Small subnet |
| /28 | 28 | 4 | 16 | 11 | Minimum AWS subnet |
| /32 | 32 | 0 | 1 | 1 | Single host |

> *Usable = Total - 5 (AWS reserves 5 IPs per subnet)

### The Key Rule: Each +1 to the Prefix Halves the Addresses

```
/24 = 256 IPs
/25 = 128 IPs  (256 ÷ 2)
/26 = 64 IPs   (128 ÷ 2)
/27 = 32 IPs   (64 ÷ 2)
/28 = 16 IPs   (32 ÷ 2)
```

**Quick mental math:** Start from /24 = 256 and halve. Or start from /32 = 1 and double.

### AWS Reserved Addresses (5 per Subnet)

In every AWS subnet, 5 IPs are reserved and cannot be used:

```
Example: 10.0.1.0/24 (256 IPs)

10.0.1.0   — Network address
10.0.1.1   — VPC router (the .1 address)
10.0.1.2   — DNS server (VPC base + 2)
10.0.1.3   — Reserved for future use
10.0.1.255 — Broadcast address (broadcast not supported in VPC, but reserved)

Usable: 256 - 5 = 251 IPs
```

For a /28 subnet (16 IPs): 16 - 5 = **11 usable** (the smallest practical AWS subnet).

### Subnet Math: How to Calculate

**Question:** What is the network range for `10.0.5.96/27`?

**Step 1:** /27 = 32 IPs  
**Step 2:** Find the block: 96 ÷ 32 = 3 (exact), so the block starts at 96  
**Step 3:** Network: 10.0.5.96, Last address: 10.0.5.96 + 32 - 1 = 10.0.5.127  
**Step 4:** Usable range: 10.0.5.97 – 10.0.5.126 (in standard networking, or minus AWS reserved)

**Shortcut:** For any /27, the blocks are: .0, .32, .64, .96, .128, .160, .192, .224  
For /26: .0, .64, .128, .192  
For /25: .0, .128  

### Subnet Design: The VPC Layout Question

Given a VPC `10.0.0.0/16`, design subnets for a 3-tier architecture:

```
VPC: 10.0.0.0/16 (65,536 IPs)

Public Subnets (ALB, NAT GW, Bastion):
├─ 10.0.0.0/24   — Public AZ-a   (251 usable)
├─ 10.0.1.0/24   — Public AZ-b   (251 usable)

Private App Subnets (ECS/EKS workloads):
├─ 10.0.10.0/24  — App AZ-a      (251 usable)
├─ 10.0.11.0/24  — App AZ-b      (251 usable)
├─ 10.0.12.0/24  — App AZ-c      (251 usable)

Private DB Subnets (RDS, ElastiCache):
├─ 10.0.20.0/24  — DB AZ-a       (251 usable)
├─ 10.0.21.0/24  — DB AZ-b       (251 usable)

Spare capacity:
└─ 10.0.22.0/24 – 10.0.255.0/24  (future growth)
```

**Key design decisions:**
1. Leave gaps between subnet groups (0.x, 10.x, 20.x) for future growth
2. Use consistent sizes (/24 for simplicity)
3. Ensure no overlaps with other VPCs you might peer with
4. Plan for 3 AZs even if you start with 2

---

## 4. Internal Working (Under the Hood)

### How IP Addresses Work in the Kernel

When a Linux machine sends a packet:

```
1. Application calls connect(destination_IP:port)
2. Kernel performs routing table lookup:
   → ip route get <destination_IP>
   → Finds the matching route (longest prefix wins)
   → Determines: output interface + next-hop gateway
3. Kernel sets Source IP:
   → If explicitly bound, use that IP
   → Otherwise, use the IP of the output interface
4. Kernel passes to L2:
   → If destination is on the same subnet → ARP for destination MAC
   → If destination is on a different subnet → ARP for gateway MAC
5. Frame sent out the interface
```

### Longest Prefix Match (The Routing Algorithm)

The routing table might have multiple matching routes. The **longest prefix match** wins:

```
Route Table:
    0.0.0.0/0      → gateway 10.0.0.1 (default route)
    10.0.0.0/16     → local
    10.0.5.0/24     → via 10.0.0.100
    10.0.5.128/25   → via 10.0.0.200

Packet destination: 10.0.5.200

Matching routes:
    0.0.0.0/0       ← matches (default, prefix length 0)
    10.0.0.0/16     ← matches (prefix length 16)
    10.0.5.0/24     ← matches (prefix length 24)
    10.0.5.128/25   ← matches (prefix length 25) ← WINS (longest prefix)

Packet goes via 10.0.0.200
```

This is the **exact same algorithm** used by:
- Linux kernel (`ip route`)
- AWS VPC route tables
- Kubernetes pod routing
- Internet BGP routing

### Subnetting in Binary (How It Actually Works)

```
IP:      10.0.5.100
Binary:  00001010.00000000.00000101.01100100

Subnet:  10.0.5.0/24
Mask:    11111111.11111111.11111111.00000000
         ◄── Network (fixed) ───►◄─ Host ─►

Network = IP AND Mask:
         00001010.00000000.00000101.00000000 = 10.0.5.0

Host    = IP AND (NOT Mask):
         00000000.00000000.00000000.01100100 = 0.0.0.100
```

Two IPs are on the **same subnet** if their network portions match. If they match → traffic goes directly (ARP). If they don't → traffic goes to the gateway.

---

## 5. Mental Models

### Mental Model 1: The Apartment Building

```
10.0.0.0/16 = A city with 65,536 apartments

10.0.1.0/24 = A building with 256 apartments
10.0.2.0/24 = Another building with 256 apartments

10.0.1.50  = Apartment 50 in building 10.0.1.x
10.0.2.50  = Apartment 50 in building 10.0.2.x (different building!)
```

The prefix length (/16, /24) tells you the building size. Same building = local delivery. Different building = need a postal route.

### Mental Model 2: The Powers-of-Two Ruler

```
/32 ─ 1 IP       (single host)
/31 ─ 2 IPs      (point-to-point)
/30 ─ 4 IPs
/29 ─ 8 IPs
/28 ─ 16 IPs     (minimum AWS subnet)
/27 ─ 32 IPs
/26 ─ 64 IPs
/25 ─ 128 IPs
/24 ─ 256 IPs    (standard subnet)
/23 ─ 512 IPs
/22 ─ 1,024 IPs
/21 ─ 2,048 IPs
/20 ─ 4,096 IPs
/16 ─ 65,536 IPs (typical VPC)
/8 ── 16.7M IPs  (10.0.0.0/8 entire range)
```

Each step up: double the IPs. Each step down: halve them.

### Mental Model 3: The Pizza Analogy

A /16 network is a whole pizza (65,536 slices).

```
/16 = 1 whole pizza
/17 = cut in half (2 pieces, 32,768 each)
/18 = cut into 4
/24 = cut into 256 equal pieces
```

CIDR notation tells you: **how many times did you cut the pizza?** Each cut doubles the number of pieces but halves their size.

---

## 6. Real-World Mapping

### Linux

```bash
# View your IP addresses
ip addr show

# View the routing table
ip route show

# Query which route a specific IP would use
ip route get 10.0.5.100

# Add a static route
ip route add 192.168.100.0/24 via 10.0.0.1

# Delete a route
ip route del 192.168.100.0/24

# Check if two IPs are on the same subnet
# 10.0.1.50 and 10.0.1.100 — both in 10.0.1.0/24 → same subnet (direct)
# 10.0.1.50 and 10.0.2.50 — different /24 subnets → need a router

# Calculate subnet info (install ipcalc)
ipcalc 10.0.5.96/27
# Output: Network, HostMin, HostMax, Broadcast, Hosts/Net
```

### AWS

| Concept | IP/CIDR Aspect |
|---------|---------------|
| **VPC CIDR** | Choose at creation. /16 recommended. Can add secondary CIDRs later, but primary is permanent. |
| **Subnet CIDR** | Must be within VPC CIDR. /24 standard. /28 minimum. Each subnet = one AZ. |
| **Public vs Private** | Determined by route table, NOT by the CIDR. Both can have any CIDR. |
| **Security Group rules** | Reference CIDRs (10.0.0.0/16) or SG IDs. /32 for single IP. 0.0.0.0/0 for anywhere. |
| **VPC Peering** | Requires non-overlapping CIDRs. Plan CIDRs across all environments before creating VPCs. |
| **ENI** | Gets a primary private IP from the subnet's CIDR. Can get secondary IPs. Can get a public IP (if subnet has auto-assign enabled). |
| **Elastic IP** | Static public IP. Attached to ENI. Persists across stop/start. |

### Kubernetes

| Concept | CIDR Aspect |
|---------|------------|
| **Pod CIDR** | Cluster-wide range for pod IPs (e.g., 10.244.0.0/16). Each node gets a slice (e.g., /24 per node = 254 pods max per node). |
| **Service CIDR** | Range for ClusterIP Services (e.g., 10.96.0.0/12). Virtual — only exists in iptables rules. |
| **MUST NOT OVERLAP** | Pod CIDR ≠ Service CIDR ≠ VPC CIDR. If they overlap, routing breaks — packets meant for a pod could be routed to the VPC, or vice versa. |
| **AWS VPC CNI** | Pods get real VPC IPs from the node's subnet. Major implication: your subnet needs enough IPs for all pods on all nodes in that subnet. |
| **EKS max pods per node** | Limited by ENI count × IPs per ENI. t3.medium = 17 pods max. m5.xlarge = 58 pods max. This is an IP addressing constraint. |

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: VPC CIDR Overlap Prevents Peering

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Cannot create VPC Peering connection. Error: "overlapping CIDR blocks." |
| **Root Cause** | Both VPCs use the same CIDR (e.g., both are 10.0.0.0/16). |
| **OSI Layer** | L3 |
| **Debug Steps** | 1. Check both VPC CIDRs. 2. Check all secondary CIDRs. 3. Peering requires no overlap in any CIDR block. |
| **Fix** | Redesign one VPC's CIDR. If VPCs can't be changed (production), use PrivateLink instead (no CIDR overlap requirement). |
| **Prevention** | Plan CIDR allocations across ALL environments before creating anything. Use a CIDR registry/spreadsheet. |

### Scenario 2: Subnet Too Small for EKS Pods (AWS VPC CNI)

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Pods stuck in `Pending` with error "FailedCreatePodSandBox: too many addresses" or "InsufficientFreeAddresses." Nodes can't schedule more pods. |
| **Root Cause** | Subnet ran out of available IPs. With AWS VPC CNI, each pod consumes a real VPC IP from the subnet. A /24 (251 usable) with 10 nodes running 25 pods each = 250 IPs used. |
| **OSI Layer** | L3 |
| **Debug Steps** | 1. Check subnet available IPs in AWS console. 2. `kubectl describe node <node>` — look at "Allocatable" pods vs "allocated". 3. Check the IPAMD logs: `kubectl logs -n kube-system -l k8s-app=aws-node`. |
| **Fix** | 1. Use larger subnets (/20 = 4,091 usable). 2. Enable prefix delegation (ENABLE_PREFIX_DELEGATION=true — each ENI slot gets a /28 instead of a single IP). 3. Use secondary CIDRs. |

### Scenario 3: Pod CIDR Overlaps with VPC CIDR

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Pods can reach some VPC resources but not others. Random connectivity failures. Some pods get IPs that conflict with VPC resources. |
| **Root Cause** | Pod CIDR was configured with a range that overlaps the VPC CIDR. The kernel doesn't know whether 10.0.5.50 is a pod or a VPC resource. |
| **OSI Layer** | L3 |
| **Debug Steps** | 1. Compare: `kubectl cluster-info dump | grep -i cidr`. 2. Compare pod CIDR, service CIDR, and VPC CIDR. 3. Any overlap = routing ambiguity. |
| **Fix** | You cannot change pod CIDR after cluster creation. The cluster must be recreated with non-overlapping CIDRs. |

### Scenario 4: Security Group Rule Uses Wrong CIDR

| Attribute | Detail |
|-----------|--------|
| **Symptom** | An EC2 instance can access some services but not others in the same VPC. |
| **Root Cause** | SG inbound rule allows 10.0.1.0/24, but the calling instance is in 10.0.2.0/24. |
| **OSI Layer** | L3/L4 |
| **Debug Steps** | 1. Check source IP of the caller. 2. Check SG inbound rules — does the CIDR include the caller? 3. VPC Flow Logs — look for REJECT. |
| **Fix** | Use SG-to-SG references instead of CIDRs where possible (e.g., allow traffic from `sg-abc123`). This automatically includes any instance attached to that SG, regardless of its IP. |

---

## 8. Debugging Playbook

### "Is It an IP/Subnet Problem?" Checklist

```
Step 1: Verify source and destination IPs
─────────────────────────────────────────
$ ip addr show          # On source machine
$ kubectl get pod -o wide  # In K8s — check pod IP
→ Do both IPs look correct?
→ Are they in the expected CIDRs?

Step 2: Are they on the same subnet?
─────────────────────────────────────
→ Same CIDR range? → Direct communication (no routing needed)
→ Different CIDRs? → Needs a route between them

Step 3: Check route tables
──────────────────────────
$ ip route get <destination-ip>
→ Does a route exist for this destination?
→ AWS console: check subnet's route table

Step 4: Check for overlapping CIDRs
────────────────────────────────────
→ Pod CIDR vs VPC CIDR vs Service CIDR
→ VPC CIDRs across peered VPCs
→ Any overlap = routing ambiguity

Step 5: Check Security Group / NACL CIDR rules
───────────────────────────────────────────────
→ Does the SG allow the source CIDR?
→ Does the NACL allow the source CIDR?
→ Pro tip: Use SG IDs instead of CIDRs when possible
```

---

## 9. Hands-On Practice

### Lab 1: CIDR Mental Math Drill

Calculate these WITHOUT a calculator:

```
1. How many usable IPs in 10.0.0.0/24?   → 256 total, 251 usable (AWS)
2. How many usable IPs in 10.0.5.0/27?   → 32 total, 27 usable
3. How many usable IPs in 10.0.10.0/28?  → 16 total, 11 usable
4. A /25 subnet starts at 10.0.1.0. What's the last IP?  → 10.0.1.127
5. A /26 subnet starts at 10.0.1.64. What's the last IP? → 10.0.1.127
6. Can 10.0.1.0/24 and 10.0.1.0/25 coexist in the same VPC? → No, /25 is a subset of /24
```

**Practice until you can answer all in under 30 seconds.**

### Lab 2: Design a VPC From Scratch

**Task:** Design subnets for a VPC with CIDR `10.100.0.0/16`:

Requirements:
- 2 public subnets (different AZs) for ALBs
- 4 private subnets (2 app, 2 DB) across 2 AZs
- EKS cluster will need ~500 pod IPs per AZ
- Plan for future 3rd AZ

**Your design (write it out before looking at the solution):**

<details>
<summary>Solution</summary>

```
VPC: 10.100.0.0/16

Public (ALB/NAT):
├─ 10.100.0.0/24   — AZ-a (251 usable)
├─ 10.100.1.0/24   — AZ-b (251 usable)
├─ 10.100.2.0/24   — AZ-c (future)

Private App (EKS nodes + pods):
├─ 10.100.10.0/22  — AZ-a (1019 usable — enough for 500+ pods)
├─ 10.100.14.0/22  — AZ-b (1019 usable)
├─ 10.100.18.0/22  — AZ-c (future)

Private DB:
├─ 10.100.30.0/24  — AZ-a (251 usable)
├─ 10.100.31.0/24  — AZ-b (251 usable)
├─ 10.100.32.0/24  — AZ-c (future)

Note: App subnets are /22 (1024 IPs) to handle  
AWS VPC CNI pod IP allocation (500+ pods per AZ).
```
</details>

### Lab 3: Check Route Decisions

```bash
# View full routing table
ip route show

# Check which route a specific IP will use
ip route get 8.8.8.8
ip route get 10.0.5.100
ip route get 192.168.1.1

# Add a route and verify
sudo ip route add 192.168.100.0/24 via 10.0.0.1
ip route get 192.168.100.50

# Delete the route
sudo ip route del 192.168.100.0/24
```

### Lab 4: Verify No CIDR Overlaps in Kubernetes

```bash
# Get pod CIDR
kubectl cluster-info dump | grep -i "cluster-cidr"

# Get service CIDR
kubectl cluster-info dump | grep -i "service-cluster-ip-range"

# Get VPC CIDR (AWS)
aws ec2 describe-vpcs --query 'Vpcs[*].CidrBlock'

# Verify: None of these should overlap!
```

---

## 10. Interview Preparation

### Q1: (Screening) How many usable IPs in a /24 subnet on AWS?

**Answer:** 256 total - 5 reserved (network address, VPC router, DNS, future use, broadcast) = **251 usable IPs**.

### Q2: Design a VPC CIDR layout for a 3-tier architecture with 2 AZs.

**Answer:** VPC: 10.0.0.0/16. Public: 10.0.0.0/24 (AZ-a), 10.0.1.0/24 (AZ-b). App: 10.0.10.0/24, 10.0.11.0/24. DB: 10.0.20.0/24, 10.0.21.0/24. Leave gaps for future AZ-c and growth. For EKS with VPC CNI, use /20 or /22 for app subnets (pod IPs consume VPC IPs).

### Q3: Why must the pod CIDR and service CIDR not overlap with the VPC CIDR?

**Answer:** Because the Linux routing table must decide where to send packets. If a pod IP (10.0.5.50) is in the same range as a VPC resource (10.0.5.50), the kernel can't distinguish them. Packets meant for a pod route to the VPC, or vice versa. This causes intermittent, hard-to-debug connectivity failures.

### Q4: You have 6 VPCs that need to communicate. What CIDR constraint exists?

**Answer:** For VPC Peering: no overlapping CIDRs between any peered pair. For Transit Gateway: same constraint. This means all 6 VPCs must have non-overlapping CIDRs. Plan ahead: use 10.0.0.0/16, 10.1.0.0/16, 10.2.0.0/16, etc.

### Q5: What is a /32 route and when is it used?

**Answer:** /32 = exactly one IP address. Used for: (1) SG rules allowing a single IP, (2) host routes in routing tables (e.g., AWS VPC CNI creates /32 routes per pod IP), (3) Elastic IP associations. In Kubernetes, you'll see /32 routes for individual pod IPs on host routing tables.

### Q6: An EKS cluster keeps running out of pod IPs. Explain the problem and solution.

**Answer:** AWS VPC CNI assigns real VPC IPs to pods. Each node's ENI has a limited number of IP slots (varies by instance type). When all slots are used, no more pods can be scheduled. Solutions: (1) Use larger instance types (more ENIs = more IPs). (2) Enable prefix delegation (ENABLE_PREFIX_DELEGATION=true) — each slot gets a /28 (16 IPs) instead of 1, increasing pod density. (3) Use larger subnets to ensure IP availability. (4) Consider secondary CIDRs for pods.

### Q7: What is the difference between 10.0.0.0/8 and 10.0.0.0/16 in an SG rule?

**Answer:** /8 allows all IPs from 10.0.0.0 to 10.255.255.255 (16.7M addresses). /16 allows only 10.0.0.0 to 10.0.255.255 (65,536 addresses). Using /8 is much more permissive — it would allow traffic from any 10.x.x.x address. For security, use the most specific CIDR possible, or better yet, reference SG IDs.

---

## 11. Advanced Insights (Senior Level)

### CIDR Planning Across an Organization

A senior engineer plans CIDRs for ALL environments before creating anything:

```
Account: Production
├─ VPC us-east-1: 10.0.0.0/16
├─ VPC eu-west-1: 10.1.0.0/16
│
Account: Staging
├─ VPC us-east-1: 10.2.0.0/16
│
Account: Development
├─ VPC us-east-1: 10.3.0.0/16
│
Account: Shared Services (CI/CD, monitoring)
├─ VPC us-east-1: 10.4.0.0/16
│
On-premises:
├─ 10.10.0.0/16 (Direct Connect)
│
EKS Pod CIDRs:
├─ 100.64.0.0/16 (per cluster, from secondary CIDR)
```

**Key considerations:**
- All VPCs are non-overlapping → can peer any pair
- On-premises is non-overlapping → can use Direct Connect or VPN
- EKS pod CIDRs use 100.64.0.0/10 (CGNAT range) as secondary CIDRs to preserve VPC IPs
- A CIDR registry document is maintained and consulted before every VPC creation

### IPv6 in AWS

AWS increasingly supports IPv6 dual-stack:
- VPCs can have both IPv4 and IPv6 CIDRs
- IPv6 addresses are globally unique (no NAT needed)
- Egress-only IGW replaces NAT GW for IPv6
- Security Groups support IPv6 rules

Senior engineers plan for eventual IPv6 adoption, especially for IoT workloads.

### Secondary CIDRs

If your VPC runs out of IPs, you can add secondary CIDR blocks:
- Up to 5 IPv4 CIDRs per VPC
- Must not overlap with existing CIDRs or peered VPCs
- Common pattern: primary CIDR for nodes, secondary CIDR (100.64.0.0/16) for EKS pods with custom networking

---

## 12. Common Mistakes

### Mistake 1: Choosing a VPC CIDR that's too small
A /20 gives you 4,096 IPs. Sounds like a lot until you run EKS with VPC CNI and each pod needs a VPC IP. **Always use /16 for production VPCs.** It costs nothing extra.

### Mistake 2: Forgetting AWS reserves 5 IPs per subnet
A /28 has 16 IPs, but only 11 are usable. New engineers forget this and are surprised when they can't launch instances in a "16 IP" subnet.

### Mistake 3: Not planning CIDR allocation before VPC creation
Creating VPCs with random CIDRs → discovering you can't peer them later because they overlap. You can't change a VPC's primary CIDR after creation.

### Mistake 4: Using 192.168.x.x for production VPCs
192.168.0.0/16 gives only 65,536 IPs — the same as a single 10.x.0.0/16. The 10.0.0.0/8 range gives 16.7M IPs. Use 10.x.x.x for production.

### Mistake 5: Confusing /24 for "256 usable IPs"
256 total, 251 usable on AWS (5 reserved), 254 usable in standard networking (2 reserved: network + broadcast). Always clarify context.

---

## 13. Cheat Sheet

### CIDR Quick Reference

```
/32 = 1 IP        /28 = 16 IPs (min AWS)
/31 = 2 IPs       /27 = 32 IPs
/30 = 4 IPs       /26 = 64 IPs
/29 = 8 IPs       /25 = 128 IPs
                   /24 = 256 IPs (standard)
/20 = 4,096 IPs   /16 = 65,536 IPs (VPC)
```

### Private IP Ranges

```
10.0.0.0/8      — 16.7M IPs  (use for VPCs)
172.16.0.0/12   — 1M IPs     (Docker default)
192.168.0.0/16  — 65K IPs    (home/lab)
```

### AWS Subnet Rules

```
5 IPs reserved per subnet (first 4 + last)
/28 = smallest subnet (11 usable)
/16 = recommended VPC size
Secondary CIDRs can be added later (up to 5 total)
```

### Essential Commands

```bash
ip addr show                    # View IPs
ip route show                   # View routes
ip route get <ip>               # Query routing decision
ipcalc 10.0.5.96/27            # Calculate subnet details
```

### The Golden Rule

**Plan your CIDRs before you create anything.** Use a spreadsheet. Document every VPC, subnet, peering connection, and Kubernetes cluster CIDR. Changing CIDRs after deployment is extremely painful.

---

## 🔗 Connections

### How This Connects to Previous Topics
**OSI Model (Topic 1.1):** IP addressing is **Layer 3 (Network)**. The OSI model taught you that L3 is where routing and IP addresses live — now you know how those addresses actually work.

**TCP (Topic 1.2):** TCP connects to a destination identified by `IP:Port`. The IP comes from what you just learned. A full connection = `(Source IP, Source Port, Destination IP, Destination Port, Protocol)`. You now understand both the IP part (L3) and the port part (L4).

### What This Prepares You For Next
**Next topic: DNS — The Thing That Breaks Most "Networking" Issues**

You now know that packets need IP addresses to reach their destination. But humans don't type IPs — they type domain names like `google.com` or `my-service.default.svc.cluster.local`. DNS translates names to IPs. When DNS breaks, nothing works — even though the network itself is fine. DNS is the most common cause of "networking" issues in production, and understanding it requires knowing about IPs (which you now do) and about TTL, caching, and resolution chains.
