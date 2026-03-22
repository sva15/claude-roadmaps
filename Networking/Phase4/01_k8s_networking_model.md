# The Kubernetes Networking Model — 4 Rules

> 🔹 **Phase:** 4 — Kubernetes Networking  
> 🔹 **Topic:** K8s Networking Model and CNI  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Kubernetes runs thousands of pods across dozens of nodes. Each pod needs a unique IP and must communicate with every other pod. But pods are ephemeral — they're created, destroyed, and moved across nodes constantly.

K8s networking solves: **How does every pod talk to every other pod, regardless of which node it's on, without NAT, and at scale?**

### The 4 Fundamental Rules

Kubernetes mandates these networking rules (the CNI plugin must implement them):

```
Rule 1: Every pod gets its own unique IP address
Rule 2: Pods on the same node can communicate directly
Rule 3: Pods on different nodes can communicate WITHOUT NAT
Rule 4: The IP a pod sees as its own is the same IP others use to reach it
```

These rules create a **flat network** — every pod can reach every other pod by IP, just like every VM on a traditional network.

> 💡 **Key Insight:** K8s doesn't implement networking itself. It defines the RULES. A CNI (Container Network Interface) plugin implements them. Choosing the right CNI is a critical architecture decision.

---

## 2. Core Concept (Deep Dive)

### Pod-to-Pod Communication — Same Node

```
Node (10.0.1.50)
┌──────────────────────────────────────────────────────┐
│                                                        │
│  Pod A (10.244.1.5)    Pod B (10.244.1.6)            │
│      │                     │                          │
│  [veth-a] ──── bridge ──── [veth-b]                  │
│            (cni0/cbr0)                                │
│                                                        │
│  Packet: src=10.244.1.5 → dst=10.244.1.6             │
│  Path: eth0(A) → veth-a → bridge → veth-b → eth0(B) │
│  No NAT. No routing needed. Pure L2 switching.        │
└──────────────────────────────────────────────────────┘
```

### Pod-to-Pod Communication — Different Nodes

**Two approaches depending on CNI:**

```
Overlay (Flannel, Calico VXLAN):
  Pod A (10.244.1.5) on Node 1 → Pod C (10.244.2.8) on Node 2
  
  Node 1: Packet encapsulated in VXLAN (outer: Node1→Node2, inner: PodA→PodC)
  Network: Regular IP packet between nodes
  Node 2: Decapsulated, delivered to Pod C
  
  Pro: Works anywhere (any network, overlapping node CIDRs)
  Con: Encapsulation overhead (MTU reduction, CPU for encap/decap)

Native/Routed (AWS VPC CNI, Calico BGP):
  Pod A (10.0.1.51) on Node 1 → Pod C (10.0.1.55) on Node 2
  
  Node 1: Packet sent directly — pod IP is a real VPC IP
  Network: VPC routing delivers to correct node
  Node 2: Route table sends to correct pod's veth
  
  Pro: No encapsulation, full MTU, VPC-native security  
  Con: Consumes VPC IPs, limited by ENI/IP quotas
```

### CNI Plugins Comparison

| CNI | Approach | Pod IPs | Use Case |
|-----|----------|---------|----------|
| **AWS VPC CNI** | Native (VPC IPs) | From VPC CIDR via ENI secondary IPs | EKS (default). Pods are first-class VPC citizens. |
| **Calico** | BGP or VXLAN | Own CIDR (10.244.x.x) | Network Policy enforcement. |
| **Flannel** | VXLAN overlay | Own CIDR (10.244.x.x) | Simple, lightweight, no Network Policy. |
| **Cilium** | eBPF | Own CIDR or VPC | Advanced: eBPF-based, L7 policies, observability. |
| **Weave** | VXLAN + mesh | Own CIDR | Easy multi-cloud. |

### AWS VPC CNI — How EKS Does It

```
Node (m5.large: max 3 ENIs × 10 IPs each = 29 pods max):

Primary ENI (eth0): 10.0.1.50 (node IP)
├── Secondary IP: 10.0.1.51 → Pod A
├── Secondary IP: 10.0.1.52 → Pod B
└── Secondary IP: 10.0.1.53 → Pod C

Secondary ENI (eth1): 10.0.1.60
├── Secondary IP: 10.0.1.61 → Pod D
└── Secondary IP: 10.0.1.62 → Pod E

Each pod gets a REAL VPC IP:
  → VPC routing delivers to the correct node
  → Node's /32 route delivers to the correct pod's veth
  → No overlay, no encapsulation
  → Security Groups can be applied directly to pods (SG for Pods)
  → VPC Flow Logs show pod traffic

Limitation: Max pods = (ENIs × IPs per ENI) - 1
  t3.micro:  2 × 2 = 3 pods
  t3.medium: 3 × 6 = 17 pods
  m5.large:  3 × 10 = 29 pods
  m5.xlarge: 4 × 15 = 58 pods
```

---

## 3. Services — The Stable Abstraction

### Why Services Exist

Pods are ephemeral — their IPs change on every restart. Services provide a **stable virtual IP (ClusterIP)** that routes to the current set of healthy pods.

```
Service: my-api (ClusterIP: 10.96.50.100)
  selector: app=my-api
  port: 80 → targetPort: 8080

Endpoints (auto-discovered):
  10.244.1.5:8080 (pod-1)
  10.244.1.6:8080 (pod-2)
  10.244.2.8:8080 (pod-3)

Packet to 10.96.50.100:80 →
  iptables DNAT → randomly selects one pod IP:8080 →
  Packet rewritten: dst=10.244.1.5:8080
```

### Service Types

| Type | Accessible From | How |
|------|----------------|-----|
| **ClusterIP** | Inside cluster only | Virtual IP → iptables DNAT to pod |
| **NodePort** | Inside + node's IP:30000-32767 | Allocates port on every node → DNAT to pod |
| **LoadBalancer** | External (internet) | Creates AWS ELB → NodePort → pod |
| **ExternalName** | Inside cluster | DNS CNAME → external DNS name |
| **Headless (clusterIP: None)** | Inside cluster | DNS returns pod IPs directly (no proxy) |

### Headless Services — When You Need Pod IPs

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  clusterIP: None    # ← Headless
  selector:
    app: my-db
  ports:
  - port: 5432
```

```
DNS: my-db.default.svc.cluster.local
  → Returns: 10.244.1.5, 10.244.1.6, 10.244.2.8 (all pod IPs)
  → No ClusterIP proxy
  → Used for: StatefulSets, databases, any app that needs direct pod-to-pod
```

---

## 4. DNS in Kubernetes — CoreDNS

```
Pod DNS resolution:
  my-service                          → my-service.default.svc.cluster.local
  my-service.other-ns                 → my-service.other-ns.svc.cluster.local
  my-service.other-ns.svc.cluster.local → fully qualified (no search)

Pod's /etc/resolv.conf:
  nameserver 10.96.0.10              ← CoreDNS ClusterIP
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
```

**ndots:5 performance issue:**
```
Querying: api.example.com (2 dots, ndots=5 means < 5 dots):
  1. api.example.com.default.svc.cluster.local → NXDOMAIN
  2. api.example.com.svc.cluster.local → NXDOMAIN  
  3. api.example.com.cluster.local → NXDOMAIN
  4. api.example.com → RESOLVED ✓

Four DNS queries instead of one! For external domains, add a trailing dot:
  api.example.com.    ← Fully qualified, skips search list
```

---

## 5. Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
  - to:                    # Allow DNS
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
```

**What this does:**
- `api` pods can only receive traffic from `frontend` pods on port 8080
- `api` pods can only send to `database` pods on 5432 and DNS (port 53)
- All other ingress AND egress is DENIED

> 🚨 **Requires CNI support!** AWS VPC CNI alone doesn't enforce Network Policies. You need Calico or Cilium alongside it.

---

## 6. Failure Scenarios

### Scenario 1: Pod Can't Reach ClusterIP

| Attribute | Detail |
|-----------|--------|
| **Symptom** | `curl 10.96.50.100:80` from a pod times out |
| **Root Cause** | 1. No endpoints (no pods match selector). 2. kube-proxy iptables rules stale. 3. Network Policy blocking. |
| **Debug** | 1. `kubectl get endpoints my-service` — are there endpoints? 2. `iptables -t nat -L KUBE-SERVICES -n | grep 10.96.50.100` — rule exists? 3. Check Network Policies: `kubectl get networkpolicy -n <ns>`. |

### Scenario 2: DNS Resolution Slow in Cluster

| Attribute | Detail |
|-----------|--------|
| **Symptom** | External API calls slow due to DNS (2-5s extra) |
| **Root Cause** | ndots:5 — every external domain triggers 4 extra queries |
| **Fix** | 1. Use trailing dots in URLs: `api.example.com.` 2. Set ndots:2 in pod spec. 3. Deploy NodeLocal DNS cache. |

### Scenario 3: Pods Running Out on a Node (EKS)

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Pending pods with "too many pods" error, but node has plenty of CPU/RAM |
| **Root Cause** | ENI IP limit reached. Instance type determines max pods. |
| **Fix** | 1. Use larger instance type. 2. Enable prefix delegation (VPC CNI feature — assigns /28 prefixes instead of individual IPs, significantly increasing max pods). |

### Scenario 4: Cross-Node Pod Communication Fails

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Pods on same node communicate but cross-node fails |
| **Root Cause** | 1. SG on nodes doesn't allow pod CIDR. 2. Missing routes (overlay CNI). 3. MTU mismatch (overlay overhead). |
| **Debug** | 1. Check node SGs: allow all traffic from nodes SG. 2. `ip route` on nodes — are routes to other node's pod CIDRs present? 3. `ip link show flannel.1` — check MTU. |

---

## 7. Debugging Playbook

```
K8s Networking Debug:

DNS fails:
  kubectl exec <pod> -- nslookup kubernetes
  kubectl exec <pod> -- cat /etc/resolv.conf
  kubectl logs -n kube-system -l k8s-app=kube-dns

Service unreachable:
  kubectl get endpoints <service>  → any endpoints?
  kubectl get svc <service>        → ClusterIP + port correct?
  iptables -t nat -L KUBE-SERVICES -n | grep <ClusterIP>

Pod-to-pod fails:
  kubectl exec <pod-a> -- ping <pod-b-ip>
  nsenter -t <pid> -n tcpdump -i eth0 -nn
  Check SGs on nodes: allow all from node SG

External access fails:
  kubectl exec <pod> -- curl -v https://example.com
  Check NAT GW, route table (0.0.0.0/0 → nat-gw)
  Check Network Policies (egress to 0.0.0.0/0 allowed?)
```

---

## 8. Interview Preparation

### Q1: Explain the Kubernetes networking model.

**Answer:** Four rules: (1) every pod gets a unique IP, (2) pods on the same node communicate directly (L2 bridge or direct routes), (3) pods on different nodes communicate without NAT (overlay or native routing), (4) the IP a pod sees is the same IP others use. A CNI plugin implements these rules. This creates a flat network — any pod reaches any pod by IP.

### Q2: How does the AWS VPC CNI work?

**Answer:** Each pod gets a real VPC IP from ENI secondary IPs. The CNI pre-allocates secondary IPs on the node's ENIs. When a pod starts, it's assigned one of these IPs. No overlay — VPC routing delivers traffic directly. Max pods per node = (number of ENIs × IPs per ENI) - 1, determined by instance type. Advantage: pods are first-class VPC citizens (SGs, Flow Logs, native monitoring).

### Q3: ClusterIP doesn't work — debug steps?

**Answer:** 1. `kubectl get endpoints <svc>` — if empty, selector doesn't match any pods (check labels). 2. `kubectl get svc` — verify port mapping. 3. `iptables -t nat -L KUBE-SERVICES -n | grep <ClusterIP>` — are rules present? 4. `kubectl logs kube-proxy` — errors? 5. Test directly: `curl <pod-ip>:<targetPort>` — if works, issue is in Service/iptables layer.

### Q4: What are Network Policies and how do they work?

**Answer:** Network Policies are K8s resources that define ingress/egress rules for pods using label selectors. By default, all traffic is allowed. Adding a policy to a pod makes it deny-all for that direction, then explicitly allows specified traffic. Implementation depends on CNI — Calico translates policies to iptables rules, Cilium uses eBPF. Always allow DNS egress (port 53) in policies.

---

## 9. Cheat Sheet

```
K8s Networking Rules: unique pod IP, no NAT, flat network
Service types: ClusterIP (internal) → NodePort → LoadBalancer
ClusterIP = iptables DNAT (not a real IP on any interface)
Headless (clusterIP: None) = DNS returns pod IPs directly

AWS VPC CNI: real VPC IPs, no overlay, ENI-limited pods
Max pods = (ENIs × IPs per ENI) - 1

DNS: <svc>.<ns>.svc.cluster.local
CoreDNS: 10.96.0.10 (check: kubectl get svc -n kube-system kube-dns)
ndots:5 → trailing dot for external domains

Network Policy: deny-all when created, then whitelist
  → Requires CNI support (Calico, Cilium — not VPC CNI alone)
  → Always allow DNS egress (port 53 UDP)
```

---

## 🔗 Connections

### Previous: All Phase 2 Linux networking topics (interfaces, routing, iptables, namespaces) directly implement K8s networking. Phase 3 VPC concepts map to how EKS pods use VPC IPs.

### Next: Topic 4.2 — Ingress Controllers, Service Mesh, and East-West Traffic — how external traffic reaches pods and how pod-to-pod traffic is managed with service meshes.
