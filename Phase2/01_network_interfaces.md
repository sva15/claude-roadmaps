# Network Interfaces and the `ip` Command

> 🔹 **Phase:** 2 — Linux Networking  
> 🔹 **Topic:** Network Interfaces and the `ip` Command  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

A computer can have multiple network connections: a wired Ethernet, a Wi-Fi adapter, a VPN tunnel, and virtual interfaces for containers. Each of these is a **network interface** — a point where the machine connects to a network.

Every packet entering or leaving the machine goes through an interface. If you don't know what interfaces exist, what IPs they have, and whether they're up or down, you can't debug any network problem.

### The Analogy: Doors of a Building

Think of your machine as a building. Each **network interface** is a **door**:
- `eth0` — The main entrance (primary network)
- `lo` — The mail slot (loopback — internal delivery only)
- `veth0` — A service entrance (virtual link to a container)
- `docker0` — An internal lobby (bridge connecting containers)
- `tun0` — A secret tunnel (VPN)

Each door has an address nailed to it (IP address) and can be open (UP) or locked (DOWN).

### Why Does It Exist?

The `ip` command (from the `iproute2` package) replaced the older `ifconfig` and `route` commands. It's the modern, canonical tool for managing network interfaces, addresses, routes, and tunnels on Linux. Every Kubernetes node, every Docker host, every EC2 instance uses these interfaces — and the `ip` command is how you inspect and manage them.

> 💡 **Key Insight:** On a Kubernetes node, `ip addr` shows dozens of `veth` interfaces — one for each pod. Understanding what you're seeing in `ip addr` output is the difference between "the network looks weird" and "I can see each pod's virtual interface and trace how its traffic flows."

---

## 2. Why This Matters in DevOps

| Scenario | Interface Concept |
|----------|------------------|
| K8s pod can't communicate | Check veth pair — is the interface UP? Correct IP? |
| EC2 instance has no connectivity | Check `eth0` — is it UP? Does it have an IP (DHCP)? |
| Container can't reach the host | Check `docker0` bridge — is it configured? |
| MTU mismatch dropping packets | Check `ip link` — MTU on all interfaces in the path |
| VPN not working | Check `tun0` — does the tunnel interface exist? |
| Node has multiple IPs | Multiple ENIs on AWS → multiple interfaces → routing matters |

---

## 3. Core Concept (Deep Dive)

### Types of Network Interfaces

```
Physical Interfaces:
├── eth0, ens5, enp0s3    ← Real NICs (ethernet)
├── wlan0                  ← Wireless
└── lo                     ← Loopback (127.0.0.1, always present)

Virtual Interfaces:
├── veth pairs             ← Virtual Ethernet (one end in container, one on host)
├── docker0                ← Bridge (connects containers to host network)
├── cni0, cbr0             ← K8s CNI bridges
├── flannel.1              ← VXLAN tunnel (Flannel CNI)
├── cali*                  ← Calico interfaces
├── tun0, wg0              ← VPN tunnels (OpenVPN, WireGuard)
└── dummy0                 ← Dummy interface for testing
```

### Reading `ip addr` Output

```bash
$ ip addr show eth0

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0a:1b:2c:3d:4e:5f brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.50/24 brd 10.0.1.255 scope global dynamic eth0
       valid_lft 3500sec preferred_lft 3500sec
    inet6 fe80::81b:2cff:fe3d:4e5f/64 scope link
       valid_lft forever preferred_lft forever
```

**Decoding each line:**

| Part | Meaning |
|------|---------|
| `2: eth0:` | Interface index 2, name `eth0` |
| `<BROADCAST,MULTICAST,UP,LOWER_UP>` | Flags: UP = administratively enabled, LOWER_UP = cable connected |
| `mtu 9001` | Maximum Transmission Unit = 9001 bytes (jumbo frames, common on AWS) |
| `state UP` | Interface is operational |
| `link/ether 0a:1b:2c:3d:4e:5f` | MAC address (L2) |
| `inet 10.0.1.50/24` | IPv4 address with /24 subnet |
| `scope global` | Routable address (vs. `link` = local only, `host` = loopback) |
| `dynamic` | Assigned via DHCP |

### veth Pairs — The Container Networking Primitive

A **veth pair** is two virtual interfaces connected like a tube — anything that enters one end comes out the other:

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│     Container / Pod          │     │        Host                  │
│                              │     │                              │
│   eth0 (10.244.1.5)   ◄═══════════►   veth1234@if5              │
│   (inside namespace)         │     │   (on host network)          │
│                              │     │                              │
└─────────────────────────────┘     └─────────────────────────────┘
```

When a pod sends a packet:
1. Packet goes out the pod's `eth0`
2. Arrives on the host's `veth1234`
3. Host routing table decides where to send it next
4. If destination is another pod on the same node → route to that pod's veth
5. If destination is external → route through `eth0` (or tunnel)

### Bridge Interfaces

A **bridge** connects multiple interfaces at L2 (like a virtual switch):

```
┌────────────────────────────────────────________________──┐
│                    docker0 / cni0 (bridge)                │
│                    IP: 172.17.0.1                         │
│                                                          │
│  veth-pod1    veth-pod2    veth-pod3    veth-pod4        │
│  10.244.1.2   10.244.1.3   10.244.1.4   10.244.1.5     │
└──────────────────────────────────────────────────────────┘
```

All pods connected to the bridge can talk to each other directly (L2 switching). Traffic to outside the bridge goes through the bridge's IP and the host's routing table.

### MTU — Maximum Transmission Unit

MTU is the maximum packet size an interface will send. Default is **1500 bytes** for Ethernet.

```
AWS EC2 default:     MTU 9001 (jumbo frames within VPC)
Standard Ethernet:   MTU 1500
VXLAN overlay:       MTU 1450 (encapsulation overhead takes 50 bytes)
VPN tunnels:         MTU 1400-1420 (IPsec/WireGuard overhead)
```

> 🚨 **MTU mismatch causes silent failures:** If one link has MTU 1500 and another has MTU 1400, packets between 1400-1500 bytes get dropped. Small requests work (under 1400); large responses fail. This is incredibly hard to diagnose without checking MTU explicitly.

---

## 4. Internal Working (Under the Hood)

### How the Kernel Manages Interfaces

```
1. Physical NIC driver registers interface (e.g., eth0)/
   Or: virtual interface created via ip link add
2. Kernel creates a net_device structure:
   - Name, MAC address, MTU, flags (UP/DOWN)
   - Statistics (bytes/packets received/transmitted)
   - Queues (transmit and receive ring buffers)
3. Interface is DOWN by default → set UP with ip link set dev X up
4. IP address assigned (DHCP or static) → creates route entries
5. Packets matched to interface by routing table decision
6. For veth pairs: kernel directly connects the two ends
   - Write to one end → read from the other end
   - No physical NIC involved — pure kernel memory copy
```

### Interface on a K8s Node (What You Actually See)

```bash
$ ip addr | grep -E "^[0-9]|inet " | head -30

1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.0.1.50/24          ← Node's VPC IP
3: eni1234@if3: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.0.1.100/32         ← Secondary ENI IP
5: vethaa11bb22@if2: <BROADCAST,MULTICAST,UP,LOWER_UP>
                                ← Pod 1's veth (host end)
7: vethcc33dd44@if2: <BROADCAST,MULTICAST,UP,LOWER_UP>
                                ← Pod 2's veth (host end)
9: vethee55ff66@if2: <BROADCAST,MULTICAST,UP,LOWER_UP>
                                ← Pod 3's veth (host end)
```

Each `veth` interface corresponds to a pod. The `@if2` part tells you which interface index this veth is paired with (inside the pod's namespace, it appears as `eth0`).

---

## 5. Mental Models

### Mental Model 1: Doors with Name Tags

```
Your Linux machine is a building:

[Door: lo]        → Mail slot. Only delivers inside the building (127.0.0.1)
[Door: eth0]      → Main entrance. IP: 10.0.1.50. Leads to the VPC network.
[Door: veth-pod1] → Service entrance to Room 1 (Pod 1). 
[Door: veth-pod2] → Service entrance to Room 2 (Pod 2).
[Door: docker0]   → Internal lobby connecting all rooms.

ip addr = "Show me all doors and their name tags"
ip link = "Show me all doors and whether they're open or locked"
```

### Mental Model 2: The Cable Analogy for veth

```
veth pair = An Ethernet cable with one end plugged into 
            the pod and the other end plugged into the host's 
            virtual switch (bridge).

Creating a veth pair = crimping a new cable
Moving one end to a namespace = plugging it into a different room
Deleting the pair = cutting the cable
```

---

## 6. Real-World Mapping

### Linux

```bash
# Show all interfaces with IPs
ip addr show
ip -br addr          # Brief format (cleaner)

# Show only interface status (UP/DOWN)
ip link show
ip -br link          # Brief format

# Show specific interface
ip addr show eth0

# Bring interface UP/DOWN
ip link set dev eth0 up
ip link set dev eth0 down

# Set IP address
ip addr add 10.0.1.50/24 dev eth0
ip addr del 10.0.1.50/24 dev eth0

# Change MTU
ip link set dev eth0 mtu 1400

# Create a veth pair
ip link add veth0 type veth peer name veth1

# Move interface to a namespace
ip link set veth1 netns mypod

# Show all veth interfaces (K8s node)
ip link | grep veth

# Count pods by counting veths
ip link | grep -c veth

# Show interface statistics
ip -s link show eth0

# Show ARP table (L2 neighbor discovery)
ip neigh show
```

### AWS

| AWS Entity | Linux Interface Equivalent |
|-----------|---------------------------|
| **Primary ENI** | `eth0` — every EC2 instance has one. Gets the primary private IP. |
| **Secondary ENI** | Additional `ethN` interfaces. Can be in different subnets. Common for multi-homed instances. |
| **ENI IP addresses** | Each ENI can have multiple private IPs. AWS VPC CNI assigns these to pods. |
| **EFA (Elastic Fabric Adapter)** | High-performance network interface for HPC workloads. Shows as a special interface. |

### Kubernetes

| K8s Entity | Interface Concept |
|-----------|-------------------|
| **Pod** | Gets its own network namespace with `eth0` inside (veth pair to host) |
| **Node** | Has `eth0` (VPC IP) + many `veth` interfaces (one per pod) |
| **CNI bridge** | `cni0` or `cbr0` — bridge connecting pod veth interfaces |
| **Overlay** | `flannel.1` (VXLAN), `vxlan.calico` — tunnel interfaces for cross-node pod traffic |
| **AWS VPC CNI** | No bridge, no overlay — each pod gets a VPC IP via ENI secondary IPs. `/32` routes per pod. |

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: Interface is DOWN

| Attribute | Detail |
|-----------|--------|
| **Symptom** | No connectivity on an interface. `ping` from/to that interface fails. |
| **Root Cause** | Interface is administratively DOWN or cable disconnected (LOWER_UP missing). |
| **Debug Steps** | 1. `ip link show eth0` — check for `UP` and `LOWER_UP` flags. 2. If no `UP` → `ip link set eth0 up`. 3. If no `LOWER_UP` → physical cable issue or hypervisor problem. |

### Scenario 2: MTU Mismatch Causing Packet Loss

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Small requests (ping, short HTTP) work. Large transfers fail or hang. HTTPS handshake works but data transfer stalls. |
| **Root Cause** | MTU on one segment is smaller than on another. Packets over the smaller MTU get silently dropped. |
| **OSI Layer** | L2 |
| **Debug Steps** | 1. `ip link show` on all interfaces in the path — check MTU values. 2. `ping -M do -s 1472 <dest>` — test with large packets (don't fragment). 3. If ping with large packets fails → MTU is the issue. 4. Reduce MTU: `ip link set dev eth0 mtu 1400`. |
| **Common in** | VPN tunnels (MTU 1400-1420), VXLAN overlays (MTU 1450), cross-region traffic through NAT/tunnel. |

### Scenario 3: Missing veth Interface for a Pod

| Attribute | Detail |
|-----------|--------|
| **Symptom** | K8s pod is in `Running` state but has no network connectivity. |
| **Root Cause** | CNI failed to set up the veth pair. Pod's network namespace is missing its interface. |
| **Debug Steps** | 1. `kubectl exec <pod> -- ip addr` — does eth0 exist inside the pod? 2. `kubectl exec <pod> -- ip link` — is eth0 UP? 3. On the node: `ip link | grep veth` — is there a corresponding veth? 4. Check CNI plugin logs: `journalctl -u kubelet | grep -i cni`. |

### Scenario 4: Duplicate IP Address

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Intermittent connectivity. Sometimes works, sometimes doesn't. ARP conflicts in logs. |
| **Root Cause** | Two interfaces/machines have the same IP address. |
| **Debug Steps** | 1. `ip neigh show` — look for "FAILED" entries or flip-flopping MACs. 2. `arping -D -I eth0 <suspect-ip>` — duplicate detection. 3. Check DHCP server for conflicts. |

---

## 8. Debugging Playbook

```
Network Interface Debug:

Step 1: Does the interface exist?
──────────────────────────────────
$ ip -br link
→ List all interfaces and their states
→ Missing interface? → CNI/driver problem

Step 2: Is it UP?
──────────────────
$ ip link show <iface>
→ Check for UP flag → if DOWN: ip link set <iface> up
→ Check for LOWER_UP → if missing: physical/hypervisor issue

Step 3: Does it have an IP?
────────────────────────────
$ ip addr show <iface>
→ No IP? → DHCP failed or manual config needed
→ Wrong IP? → ip addr add <ip>/<prefix> dev <iface>

Step 4: Check MTU
──────────────────
$ ip link show <iface> | grep mtu
→ Match with other interfaces in the path
→ Overlay/VPN should have smaller MTU than physical

Step 5: Check statistics
─────────────────────────
$ ip -s link show <iface>
→ TX errors? RX drops? → Hardware or driver issue
→ Increasing drops? → Ring buffer overflow or CPU can't keep up
```

---

## 9. Hands-On Practice

### Lab 1: Explore Your Interfaces

```bash
# List all interfaces (brief format)
ip -br addr

# Show detailed info for main interface
ip addr show eth0

# Show link-layer info (MAC, MTU, state)
ip -br link

# Check interface statistics
ip -s link show eth0
```

### Lab 2: Create a veth Pair (What Docker/K8s Does)

```bash
# Create a veth pair
sudo ip link add veth-host type veth peer name veth-guest

# Both interfaces are DOWN by default
ip link show veth-host
ip link show veth-guest

# Assign IPs
sudo ip addr add 10.200.1.1/24 dev veth-host
sudo ip addr add 10.200.1.2/24 dev veth-guest

# Bring both UP
sudo ip link set veth-host up
sudo ip link set veth-guest up

# Ping between them
ping -c 3 10.200.1.2  # From host to guest end

# Clean up
sudo ip link del veth-host  # Deleting one end removes both
```

### Lab 3: Inspect a K8s Node's Interfaces

```bash
# On a K8s node:
# Count pod interfaces
ip link | grep -c veth

# List all pod interfaces with their IPs
ip addr | grep -A 2 veth

# Correlate with kubectl
kubectl get pods -o wide --field-selector spec.nodeName=$(hostname)

# Find which veth belongs to which pod:
# Get pod's interface index from inside the pod:
kubectl exec <pod> -- cat /sys/class/net/eth0/iflink
# That index number corresponds to the veth on the host
```

### Lab 4: MTU Testing

```bash
# Check current MTU
ip link show eth0 | grep mtu

# Test with large packets (1472 + 28 bytes IP/ICMP header = 1500 MTU)
ping -M do -s 1472 8.8.8.8    # Should work with MTU 1500
ping -M do -s 1473 8.8.8.8    # Should fail if MTU is 1500

# For overlay networks, test with smaller sizes:
ping -M do -s 1422 <pod-ip>   # Adjusted for VXLAN overhead
```

---

## 10. Interview Preparation

### Q1: How does a container get its own network interface while sharing the host kernel?

**Answer:** Linux network namespaces. When a container starts, Docker/K8s creates a new network namespace (isolated network stack with its own interfaces, routing table, and iptables). A veth pair is created — one end placed in the container's namespace (appears as `eth0`), the other stays on the host. The container sees only its own `eth0`, but the host sees the corresponding `veth` interface. Both sides share the same kernel but have completely isolated network views.

### Q2: What does `ip addr show` tell you that `ip link show` doesn't?

**Answer:** `ip link` shows Layer 2 info: interface names, states (UP/DOWN), MTU, MAC addresses. `ip addr` shows everything `ip link` shows PLUS Layer 3 info: IP addresses, subnet masks, scope (global/link/host). Use `ip link` to check "is the interface alive?" and `ip addr` to check "what IP does it have?"

### Q3: You see 50 veth interfaces on a K8s node. What does that mean?

**Answer:** Each veth interface is one end of a veth pair connected to a pod. 50 veth interfaces = approximately 50 pods on this node. Each pod's network namespace has the other end of the pair as its `eth0`. You can correlate them with `kubectl get pods -o wide --field-selector spec.nodeName=<this-node>`.

### Q4: Explain MTU and when it causes problems in production.

**Answer:** MTU is the maximum packet size an interface sends. Default is 1500 bytes. Problems occur when the path has mixed MTUs: if host sends 1500-byte packets through a VPN tunnel with MTU 1400, packets get dropped. Symptoms: small requests work, large responses fail. Fix: set MTU to the smallest value in the path, or configure Path MTU Discovery (requires ICMP type 3 code 4 to be allowed).

---

## 11. Advanced Insights (Senior Level)

### AWS ENI Architecture

```
EC2 Instance:
├── Primary ENI (eth0): 10.0.1.50
│   ├── Secondary IP: 10.0.1.51  → assigned to Pod A
│   ├── Secondary IP: 10.0.1.52  → assigned to Pod B
│   └── Secondary IP: 10.0.1.53  → assigned to Pod C
│
├── Secondary ENI (eth1): 10.0.1.60
│   ├── Secondary IP: 10.0.1.61  → assigned to Pod D
│   └── Secondary IP: 10.0.1.62  → assigned to Pod E

Max pods = (Number of ENIs × IPs per ENI) - 1
t3.medium:  3 ENIs × 6 IPs = 17 pods
m5.xlarge:  4 ENIs × 15 IPs = 58 pods
```

This is why instance type matters for EKS pod density — it's an L2/L3 constraint.

### Interface Bonding / Teaming

For high-availability bare-metal servers:
```bash
# Two physical NICs bonded into one logical interface:
bond0: <BROADCAST,MULTICAST,MASTER,UP>
  ├── eth0 (SLAVE)
  └── eth1 (SLAVE)

# If eth0 fails, traffic automatically shifts to eth1
# In cloud: multi-AZ ENI placement serves a similar purpose
```

---

## 12. Common Mistakes

### Mistake 1: Using `ifconfig` instead of `ip`
`ifconfig` is deprecated and not installed by default on modern Linux. Always use `ip addr`, `ip link`, `ip route`.

### Mistake 2: Ignoring MTU on overlay networks
VXLAN adds 50 bytes of encapsulation. If your host MTU is 1500, set overlay MTU to 1450. If host is 9001 (AWS), overlay can be 8951. Mismatched MTU causes subtle, hard-to-diagnose packet loss.

### Mistake 3: Not checking if an interface is UP
`ip addr show eth0` might show an IP, but if the interface is DOWN, no traffic flows. Always check the flags.

### Mistake 4: Confusing interface names across distros
`eth0` vs `ens5` vs `enp0s3` — modern Linux uses predictable names based on hardware location. The name doesn't matter; the IP and state do.

---

## 13. Cheat Sheet

### Essential `ip` Commands

```bash
ip addr show              # All interfaces + IPs
ip -br addr               # Brief format (cleaner)
ip link show              # All interfaces + L2 info
ip -br link               # Brief link status
ip addr show eth0         # Specific interface
ip link set eth0 up       # Enable interface
ip link set eth0 down     # Disable interface
ip addr add 10.0.1.50/24 dev eth0    # Add IP
ip addr del 10.0.1.50/24 dev eth0    # Remove IP
ip link set eth0 mtu 1400            # Change MTU
ip link | grep veth                   # List pod interfaces
ip -s link show eth0                  # Interface statistics
ip neigh show                         # ARP table
```

### Interface Types Quick Reference

```
eth0/ens5    → Physical NIC (main network)
lo           → Loopback (127.0.0.1)
vethXXXX     → Virtual Ethernet (container/pod link)
docker0/cni0 → Bridge (connects containers)
flannel.1    → VXLAN overlay tunnel
tun0/wg0     → VPN tunnel
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**OSI Model (Topic 1.1):** Network interfaces operate at **Layer 1-2** (Physical/Data Link). MAC addresses, MTU, interface state — all L2 concepts. IP addresses assigned to interfaces are L3.

**IP Addressing (Topic 1.3):** Every interface has (or can have) an IP address. The IP address + subnet mask tells the kernel which traffic belongs to this local network (direct delivery via ARP) vs. which needs a router (next topic).

**TCP (Topic 1.2):** TCP connections bind to specific interfaces via IP addresses. A server listening on `0.0.0.0:8080` listens on ALL interfaces. A server on `10.0.1.50:8080` listens only on that specific interface.

### What This Prepares You For Next
**Next topic: Linux Routing Tables and How Packets Find Their Destination**

Now that you can see and understand interfaces, the next question is: when a packet arrives, how does Linux decide which interface to send it out? That's the routing table — the kernel's rulebook for "if the destination is X, send through interface Y via gateway Z." Every VPC route table, every K8s pod route, every NAT decision starts with the Linux routing table.
