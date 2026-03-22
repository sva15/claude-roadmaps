# Linux Routing Tables and How Packets Find Their Destination

> 🔹 **Phase:** 2 — Linux Networking  
> 🔹 **Topic:** Linux Routing Tables  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

When a packet needs to go somewhere, the Linux kernel asks: **"Which door (interface) should this packet go through, and who should I hand it to next?"**

The **routing table** is the answer sheet. It's a list of rules that says:
- "Packets for 10.0.1.0/24 → send through eth0 directly (same subnet)"
- "Packets for 10.0.5.0/24 → send through eth0 to gateway 10.0.1.1"
- "Everything else → send to the default gateway"

### The Analogy: Highway Signs

Routing tables are like highway exit signs:
- "Exit 5: Downtown (10.0.1.0/24)" → take this exit
- "Exit 12: Airport (10.0.5.0/24)" → take this exit
- "All other destinations → stay on the highway (default route)"

**Longest prefix match** = the most specific sign wins. If there's a sign for "Downtown" AND a sign for "Downtown East District," a car going to the East District takes the more specific sign.

### Why Does It Exist?

Every networked device — your laptop, an EC2 instance, a K8s node, a VPC — must decide where to send each packet. The routing table makes this decision. It's the same fundamental algorithm everywhere:
- `ip route` on Linux
- VPC route tables in AWS
- BGP routing on the internet

> 💡 **Key Insight:** AWS VPC route tables are just a cloud-managed version of Linux routing tables. Same longest-prefix-match logic. Same concept of a default route. If you understand `ip route`, you understand VPC route tables.

---

## 2. Why This Matters in DevOps

| Scenario | Routing Concept |
|----------|----------------|
| AWS VPC "local" route | Automatic route for intra-VPC traffic (like the connected route on Linux) |
| Private subnet → NAT GW | 0.0.0.0/0 → NAT Gateway in the route table |
| Public subnet → IGW | 0.0.0.0/0 → Internet Gateway in the route table |
| K8s pod routing | CNI creates routes: pod_IP/32 → veth interface |
| VPC Peering traffic not flowing | Missing route in the route table pointing to the peering connection |
| "No route to host" error | Routing table has no entry matching the destination |
| Policy routing in K8s | Multiple routing tables selected by `ip rule` |

---

## 3. Core Concept (Deep Dive)

### Reading the Routing Table

```bash
$ ip route show

default via 10.0.1.1 dev eth0 proto dhcp metric 100
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.50
10.244.1.0/24 dev cni0 proto kernel scope link src 10.244.1.1
10.244.2.0/24 via 10.0.1.100 dev eth0
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

**Decoding each line:**

| Route | Meaning |
|-------|---------|
| `default via 10.0.1.1 dev eth0` | Default route: anything not matching another rule goes to gateway 10.0.1.1 through eth0 |
| `10.0.1.0/24 dev eth0 scope link` | Connected route: 10.0.1.x is directly reachable on eth0 (no gateway needed — use ARP) |
| `10.244.1.0/24 dev cni0` | Pod subnet: pods in 10.244.1.x are reachable via the cni0 bridge (local pods) |
| `10.244.2.0/24 via 10.0.1.100` | Remote pod subnet: pods in 10.244.2.x are on node 10.0.1.100 (cross-node pod traffic) |
| `172.17.0.0/16 dev docker0` | Docker: containers in 172.17.x.x are reachable via docker0 bridge |

### Longest Prefix Match — The Core Algorithm

When the kernel needs to route a packet to `10.0.1.50`:

```
Route Table:
  0.0.0.0/0       via 10.0.1.1    ← Prefix length: 0  (matches everything)
  10.0.0.0/8      via 10.0.0.1    ← Prefix length: 8  (matches)
  10.0.1.0/24     dev eth0        ← Prefix length: 24 (matches) ← WINNER

The /24 route wins because 24 > 8 > 0. Most specific match.
```

**This is the exact same algorithm used by:**
- Linux kernel
- AWS VPC route tables
- Internet BGP routers
- Kubernetes CNI routing

### Route Types

```
Connected routes (automatically created):
  When you assign 10.0.1.50/24 to eth0, the kernel automatically
  adds: 10.0.1.0/24 dev eth0 scope link
  → "Anything in 10.0.1.x is directly reachable on eth0"

Default route:
  0.0.0.0/0 via <gateway>
  → "If nothing else matches, send here"
  → This is the "door to the internet" (via NAT GW, IGW, or router)

Static routes:
  Added manually or by CNI plugins:
  ip route add 10.244.2.0/24 via 10.0.1.100
  → "Pods in 10.244.2.x are on node 10.0.1.100"

Host routes (/32):
  10.244.1.5/32 dev veth1234
  → "This specific pod IP is reachable through this specific veth"
  → AWS VPC CNI creates one /32 route per pod
```

### Querying the Routing Decision

```bash
# Which route will be used for a specific destination?
$ ip route get 8.8.8.8
8.8.8.8 via 10.0.1.1 dev eth0 src 10.0.1.50 uid 1000
    cache

# Tells you:
# → Gateway: 10.0.1.1
# → Interface: eth0
# → Source IP: 10.0.1.50
```

### Mapping to AWS VPC Route Tables

```
Linux Route Table               AWS VPC Route Table
────────────────────            ────────────────────
10.0.0.0/16 dev eth0  ◄═══►    10.0.0.0/16 → local
    (connected route)                (auto, can't delete)

0.0.0.0/0 via 10.0.1.1 ◄═══►  0.0.0.0/0 → igw-xxxx (public)
                                0.0.0.0/0 → nat-xxxx (private)

10.1.0.0/16 via peer   ◄═══►  10.1.0.0/16 → pcx-xxxx (peering)

10.244.0.0/16 via node ◄═══►  (CNI handles pod routing)
```

---

## 4. Internal Working (Under the Hood)

### Kernel Routing Decision Flow

```
Packet arrives at the kernel:
│
├── 1. Check destination IP
├── 2. Lookup routing table (FIB — Forwarding Information Base)
│      → Longest prefix match
│      → Determine: output interface + next-hop
├── 3. If next-hop is a gateway:
│      → ARP for gateway's MAC address
│      → Send to gateway (who forwards further)
├── 4. If destination is on a connected subnet:
│      → ARP for destination's MAC directly
│      → Send frame directly on the interface
├── 5. If no route matches:
│      → Drop packet
│      → Return ICMP "Network unreachable" or "No route to host"
│
└── Between steps 2 and 4: netfilter hooks fire (iptables)
    → PREROUTING → routing decision → FORWARD/INPUT
    → This is where iptables can intercept and modify packets
```

### Policy Routing (Advanced)

Linux supports **multiple routing tables** selected by rules:

```bash
# View routing rules (priorities)
$ ip rule show
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default

# The kernel checks rules in order:
# 1. "local" table — loopback, broadcast addresses
# 2. "main" table — what ip route show displays
# 3. "default" table — rarely used

# You can add custom rules:
ip rule add from 10.244.1.0/24 table 100
ip route add default via 10.0.1.1 table 100

# → "Any packet from pod subnet 10.244.1.0/24 uses routing table 100"
# → This is how some CNIs route pod traffic differently from host traffic
```

---

## 5. Mental Models

### Mental Model 1: The GPS with Preference for Specific Routes

```
Routing table = GPS navigation with multiple route options:

1. "10.0.1.0/24 → Take local road" (connected, most specific)
2. "10.0.0.0/16 → Take the highway" (broader match)
3. "0.0.0.0/0 → Take any major road to the interstate" (default)

GPS picks the most specific route that matches your destination.
That's longest prefix match.
```

### Mental Model 2: VPC as a Linux Router

```
A VPC is just a big Linux router managed by AWS:

  Subnet route table = one Linux routing table per subnet
  Local route = connected route (auto-created, always there)
  IGW route = default gateway pointing to the internet
  NAT GW route = default gateway pointing to NAT (SNAT)
  Peering route = static route pointing to the peer VPC

Same concepts. Same logic. Different management interface (AWS Console vs ip route).
```

---

## 6. Real-World Mapping

### Linux Commands

```bash
# View routing table
ip route show
ip route list

# Query routing for a specific destination
ip route get 10.0.5.100
ip route get 8.8.8.8

# Add a static route
ip route add 192.168.100.0/24 via 10.0.1.1
ip route add 10.244.2.0/24 via 10.0.1.100 dev eth0

# Add a host route (/32)
ip route add 10.244.1.5/32 dev veth1234

# Delete a route
ip route del 192.168.100.0/24

# Add a default route
ip route add default via 10.0.1.1

# Replace default route
ip route replace default via 10.0.1.2

# Show route cache / FIB
ip route show cache

# Traceroute — show each hop (each routing decision)
traceroute 8.8.8.8
mtr --report 8.8.8.8
```

### AWS VPC Route Tables

| VPC Route Entry | Linux Equivalent | Purpose |
|----------------|-----------------|---------|
| 10.0.0.0/16 → local | 10.0.0.0/16 dev eth0 scope link | Intra-VPC traffic |
| 0.0.0.0/0 → igw-xxx | default via gateway | Internet access (public) |
| 0.0.0.0/0 → nat-xxx | default via nat-gateway | Internet access (private, outbound only) |
| 10.1.0.0/16 → pcx-xxx | 10.1.0.0/16 via peering-ip | VPC peering traffic |
| 0.0.0.0/0 → tgw-xxx | default via transit-gw | Transit Gateway routing |
| pl-xxx → vpce-xxx | (prefix list route) | VPC endpoint (S3, DynamoDB) |

### Kubernetes Routing

```
On a K8s node with AWS VPC CNI:
$ ip route
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0 scope link
10.0.1.50 dev eniXXX scope link    ← Pod IP via ENI
10.0.1.51 dev eniXXX scope link    ← Another pod IP
10.0.1.52 dev eniYYY scope link    ← Pod on different ENI

On a K8s node with Flannel (overlay):
$ ip route
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0 scope link
10.244.0.0/24 dev cni0              ← Local pods (this node)
10.244.1.0/24 via 10.244.1.0 dev flannel.1  ← Remote pods (other node)
10.244.2.0/24 via 10.244.2.0 dev flannel.1  ← Remote pods (another node)
```

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: "No Route to Host"

| Attribute | Detail |
|-----------|--------|
| **Symptom** | `ping: connect: No route to host` or `Network is unreachable` |
| **Root Cause** | No routing table entry matches the destination IP |
| **Debug Steps** | 1. `ip route get <destination>` — what route does the kernel select? 2. If "unreachable" → no route exists. 3. Check: is there a default route? `ip route | grep default`. 4. In AWS: check subnet's route table — is there a route for this CIDR? |

### Scenario 2: VPC Peering Traffic Not Flowing

| Attribute | Detail |
|-----------|--------|
| **Symptom** | EC2 in VPC-A can't reach EC2 in VPC-B even though peering connection exists |
| **Root Cause** | Route table entries missing. Peering connection exists but no routes point to it. |
| **Debug Steps** | 1. Check VPC-A's route table: is there a route `VPC-B-CIDR → pcx-xxx`? 2. Check VPC-B's route table: is there a route `VPC-A-CIDR → pcx-xxx`? 3. BOTH sides need routes. This is the #1 VPC Peering mistake. |

### Scenario 3: Asymmetric Routing Causing Drops

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Connections intermittently fail. Packets going out one interface, responses coming in another. |
| **Root Cause** | Multiple interfaces/gateways. Request goes out eth0 but response tries to come back through eth1. Stateful firewalls drop it because they didn't see the original request on eth1. |
| **Debug Steps** | 1. `ip route get <dest>` — check which interface is used. 2. Check for multiple default routes (`ip route | grep default`). 3. Use policy routing to ensure symmetric paths. |

### Scenario 4: Black Hole Route

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Packets silently disappear. No error, no timeout, just gone. |
| **Root Cause** | Route exists but points to a dead-end (removed NAT GW, deleted peering, interface that's DOWN). |
| **Debug Steps** | 1. `ip route get <dest>` — is the route valid? 2. Is the next-hop reachable? `ping <gateway>`. 3. In AWS: does the target of the route still exist? (NAT GW deleted, peering connection removed). |

---

## 8. Debugging Playbook

```
Routing Problem? Follow this:

Step 1: What route is being used?
──────────────────────────────────
$ ip route get <destination-ip>
→ Shows: via (gateway), dev (interface), src (source IP)
→ If "unreachable" → no route exists

Step 2: Does a route exist?
────────────────────────────
$ ip route show | grep <network>
→ Is there a specific route? Default route?
→ AWS: check subnet's route table in console

Step 3: Is the gateway reachable?
──────────────────────────────────
$ ping <gateway-ip>
→ If gateway is unreachable → gateway is the problem

Step 4: Trace the path
────────────────────────
$ traceroute <destination>
→ Shows each hop. Where does it stop? That's where routing breaks.

Step 5: Check for missing routes (VPC Peering)
───────────────────────────────────────────────
→ Routes must exist on BOTH sides
→ AWS makes you add them manually after creating peering
```

---

## 9. Hands-On Practice

### Lab 1: Explore Your Routing Table

```bash
ip route show
ip route get 8.8.8.8
ip route get 10.0.1.100
ip route get 127.0.0.1
# Compare the gateway and interface for each
```

### Lab 2: Add and Remove Static Routes

```bash
# Add a route
sudo ip route add 192.168.100.0/24 via 10.0.0.1

# Verify
ip route get 192.168.100.50

# Delete
sudo ip route del 192.168.100.0/24

# Verify it's gone
ip route get 192.168.100.50  # Should fall back to default
```

### Lab 3: Trace Network Path

```bash
# See every hop between you and the destination
traceroute 8.8.8.8

# Better: continuous measurement with packet loss stats
mtr --report 8.8.8.8
```

---

## 10. Interview Preparation

### Q1: A packet is destined for 10.0.5.100. Routes exist for 10.0.0.0/8 and 10.0.5.0/24. Which wins?

**Answer:** 10.0.5.0/24 wins. Longest prefix match: /24 (24 bits) is more specific than /8 (8 bits). The routing table always picks the most specific match. This is the fundamental routing algorithm used everywhere — Linux, AWS, BGP.

### Q2: What is the default route and what happens without one?

**Answer:** The default route (0.0.0.0/0) is the "catch-all" — packets that don't match any more specific route go here. Without a default route, the kernel drops packets for unknown destinations and returns "Network is unreachable." In AWS, a private subnet's default route points to NAT GW; a public subnet's points to IGW.

### Q3: VPC peering is established but traffic doesn't flow. What's wrong?

**Answer:** The most common mistake: route table entries are missing on one or both sides. Creating a VPC peering connection does NOT automatically add routes. You must manually add a route in VPC-A's route table pointing VPC-B's CIDR to the peering connection, AND vice versa. Also check: Security Groups, NACLs, and that CIDRs don't overlap.

### Q4: How does VPC routing relate to Linux routing?

**Answer:** They're the same algorithm. VPC route tables use longest prefix match, have a "local" route (connected route), support default routes, and custom static routes. The VPC is essentially a managed Linux router. Understanding `ip route` means you understand VPC route tables — just different management interface.

---

## 11. Advanced Insights (Senior Level)

### Source-Based Routing (Policy Routing)

On multi-homed instances (multiple ENIs):
```bash
# Problem: traffic from ENI-2 must return through ENI-2, not ENI-1
# Solution: policy routing

# Create a custom routing table for ENI-2
echo "100 eni2table" >> /etc/iproute2/rt_tables

# Add routes to custom table
ip route add 10.0.2.0/24 dev eth1 table eni2table
ip route add default via 10.0.2.1 table eni2table

# Rule: traffic from ENI-2's IP uses the custom table
ip rule add from 10.0.2.50 table eni2table
```

### Route Metrics and Multipath

```bash
# Two default routes with different metrics (priority):
ip route add default via 10.0.1.1 metric 100  # Primary
ip route add default via 10.0.2.1 metric 200  # Backup

# ECMP (Equal-Cost Multipath) for load balancing:
ip route add default nexthop via 10.0.1.1 weight 1 nexthop via 10.0.2.1 weight 1
```

---

## 12. Common Mistakes

### Mistake 1: Forgetting routes on BOTH sides of peering
VPC peering requires routes in both VPCs. This is the #1 peering mistake.

### Mistake 2: Confusing "connected route" with "static route"
Connected routes appear automatically when you assign an IP. Static routes must be added explicitly. Deleting an IP removes its connected route.

### Mistake 3: Not checking the default route
"No internet access" → first check: `ip route | grep default`. Missing default route is the most basic routing problem.

### Mistake 4: Adding routes that don't persist across reboot
`ip route add` is temporary. For persistence, configure in `/etc/network/interfaces`, netplan, or NetworkManager.

---

## 13. Cheat Sheet

```bash
ip route show                         # View all routes
ip route get <ip>                     # Query routing decision
ip route add <cidr> via <gw>          # Add static route
ip route add <cidr> dev <iface>       # Add connected route
ip route del <cidr>                   # Delete route
ip route add default via <gw>         # Set default gateway
ip route replace default via <gw>     # Replace default
traceroute <ip>                       # Trace route path
mtr --report <ip>                     # Better traceroute
ip rule show                          # View policy routing rules
```

### Routing Decision Flow

```
Packet → Routing Table Lookup → Longest Prefix Match → Output Interface + Gateway
                                     │
                     No match? → "No route to host" → DROP
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**IP Addressing (Topic 1.3):** Routes use CIDRs. The subnet math you learned (/24, /16, longest prefix match) is exactly how the routing algorithm works.

**Network Interfaces (Topic 2.1):** Routes specify which interface to use. Each interface has connected routes automatically. Understanding interfaces tells you what doors exist; routing tells you which door to use for each destination.

### What This Prepares You For Next
**Next topic: iptables — The Firewall That Runs Everything**

Routing decides WHERE to send the packet. But BEFORE the packet goes there, iptables can intercept it — block it, modify it, redirect it. Kubernetes Services are implemented entirely in iptables (DNAT rules). Understanding routing first is essential because iptables hooks fire at specific points in the routing path (PREROUTING, FORWARD, POSTROUTING).
