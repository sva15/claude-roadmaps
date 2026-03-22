# DNS — The Thing That Breaks Most "Networking" Issues

> 🔹 **Phase:** 1 — Network Foundations  
> 🔹 **Topic:** DNS (Domain Name System)  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Computers talk to each other using IP addresses: `142.250.190.78`. But humans can't remember `142.250.190.78` — they remember `google.com`.

**DNS is the phone book of the internet.** You give it a name, it gives you an IP address.

But here's why it's critical for DevOps: DNS isn't just for websites. In Kubernetes, when `payment-service` calls `order-service`, it uses DNS to find it. When your app connects to `my-database.cluster-xyz.us-east-1.rds.amazonaws.com`, it uses DNS. When Route53 does failover, it uses DNS.

**80% of "I can't connect to service X" incidents are DNS failures**, not network failures.

### The Analogy: The Phone Book Chain

Imagine you need to call "John Smith in New York." You don't have a giant phone book with every person on Earth. Instead:

1. You call **Information (411)** → "I need John Smith in New York"
2. Information calls the **New York directory** → "Do you have John Smith?"
3. New York directory has the number → returns it
4. Information tells you → you dial John

DNS works exactly this way, but with a chain of servers:

```
You → Local Resolver → Root Servers → .com Servers → google.com's Servers → IP address
```

### Why Does It Exist?

In the early internet (ARPANET), all hostnames were mapped in a single file: `/etc/hosts`. Every computer downloaded this file periodically. When the internet had 100 hosts, this was fine. With millions of hosts, it became impossible.

DNS was created in 1983 (RFC 882/883) as a **distributed, hierarchical** naming system. No single server has all the answers — they cooperate by following a chain from the top (root servers) down to the authoritative source.

> 💡 **Key Insight:** When something "can't connect," always check DNS first. Before blaming the network, before blaming the firewall, before blaming the application — run `dig` or `nslookup`. If the name doesn't resolve to the right IP, nothing else matters.

---

## 2. Why This Matters in DevOps

### Where This Appears in Real Systems

| Scenario | DNS at Work |
|----------|-------------|
| K8s pod calls another service by name | CoreDNS resolves `service.namespace.svc.cluster.local` → ClusterIP |
| ALB health check hits a domain | Route53 / external DNS resolves the endpoint |
| Application connects to RDS | DNS resolves the RDS endpoint to the current primary IP |
| Blue/green deployment via DNS | Route53 weighted/failover policy switches traffic |
| VPN connection to on-premises | Split-DNS or DNS forwarding to resolve internal domains |
| Service migration | Lower TTL → switch DNS → old clients drain → raise TTL |
| Private services within AWS | Route53 Private Hosted Zone resolves internal names |

### Why DevOps Engineers Must Know This

1. **DNS is the first hop in every connection.** Before TCP, before TLS, before HTTP — DNS must resolve the name to an IP. If DNS fails, nothing works.

2. **TTL caching causes "stale" routing.** You change a DNS record, but clients still go to the old IP for minutes/hours because they cached the old answer. Understanding TTL is critical for migrations, failovers, and blue/green deployments.

3. **CoreDNS is the backbone of Kubernetes service discovery.** Every pod uses CoreDNS to find other services. When CoreDNS fails, the entire cluster's service-to-service communication breaks.

4. **Debugging DNS is often the fastest path to resolution.** Running `dig` takes 2 seconds and immediately tells you if the problem is DNS or something else.

---

## 3. Core Concept (Deep Dive)

### The DNS Resolution Chain

When you type `google.com`, this happens:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Step 1: Browser/OS Cache Check                                      │
│ Browser has its own DNS cache. OS has a cache.                      │
│ If found → use cached IP immediately (no network call)              │
└────────────────────────────┬────────────────────────────────────────┘
                             │ Cache miss
┌────────────────────────────▼────────────────────────────────────────┐
│ Step 2: Local Resolver (/etc/resolv.conf)                           │
│ OS sends query to the recursive resolver configured here            │
│ On AWS EC2: VPC DNS (x.x.x.2 — VPC base + 2)                      │
│ On K8s pod: CoreDNS IP (typically 10.96.0.10)                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│ Step 3: Recursive Resolver                                          │
│ Your ISP's DNS or a public one (8.8.8.8, 1.1.1.1)                 │
│ It does the heavy lifting — queries the DNS hierarchy               │
│ Has its own cache (honors TTL)                                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │ Cache miss — recursive resolution
┌────────────────────────────▼────────────────────────────────────────┐
│ Step 4: Root Servers (13 clusters worldwide: a.root-servers.net)    │
│ "I don't know google.com, but .com is handled by these servers"    │
│ Returns: NS records for .com TLD servers                            │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│ Step 5: TLD Servers (.com, .org, .io, etc.)                        │
│ "I don't know google.com, but google.com's NS are these servers"   │
│ Returns: NS records for google.com's authoritative servers          │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│ Step 6: Authoritative Name Server (ns1.google.com)                  │
│ "Yes! google.com's A record is 142.250.190.78"                     │
│ Returns: A record with IP and TTL                                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                    IP returned to your app
                    Cached at each level for TTL seconds
```

### DNS Record Types

| Type | Purpose | Example | When You Care |
|------|---------|---------|--------------|
| **A** | Maps name → IPv4 address | `google.com → 142.250.190.78` | Most lookups |
| **AAAA** | Maps name → IPv6 address | `google.com → 2607:f8b0:4004:800::200e` | IPv6 environments |
| **CNAME** | Alias to another name | `www.example.com → example.com` | Subdomains, CDN |
| **MX** | Mail server | `example.com → mail.example.com (priority 10)` | Email routing |
| **TXT** | Arbitrary text | `example.com → "v=spf1 include:..."` | SPF, DKIM, domain verification |
| **NS** | Nameserver for a zone | `example.com → ns1.awsdns-00.com` | DNS delegation |
| **SRV** | Service discovery | `_http._tcp.example.com → port 80, host.example.com` | Service mesh, SIP |
| **PTR** | Reverse DNS (IP → name) | `78.190.250.142.in-addr.arpa → google.com` | Email verification, logging |
| **SOA** | Start of Authority | Zone metadata (serial, refresh, retry) | Zone management |
| **ALIAS/ANAME** | Like CNAME, but works on apex | `example.com → d1234.cloudfront.net` | AWS ALB on apex domain |

### CNAME Restrictions

**CNAME cannot be used on the apex domain** (also called zone apex or naked domain):

```
✅ www.example.com  CNAME  my-alb-1234.us-east-1.elb.amazonaws.com
❌ example.com      CNAME  my-alb-1234.us-east-1.elb.amazonaws.com  ← INVALID

✅ example.com      ALIAS  my-alb-1234.us-east-1.elb.amazonaws.com  ← AWS-specific
```

This is why Route53 has **ALIAS records** — they work like CNAME but can be used on the apex domain.

### TTL (Time To Live)

TTL tells resolvers how long to cache the answer (in seconds):

```
dig google.com

;; ANSWER SECTION:
google.com.   300   IN   A   142.250.190.78
              ^^^
              TTL = 300 seconds (5 minutes)
```

**TTL implications:**

| TTL | Behavior | Use Case |
|-----|----------|----------|
| 60s (low) | Resolvers re-query every minute. Fast failover but more DNS queries. | During migrations, failovers |
| 300s | Balance between freshness and query volume | Standard web services |
| 3600s (high) | Fewer queries but slow to propagate changes | Stable records that rarely change |
| 86400s (1 day) | Very long cache. Changes take up to 24 hours to propagate | NS records, MX records |

> 🚨 **Migration strategy:** Before changing a DNS record, lower the TTL to 60s and wait for the OLD TTL to expire. Then make the change. This ensures clients pick up the new record quickly. After migration, raise TTL back.

### DNS in Kubernetes: CoreDNS

Every K8s pod has DNS configured via `/etc/resolv.conf`:

```
# Inside a K8s pod:
$ cat /etc/resolv.conf

nameserver 10.96.0.10              ← CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**How search domains work:**

When a pod calls just `my-service`, the resolver appends search domains:

```
"my-service" → Try:
  1. my-service.default.svc.cluster.local   ← Found! Return ClusterIP
  2. my-service.svc.cluster.local           ← (only if #1 fails)
  3. my-service.cluster.local               ← (only if #2 fails)
  4. my-service.                            ← (only if #3 fails, external DNS)
```

The `ndots:5` option means: if the name has fewer than 5 dots, try search domains first. This is why `my-service` (0 dots) tries internal first, but `google.com` (1 dot, < 5) ALSO tries internal first — which can slow down external DNS resolution.

**Cross-namespace resolution:**

```
# Same namespace: short name works
curl http://my-service:8080

# Different namespace: must include namespace
curl http://my-service.other-namespace:8080

# Fully qualified (works from anywhere):
curl http://my-service.other-namespace.svc.cluster.local:8080
```

### DNS in AWS: Route53

**Public Hosted Zone:** Resolves on the public internet.

**Private Hosted Zone:** Resolves ONLY inside associated VPCs. Used for internal service discovery.

**Routing Policies:**

| Policy | How It Works | Use Case |
|--------|-------------|----------|
| **Simple** | Returns one record | Basic, single-endpoint services |
| **Weighted** | Returns records proportionally (e.g., 10% new, 90% old) | A/B testing, gradual migration |
| **Latency** | Returns the record with lowest latency from the client | Multi-region deployments |
| **Failover** | Primary + secondary. Switches on health check failure | Disaster recovery |
| **Geolocation** | Returns record based on client location (EU → EU server) | Data residency, localization |
| **Multivalue** | Returns up to 8 IPs, removes unhealthy ones | Basic load balancing + health checks |

---

## 4. Internal Working (Under the Hood)

### How CoreDNS Works Inside Kubernetes

```
┌────────────────────────────────────────────────────────────┐
│ Pod: my-app (namespace: production)                        │
│                                                            │
│  Application calls: http://payment-svc:8080                │
│  → glibc resolver reads /etc/resolv.conf                   │
│  → Sends UDP DNS query to 10.96.0.10 (CoreDNS)            │
└──────────────────────────┬─────────────────────────────────┘
                           │ UDP port 53
┌──────────────────────────▼─────────────────────────────────┐
│ CoreDNS Pods (kube-system namespace, Deployment)           │
│                                                            │
│ 1. Check: is "payment-svc.production.svc.cluster.local"    │
│    in the cluster? → Query K8s API for Service             │
│                                                            │
│ 2. If Service exists → return ClusterIP                    │
│    If not → NXDOMAIN (name not found)                      │
│                                                            │
│ 3. For external domains (google.com) → forward to          │
│    upstream resolver (VPC DNS: x.x.x.2)                    │
│                                                            │
│ Corefile config:                                           │
│   .:53 {                                                   │
│     kubernetes cluster.local in-addr.arpa ip6.arpa         │
│     forward . /etc/resolv.conf                             │
│     cache 30                                               │
│     loop                                                   │
│     reload                                                 │
│   }                                                        │
└────────────────────────────────────────────────────────────┘
```

### The ndots Problem (Performance Impact)

With `ndots:5`, any name with fewer than 5 dots tries all search domains FIRST:

```
Query for: api.external-service.com (2 dots, < 5)

DNS queries actually sent:
1. api.external-service.com.production.svc.cluster.local  → NXDOMAIN
2. api.external-service.com.svc.cluster.local             → NXDOMAIN
3. api.external-service.com.cluster.local                 → NXDOMAIN
4. api.external-service.com.                              → SUCCESS

That's 4 DNS queries instead of 1!
```

**Fix:** Append a trailing dot to external FQDNs: `api.external-service.com.` — this tells the resolver it's already fully qualified, skip search domains.

### VPC DNS Resolution Order

```
EC2 instance queries DNS:
│
├─ 1. Check /etc/hosts
├─ 2. Query VPC DNS (VPC_BASE + 2, e.g., 10.0.0.2)
│     │
│     ├─ Is it a Route53 Private Hosted Zone? → Resolve internally
│     ├─ Is it a VPC DHCP options custom DNS? → Forward there
│     └─ Otherwise → Forward to public recursive resolver
│
└─ Result returned to application
```

---

## 5. Mental Models

### Mental Model 1: The Library Reference System

```
You need a specific book:
"Networking Fundamentals by John Doe"

Library → "Check the Computer Science section" (Root → TLD)
CS Section → "Check shelf 'Networking'" (TLD → Authoritative)
Shelf → "Here's the book" (Authoritative → IP)

The librarian remembers the location for 5 minutes (TTL).
Within those 5 minutes, they don't have to look it up again.
```

### Mental Model 2: The Caching Waterfall

```
Request: "What is the IP of my-service?"

Level 1: App cache → Hit? → Use it       TTL: varies
              │
Level 2: OS cache → Hit? → Use it        TTL: from record
              │
Level 3: Local resolver → Hit? → Use it  TTL: from record
              │
Level 4: Recursive resolver → Hit? → Use it  TTL: from record
              │
Level 5: Authoritative → Always has the answer  TTL: set here
```

Stale data at ANY level means you see the old answer. This is why TTL matters so much during migrations.

### Mental Model 3: The K8s DNS Decision Tree

```
Pod calls a name:
│
├─ Has a dot at the end? (FQDN)
│   └─ Query directly, no search domains
│
├─ Has ≥ 5 dots?
│   └─ Query directly first, THEN try search domains
│
└─ Has < 5 dots? (most cases)
    └─ Try search domains IN ORDER:
       1. name.NAMESPACE.svc.cluster.local
       2. name.svc.cluster.local
       3. name.cluster.local
       4. name. (external)
```

---

## 6. Real-World Mapping

### Linux

```bash
# Basic DNS lookup
dig google.com

# Short answer only
dig +short google.com

# Full trace (root → TLD → authoritative)
dig +trace google.com

# Query specific nameserver
dig @8.8.8.8 google.com

# Query specific record type
dig google.com MX
dig google.com TXT
dig google.com NS

# Reverse lookup (IP → name)
dig -x 142.250.190.78

# Watch TTL decrease
dig google.com | grep -i "^google"
# Wait 30 seconds, repeat — TTL will be lower

# Check what DNS server your system uses
cat /etc/resolv.conf

# nslookup (simpler, less detail)
nslookup google.com
nslookup -type=MX google.com

# Clear DNS cache (systemd-resolved)
sudo systemd-resolve --flush-caches

# Test DNS resolution timing
dig google.com | grep "Query time"
```

### AWS

| Service/Feature | DNS Aspect |
|---------------|-----------|
| **Route53 Public Hosted Zone** | Authoritative DNS for your domain. NS records point here. |
| **Route53 Private Hosted Zone** | Internal DNS, resolves inside VPCs only. Must associate VPC. |
| **VPC DNS (enableDnsSupport)** | VPC provides DNS at VPC_BASE+2. Must be enabled for Route53 Private Hosted Zones to work. |
| **enableDnsHostnames** | EC2 instances get public DNS hostnames (e.g., ec2-1-2-3-4.compute.amazonaws.com). |
| **Route53 Health Checks** | Monitor endpoints. Used with failover/weighted routing policies. |
| **Route53 ALIAS records** | Like CNAME but for apex domains. Free intra-AWS lookups. |
| **DHCP Options Set** | Can set custom DNS servers for a VPC. Affects all instances. |

### Kubernetes

| Component | DNS Aspect |
|-----------|-----------|
| **CoreDNS** | Deployment in kube-system. Resolves `svc.cluster.local` names. |
| **Service** | ClusterIP is the DNS answer for `service-name.namespace.svc.cluster.local`. |
| **Headless Service** | DNS returns pod IPs directly (no ClusterIP). Used by StatefulSets. |
| **StatefulSet DNS** | Each pod gets a stable DNS name: `pod-0.service-name.namespace.svc.cluster.local`. |
| **ExternalName Service** | DNS CNAME to an external domain. E.g., `my-rds IN CNAME my-db.cluster-xyz.rds.amazonaws.com`. |
| **/etc/resolv.conf** | Configured by kubelet. Contains CoreDNS IP, search domains, ndots. |
| **dnsPolicy** | `ClusterFirst` (default), `Default` (use node's DNS), `None` (custom). |

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: "Name or Service Not Known" in a K8s Pod

| Attribute | Detail |
|-----------|--------|
| **Symptom** | `curl http://my-service:8080` returns "Could not resolve host: my-service" |
| **Root Cause** | DNS cannot resolve the service name |
| **OSI Layer** | L7 (DNS is an application-layer protocol) |
| **Possible Causes** | 1. Service doesn't exist in this namespace. 2. Service name is misspelled. 3. CoreDNS pods are down/crashlooping. 4. Cross-namespace call without specifying namespace. 5. NetworkPolicy blocking DNS (UDP 53) to kube-system. |
| **Debug Steps** | 1. `kubectl get svc my-service` — does it exist? 2. `kubectl get pods -n kube-system -l k8s-app=kube-dns` — is CoreDNS running? 3. From inside the pod: `nslookup my-service` → `nslookup my-service.default.svc.cluster.local` 4. Try FQDN with trailing dot: `nslookup my-service.default.svc.cluster.local.` 5. Check if NetworkPolicy blocks UDP 53 egress. |

### Scenario 2: DNS Resolution Is Slow (5+ Seconds)

| Attribute | Detail |
|-----------|--------|
| **Symptom** | HTTP requests take 5+ seconds in the DNS phase. `curl -w "%{time_namelookup}"` shows high DNS time. |
| **Root Cause** | CoreDNS is overloaded, or the ndots:5 setting causes 4 unnecessary queries for external domains. |
| **OSI Layer** | L7 |
| **Possible Causes** | 1. CoreDNS has too few replicas for cluster size. 2. ndots:5 causing search domain queries for external names. 3. Upstream DNS (VPC DNS) rate limiting. 4. CoreDNS pod on a node with high CPU pressure. |
| **Debug Steps** | 1. `kubectl logs -n kube-system -l k8s-app=kube-dns` — check for errors/throttling. 2. `kubectl top pods -n kube-system` — CPU/memory of CoreDNS pods. 3. Test: `time nslookup google.com` from inside a pod. 4. Test with trailing dot: `time nslookup google.com.` — if much faster, ndots is the issue. |
| **Fix** | 1. Scale CoreDNS: increase replicas. 2. Use dnsConfig to set ndots:2 on pods that make many external calls. 3. Use NodeLocal DNS Cache (runs DNS cache on each node). 4. Append trailing dots to external FQDNs in app config. |

### Scenario 3: DNS Changes Don't Take Effect

| Attribute | Detail |
|-----------|--------|
| **Symptom** | You changed a DNS record (e.g., Route53), but clients still go to the old IP. |
| **Root Cause** | TTL caching. The old record is cached at multiple levels. |
| **OSI Layer** | L7 |
| **Possible Causes** | 1. TTL on the old record was high (3600s = 1 hour of stale caching). 2. Application-level DNS caching (JVM caches DNS forever by default). 3. Browser cache. 4. Corporate proxy/resolver cache. |
| **Debug Steps** | 1. `dig +short your-domain.com` — what does the authoritative DNS return? 2. `dig +short your-domain.com @8.8.8.8` — what does Google's resolver return? If different → cache still has old value. 3. Check the TTL on the new record. 4. Wait TTL seconds and check again. |
| **Fix** | 1. Before migration: lower TTL to 60s. Wait for the OLD TTL to expire. 2. Then make the change. 3. After migration: raise TTL back. 4. For JVM apps: set `networkaddress.cache.ttl=30` in JVM settings. |

### Scenario 4: Route53 Private Hosted Zone Not Resolving

| Attribute | Detail |
|-----------|--------|
| **Symptom** | EC2 instances can't resolve names from a Private Hosted Zone. `dig internal.mycompany.com` → NXDOMAIN. |
| **Root Cause** | The Private Hosted Zone is not associated with the VPC, or DNS support is disabled on the VPC. |
| **OSI Layer** | L7 |
| **Debug Steps** | 1. Check VPC settings: `enableDnsSupport=true`, `enableDnsHostnames=true`. 2. Route53 → Private Hosted Zone → check VPC associations → is this VPC listed? 3. From EC2: `dig @169.254.169.253 internal.mycompany.com` — query the VPC DNS directly. |

### Scenario 5: K8s DNS Breaks After Applying NetworkPolicy

| Attribute | Detail |
|-----------|--------|
| **Symptom** | After applying a default-deny NetworkPolicy, pods can't resolve any DNS names. Services are unreachable by name (but work by IP). |
| **Root Cause** | The NetworkPolicy blocks DNS traffic (UDP port 53) from pods to CoreDNS in kube-system namespace. |
| **OSI Layer** | L7 (DNS) / L4 (port blocked) |
| **Debug Steps** | 1. `kubectl exec pod -- nslookup kubernetes` — does DNS work? 2. `kubectl exec pod -- ping 10.96.0.10` — can you reach CoreDNS IP? 3. Check NetworkPolicy — does it allow egress to kube-system on UDP 53? |
| **Fix** | Add a NetworkPolicy rule allowing egress to CoreDNS: ```yaml egress: - to: - namespaceSelector: matchLabels: kubernetes.io/metadata.name: kube-system ports: - protocol: UDP port: 53 - protocol: TCP port: 53 ``` |

---

## 8. Debugging Playbook

### The DNS Debug Ladder

```
Step 1: Does DNS resolve at all?
──────────────────────────────────
$ dig <hostname>
$ nslookup <hostname>
  ✓ Returns IP → DNS works, problem is elsewhere
  ✗ NXDOMAIN   → Name doesn't exist or wrong domain
  ✗ SERVFAIL   → DNS server error
  ✗ Timeout    → Can't reach DNS server

Step 2: Is it the right IP?
────────────────────────────
$ dig +short <hostname>
→ Compare with expected IP
→ If wrong IP → stale cache or wrong record

Step 3: Check different resolvers
──────────────────────────────────
$ dig @8.8.8.8 <hostname>          # Google's resolver
$ dig @1.1.1.1 <hostname>          # Cloudflare's resolver
$ dig @10.96.0.10 <hostname>       # CoreDNS (in K8s)
→ If different answers → cache inconsistency

Step 4: Check the authoritative answer
───────────────────────────────────────
$ dig +trace <hostname>
→ Follow the entire chain from root to authoritative
→ The authoritative answer is the truth

Step 5: Check TTL
──────────────────
$ dig <hostname> | grep "IN"
→ Look at TTL value
→ Low TTL = refresh soon. High TTL = stale for a while.

Step 6: In K8s — check CoreDNS
────────────────────────────────
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
$ kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
$ kubectl exec <any-pod> -- cat /etc/resolv.conf
$ kubectl exec <any-pod> -- nslookup kubernetes
```

---

## 9. Hands-On Practice

### Lab 1: Trace DNS Resolution

```bash
# Full trace from root servers to authoritative
dig +trace google.com

# Observe:
# 1. Root servers (.) → delegate to .com servers
# 2. .com servers → delegate to google.com's NS
# 3. google.com's NS → return the A record with IP
```

### Lab 2: Watch TTL Decrease

```bash
# Run twice quickly:
dig google.com | grep "^google"
# Note the TTL (e.g., 300)

# Wait 30 seconds, run again:
dig google.com | grep "^google"
# TTL is now ~270 (300 - 30)

# When TTL hits 0, the resolver re-queries the authoritative server
```

### Lab 3: K8s DNS Exploration

```bash
# Inside a K8s pod:
cat /etc/resolv.conf

# Test resolution at different levels:
nslookup kubernetes
nslookup kubernetes.default
nslookup kubernetes.default.svc.cluster.local

# All three should resolve. Why? Search domains.

# Test cross-namespace:
nslookup my-service                    # Only works same namespace
nslookup my-service.other-namespace    # Works cross-namespace
```

### Lab 4: Simulate DNS Failure

```bash
# In a test container/pod, break DNS:
# Override resolv.conf to point at a non-existent DNS server

kubectl run dns-test --image=busybox --rm -it --overrides='
{
  "spec": {
    "dnsPolicy": "None",
    "dnsConfig": {
      "nameservers": ["192.0.2.1"]
    }
  }
}' -- sh

# Inside:
nslookup google.com
# Result: timeout — this is what a DNS failure looks like

curl http://google.com
# Result: "Could not resolve host" — common production error
```

### Lab 5: External DNS Performance (ndots Impact)

```bash
# Inside a K8s pod:
# Time a query without trailing dot (ndots:5 kicks in):
time nslookup api.example.com

# Time a query WITH trailing dot (skip search domains):
time nslookup api.example.com.

# The second should be faster (1 query vs 4+)
```

---

## 10. Interview Preparation

### Q1: (Must-know) A K8s pod calls another service by name and gets "Name or service not known." The service exists. Walk through your diagnosis.

**Answer:**
1. Verify the service: `kubectl get svc my-service -n <namespace>` — exists?
2. Check namespace: is the pod in the same namespace as the service? If not, use `my-service.other-namespace`.
3. Check CoreDNS: `kubectl get pods -n kube-system -l k8s-app=kube-dns` — running?
4. Test from inside the pod: `nslookup my-service.namespace.svc.cluster.local` — resolves?
5. Check `/etc/resolv.conf` in the pod — correct nameserver (CoreDNS IP)?
6. Check NetworkPolicy — blocking UDP 53 egress?
7. Check CoreDNS logs for errors.

### Q2: Explain how DNS TTL works and why it matters during migrations.

**Answer:** TTL tells caching resolvers how long to keep the answer. During a migration (changing IP), if TTL is 3600s (1 hour), clients will use the OLD IP for up to an hour after you change the record. Strategy: lower TTL to 60s, wait for the old TTL to expire (1 hour for 3600s records), make the change, verify, then raise TTL back.

### Q3: What's the difference between Route53 ALIAS and CNAME records?

**Answer:** CNAME creates an alias that returns another name (requires a second DNS lookup). Cannot be used on the apex domain (example.com). ALIAS is AWS-specific — works like CNAME but can be used on apex domains, and Route53 resolves it internally (no extra lookup, no charge for intra-AWS queries).

### Q4: Explain ndots:5 in Kubernetes DNS. What's the performance implication?

**Answer:** `ndots:5` means: if a name has fewer than 5 dots, try search domains first. A call to `api.external.com` (2 dots, < 5) generates 4 DNS queries (appending each search domain) before trying the actual name. This multiplies DNS traffic 4x for external domains. Fix: use FQDNs with trailing dots, reduce ndots to 2, or deploy NodeLocal DNS Cache.

### Q5: Your Route53 failover routing isn't working. Primary is down but DNS still returns the primary IP. Why?

**Answer:** Possible causes: (1) Health check hasn't failed yet (check interval + failure threshold). (2) TTL is still active — clients cached the primary IP. (3) Health check is checking the wrong port/path. (4) Health check is in a different region than the endpoint. Debug: check Route53 health check status, verify TTL, test with `dig @8.8.8.8` to see what DNS returns NOW.

### Q6: How does service discovery work in Kubernetes?

**Answer:** CoreDNS watches the K8s API for Service objects. When a Service is created, CoreDNS automatically creates DNS records: `service-name.namespace.svc.cluster.local → ClusterIP`. Pods use search domains in `/etc/resolv.conf` to resolve short names. For StatefulSets with headless Services, CoreDNS creates per-pod DNS records: `pod-0.service-name.namespace.svc.cluster.local → Pod IP`.

---

## 11. Advanced Insights (Senior Level)

### NodeLocal DNS Cache

At scale, all DNS queries from pods go to CoreDNS pods (small deployment). This becomes a bottleneck. NodeLocal DNS Cache runs a DNS cache on every node:

```
Without NodeLocal DNS:
Pod → (network) → CoreDNS pods → answer

With NodeLocal DNS:
Pod → NodeLocal cache (localhost) → cache hit? → return
                                 → cache miss? → CoreDNS → answer
```

Benefits: reduced CoreDNS load, lower DNS latency, UDP over localhost is more reliable than cross-node UDP.

### DNS-Based Service Discovery Patterns

```
Pattern: ExternalName Service for Database Abstraction
─────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: my-database
spec:
  type: ExternalName
  externalName: my-db.cluster-xyz.us-east-1.rds.amazonaws.com

# Pods call "my-database" → CoreDNS returns CNAME to RDS endpoint
# To migrate databases: change the externalName — zero code changes
```

### DNS Rate Limiting in AWS VPC

AWS VPC DNS resolver has a soft limit of **1,024 packets per second per ENI**. Large EKS clusters with many pods making DNS queries can hit this. Symptoms: intermittent DNS timeouts. Fix: NodeLocal DNS Cache, or increase the limit via support.

---

## 12. Common Mistakes

### Mistake 1: "DNS is working, the record exists" — but TTL is stale
The record exists in Route53, but resolvers are still returning the old IP because they cached it. Always verify with `dig @8.8.8.8` to see what the public internet sees right now.

### Mistake 2: Not lowering TTL before migration
Changing a record with TTL=3600s means up to 1 hour of split traffic. Lower TTL BEFORE the change, wait for the old TTL to expire, then make the change.

### Mistake 3: Calling services by short name across namespaces
`curl http://my-service:8080` only works within the same namespace. Cross-namespace requires: `curl http://my-service.other-namespace:8080`.

### Mistake 4: Forgetting DNS when setting up NetworkPolicy
Default-deny NetworkPolicy blocks ALL egress, including DNS (UDP 53). Your first allow rule should always include DNS access to kube-system.

### Mistake 5: JVM caching DNS forever
Java's default DNS cache TTL is infinite (JVM 8 and earlier). This means the JVM will NEVER pick up a DNS change. Set `networkaddress.cache.ttl=30` in `$JAVA_HOME/jre/lib/security/java.security` or via system property.

---

## 13. Cheat Sheet

### DNS Record Types Quick Reference

```
A     → Name → IPv4
AAAA  → Name → IPv6
CNAME → Name → Name (not on apex!)
ALIAS → Name → AWS resource (works on apex)
MX    → Name → Mail server
TXT   → Name → Text (SPF, DKIM, verification)
NS    → Name → Nameserver
PTR   → IP → Name (reverse)
SRV   → Name → Port + Host (service discovery)
```

### Essential Commands

```bash
dig <hostname>                    # Full DNS lookup
dig +short <hostname>             # IP only
dig +trace <hostname>             # Full resolution chain
dig @8.8.8.8 <hostname>           # Specific resolver
dig <hostname> MX                 # Specific record type
nslookup <hostname>               # Simple lookup
host <hostname>                   # Another simple lookup
cat /etc/resolv.conf              # Check DNS config
```

### K8s DNS Names

```
service.namespace.svc.cluster.local          → ClusterIP
pod-0.service.namespace.svc.cluster.local    → Pod IP (StatefulSet)
```

### The DNS Debug First Rule

```
ALWAYS check DNS first:
  dig <hostname>
  nslookup <hostname>

If DNS doesn't resolve → fix DNS
If DNS resolves to wrong IP → fix the record or wait for TTL
If DNS resolves correctly → problem is NOT DNS, check L3/L4
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**OSI Model (Topic 1.1):** DNS is a **Layer 7 (Application)** protocol. It runs over **UDP port 53** (Layer 4) for queries, falling back to TCP for large responses. The diagnostic ladder taught you to check L3 → L4 → L7 — DNS is the L7 check.

**TCP (Topic 1.2):** DNS usually runs over UDP for speed, but zone transfers and large responses use TCP. Understanding the UDP vs TCP trade-off (speed vs reliability) from Topic 2 explains why DNS chose UDP for queries.

**IP Addressing (Topic 1.3):** DNS translates human-readable names into the IP addresses you learned about in Topic 3. Without DNS, you'd need to know every IP by heart. DNS is the bridge between the human world and the IP world.

### What This Prepares You For Next
**Next topic: TLS/HTTPS — What Happens Before the First HTTP Byte**

Now that you understand how DNS resolves a name to an IP, and how TCP establishes a connection to that IP, the next layer is encryption. TLS wraps around TCP to provide confidentiality and authentication. Every HTTPS URL, every `kubectl` command, every service mesh connection uses TLS. Understanding TLS requires knowing about certificates, certificate authorities, and handshake flows — and how TLS errors are one of the most common production issues after DNS.
