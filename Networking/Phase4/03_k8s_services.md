# Kubernetes Services — ClusterIP, NodePort, LoadBalancer

> 🔹 **Phase:** 4 — Kubernetes Networking  
> 🔹 **Topic:** K8s Services Deep Dive  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Pods are ephemeral — they get new IPs on every restart, scale up/down, and move across nodes. You can't hardcode a pod IP. **Services** provide a stable endpoint (DNS name + virtual IP) that always routes to the current set of healthy pods.

---

## 2. Core Concept (Deep Dive)

### ClusterIP — Internal Load Balancing

```
Service: my-api (ClusterIP: 10.96.50.100:80)
Selector: app=my-api
Endpoints: [10.244.1.5:8080, 10.244.1.6:8080, 10.244.2.8:8080]

What happens when a pod calls 10.96.50.100:80:
  1. Packet enters PREROUTING chain
  2. iptables matches KUBE-SERVICES rule for 10.96.50.100:80
  3. Jumps to KUBE-SVC-XXXX chain (load balancing)
  4. Random probability selects one KUBE-SEP endpoint
  5. DNAT rewrites destination: 10.96.50.100:80 → 10.244.1.5:8080
  6. Packet delivered to selected pod

Key facts:
  • ClusterIP doesn't exist on any interface — it's purely iptables
  • Can't ping ClusterIP (no host responds, only TCP/UDP DNAT works)
  • kube-proxy maintains these rules on every node
  • Load balancing is random (iptables) or round-robin (IPVS)
```

### NodePort — External Access (Simple)

```
Service type: NodePort
  ClusterIP: 10.96.50.100:80
  NodePort: 30080 (allocated from 30000-32767)

External client → <any-node-ip>:30080
  1. iptables KUBE-NODEPORTS matches port 30080
  2. DNAT → random pod (same as ClusterIP)
  3. If pod is on a different node → forward there

Downsides:
  • Ugly ports (30000-32767)
  • Must open these ports in Security Groups
  • No TLS termination
  • Use only for development/testing
```

### LoadBalancer — Production External Access

```
Service type: LoadBalancer

K8s creates:
  1. ClusterIP (internal)
  2. NodePort (per node)
  3. Cloud provider creates an ALB/NLB
     → Points to NodePort on all nodes
     → Or: directly to pod IPs (IP target type)

For EKS:
  → AWS Load Balancer Controller watches Service annotations
  → Creates NLB (default for type: LoadBalancer)
  → Registers node IPs and NodePorts as targets
  → With service.beta.kubernetes.io/aws-load-balancer-type: "external"
     and service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
     → Registers pod IPs directly (more efficient)
```

### ExternalName — DNS CNAME

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  type: ExternalName
  externalName: mydb.abcdef.us-east-1.rds.amazonaws.com
```
```
DNS: my-db.default.svc.cluster.local
  → CNAME → mydb.abcdef.us-east-1.rds.amazonaws.com
  → App uses K8s DNS name, infrastructure manages the actual endpoint
```

### Headless Service (clusterIP: None)

```
DNS returns pod IPs directly (no ClusterIP proxy):
  dig my-db.default.svc.cluster.local
  → 10.244.1.5
  → 10.244.1.6
  → 10.244.2.8

Used for:
  • StatefulSets (each pod gets a predictable DNS name: pod-0.my-db.ns.svc)
  • Databases (client needs direct pod connections)
  • Custom load balancing (app decides which pod to connect to)
```

### externalTrafficPolicy

```
Cluster (default):
  Traffic → any node → SNAT to node IP → forward to any pod
  ✗ Source IP lost
  ✓ Even distribution

Local:
  Traffic → only nodes WITH local pods → direct to local pod
  ✓ Source IP preserved
  ✗ Uneven if pods aren't spread evenly
  ✗ Nodes without pods get traffic but drop it (configure health checks)
```

---

## 3. Failure Scenarios

### Scenario 1: Service Has No Endpoints

| Attribute | Detail |
|-----------|--------|
| **Symptom** | ClusterIP reachable but connection refused or timeout |
| **Root Cause** | No pods match the Service's label selector |
| **Debug** | `kubectl get endpoints <service>` → empty? `kubectl get pods -l <selector>` → do labels match? |

### Scenario 2: Source IP Needed but Lost

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Application logs show node IPs instead of client IPs |
| **Root Cause** | `externalTrafficPolicy: Cluster` (default) SNAT's the source |
| **Fix** | Set `externalTrafficPolicy: Local` on the Service |

---

## 4. Interview Preparation

### Q1: How does ClusterIP work under the hood?

**Answer:** kube-proxy creates iptables DNAT rules on every node. The ClusterIP doesn't exist on any interface — it's matched by iptables. When a packet's destination matches ClusterIP:port, iptables rewrites the destination to a randomly selected pod IP:targetPort. conntrack tracks the mapping for return traffic.

### Q2: What's the difference between NodePort and LoadBalancer?

**Answer:** NodePort exposes the service on every node at a static port (30000-32767). LoadBalancer creates NodePorts PLUS provisions a cloud load balancer (ALB/NLB) that distributes external traffic to the NodePorts. In production, always use LoadBalancer (or Ingress) — NodePort is for testing.

---

## 5. Cheat Sheet

```
ClusterIP:     Internal only, iptables DNAT, default type
NodePort:      ClusterIP + port on every node (30000-32767)
LoadBalancer:  NodePort + cloud LB (NLB in EKS)
ExternalName:  DNS CNAME to external service
Headless:      clusterIP: None → DNS returns pod IPs

Debug:
  kubectl get svc                    # List services
  kubectl get endpoints <svc>        # Check endpoints  
  kubectl describe svc <svc>         # Full details
  iptables -t nat -L KUBE-SERVICES   # View DNAT rules
```

---

## 🔗 Connections

### Previous: Topics 4.1-4.2 covered how pods get IPs and communicate. Services add a stable abstraction layer.
### Next: Topic 4.4 — Ingress Controllers expose Services to external traffic with HTTP routing.
