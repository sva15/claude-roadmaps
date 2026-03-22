# CNI Plugins — How Pods Get Their IP and Connectivity

> 🔹 **Phase:** 4 — Kubernetes Networking  
> 🔹 **Topic:** Container Network Interface (CNI) Deep Dive  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Kubernetes defines networking rules but doesn't implement them. The **CNI (Container Network Interface)** plugin is the software that:
1. Creates the network namespace for each pod
2. Creates the veth pair and bridge/routes
3. Assigns an IP address
4. Sets up routing so pods can communicate across nodes
5. Cleans up when the pod is deleted

CNI is a specification — a contract between K8s and the networking plugin. Any plugin that follows the spec can be used.

### The CNI Lifecycle

```
Pod Creation:
  kubelet → calls CNI plugin with ADD command
  → CNI creates veth pair
  → Moves one end into pod namespace
  → Assigns IP address
  → Sets up routes
  → Returns IP to kubelet
  → kubelet reports pod IP to API server

Pod Deletion:
  kubelet → calls CNI plugin with DEL command
  → CNI removes veth pair
  → Releases IP address
  → Cleans up routes
```

---

## 2. Core Concept (Deep Dive)

### AWS VPC CNI — The EKS Default

```
Architecture:
  Two components on each node:
  ├── ipamd (IP Address Management Daemon)
  │   → Pre-allocates ENIs and secondary IPs
  │   → Maintains a "warm pool" of ready IPs
  │   → Handles ENI attachment/detachment
  │
  └── CNI binary (invoked by kubelet)
      → Assigns IP from warm pool to pod
      → Creates veth pair and routes
      → Configures pod namespace

IP allocation:
  Node starts → ipamd attaches ENIs based on instance limits
  → Pre-allocates secondary IPs on each ENI (warm pool)
  → Pod created → CNI binary grabs IP from warm pool
  → Pool runs low → ipamd allocates more IPs/ENIs
  → Pod deleted → IP returned to warm pool (reusable)
```

**Configuration tuning:**
```bash
# Environment variables on aws-node DaemonSet:
WARM_ENI_TARGET=1           # Keep 1 spare ENI ready
WARM_IP_TARGET=5            # Keep 5 spare IPs ready
MINIMUM_IP_TARGET=10        # Ensure at least 10 IPs available
ENABLE_PREFIX_DELEGATION=true  # /28 prefixes instead of individual IPs
                               # → Dramatically increases max pods
```

**Prefix delegation mode:**
```
Without prefix delegation:
  ENI gets individual IPs: 10.0.1.51, 10.0.1.52, 10.0.1.53 ...
  t3.medium: 3 ENIs × 6 IPs = 17 pods max

With prefix delegation:
  ENI gets /28 prefixes: 10.0.1.48/28 (16 IPs), 10.0.1.64/28 (16 IPs) ...
  t3.medium: 3 ENIs × 6 prefixes × 16 IPs = up to 110 pods
  → Massively increases pod density
```

### Calico

```
Modes:
├── BGP (native routing)
│   → Calico announces pod CIDRs via BGP
│   → Routers learn pod routes automatically
│   → Best performance (no encapsulation)
│   → Requires BGP-capable infrastructure
│
├── VXLAN (overlay)
│   → Encapsulates pod traffic in VXLAN tunnels
│   → Works on any network (no BGP needed)
│   → Slight performance overhead (encap/decap)
│
└── IPIP (overlay)
    → Encapsulates in IP-in-IP tunnels
    → Simpler than VXLAN
    → Higher MTU than VXLAN (less overhead)

Key features:
  ✅ Network Policy enforcement (iptables or eBPF)
  ✅ Works with AWS VPC CNI (Calico for policies, VPC CNI for IPs)
  ✅ Can run in "policy-only" mode alongside other CNIs
```

### Cilium

```
eBPF-based (no iptables for data plane):
  ✅ O(1) Service routing (vs iptables O(n))
  ✅ L7 Network Policies (HTTP path, gRPC method)
  ✅ Built-in observability (Hubble)
  ✅ Transparent encryption (WireGuard/IPsec)
  ✅ Multi-cluster connectivity (Cluster Mesh)
  ✅ Bandwidth management
  
  Architecture:
    Pod traffic → eBPF programs on NIC → direct routing
    (Bypasses iptables entirely)
```

### CNI Selection Decision

```
EKS (AWS)?
├── Default: AWS VPC CNI (native VPC IPs)
├── Need Network Policies? → Add Calico policy-only mode
├── Want eBPF/L7 policies? → Cilium as primary CNI
└── Need more pods? → Enable prefix delegation

Self-managed K8s?
├── Simple overlay → Flannel (easy, but no Network Policy)
├── Need policies → Calico (VXLAN or BGP)
├── Advanced features → Cilium (eBPF, observability)
└── Multi-cloud → Weave Net
```

---

## 3. Failure Scenarios

### Scenario 1: ipamd IP Exhaustion (VPC CNI)

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Pods stuck in `ContainerCreating`, events show "no available IPs" |
| **Root Cause** | Subnet CIDR exhausted or ENI limit reached |
| **Debug** | 1. `kubectl describe pod` → event message. 2. `kubectl logs -n kube-system -l k8s-app=aws-node` → ipamd logs. 3. Check subnet: `aws ec2 describe-subnets` → AvailableIpAddressCount. |
| **Fix** | Larger subnets (/20), enable prefix delegation, or add secondary CIDR to VPC. |

### Scenario 2: CNI Binary Not Found

| Attribute | Detail |
|-----------|--------|
| **Symptom** | All pods stuck in `ContainerCreating`, node NotReady for networking |
| **Root Cause** | CNI binary missing from `/opt/cni/bin/` or CNI config missing from `/etc/cni/net.d/` |
| **Debug** | `ls /opt/cni/bin/` and `ls /etc/cni/net.d/`. Check aws-node DaemonSet is running. |

---

## 4. Interview Preparation

### Q1: Compare the AWS VPC CNI with an overlay CNI like Flannel.

**Answer:** VPC CNI assigns real VPC IPs — pods are first-class VPC citizens (SGs, Flow Logs, native routing). No encapsulation overhead. But limited by ENI/IP quotas per instance type. Flannel uses VXLAN overlay — pods get IPs from a separate CIDR (10.244.x.x), packets are encapsulated between nodes. No IP limit issues but adds encapsulation overhead (+50 bytes header, reduced MTU).

### Q2: How would you increase max pods per node in EKS?

**Answer:** Enable prefix delegation: set `ENABLE_PREFIX_DELEGATION=true` on the aws-node DaemonSet. Instead of individual secondary IPs, VPC CNI assigns /28 prefixes (16 IPs each) to ENIs. A t3.medium goes from 17 pods to 110. Also: use larger instance types (more ENIs × more IPs per ENI).

---

## 5. Cheat Sheet

```
CNI location: /opt/cni/bin/ (binaries), /etc/cni/net.d/ (config)
VPC CNI logs: kubectl logs -n kube-system -l k8s-app=aws-node
VPC CNI max pods: (ENIs × IPs per ENI) - 1
Prefix delegation: ENABLE_PREFIX_DELEGATION=true → 5-10x more pods
Calico + VPC CNI: Calico for policies, VPC CNI for IPs
Cilium: eBPF-based, no iptables, L7 policies
```

---

## 🔗 Connections

### Previous: Topic 4.1 introduced the 4 networking rules. CNI plugins are what implement those rules.
### Next: Topic 4.3 — K8s Services deep dive into ClusterIP, NodePort, LoadBalancer mechanics.
