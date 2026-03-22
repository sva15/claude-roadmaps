# Network Namespaces — The Container Isolation Primitive

> 🔹 **Phase:** 2 — Linux Networking  
> 🔹 **Topic:** Network Namespaces  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Containers need their own isolated network — their own IP address, their own routing table, their own firewall rules — while sharing the same kernel. But a kernel has ONE network stack. How do you create multiple isolated network stacks on one kernel?

**Network namespaces.** Each namespace is a completely independent copy of the network stack: its own interfaces, routing table, iptables rules, and port space.

### The Analogy: Apartments in a Building

Your Linux machine is an apartment building. The building has one water main (kernel), but each apartment (namespace) has its own:
- Mailbox with a unique address (interfaces + IPs)
- Internal plumbing layout (routing table)
- Security system (iptables rules)
- Doorbells on different ports (port space)

Two tenants can both run a web server on port 80 without conflict — they're in separate apartments.

### Why Does It Exist?

Network namespaces are a Linux kernel feature (since kernel 2.6.24, 2008). They enable:
- **Containers** — every Docker container and K8s pod runs in its own network namespace
- **Multi-tenant isolation** — guarantee one tenant can't see another's traffic
- **Testing** — create isolated virtual networks on a single machine

> 💡 **Key Insight:** When you run `kubectl exec <pod> -- ip addr`, you're looking at the network namespace of that pod. It has its OWN `eth0`, its OWN routing table, and its OWN view of the network. The node's network namespace is completely separate.

---

## 2. Why This Matters in DevOps

| Scenario | Namespace Concept |
|----------|------------------|
| Pod has its own IP separate from the node | Pod runs in isolated network namespace |
| Two pods both listen on port 80 | Different namespaces → different port spaces |
| Container can't reach the internet | Namespace missing default route or NAT rules |
| `nsenter` to debug a pod's network | Entering the pod's network namespace from the node |
| Docker bridge networking | Container namespace → veth → bridge in host namespace |
| Kubernetes pause container | Holds the network namespace alive for the pod |

---

## 3. Core Concept (Deep Dive)

### What Exactly Is a Network Namespace?

A namespace contains its own isolated copy of:

```
┌── Network Namespace ──────────────────────────────────┐
│                                                        │
│  Network Interfaces:  eth0, lo                        │
│  IP Addresses:        10.244.1.5/32                   │
│  Routing Table:       default via 169.254.1.1         │
│  iptables Rules:      (own set of rules)              │
│  Port Space:          0-65535 (independent)            │
│  ARP Table:           (own neighbor cache)             │
│  /proc/net/ :         (own /proc/net statistics)       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**The host (root) namespace** is the default — what you see when you log into the node. Every new namespace starts empty (no interfaces, no routes) and must be configured.

### How Containers Use Network Namespaces

```
┌──────── Host (Root) Namespace ────────────────────────────┐
│                                                            │
│  eth0 (10.0.1.50)  ─── node's main interface              │
│  cni0 (10.244.1.1) ─── bridge for local pods              │
│  vethAAAA           ─── paired with Pod A's eth0           │
│  vethBBBB           ─── paired with Pod B's eth0           │
│                                                            │
│  Routing: Pod A IP/32 → vethAAAA                          │
│           Pod B IP/32 → vethBBBB                          │
│           0.0.0.0/0  → 10.0.1.1                          │
│                                                            │
│  iptables: KUBE-SERVICES, MASQUERADE for pods, etc.       │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌── Pod A Namespace ──────┐  ┌── Pod B Namespace ──────┐ │
│  │ eth0 (10.244.1.5)      │  │ eth0 (10.244.1.6)      │ │
│  │ lo   (127.0.0.1)       │  │ lo   (127.0.0.1)       │ │
│  │ Route: default→gw      │  │ Route: default→gw      │ │
│  │ Port 80: nginx         │  │ Port 80: nginx         │ │
│  │ (no conflict!)         │  │ (no conflict!)         │ │
│  └────────────────────────┘  └────────────────────────┘ │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### The K8s Pod Network Namespace Lifecycle

```
1. kubelet creates a new pod
2. CRI (containerd) creates a network namespace
3. CNI plugin is invoked with the namespace:
   a. Creates a veth pair
   b. Moves one end into the pod's namespace (becomes eth0)
   c. Configures IP address on the pod's eth0
   d. Sets up routing inside the namespace (default gateway)
   e. Connects the host end to the bridge or route table
4. Pause container is started in this namespace
   → It does nothing except hold the namespace alive
5. App containers join the SAME namespace (shared network)
   → All containers in a pod share the same IP and port space
6. Pod deletion:
   → CNI plugin cleans up veth pair
   → Namespace is destroyed
```

### The Pause Container — Why It Matters

The **pause container** (`k8s.gcr.io/pause`) is a tiny container that:
1. Creates and holds the network namespace for the pod
2. Does absolutely nothing else (infinite sleep)

**Why?** If the app container crashes, the network namespace survives (pause is still running). When the app container restarts, it rejoins the same namespace with the same IP. Without pause, a container restart would destroy the namespace, change the IP, and break connections.

### Containers in a Pod Share Everything Network

```
Pod with 2 containers (app + sidecar):

┌── Pod Network Namespace ───────────────────┐
│                                             │
│  Container 1 (nginx on port 80)            │
│  Container 2 (envoy proxy on port 15001)   │
│                                             │
│  Both share:                                │
│  • eth0 (same IP: 10.244.1.5)             │
│  • lo (container 2 can call container 1     │
│        via localhost:80)                    │
│  • Routing table                            │
│  • Port space (both can't use same port!)   │
│                                             │
└─────────────────────────────────────────────┘
```

This is why sidecar containers (like Istio envoy) can intercept traffic by listening on localhost — they're in the same network namespace.

---

## 4. Internal Working (Under the Hood)

### Creating and Managing Namespaces

```bash
# Create a new network namespace
ip netns add mypod

# List all namespaces
ip netns list

# Execute command inside the namespace
ip netns exec mypod ip addr show
# → Only 'lo' exists, and it's DOWN

# Create a veth pair and move one end inside
ip link add veth-host type veth peer name veth-pod
ip link set veth-pod netns mypod

# Configure inside the namespace
ip netns exec mypod ip addr add 10.200.1.2/24 dev veth-pod
ip netns exec mypod ip link set veth-pod up
ip netns exec mypod ip link set lo up

# Configure on the host
ip addr add 10.200.1.1/24 dev veth-host
ip link set veth-host up

# Now they can communicate:
ip netns exec mypod ping 10.200.1.1   # Pod → Host
ping 10.200.1.2                          # Host → Pod
```

### nsenter — Entering a Container's Namespace

```bash
# Find the container's PID:
# Docker:
docker inspect <container-id> --format '{{.State.Pid}}'

# Kubernetes (via crictl):
crictl inspect <container-id> | grep pid

# Enter only the network namespace of that process:
nsenter -t <PID> -n ip addr show
nsenter -t <PID> -n ip route show
nsenter -t <PID> -n iptables -L -n
nsenter -t <PID> -n ss -tnlp
nsenter -t <PID> -n tcpdump -i eth0 -nn
```

> 🚨 **Production debugging superpower:** `nsenter -t <PID> -n` lets you run ANY network diagnostic tool inside a pod's namespace — even if the pod doesn't have those tools installed. You use the host's netstat, tcpdump, ss, etc., but see the pod's network view.

---

## 5. Mental Models

### Mental Model 1: Parallel Universes

```
Root namespace = Earth (the real world)
Pod namespace = A parallel universe

Each universe has its own:
  • Map (routing table)
  • Address system (IPs)  
  • Laws (iptables)
  • Roads (interfaces)

The veth pair is a portal between universes.
Traffic enters the portal on one side (pod eth0) 
and emerges on the other side (host veth).
```

### Mental Model 2: The Pod Apartment

```
Pod = Apartment with shared utilities

Container 1 (app) and Container 2 (sidecar) are roommates:
  • Same address (shared IP)
  • Same phone line (shared port space)
  • Can talk face-to-face (localhost)
  • Share the front door (eth0)

But they CANNOT both answer the phone on the same number (same port).
The pause container is the landlord who holds the lease (namespace).
```

---

## 6. Real-World Mapping

### Linux Commands

```bash
# List network namespaces
ip netns list

# Create/delete namespace
ip netns add test-ns
ip netns del test-ns

# Run command in namespace
ip netns exec test-ns <command>

# Enter a container's network namespace
nsenter -t <PID> -n <command>

# Find a pod's network namespace PID
# Option 1: via crictl
crictl ps | grep <pod-name>
crictl inspect <container-id> | grep pid

# Option 2: via /proc
ls /proc/<pid>/ns/net   # Each namespace has a unique inode

# Compare namespaces
readlink /proc/1/ns/net          # Host namespace
readlink /proc/<pid>/ns/net      # Container namespace
# Different inodes = different namespaces
```

### Kubernetes

| K8s Concept | Namespace Connection |
|------------|---------------------|
| **Pod** | Each pod has its own network namespace |
| **Pause container** | Creates and holds the network namespace |
| **Multi-container pod** | All containers share ONE network namespace |
| **kubectl exec** | Enters the pod's PID and mount namespace (network is already shared) |
| **hostNetwork: true** | Pod uses the HOST's network namespace (no isolation) |
| **CNI plugin** | Responsible for configuring the pod's namespace (IP, routes, veth) |

### hostNetwork: true — When and Why

```yaml
spec:
  hostNetwork: true  # Pod uses the node's network namespace
```

**When to use:** kube-proxy, CNI plugins, monitoring agents that need to see all node traffic. The pod gets the node's IP and interfaces.

**Risk:** No network isolation. Pod can see/bind to any port on the node. Security concern.

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: Pod Has No Network Connectivity

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Pod is Running but can't reach any network — not even other pods |
| **Root Cause** | CNI failed to configure the pod's network namespace |
| **Debug Steps** | 1. `kubectl exec <pod> -- ip addr` — does eth0 exist? Has an IP? 2. `kubectl exec <pod> -- ip route` — is there a default route? 3. On the node: `ip link | grep <pod's veth>` — does the host-side veth exist? 4. Check CNI logs: `journalctl -u kubelet | grep -i cni` |

### Scenario 2: Two Containers in a Pod Can't Communicate

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Sidecar can't reach the app container via localhost |
| **Root Cause** | 1. App isn't listening on 0.0.0.0 (bound to a specific interface). 2. Wrong port. 3. App hasn't started yet (race condition). |
| **Debug Steps** | 1. `kubectl exec <pod> -c <sidecar> -- curl localhost:<port>` 2. `kubectl exec <pod> -c <app> -- ss -tnlp` — what's actually listening? 3. Check startup order — readiness probe on the app before sidecar sends traffic. |

### Scenario 3: nsenter Shows Different Network Than Expected

| Attribute | Detail |
|-----------|--------|
| **Symptom** | `nsenter` output shows the host network instead of the container network |
| **Root Cause** | Wrong PID used. The PID belongs to a process in the host namespace (or hostNetwork pod). |
| **Debug Steps** | `readlink /proc/<pid>/ns/net` — compare with host namespace (`readlink /proc/1/ns/net`). If same inode → you're in the host namespace. |

---

## 8. Debugging Playbook

```
Pod Network Issues? Use nsenter:

Step 1: Find the container PID
────────────────────────────────
$ crictl ps | grep <pod-name>
$ crictl inspect <container-id> | grep -i pid

Step 2: Enter the namespace and inspect
────────────────────────────────────────
$ nsenter -t <PID> -n ip addr show         # Interfaces
$ nsenter -t <PID> -n ip route show        # Routes
$ nsenter -t <PID> -n ss -tnlp            # Listening sockets
$ nsenter -t <PID> -n ss -tnp             # Active connections

Step 3: Test connectivity from inside
──────────────────────────────────────
$ nsenter -t <PID> -n ping <target>
$ nsenter -t <PID> -n curl <service-url>
$ nsenter -t <PID> -n nslookup <service>

Step 4: Capture packets (the ultimate debug)
──────────────────────────────────────────────
$ nsenter -t <PID> -n tcpdump -i eth0 -nn -c 50
→ See exactly what the pod is sending and receiving
```

---

## 9. Hands-On Practice

### Lab 1: Create Your Own Container Network

```bash
# Create two namespaces (simulating two pods)
sudo ip netns add pod-a
sudo ip netns add pod-b

# Create veth pairs
sudo ip link add veth-a type veth peer name veth-a-br
sudo ip link add veth-b type veth peer name veth-b-br

# Move one end into each namespace
sudo ip link set veth-a netns pod-a
sudo ip link set veth-b netns pod-b

# Create a bridge (simulating cni0)
sudo ip link add br0 type bridge
sudo ip link set br0 up
sudo ip addr add 10.200.0.1/24 dev br0

# Attach host-side veths to the bridge
sudo ip link set veth-a-br master br0
sudo ip link set veth-b-br master br0
sudo ip link set veth-a-br up
sudo ip link set veth-b-br up

# Configure inside namespaces
sudo ip netns exec pod-a ip addr add 10.200.0.2/24 dev veth-a
sudo ip netns exec pod-a ip link set veth-a up
sudo ip netns exec pod-a ip link set lo up
sudo ip netns exec pod-a ip route add default via 10.200.0.1

sudo ip netns exec pod-b ip addr add 10.200.0.3/24 dev veth-b
sudo ip netns exec pod-b ip link set veth-b up
sudo ip netns exec pod-b ip link set lo up
sudo ip netns exec pod-b ip route add default via 10.200.0.1

# Test: pod-a → pod-b
sudo ip netns exec pod-a ping -c 3 10.200.0.3
# ✓ This is exactly how K8s pods communicate on the same node!

# Cleanup
sudo ip netns del pod-a
sudo ip netns del pod-b
sudo ip link del br0
```

### Lab 2: Inspect a K8s Pod's Namespace

```bash
# Find a pod's container PID
POD=$(kubectl get pod -o name | head -1 | cut -d/ -f2)
CONTAINER_ID=$(crictl ps --name $POD -q | head -1)
PID=$(crictl inspect $CONTAINER_ID | jq .info.pid)

# Compare with host namespace
echo "Host NS: $(readlink /proc/1/ns/net)"
echo "Pod NS:  $(readlink /proc/$PID/ns/net)"
# Different → they're in separate namespaces

# View inside the pod's namespace
nsenter -t $PID -n ip addr show
nsenter -t $PID -n ip route show
nsenter -t $PID -n ss -tnlp
```

---

## 10. Interview Preparation

### Q1: How does a Kubernetes pod get its own IP address and network?

**Answer:** Each pod runs in its own Linux network namespace. The CNI plugin creates a veth pair — one end in the pod's namespace (appears as eth0), one end on the host. The CNI assigns an IP and sets up routing in the pod's namespace. The pod sees its own isolated network with its own interfaces, routes, and port space. A pause container holds the namespace alive for the pod's lifetime.

### Q2: What is the pause container and why does it exist?

**Answer:** The pause container (`k8s.gcr.io/pause`) creates and holds the network namespace for the pod. It does nothing else (sleeps forever). Purpose: if the app container crashes and restarts, the network namespace persists (because pause is still running). The restarted container rejoins the same namespace with the same IP. Without pause, a restart would destroy the namespace, creating a new IP and breaking existing connections.

### Q3: Two containers in the same pod can communicate via localhost. Explain why.

**Answer:** All containers in a pod share the same network namespace. They have the same `eth0`, same IP, same routing table, and same port space. Since they share the `lo` (loopback) interface, one container can reach another via `localhost:<port>`. This is the same as two processes on the same machine communicating via loopback.

### Q4: How would you debug a pod's network issues without installing tools in the pod?

**Answer:** Use `nsenter -t <PID> -n` to enter the pod's network namespace while using the host's tools. Find the PID via `crictl inspect <container-id>`, then run `nsenter -t <PID> -n tcpdump`, `nsenter -t <PID> -n ss`, `nsenter -t <PID> -n ip route`, etc. You get the pod's network view with the host's diagnostic tools.

### Q5: What is `hostNetwork: true` and when would you use it?

**Answer:** `hostNetwork: true` makes the pod use the host's network namespace instead of creating its own. The pod gets the node's IP and can bind to any port on the node. Use cases: kube-proxy (needs access to node-level iptables), CNI plugins, monitoring agents. Risk: no network isolation, port conflicts with other pods and host processes.

---

## 11. Advanced Insights (Senior Level)

### Network Namespace vs Other Namespaces

Linux has 8 namespace types. For containers, the key ones are:

| Namespace | Isolates | K8s Pod Behavior |
|-----------|----------|-----------------|
| **Network** | Interfaces, IPs, routes, iptables, ports | Shared by all containers in the pod |
| **PID** | Process IDs | Each container has its own PID space (usually) |
| **Mount** | Filesystem mounts | Each container has its own filesystem |
| **UTS** | Hostname | Shared per pod |
| **IPC** | Shared memory, semaphores | Shared per pod |

### Performance: Network Namespace Overhead

Network namespaces add minimal overhead — the kernel uses the same networking code, just with different context (routing table, iptables). The veth pair has ~1-3μs added latency compared to direct kernel networking. At scale, the overhead is negligible compared to network latency.

---

## 12. Common Mistakes

### Mistake 1: Thinking each container in a pod has its own IP
All containers in a pod share ONE network namespace → same IP. A port conflict between containers is a common misconfiguration.

### Mistake 2: Using the wrong PID for nsenter
Using a PID from a different container or the host gives you the wrong namespace. Always verify with `readlink /proc/<pid>/ns/net`.

### Mistake 3: Forgetting that network namespaces start empty
A new namespace has no interfaces (not even `lo`), no routes, no IPs. Everything must be explicitly configured — this is what the CNI plugin does.

---

## 13. Cheat Sheet

```bash
# Namespace management
ip netns list                        # List namespaces
ip netns add <name>                  # Create
ip netns del <name>                  # Delete
ip netns exec <name> <cmd>           # Run in namespace

# Container debugging
nsenter -t <PID> -n ip addr         # View pod interfaces
nsenter -t <PID> -n ip route        # View pod routes
nsenter -t <PID> -n ss -tnlp       # View pod listeners
nsenter -t <PID> -n tcpdump -i eth0 # Capture pod traffic

# Verify isolation
readlink /proc/<pid>/ns/net          # Namespace inode

# K8s namespace facts
Pod = own network namespace
Containers in pod = shared namespace
Pause container = holds namespace
hostNetwork: true = uses host namespace
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**Network Interfaces (Topic 2.1):** veth pairs are interfaces — one end in the host namespace, one in the pod namespace. Everything you learned about `ip addr`, `ip link`, and MTU applies within each namespace.

**Routing (Topic 2.2):** Each namespace has its own routing table. The pod's default route and the host's routing table are independent.

**iptables (Topic 2.3):** Each namespace can have its own iptables rules. Network Policies (Calico) create iptables rules inside the pod's namespace or on the host's namespace to filter traffic.

### What This Prepares You For Next
**Next topic: Network Debugging Tools — Your Production Toolkit**

Now that you understand interfaces, routing, iptables, and namespaces — the four building blocks of Linux networking — the next topic brings them together with the debugging tools that make you a production-level troubleshooter: `tcpdump`, `ss`, `curl`, `dig`, `mtr`, and how to use them systematically.
