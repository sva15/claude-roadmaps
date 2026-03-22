# NetworkPolicy — Kubernetes Firewall

> 🔹 **Phase:** 4 — Kubernetes Networking  
> 🔹 **Topic:** Network Policies  
> 🔹 **Priority:** HIGH

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

By default, every pod can talk to every other pod in the cluster — no firewall, no segmentation. In production, this is a security risk: a compromised pod can reach the database, other namespaces, or the internet.

**NetworkPolicy** is the K8s firewall. It defines rules that restrict which pods can send or receive traffic and on which ports.

> 💡 **Key Insight:** NetworkPolicy is **default allow, opt-in deny.** Adding a policy to a pod makes it deny-all for that direction, then explicitly allows specified sources/destinations.

---

## 2. Core Concept (Deep Dive)

### How NetworkPolicy Works

```
Default (no NetworkPolicy):
  ALL ingress allowed
  ALL egress allowed
  → Every pod can talk to every other pod

After applying a NetworkPolicy with spec.policyTypes=["Ingress"]:
  → Matching pods: ALL ingress DENIED except what the policy allows
  → Unmatched pods: still allow all (unchanged)
  → Egress: unchanged (not specified in policyTypes)

After applying a NetworkPolicy with spec.policyTypes=["Ingress","Egress"]:
  → Matching pods: ALL ingress AND egress DENIED except what policy allows
  → MUST explicitly allow DNS (port 53 UDP) or all DNS will break!
```

### Complete Example: Three-Tier Application

```yaml
# 1. Default deny all in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # Matches ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
---
# 2. Frontend: allow from internet, allow to API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels: { app: frontend }
  policyTypes: [Ingress, Egress]
  ingress:
  - from: []              # From anywhere (internet via Ingress)
    ports: [{ port: 3000 }]
  egress:
  - to:
    - podSelector: { matchLabels: { app: api } }
    ports: [{ port: 8080 }]
  - to: [{ namespaceSelector: {} }]     # DNS (any namespace)
    ports: [{ port: 53, protocol: UDP }]
---
# 3. API: allow from frontend only, allow to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: { matchLabels: { app: frontend } }
    ports: [{ port: 8080 }]
  egress:
  - to:
    - podSelector: { matchLabels: { app: database } }
    ports: [{ port: 5432 }]
  - to: [{ namespaceSelector: {} }]
    ports: [{ port: 53, protocol: UDP }]
---
# 4. Database: allow from API only, no egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels: { app: database }
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: { matchLabels: { app: api } }
    ports: [{ port: 5432 }]
  egress:
  - to: [{ namespaceSelector: {} }]
    ports: [{ port: 53, protocol: UDP }]
```

### CNI Requirement

```
CNI Plugin          NetworkPolicy Support
─────────────────────────────────────────
AWS VPC CNI         ❌ (alone) — need Calico alongside
Calico              ✅ (iptables or eBPF)
Cilium              ✅ (eBPF, supports L7 policies too)
Flannel             ❌ (no policy support)
Weave               ✅
```

---

## 3. Failure Scenarios

### Scenario 1: DNS Breaks After Adding NetworkPolicy

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Pods can't resolve DNS after NetworkPolicy applied |
| **Root Cause** | Egress policy blocks DNS (CoreDNS is in kube-system namespace) |
| **Fix** | Add egress rule allowing port 53 UDP to any namespace |

### Scenario 2: Policy Applied But Traffic Still Flows

| Attribute | Detail |
|-----------|--------|
| **Symptom** | NetworkPolicy exists but pods still communicate freely |
| **Root Cause** | CNI doesn't support NetworkPolicy (Flannel, VPC CNI alone) |
| **Fix** | Install Calico (in policy-only mode alongside VPC CNI) |

---

## 4. Interview Preparation

### Q1: Explain the default behavior and how NetworkPolicy changes it.

**Answer:** Default: all traffic allowed. NetworkPolicy is opt-in. When a policy selects a pod (via podSelector) and specifies a policyType (Ingress/Egress), that pod becomes deny-all for that direction, with explicit allow rules. Pods without any policy are unaffected.

### Q2: What's the most common mistake with NetworkPolicy?

**Answer:** Forgetting DNS. When you add an Egress policy, ALL egress is denied except what's explicitly allowed. If you don't allow port 53 UDP, the pod can't resolve any DNS names — including service names. Always include a DNS egress rule.

---

## 5. Cheat Sheet

```
No policy → all allowed
Policy applied → deny-all + explicit allows
Always allow DNS (port 53 UDP) in egress policies
Requires CNI support: Calico, Cilium (not Flannel, not VPC CNI alone)
EKS: install Calico policy engine alongside VPC CNI
Default deny: podSelector: {} with policyTypes: [Ingress, Egress]
```

---

## 🔗 Connections

### Previous: iptables (2.3) showed how the kernel filters packets. NetworkPolicy is the K8s abstraction that generates iptables/eBPF rules.
### Next: Topic 4.6 — CoreDNS provides the DNS service that NetworkPolicy must always allow.
