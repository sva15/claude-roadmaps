# eBPF — The Future of Kubernetes Networking

> 🔹 **Phase:** 5 — Advanced (Senior Differentiator)  
> 🔹 **Topic:** eBPF  
> 🔹 **Priority:** MEDIUM

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Traditional K8s networking relies on **iptables** for Service routing, **conntrack** for connection tracking, and **sidecar proxies** for L7 policies. At scale, all three become bottlenecks:
- iptables: O(n) rule matching (slow with 5000+ Services)
- conntrack: table exhaustion under high connection rates
- Sidecars: RAM overhead and added latency per hop

**eBPF (extended Berkeley Packet Filter)** lets you run custom programs directly in the Linux kernel — at the network interface level — without modifying kernel code or using iptables.

### The Analogy

iptables = a bouncer checking a long guest list name by name (slow).
eBPF = a smart scanner that checks your ID against a hash table in microseconds.

---

## 2. Core Concept (Deep Dive)

### What eBPF Is

```
Traditional networking:
  Packet → NIC → kernel network stack → iptables chains → routing → app
  (Many hooks, rule-by-rule matching, slow at scale)

eBPF networking:
  Packet → NIC → eBPF program runs (kernel space) → direct action
  (Programmable, O(1) hash lookups, runs at wire speed)

eBPF programs:
  → Written in restricted C, compiled to bytecode
  → Loaded into kernel and attached to hooks (XDP, TC, cgroups)
  → Verified by kernel (safety guarantees — can't crash kernel)
  → Use BPF maps (hash tables) for O(1) lookups
  → JIT-compiled to native machine code for speed
```

### How Cilium Uses eBPF

```
Cilium replaces:

1. kube-proxy (iptables) → eBPF Service routing
   → O(1) hash lookup instead of O(n) iptables chain
   → No conntrack table exhaustion
   → Direct pod-to-pod forwarding (no bridge needed)

2. NetworkPolicy (iptables filter) → eBPF policy enforcement
   → Per-packet identity-based filtering
   → L3/L4 AND L7 policies (HTTP path, gRPC method)
   → No iptables rules to manage

3. Sidecar proxies (optional) → eBPF L7 interception
   → Some L7 features without sidecar overhead
   → "Sidecar-less" service mesh model
```

### XDP (eXpress Data Path)

```
Fastest eBPF hook — runs BEFORE kernel network stack:
  Packet → NIC driver → XDP program → action

Actions:
  XDP_PASS  → Let packet through to kernel stack
  XDP_DROP  → Drop immediately (fastest possible firewall)
  XDP_TX    → Send back out the same NIC (reflect)
  XDP_REDIRECT → Send to another NIC or CPU

Use cases:
  → DDoS mitigation at line rate
  → Load balancing without kernel stack overhead
  → Packet filtering at millions of packets/sec
```

### Performance Comparison

| Operation | iptables | eBPF (Cilium) |
|-----------|---------|---------------|
| Service routing (1K services) | ~0.5ms | ~0.01ms |
| Service routing (10K services) | ~5ms | ~0.01ms |
| Network Policy evaluation | O(n) rules | O(1) lookup |
| Connection tracking | conntrack table (limited) | BPF map (configurable) |
| Memory per policy | ~100 bytes/rule | ~64 bytes/entry |
| L7 policy (HTTP) | Requires sidecar proxy | Native (no sidecar) |

---

## 3. Cilium Architecture

```
┌──────── Node ──────────────────────────────┐
│                                              │
│  ┌── Pod A ──┐    ┌── Pod B ──┐             │
│  │   App     │    │   App     │             │
│  │   eth0    │    │   eth0    │             │
│  └──┬────────┘    └──┬────────┘             │
│     │                │                       │
│     ▼                ▼                       │
│  [eBPF programs on veth interfaces]         │
│     │                │                       │
│     ▼                ▼                       │
│  [eBPF at tc/XDP level — routing, policy]   │
│     │                                        │
│     ▼                                        │
│  eth0 (host) ← [eBPF: encap/decap, LB]     │
│                                              │
│  cilium-agent (DaemonSet):                   │
│    → Watches K8s API for Services, Pods,     │
│      NetworkPolicies, CiliumNetworkPolicies  │
│    → Compiles and loads eBPF programs        │
│    → Manages BPF maps (Service→Pod mapping)  │
│                                              │
│  Hubble (observability):                     │
│    → Captures eBPF events                    │
│    → Provides flow visibility (like Flow Logs)│
│    → Service dependency map                  │
│    → DNS query logging                       │
└──────────────────────────────────────────────┘
```

### Cilium Network Policies (L7)

```yaml
# Standard K8s NetworkPolicy: L3/L4 only
# → "Allow from frontend to api on port 8080"

# CiliumNetworkPolicy: L3/L4 + L7
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-policy
spec:
  endpointSelector:
    matchLabels: { app: api }
  ingress:
  - fromEndpoints:
    - matchLabels: { app: frontend }
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/api/users"     # Only allow GET /api/users
        - method: POST
          path: "/api/orders"    # And POST /api/orders
```

---

## 4. Interview Preparation

### Q1: What is eBPF and why is it important for Kubernetes?

**Answer:** eBPF allows running verified programs directly in the Linux kernel. For K8s, it replaces iptables-based routing (O(n)) with hash-map lookups (O(1)), scales to 10,000+ Services without latency degradation, enables L7 policies without sidecars, and provides deep observability (Hubble). Cilium is the leading eBPF-based CNI.

### Q2: When would you choose Cilium over Calico?

**Answer:** Cilium for: large clusters (1000+ Services — eBPF scales better than iptables), need L7 network policies (HTTP path-based), want to avoid iptables complexity, want built-in observability (Hubble), or exploring sidecar-less service mesh. Calico for: simpler needs, well-understood iptables model, or when the team has Calico expertise.

---

## 5. Cheat Sheet

```
eBPF: programmable kernel-space packet processing
Cilium: eBPF-based CNI for K8s (replaces kube-proxy + iptables)
XDP: fastest hook, runs before kernel stack
BPF maps: O(1) hash tables for Service routing
Hubble: Cilium's observability platform (flow logs, service map)

Cilium replaces:
  kube-proxy → eBPF Service routing
  iptables policies → eBPF identity-based filtering
  sidecar L7 → eBPF L7 interception (partial)

Performance: O(1) at all scales vs iptables O(n)
```

---

## 🔗 Connections

### Previous: iptables (2.3) and kube-proxy (4.1) showed the traditional approach. eBPF is the next-generation replacement.
### Next: Multi-Region and Hybrid Connectivity — the final topic bringing all networking concepts together at global scale.
