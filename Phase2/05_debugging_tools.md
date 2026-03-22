# Network Debugging Tools — Your Production Toolkit

> 🔹 **Phase:** 2 — Linux Networking  
> 🔹 **Topic:** Network Debugging Tools  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

Something is broken. A request is failing. Latency is high. Connections are dropping. You need to find out **where** in the stack the problem is and **why** it's happening.

Network debugging tools give you visibility into every layer:
- **L3** — Can I reach the host? (`ping`, `traceroute`, `mtr`)
- **L4** — Can I connect to the port? (`telnet`, `nc`, `ss`)
- **L7** — Does the application respond? (`curl`, `dig`)
- **Packet level** — What's actually on the wire? (`tcpdump`, `Wireshark`)

### The Analogy: Medical Diagnostics

Debugging a network is like diagnosing a patient:
- `ping` = Checking pulse — "Is the patient alive?"
- `traceroute` = X-ray — "Where exactly is the blockage?"
- `ss` = Blood test — "What's the internal state?"
- `tcpdump` = MRI — "Show me everything that's happening in detail"
- `curl -v` = Full physical exam — "Test every system end-to-end"

### Why This Matters

> 💡 **The Debugging Ladder:** Always start with the simplest tool and escalate. Don't jump to tcpdump when ping would have told you the answer in 2 seconds. The ladder: `ping` → `telnet`/`nc` → `curl` → `ss` → `tcpdump`.

---

## 2. The Debugging Ladder

```
L3: Can I reach the host?
│   Tools: ping, traceroute, mtr
│   ✗ Fails → Routing, firewall, or host-down problem
│   ✓ Works ↓
│
L4: Can I connect to the port?
│   Tools: telnet, nc, ss
│   ✗ "Connection refused" → Nothing listening on that port
│   ✗ "Connection timed out" → Firewall dropping packets
│   ✓ Works ↓
│
L7: Does the application respond correctly?
│   Tools: curl -v, dig, openssl s_client
│   ✗ HTTP 5xx → Application error
│   ✗ TLS error → Certificate problem
│   ✗ DNS error → Name resolution problem
│   ✓ Works ↓
│
Packet Level: What's actually happening on the wire?
    Tools: tcpdump, Wireshark
    → Use when higher-level tools aren't enough
    → See exact packets, flags, timings
```

---

## 3. Core Tools (Deep Dive)

### Tool 1: `ping` — L3 Reachability

```bash
# Basic ping
ping -c 5 10.0.1.100

# What it tells you:
# ✓ Response → Host is reachable at L3
# ✗ "Destination Host Unreachable" → Routing problem (no route to host)
# ✗ "Request timed out" → Host is down or ICMP is blocked
# ✗ Packet loss % → Network congestion or intermittent issue
# ✓ RTT values → Network latency (high RTT = distant or congested)

# Test MTU
ping -M do -s 1472 10.0.1.100   # Max for 1500 MTU (1472 + 28 header)

# IMPORTANT: ping NOT working does NOT mean the network is broken.
# Many Security Groups block ICMP. Always test L4 too.
```

### Tool 2: `traceroute` / `mtr` — Path Analysis

```bash
# Show each hop between you and the destination
traceroute 8.8.8.8

# Better version: continuous with packet loss statistics
mtr --report -c 20 8.8.8.8

# Sample output:
HOST                Loss%  Snt   Last   Avg  Best  Wrst
1. 10.0.1.1         0.0%   20    0.5   0.4   0.2   0.8   ← Gateway
2. ???              100.0   20    0.0   0.0   0.0   0.0   ← ICMP blocked (normal in AWS)
3. 52.93.x.x        0.0%   20    1.2   1.3   0.9   2.1   ← AWS backbone
4. 8.8.8.8          0.0%   20    1.5   1.6   1.1   2.4   ← Destination

# If loss spikes at a specific hop → bottleneck at that device
# If ??? appears → ICMP blocked (common in cloud, often ignorable)
# If loss is only at the last hop → destination is overloaded
```

### Tool 3: `ss` — Socket Statistics (Replace netstat)

```bash
# Show all TCP connections
ss -tnp

# Show listening sockets
ss -tnlp

# Show connection summary (QUICK health check)
ss -s
# Output:
# TCP:   250 (estab 180, closed 10, timewait 50, ...)
# → High TIME_WAIT? → Connection churn
# → High CLOSE_WAIT? → App not closing sockets (bug!)

# Show connections by state
ss -tnp state established
ss -tnp state time-wait
ss -tnp state close-wait

# Show connections to a specific port
ss -tnp dst :5432         # Connections to PostgreSQL
ss -tnp src :8080         # Connections from port 8080

# Show detailed TCP info (window, rtt, retransmits)
ss -ti dst :443

# Count connections by state
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
```

### Tool 4: `curl` — End-to-End HTTP Testing

```bash
# Verbose output (shows entire request/response)
curl -v https://api.example.com/health

# Timing breakdown (THE most useful curl trick)
curl -w "\n\
  dns:        %{time_namelookup}s\n\
  tcp:        %{time_connect}s\n\
  tls:        %{time_appconnect}s\n\
  ttfb:       %{time_starttransfer}s\n\
  total:      %{time_total}s\n\
  http_code:  %{http_code}\n\
  size:       %{size_download} bytes\n" \
  -o /dev/null -s https://api.example.com

# Interpretation:
# dns high?  → DNS resolution problem
# tcp high?  → Network latency or routing problem
# tls high?  → Certificate/crypto issue
# ttfb high? → Server processing slow
# total high but ttfb low? → Large response / slow transfer

# Test with specific method and body
curl -X POST -H "Content-Type: application/json" -d '{"key":"val"}' https://api.example.com

# Test without TLS verification (debugging only!)
curl -k https://self-signed-server.local

# Follow redirects
curl -L https://example.com

# Show only response headers
curl -I https://api.example.com
```

### Tool 5: `dig` / `nslookup` — DNS Debugging

```bash
# Basic lookup
dig api.example.com
dig +short api.example.com

# Full resolution trace
dig +trace api.example.com

# Query specific resolver
dig @8.8.8.8 api.example.com
dig @10.96.0.10 my-service.default.svc.cluster.local   # CoreDNS

# Check specific record type
dig api.example.com MX
dig api.example.com TXT
dig api.example.com NS

# Reverse lookup
dig -x 142.250.190.78

# Measure DNS time
dig api.example.com | grep "Query time"
```

### Tool 6: `tcpdump` — Packet Capture (The Nuclear Option)

```bash
# Capture all traffic on port 80
tcpdump -i any port 80 -nn

# Capture traffic to/from a specific host
tcpdump -i any host 10.0.1.100 -nn

# Capture SYN packets only (connection attempts)
tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0' -nn

# Capture and show ASCII content
tcpdump -i any port 80 -nn -A

# Save to file (analyze in Wireshark later)
tcpdump -i any port 443 -nn -w /tmp/capture.pcap -c 1000

# tcpdump in a pod's namespace
nsenter -t <PID> -n tcpdump -i eth0 -nn -c 50

# Common filters:
tcpdump -i any 'dst port 53'                  # DNS queries
tcpdump -i any 'tcp[tcpflags] & tcp-rst != 0' # RST packets
tcpdump -i any 'host 10.0.1.5 and port 8080'  # Specific conversation
```

**Reading tcpdump output:**
```
10:15:32.123456 IP 10.0.1.50.54321 > 10.0.1.100.80: Flags [S], seq 12345
│               │  │                  │               │         │
│               │  │                  │               │         └─ Sequence number
│               │  │                  │               └─ TCP Flags: [S]=SYN [.]=ACK [P.]=PSH+ACK [F.]=FIN+ACK [R.]=RST
│               │  │                  └─ Destination IP:Port
│               │  └─ Source IP:Port
│               └─ IP protocol
└─ Timestamp

Flags:
  [S]     = SYN (connection start)
  [S.]    = SYN-ACK (connection accepted)
  [.]     = ACK
  [P.]    = PSH+ACK (data packet)
  [F.]    = FIN+ACK (connection close)
  [R.]    = RST (connection reset)
```

### Tool 7: `nc` (netcat) — Swiss Army Knife

```bash
# Test if a TCP port is open
nc -zv 10.0.1.100 80
nc -zv 10.0.1.100 22

# Scan a range of ports
nc -zv 10.0.1.100 80-90

# Start a basic TCP listener (for testing)
nc -l -p 8080

# Send data to a listener
echo "hello" | nc 10.0.1.100 8080

# Test UDP connectivity
nc -u -zv 10.0.1.100 53
```

### Tool 8: `openssl s_client` — TLS Debugging

```bash
# Connect and show TLS details
openssl s_client -connect api.example.com:443

# Show certificate chain
openssl s_client -connect api.example.com:443 -showcerts

# Check certificate expiry
echo | openssl s_client -connect api.example.com:443 2>/dev/null | openssl x509 -noout -enddate

# Test specific TLS version
openssl s_client -connect api.example.com:443 -tls1_2
openssl s_client -connect api.example.com:443 -tls1_3
```

---

## 4. The Production Debugging Workflow

### Scenario: "Service X can't reach Service Y"

```
Step 1: DNS — Can we resolve the name?
─────────────────────────────────────────
$ dig service-y.namespace.svc.cluster.local
  ✓ Returns IP → DNS works
  ✗ NXDOMAIN → DNS issue (check CoreDNS, namespace, service name)
  ✗ Timeout → CoreDNS down or UDP 53 blocked (NetworkPolicy)

Step 2: L3 — Can we reach the IP?
──────────────────────────────────
$ ping <service-y-ip>
  ✓ Responds → L3 connectivity OK
  ✗ Timeout → Routing or SG issue
  Note: ping may be blocked but TCP works — proceed to L4

Step 3: L4 — Can we connect to the port?
──────────────────────────────────────────
$ nc -zv <service-y-ip> <port>
$ telnet <service-y-ip> <port>
  ✓ "Connected" → L4 works, port is open
  ✗ "Connection refused" → Nothing listening on that port
  ✗ "Connection timed out" → Firewall/SG blocking

Step 4: L7 — Does the app respond?
────────────────────────────────────
$ curl -v http://service-y:<port>/health
  ✓ HTTP 200 → Application working
  ✗ HTTP 5xx → Application error (check app logs)
  ✗ TLS error → Certificate issue (check with openssl)

Step 5: Timing — Where is the latency?
────────────────────────────────────────
$ curl -w "dns:%{time_namelookup} tcp:%{time_connect} ttfb:%{time_starttransfer} total:%{time_total}\n" -o /dev/null -s http://service-y:<port>/
  → Identify which phase is slow

Step 6: Packets — What's actually happening?
──────────────────────────────────────────────
$ nsenter -t <PID> -n tcpdump -i eth0 host <service-y-ip> -nn -c 50
  → See exact packets, retransmissions, RSTs, etc.
```

---

## 5. Real-World Mapping

### AWS-Specific Debugging

```bash
# VPC Flow Logs — see allowed/rejected traffic
# Enable in VPC console → analyze in CloudWatch Logs / S3
# Format: srcaddr dstaddr srcport dstport protocol action

# ALB Access Logs — HTTP-level debugging
# Enable in ALB console → logs to S3
# Contains: request_processing_time, target_processing_time, elb_status_code, target_status_code

# CloudWatch metrics for networking:
# NAT Gateway: ErrorPortAllocation, PacketsDropCount
# ALB: HTTPCode_Target_5XX_Count, TargetResponseTime, RejectedConnectionCount
# NLB: TCP_Target_Reset_Count
```

### K8s-Specific Debugging

```bash
# Debug network from an ephemeral container (no tools in pod)
kubectl debug -it <pod> --image=nicolaka/netshoot -- bash

# Inside netshoot container you get: curl, dig, ss, tcpdump, mtr, iperf, etc.

# Or use nsenter from the node:
nsenter -t <PID> -n tcpdump -i eth0 -nn -c 100

# Check CoreDNS
kubectl run tmp-dns --image=busybox:1.28 --rm -it -- nslookup kubernetes

# Check kube-proxy iptables
iptables -t nat -L KUBE-SERVICES -n | grep <service-cluster-ip>
```

---

## 6. Quick Diagnosis Table

| Symptom | First Tool | What to Look For |
|---------|-----------|-----------------|
| "Connection refused" | `ss -tnlp` on target | Is anything listening on that port? |
| "Connection timed out" | `nc -zv host port` + VPC Flow Logs | Firewall/SG blocking? Route exists? |
| "Name not resolved" | `dig @10.96.0.10 service.ns.svc.cluster.local` | CoreDNS healthy? Service exists? |
| Slow response | `curl -w timing` | DNS/TCP/TLS/TTFB — which phase is slow? |
| Intermittent failures | `ss -s` + `conntrack -C` | TIME_WAIT exhaustion? conntrack full? |
| 502 from ALB | ALB access logs | Backend timeout? Protocol mismatch? |
| 504 from ALB | ALB access logs | `target_processing_time` near ALB timeout? |
| "Connection reset" | `tcpdump` for RST packets | Who sends RST? When? After how long? |

---

## 7. Hands-On Practice

### Lab 1: The curl Timing Drill

```bash
# Run against 5 different endpoints and compare:
for url in https://google.com https://github.com https://aws.amazon.com https://cloudflare.com https://example.com; do
  echo "=== $url ==="
  curl -w "dns:%{time_namelookup}s tcp:%{time_connect}s tls:%{time_appconnect}s ttfb:%{time_starttransfer}s total:%{time_total}s code:%{http_code}\n" -o /dev/null -s $url
done
```

### Lab 2: tcpdump a 3-Way Handshake

```bash
# Terminal 1:
sudo tcpdump -i any port 80 -nn -c 20

# Terminal 2:
curl http://example.com

# In tcpdump output, identify:
# 1. SYN packet [S]
# 2. SYN-ACK packet [S.]
# 3. ACK packet [.]
# 4. HTTP GET [P.]
# 5. Response [P.]
# 6. FIN packets [F.]
```

### Lab 3: Debug a Simulated Failure

```bash
# Start a server
python3 -m http.server 8080 &

# Test it works
curl localhost:8080

# Simulate firewall block
sudo iptables -A INPUT -p tcp --dport 8080 -j DROP

# Now test again — observe timeout vs refused
curl -m 5 localhost:8080
# → Timeout (DROP = silent)

# Change to REJECT
sudo iptables -D INPUT -p tcp --dport 8080 -j DROP
sudo iptables -A INPUT -p tcp --dport 8080 -j REJECT

curl -m 5 localhost:8080
# → Immediate "Connection refused" (REJECT = loud)

# Cleanup
sudo iptables -D INPUT -p tcp --dport 8080 -j REJECT
kill %1
```

---

## 8. Interview Preparation

### Q1: A service is returning intermittent 502 errors from the ALB. Walk through your debugging steps.

**Answer:** 1. Check ALB access logs: `elb_status_code=502`, is `target_status_code` present? If target shows `-`, backend didn't respond. 2. Check `target_processing_time` — if near ALB timeout, backend is too slow. 3. Check backend keepalive settings — must be > ALB idle timeout (60s). 4. `ss -s` on backend nodes — TIME_WAIT or CLOSE_WAIT pileup? 5. Direct curl to backend IP — skip ALB to isolate. 6. `tcpdump` on backend — look for RST packets (backend closing connections).

### Q2: How would you diagnose "DNS is slow" in a Kubernetes cluster?

**Answer:** 1. `curl -w "dns:%{time_namelookup}s"` — confirm DNS is the slow phase. 2. `kubectl exec <pod> -- time nslookup service-name` — measure DNS directly. 3. Test with trailing dot: `time nslookup service-name.ns.svc.cluster.local.` — if faster, ndots:5 is the issue. 4. `kubectl top pod -n kube-system -l k8s-app=kube-dns` — CoreDNS CPU/memory. 5. `kubectl logs -n kube-system -l k8s-app=kube-dns` — errors? 6. Solutions: NodeLocal DNS Cache, reduce ndots, scale CoreDNS replicas.

### Q3: When would you use tcpdump vs curl for debugging?

**Answer:** `curl` is for L7 testing — "does the HTTP endpoint work and how long do the phases take?" Use it first. `tcpdump` is for L3/L4 packet-level analysis — "what's actually on the wire?" Use it when curl shows something wrong but you need to see exactly which packets are being sent/received. Example: curl shows timeout → tcpdump reveals SYN packets going out but no SYN-ACK coming back → firewall is blocking.

### Q4: What's the difference between DROP and REJECT, and how do you tell them apart when debugging?

**Answer:** `DROP` silently discards — the client gets no response and times out slowly. `REJECT` sends an ICMP error or TCP RST — the client immediately gets "Connection refused." When debugging: timeout = DROP somewhere in the path. Instant "refused" = REJECT or nothing listening. `tcpdump` confirms: you'll see outgoing SYN but no response (DROP), or you'll see an RST or ICMP reply (REJECT/nothing listening).

---

## 9. Cheat Sheet

### The Debugging Ladder

```
1. DNS:  dig <hostname>              → resolves?
2. L3:   ping <ip>                   → reachable?
3. L4:   nc -zv <ip> <port>         → port open?
4. L7:   curl -v <url>              → app responds?
5. Time: curl -w "timing" <url>     → which phase slow?
6. Wire: tcpdump -i any host <ip>   → packet analysis
```

### One-Liner Diagnosis

```bash
# Quick health check
ss -s                                 # Connection state summary
curl -w "dns:%{time_namelookup}s tcp:%{time_connect}s tls:%{time_appconnect}s ttfb:%{time_starttransfer}s total:%{time_total}s\n" -o /dev/null -s https://target

# Find what's listening
ss -tnlp | grep <port>

# Find connection leaks
ss -tnp state close-wait | wc -l

# DNS check
dig +short <hostname> @10.96.0.10

# Packet capture
tcpdump -i any host <ip> and port <port> -nn -c 50
```

---

## 🔗 Connections

### How This Connects to Previous Topics
This topic synthesizes ALL previous topics into a practical toolkit:
- **OSI Model** → The debugging ladder follows the layers
- **TCP** → `ss` shows connection states from Topic 1.2
- **DNS** → `dig` tests resolution from Topic 1.4
- **TLS** → `openssl s_client` tests certificates from Topic 1.5
- **Interfaces** → `ip addr/link` from Topic 2.1
- **Routing** → `ip route get` from Topic 2.2
- **iptables** → Rule inspection from Topic 2.3
- **Namespaces** → `nsenter` debugging from Topic 2.4

### What This Prepares You For Next
**Phase 3: Cloud Networking / AWS** — With Phases 1-2 complete, you understand protocols and how Linux implements them. Phase 3 maps these concepts to AWS: VPC is Linux routing, Security Groups are stateful iptables, NACLs are stateless iptables, NAT Gateway is MASQUERADE, and the debugging tools you just learned are how you troubleshoot cloud networking.
