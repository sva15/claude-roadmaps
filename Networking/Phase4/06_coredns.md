# CoreDNS — DNS Inside Kubernetes

> 🔹 **Phase:** 4 — Kubernetes Networking  
> 🔹 **Topic:** CoreDNS  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Every K8s Service has a name (e.g., `my-api`). When a pod calls `my-api:8080`, something must translate that name to the Service's ClusterIP. That something is **CoreDNS** — the cluster's internal DNS server.

CoreDNS runs as a Deployment in the `kube-system` namespace, exposed via a ClusterIP Service (typically `10.96.0.10`). Every pod's `/etc/resolv.conf` points to CoreDNS.

---

## 2. Core Concept (Deep Dive)

### DNS Resolution Inside K8s

```
Pod resolves "my-api":

/etc/resolv.conf:
  nameserver 10.96.0.10
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5

Resolution order (because "my-api" has 0 dots, < ndots:5):
  1. my-api.default.svc.cluster.local → A record → 10.96.50.100 ✓ FOUND

Pod resolves "my-api.other-ns":
  1. my-api.other-ns.default.svc.cluster.local → NXDOMAIN
  2. my-api.other-ns.svc.cluster.local → A record ✓ FOUND

Pod resolves "api.example.com" (external, 2 dots < 5):
  1. api.example.com.default.svc.cluster.local → NXDOMAIN
  2. api.example.com.svc.cluster.local → NXDOMAIN
  3. api.example.com.cluster.local → NXDOMAIN
  4. api.example.com → A record ✓ (finally!)
  → 4 wasted queries! Use trailing dot: api.example.com.
```

### CoreDNS Corefile

```
.:53 {
    errors                     # Log errors
    health {                   # Health check endpoint on :8080
        lameduck 5s
    }
    ready                      # Readiness check on :8181
    kubernetes cluster.local in-addr.arpa ip6.arpa {  # K8s plugin
        pods insecure          # A records for pod IPs
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153           # Metrics endpoint
    forward . /etc/resolv.conf # Forward external queries upstream
    cache 30                   # Cache responses for 30s
    loop                       # Detect infinite forwarding loops
    reload                     # Auto-reload on config change
    loadbalance                # Randomize A record order
}
```

### DNS Record Types in K8s

| Query | Record | Answer |
|-------|--------|--------|
| `my-svc.ns.svc.cluster.local` | A | ClusterIP (10.96.50.100) |
| `my-svc.ns.svc.cluster.local` | SRV | Port + target host |
| `pod-0.my-svc.ns.svc.cluster.local` | A | Pod IP (StatefulSet) |
| `10-244-1-5.ns.pod.cluster.local` | A | Pod IP (pod DNS record) |
| `my-svc.ns.svc.cluster.local` (headless) | A | Multiple pod IPs |

### ndots:5 Performance Fix

```yaml
# Option 1: Trailing dot in application URLs
# api.example.com.  ← Fully qualified, skips search list

# Option 2: Override in pod spec
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"    # Only search for names with < 2 dots

# Option 3: NodeLocal DNS Cache (best for production)
# Deploy as DaemonSet → runs on every node
# Pods use local cache (169.254.20.10) instead of CoreDNS Service
# Reduces: cross-node DNS traffic, CoreDNS load, latency
```

### NodeLocal DNS Cache

```
Without NodeLocal DNS: Pod → CoreDNS Service → CoreDNS pod (possibly cross-node)

With NodeLocal DNS:    Pod → Local cache (same node) → CoreDNS (only on miss)

Benefits:
  ✅ Reduces CoreDNS load by 80%+
  ✅ Uses TCP (conntrack) instead of UDP (avoids conntrack race condition)
  ✅ DNS queries never leave the node (for cached entries)
  ✅ Reduces latency significantly
```

---

## 3. Failure Scenarios

### Scenario 1: CoreDNS Pods Crashing (OOMKilled)

| Attribute | Detail |
|-----------|--------|
| **Symptom** | DNS resolution intermittently fails across the cluster |
| **Root Cause** | CoreDNS memory exhausted from too many DNS queries/records |
| **Fix** | 1. Increase CoreDNS memory limits. 2. Scale CoreDNS replicas. 3. Deploy NodeLocal DNS Cache. 4. Fix applications doing excessive DNS queries. |

### Scenario 2: DNS Resolution Timeout

| Attribute | Detail |
|-----------|--------|
| **Symptom** | DNS queries take 5+ seconds |
| **Root Cause** | conntrack race condition with UDP (kernel drops packets). Or CoreDNS overloaded. |
| **Fix** | Deploy NodeLocal DNS Cache (uses TCP, avoids conntrack race). Scale CoreDNS. |

### Scenario 3: External DNS Not Resolving

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Internal K8s DNS works but external domains (google.com) fail |
| **Root Cause** | CoreDNS `forward` directive can't reach upstream DNS. NAT GW missing or SG blocking UDP 53 outbound. |
| **Debug** | `kubectl exec <coredns-pod> -- nslookup google.com 8.8.8.8`. Check VPC DNS resolution (enableDnsSupport). |

---

## 4. Interview Preparation

### Q1: Walk through what happens when a pod calls `my-svc.other-ns:8080`.

**Answer:** 1. Pod's libc reads `/etc/resolv.conf` → nameserver 10.96.0.10 (CoreDNS). 2. `my-svc.other-ns` has 1 dot, ndots=5, so search list is tried. 3. First: `my-svc.other-ns.default.svc.cluster.local` → NXDOMAIN. 4. Second: `my-svc.other-ns.svc.cluster.local` → CoreDNS looks up Service → returns ClusterIP. 5. Pod connects to ClusterIP → iptables DNAT to pod IP.

### Q2: How do you fix slow DNS in Kubernetes?

**Answer:** Deploy NodeLocal DNS Cache DaemonSet — each node gets a local cache, queries never cross the network for cached entries. Also: use trailing dots for external domains (`api.example.com.`), reduce ndots from 5 to 2, scale CoreDNS replicas, optimize autopath plugin.

### Q3: Why does DNS sometimes fail under high load in K8s?

**Answer:** Linux kernel conntrack race condition with UDP. Two DNS queries (A and AAAA) from the same socket can get the same conntrack tuple, causing one to be dropped → 5-second timeout before retry. NodeLocal DNS Cache fixes this by using TCP to CoreDNS.

---

## 5. Cheat Sheet

```
CoreDNS Service: kubectl get svc -n kube-system kube-dns
CoreDNS pods:    kubectl get pods -n kube-system -l k8s-app=kube-dns
CoreDNS config:  kubectl get cm -n kube-system coredns -o yaml
CoreDNS logs:    kubectl logs -n kube-system -l k8s-app=kube-dns

DNS format:      <svc>.<ns>.svc.cluster.local
Headless:        Returns pod IPs directly
StatefulSet:     <pod-name>.<svc>.<ns>.svc.cluster.local

ndots:5 → 4 extra queries for external domains
Fix: trailing dot (api.example.com.) or ndots:2 or NodeLocal DNS
```

---

## 🔗 Connections

### Previous: DNS fundamentals (1.4), CoreDNS introduction in K8s networking model (4.1). This deep dive covers configuration, performance, and troubleshooting.
### Next: Phase 5 — Advanced Topics (BGP, Service Mesh, eBPF, Multi-Region) take everything you've learned to senior-level depth.
