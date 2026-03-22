# TCP — The Protocol Everything Runs On

> 🔹 **Phase:** 1 — Network Foundations  
> 🔹 **Topic:** TCP (Transmission Control Protocol)  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

IP (Layer 3) can send packets from one computer to another — but it makes **no promises**:
- Packets might arrive out of order
- Packets might get lost
- Packets might be duplicated
- There's no way to know if they arrived at all

This is like sending 10 postcards — some might arrive out of order, some might get lost, and you have no idea which ones made it.

**TCP solves this.** It turns the unreliable IP network into a **reliable, ordered, error-checked** data stream. After TCP does its job, an application can just "send data" and trust it arrives correctly, in order, without gaps.

### The Analogy: A Phone Call vs. Dropping Letters

**UDP** (the alternative to TCP) is like dropping letters in a mailbox — you send them and hope for the best. No confirmation, no order guarantee.

**TCP** is like a phone call:
1. You **dial** first (connection setup — SYN)
2. The other person **picks up** (SYN-ACK)
3. You **confirm you can hear them** (ACK)
4. Now you have a **conversation** — both sides know the other is there
5. Either side can **hang up** when done (FIN)
6. If one side goes silent, the other eventually notices (timeout)

### Why Does It Exist?

The internet was designed as a packet-switched network where individual packets take different paths and arrive independently. This is great for resilience (if a path fails, packets reroute), but terrible for applications that need reliable, ordered delivery.

TCP was created (1974, Vint Cerf and Bob Kahn) to bridge this gap: provide applications with a reliable stream while using the unreliable IP network underneath.

> 💡 **Key insight:** Every HTTP request, every database query, every `kubectl` command, every SSH session, every microservice call — they all run on TCP. If you don't understand TCP, you cannot debug distributed systems.

---

## 2. Why This Matters in DevOps

### Where This Appears in Real Systems

| Real-World Scenario | TCP Concept |
|---------------------|------------|
| A service is slow under load → TIME_WAIT exhaustion | TCP connection states |
| "Connection reset by peer" in logs | RST packets |
| Microservice latency spikes after deploy | TCP slow start, connection reuse |
| Port exhaustion on a NAT Gateway | Ephemeral ports |
| Health check shows healthy but requests fail | Half-open connections |
| NACL blocking responses → timeouts | Ephemeral port ranges |
| Connection pool sizing for databases | TCP connection lifecycle |
| gRPC streams dropping | TCP keepalive, idle timeout |

### Why DevOps Engineers Must Know This

1. **Connection states tell you what's wrong.** 10,000 TIME_WAIT connections = ephemeral port exhaustion coming. 500 CLOSE_WAIT = app isn't closing connections (memory leak pattern).

2. **Firewall rules require TCP understanding.** Security Groups are stateful (allow response automatically). NACLs are stateless (you MUST allow ephemeral ports for responses). If you don't know this, NACLs will break your architecture mysteriously.

3. **Performance tuning.** Connection reuse (HTTP keep-alive, connection pooling) avoids the overhead of repeated handshakes. TCP slow start means new connections start slow — persistent connections are critical for latency-sensitive apps.

4. **Production incidents.** "Connection reset by peer," "broken pipe," "connection timed out" — these are TCP telling you something specific. Each one maps to a different root cause.

---

## 3. Core Concept (Deep Dive)

### The TCP 3-Way Handshake

Before any data flows, TCP establishes a connection in 3 packets:

```
Client                          Server
  │                                │
  │──── SYN (seq=100) ───────────►│  "I want to connect. My sequence starts at 100."
  │                                │
  │◄─── SYN-ACK (seq=300,ack=101)─│  "OK. My sequence starts at 300. I acknowledge your 100."
  │                                │
  │──── ACK (ack=301) ────────────►│  "Got it. Let's go."
  │                                │
  │         CONNECTION ESTABLISHED │
  │                                │
  │──── Data (seq=101) ──────────►│  Now data can flow
```

**Why 3 packets?** Both sides need to:
1. Announce their initial sequence number (ISN)
2. Acknowledge the other side's ISN
3. Confirm the connection is bidirectional

Two packets would mean the server never knows if the client received the SYN-ACK.

### TCP Connection States

TCP connections move through well-defined states. The ones you MUST know for debugging:

```
         ┌──────────────┐
         │  LISTEN       │ ← Server waiting for connections (ss -tnlp)
         └──────┬───────┘
                │ SYN received
         ┌──────▼───────┐
         │  SYN_RECEIVED │ ← Very brief, rarely seen
         └──────┬───────┘
                │ ACK received
         ┌──────▼───────┐
         │  ESTABLISHED  │ ← Active connection, data flowing
         └──────┬───────┘
                │ One side sends FIN
         ┌──────▼───────┐
         │  FIN_WAIT_1   │ ← Initiator sent FIN, waiting for ACK
         └──────┬───────┘
                │
         ┌──────▼───────┐
         │  FIN_WAIT_2   │ ← Got ACK for FIN, waiting for other side's FIN
         └──────┬───────┘
                │ Other side's FIN received
         ┌──────▼───────┐
         │  TIME_WAIT    │ ← Wait 2×MSL (60s on Linux) before releasing port
         └──────┬───────┘
                │ Timer expires
         ┌──────▼───────┐
         │  CLOSED       │
         └──────────────┘
```

### The Critical States for Production Debugging

#### ESTABLISHED
- **What:** Active, healthy connection. Data can flow both directions.
- **Normal?** Yes. Most connections should be here.
- **Check:** `ss -tnp state established`

#### TIME_WAIT
- **What:** Connection was closed, but the port is held for 60 seconds (Linux default) to handle any late-arriving packets.
- **Normal?** Yes — in moderation.
- **Problem:** Under high request load, if you create and close thousands of connections per second, you run out of ephemeral ports because they're all stuck in TIME_WAIT.
- **Symptom:** "Cannot assign requested address" or "Too many open files"
- **Fix:** HTTP keep-alive (connection reuse), connection pooling, or `net.ipv4.tcp_tw_reuse=1`
- **Check:** `ss -s` shows TIME_WAIT count

#### CLOSE_WAIT
- **What:** The *remote* side closed the connection, but the *local* application hasn't called close() yet.
- **Normal?** **NO.** This almost always indicates a bug in the application — it's not properly closing connections after the other side disconnected.
- **Symptom:** Connections pile up indefinitely, memory leak, file descriptor leak.
- **Fix:** Fix the application code to properly close connections.
- **Check:** `ss -tnp state close-wait`

### TCP Ports

A port is a 16-bit number (0–65535) that identifies a specific application on a host.

```
┌─────────────────────────────────────────────────────┐
│ Well-Known Ports (0–1023) — Require root/admin      │
│ 22=SSH  80=HTTP  443=HTTPS  53=DNS                  │
│ 3306=MySQL  5432=PostgreSQL  6379=Redis             │
│ 6443=K8s API  2379=etcd  10250=kubelet              │
├─────────────────────────────────────────────────────┤
│ Registered Ports (1024–49151)                       │
│ 8080=alt-HTTP  3000=dev-servers  8443=alt-HTTPS     │
├─────────────────────────────────────────────────────┤
│ Ephemeral Ports (32768–60999 on Linux)              │
│ Used for OUTBOUND connections. Assigned by the OS.  │
│ CRITICAL for NACL rules — responses come back here. │
└─────────────────────────────────────────────────────┘
```

> 🚨 **The NACL trap:** NACLs are stateless. If you allow inbound port 443, you MUST also allow outbound ephemeral ports (1024–65535) for the response to get back. This catches everyone at least once.

### TCP vs. UDP

| Property | TCP | UDP |
|----------|-----|-----|
| Connection | Yes (handshake first) | No (just send) |
| Reliability | Guaranteed delivery, retransmits | No guarantee |
| Ordering | In-order delivery | No ordering |
| Overhead | Higher (headers, state tracking) | Lower |
| Speed | Slower (reliability has a cost) | Faster |
| Use cases | HTTP, SSH, databases, APIs | DNS, video streaming, gaming, QUIC |

**DNS uses UDP** (port 53) for queries because they're small and fast. If the response is too large (>512 bytes), it falls back to TCP.

**QUIC (HTTP/3)** uses UDP with its own reliability layer built on top — getting UDP's speed with TCP's reliability at the application level.

---

## 4. Internal Working (Under the Hood)

### The TCP Header

Every TCP segment has a header containing:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port          │       Destination Port         │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                    Sequence Number                             │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                  Acknowledgment Number                         │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│Offset│ Res │U│A│P│R│S│F│         Window Size                  │
│      │     │R│C│S│S│Y│I│                                      │
│      │     │G│K│H│T│N│N│                                      │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Checksum             │       Urgent Pointer           │
└─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
```

**Key flags:**
- **SYN** — Synchronize sequence numbers (handshake start)
- **ACK** — Acknowledge received data
- **FIN** — Initiator wants to close
- **RST** — Hard reset, abort the connection immediately
- **PSH** — Push data to the application immediately (don't buffer)

### Flow Control: Window Size

TCP uses a **sliding window** to prevent the sender from overwhelming the receiver:

```
Sender                                    Receiver
  │                                          │
  │  "My receive window is 64KB"             │
  │◄──────────────────────────────────────── │
  │                                          │
  │  Sends 64KB of data                      │
  │──────────────────────────────────────── ►│
  │                                          │
  │  "I processed 32KB. Window is now 32KB"  │
  │◄──────────────────────────────────────── │
  │                                          │
  │  Can only send 32KB more until ACKed     │
```

- **Window size = 0** → The receiver is overwhelmed, sender must stop (called **zero window**)
- This causes "slow" connections that aren't actually a network problem — the application is slow to consume data

### Congestion Control: Slow Start

TCP doesn't blast data at full speed from the start. It begins slowly and ramps up:

```
Congestion Window (cwnd):

cwnd = 1  │█
cwnd = 2  │██
cwnd = 4  │████
cwnd = 8  │████████
cwnd = 16 │████████████████
          └──────────────────────────────►
                Time (RTTs)
```

1. **Slow start:** Start with 1 segment, double every RTT
2. **Congestion avoidance:** After threshold, increase by 1 per RTT
3. **Fast retransmit:** If 3 duplicate ACKs, retransmit immediately (don't wait for timeout)

**Why this matters for DevOps:**
- New connections start slow → first request latency is higher
- Connection reuse (HTTP keep-alive) avoids repeated slow starts
- This is why connection pooling to databases matters — each new connection starts at the bottom of the ramp

### Kernel: What Happens When a Packet Arrives

```
1. NIC receives frame
2. Kernel interrupt handler:
   → Validate checksum
   → Look up connection in conntrack table (5-tuple hash)
   → If SYN → check if a process is LISTENing → create new entry
   → If ACK/DATA → find existing connection → update sequence/window
   → If RST → tear down connection immediately
   → If FIN → start graceful shutdown state machine
3. Deliver data to socket buffer
4. Application calls read() → gets data from buffer
```

### Connection Termination: The 4-Way Close

```
Client                          Server
  │                                │
  │──── FIN ──────────────────────►│  "I'm done sending."
  │                                │
  │◄─── ACK ──────────────────────│  "Got it."
  │                                │
  │◄─── FIN ──────────────────────│  "I'm done too."
  │                                │
  │──── ACK ──────────────────────►│  "Got it. Going to TIME_WAIT."
  │                                │
  │  (TIME_WAIT: 60s)             │  (CLOSED immediately)
  │                                │
  │  CLOSED                       │
```

**Why TIME_WAIT?** To handle delayed packets. If the connection were immediately reused (same 5-tuple), a delayed packet from the old connection could be confused with the new one.

---

## 5. Mental Models

### Mental Model 1: The Reliable Postal Service

TCP is like a registered-mail postal service:
- **SYN/SYN-ACK/ACK** = You sign up for the service, they confirm, you confirm back
- **Sequence numbers** = Each letter is numbered (letter 1, 2, 3...)
- **ACK** = Delivery confirmation receipt ("I received letter 1, 2, 3")
- **Retransmission** = If no receipt for letter 5 after a timeout, send it again
- **Window** = "I can only handle 10 letters at a time, slow down"
- **FIN** = "No more letters coming from me"
- **RST** = "CANCEL EVERYTHING. RIGHT NOW."

### Mental Model 2: The Traffic Cop

Think of TCP as a traffic cop managing flow:
- **Green light (ESTABLISHED)** — Traffic flows freely
- **Yellow light (FIN_WAIT)** — Slowing down, about to close
- **Red light (CLOSE_WAIT)** — Traffic stopped, waiting for the application to clean up
- **Road closed (TIME_WAIT)** — Road is blocked for 60 seconds to clear any remaining vehicles

When you see tons of cars stuck at "road closed" (TIME_WAIT), it means too many connections are opening and closing — you need a highway (persistent connections) instead of back roads.

### Mental Model 3: The Connection State Machine

```
                     LISTEN
                       │
            SYN received │ SYN sent
                       │        │
               SYN_RECEIVED  SYN_SENT
                       │        │
                ACK received │  SYN-ACK received
                       │        │
                    ESTABLISHED ◄─┘
                       │
              FIN sent │ FIN received
                  │         │
            FIN_WAIT_1  CLOSE_WAIT ← Danger Zone!
                  │         │
            FIN_WAIT_2  LAST_ACK
                  │
             TIME_WAIT (60s timer)
                  │
               CLOSED
```

---

## 6. Real-World Mapping

### Linux

```bash
# View all TCP connections with process info
ss -tnp

# View listening sockets
ss -tnlp

# View connection state summary (quick health check)
ss -s

# View only TIME_WAIT connections
ss -tnp state time-wait

# View only CLOSE_WAIT connections (find the bug!)
ss -tnp state close-wait

# View connections to a specific port
ss -tnp dst :5432    # All connections to PostgreSQL

# Count connections by state
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# TCP kernel parameters
sysctl net.ipv4.tcp_tw_reuse           # Reuse TIME_WAIT sockets
sysctl net.ipv4.ip_local_port_range    # Ephemeral port range
sysctl net.core.somaxconn              # Max listen backlog
sysctl net.ipv4.tcp_max_syn_backlog    # Max SYN backlog
sysctl net.netfilter.nf_conntrack_max  # Max tracked connections

# Watch the handshake in real time
tcpdump -i any port 80 -nn -c 10
```

### AWS

| AWS Concept | TCP Underpinning |
|------------|-----------------|
| **Security Groups** | Stateful — if you allow inbound TCP 443, the response is automatically allowed. Internally, AWS tracks the connection state (like conntrack). |
| **NACLs** | Stateless — you must allow ephemeral ports (1024-65535) for TCP responses. If you forget, responses are silently dropped → client sees timeout. |
| **NAT Gateway** | Performs SNAT (changes source IP to NAT GW's IP). Tracks connections. Has a limit of 55,000 concurrent connections per destination. Exceeding this → intermittent failures. |
| **ALB idle timeout** | Default 60s. If no data passes for 60s, ALB closes the TCP connection. Backend must have a higher idle timeout than ALB, or you get 502 errors. |
| **NLB** | Passes TCP through. Preserves source IP. TCP idle timeout = 350s (not configurable for TCP). |
| **Connection draining** | During deploys, ALB stops sending new connections but lets existing TCP connections finish. Default 300s. Reduce for faster deploys. |

### Kubernetes

| K8s Concept | TCP Underpinning |
|------------|-----------------|
| **ClusterIP Service** | kube-proxy creates iptables DNAT rules that intercept TCP connections to the Service IP and rewrite the destination to a pod IP. The TCP connection state is tracked by conntrack. |
| **Readiness probe** | If a pod fails its readiness probe, its IP is removed from EndpointSlices → kube-proxy removes the iptables DNAT rule → no new TCP connections are sent to that pod. |
| **Liveness probe** | TCP socket check: kubelet does a TCP SYN to the port. If it connects (SYN-ACK), the pod is alive. If connection refused (RST) or timeout → kill the pod. |
| **Pod termination** | SIGTERM → prestop hook → remove from EndpointSlices → deregistration delay → SIGKILL. Existing TCP connections should complete. New ones go elsewhere. |
| **NodePort** | Opens a TCP port (30000-32767) on every node. kube-proxy uses iptables to forward TCP connections from NODE_IP:NodePort to a pod. |

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: TIME_WAIT Exhaustion Under High Load

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Service intermittently fails to connect to backend. Error: "Cannot assign requested address" or "99: Cannot assign requested address." |
| **Root Cause** | Too many short-lived TCP connections creating and closing rapidly. Each closed connection stays in TIME_WAIT for 60s, consuming an ephemeral port. With ~28,000 ephemeral ports (default range), you run out. |
| **OSI Layer** | L4 |
| **Real-world trigger** | Microservice making HTTP/1.0 calls (no keep-alive) to another service at 500 req/s. Each request creates a new TCP connection, closes it, TIME_WAIT for 60s. 500 × 60 = 30,000 ports needed. |
| **Debug Steps** | 1. `ss -s` — Look at TIME_WAIT count. 2. `ss -tnp state time-wait | wc -l` — Exact count. 3. If > 20,000 → you're approaching exhaustion. |
| **Fix** | 1. Enable HTTP keep-alive / connection reuse. 2. `sysctl net.ipv4.tcp_tw_reuse=1` — Allow reuse of TIME_WAIT sockets for new connections. 3. Expand port range: `sysctl net.ipv4.ip_local_port_range="1024 65535"`. |

### Scenario 2: CLOSE_WAIT Pile-Up (Application Bug)

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Application gradually slows down over hours/days. Eventually stops accepting new connections. Memory usage climbs. |
| **Root Cause** | The application isn't properly calling close() on sockets after the remote side disconnects. Connections stay in CLOSE_WAIT forever. |
| **OSI Layer** | L4 |
| **Real-world trigger** | Connection pool library has a bug. Remote database restarts → all connections get FIN → application receives FIN → enters CLOSE_WAIT → application never closes them → file descriptors leak. |
| **Debug Steps** | 1. `ss -tnp state close-wait | wc -l` — Count. If growing, you have a leak. 2. `ss -tnp state close-wait` — Look at the process name (the `users:` column). 3. Check that process's file descriptors: `ls /proc/<pid>/fd | wc -l`. |
| **Fix** | Fix the application code. Review connection pool configuration. Ensure health checks on pooled connections. As a band-aid: restart the application. |

### Scenario 3: "Connection Reset by Peer"

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Client sees "Connection reset by peer" or "broken pipe" error. |
| **Root Cause** | The remote side sent a RST (hard close) instead of a graceful FIN. |
| **OSI Layer** | L4 |
| **Possible Causes** | 1. Remote app crashed. 2. Remote firewall killed the idle connection. 3. ALB idle timeout (60s default) expired. 4. The server received data on a socket it had already closed. 5. NAT Gateway connection tracking timed out (350s for TCP). |
| **Debug Steps** | 1. Check timing — does it happen after a specific idle period? (Timeout) 2. Check ALB access logs for 460 error code (client idle timeout). 3. Check application logs on the remote side. 4. `tcpdump` — confirm RST is coming from the server. |

### Scenario 4: NAT Gateway Port Exhaustion

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Random connection failures from private subnet instances/pods to an external API. Some requests succeed, some fail. |
| **Root Cause** | NAT Gateway has a limit of 55,000 concurrent connections per unique destination (IP+port combo). Lambda or K8s pods behind same NAT GW all connecting to one API endpoint. |
| **OSI Layer** | L4 |
| **Debug Steps** | 1. CloudWatch: `ErrorPortAllocation` metric on NAT GW. 2. Count connections: `ss -tnp dst <api-ip>:443 | wc -l` on each node behind NAT. |
| **Fix** | 1. Use multiple NAT Gateways (one per AZ). 2. Use VPC endpoints for AWS services (no NAT needed). 3. Connection pooling to reduce concurrent connections. 4. Add multiple Elastic IPs to the NAT Gateway. |

### Scenario 5: SYN Flood / Half-Open Connections

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Server can no longer accept new connections. Existing connections work fine. |
| **Root Cause** | SYN backlog is full. Too many half-open connections (SYN_RECEIVED state). Could be a SYN flood attack or legitimate overload. |
| **OSI Layer** | L4 |
| **Debug Steps** | 1. `ss -tnp state syn-recv | wc -l` — count half-open. 2. `netstat -s | grep -i syn` — look for "SYNs to LISTEN sockets dropped." 3. Check `sysctl net.ipv4.tcp_max_syn_backlog`. |
| **Fix** | 1. Enable SYN cookies: `sysctl net.ipv4.tcp_syncookies=1`. 2. Increase backlog. 3. In AWS, NLB/ALB handle SYN floods for you (AWS Shield). |

---

## 8. Debugging Playbook

### "Is It a TCP Problem?" Decision Tree

```
Connection attempt:
│
├─ Immediate "Connection refused"
│  → RST received
│  → Nothing listening on that port
│  → Check: ss -tnlp on target
│
├─ Long wait → "Connection timed out"  
│  → Packet silently dropped
│  → Firewall/SG blocking, wrong route, host down
│  → Check: SG rules, route table, VPC Flow Logs
│
├─ Connects, then "Connection reset by peer"
│  → RST received after ESTABLISHED
│  → App crash, idle timeout, firewall kill
│  → Check: ALB idle timeout, app logs, NAT GW timeout
│
├─ Connects, but very slow
│  → Possible: slow start (new connection)
│  → Possible: zero window (receiver overwhelmed)
│  → Possible: high RTT (network latency)
│  → Check: curl -w timing, ss -ti for window info
│
└─ Works then gradually stops
   → TIME_WAIT exhaustion or CLOSE_WAIT leak
   → Check: ss -s, ss state close-wait
```

### TCP Performance Diagnostic

```bash
# Step 1: Check connection states
ss -s
# Look for: large TIME_WAIT count, any CLOSE_WAIT

# Step 2: Measure timing
curl -w "\ndns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" -o /dev/null -s https://target
# tcp connect >> 50ms for same-region? → network latency issue
# ttfb - tls >> long? → application processing slow

# Step 3: Check for retransmissions
ss -ti dst <target-ip>
# Look for: retrans, rto, rtt values

# Step 4: Packet-level analysis
tcpdump -i any host <target-ip> and port <port> -nn -c 100
# Look for: RST, retransmissions, zero window
```

---

## 9. Hands-On Practice

### Lab 1: Watch the TCP Handshake

**Setup:**
```bash
# Terminal 1: Start packet capture
sudo tcpdump -i any port 80 -nn

# Terminal 2: Make a request
curl http://example.com
```

**What to observe:**
```
# SYN — Client initiates
IP 10.0.1.5.54321 > 93.184.216.34.80: Flags [S], seq 12345
#                                            ^^^ SYN flag

# SYN-ACK — Server responds
IP 93.184.216.34.80 > 10.0.1.5.54321: Flags [S.], seq 67890, ack 12346
#                                            ^^^^ SYN+ACK flags

# ACK — Client confirms
IP 10.0.1.5.54321 > 93.184.216.34.80: Flags [.], ack 67891
#                                            ^^^ ACK only

# Now data flows...
# PSH,ACK — HTTP GET request
# FIN — Connection close
```

### Lab 2: TIME_WAIT Pile-Up

**Setup:**
```bash
# Terminal 1: Start a simple web server
python3 -m http.server 8080

# Terminal 2: Hammer it with requests
ab -n 1000 -c 10 http://localhost:8080/

# Terminal 3: Watch TIME_WAIT build up
watch -n 1 'ss -s | grep -i time'
```

**Expected result:** TIME_WAIT count climbs to ~1000 then slowly decreases (60s per entry).

**What to observe:** After the test ends, the port entries don't disappear immediately. Each one waits 60 seconds. Under sustained load, this depletes ephemeral ports.

### Lab 3: Connection Refused vs. Timeout

**Setup:**
```bash
# Refused — port exists but nothing listens
telnet localhost 9999
# Result: "Connection refused" — instant

# Timeout — packet dropped (use non-routable IP)
telnet 192.0.2.1 80
# Result: Hangs, then "Connection timed out" — slow
```

**What to observe:** The *speed* of failure tells you the cause. Instant = RST = host is reachable. Slow = DROP = something between you and the target is eating packets.

### Lab 4: Find CLOSE_WAIT Problems

**Setup:**
```bash
# Check current CLOSE_WAIT connections
ss -tnp state close-wait

# If any exist, find the process:
ss -tnp state close-wait | awk '{print $6}' | sort | uniq -c | sort -rn
```

**What to observe:** In a healthy system, CLOSE_WAIT should be near zero. Any process with a significant number of CLOSE_WAIT connections has a socket leak.

### Lab 5: Measure TCP Handshake Overhead

**Setup:**
```bash
# Measure time for new connection (includes DNS + TCP + TLS)
curl -w "\ndns:%{time_namelookup}s\ntcp:%{time_connect}s\ntls:%{time_appconnect}s\nfirst-byte:%{time_starttransfer}s\ntotal:%{time_total}s\n" -o /dev/null -s https://google.com

# Compare: repeat immediately (DNS cached, connection might be reused)
curl -w "\ndns:%{time_namelookup}s\ntcp:%{time_connect}s\ntls:%{time_appconnect}s\nfirst-byte:%{time_starttransfer}s\ntotal:%{time_total}s\n" -o /dev/null -s https://google.com
```

**What to observe:** TCP connect time shows the RTT to the server. TLS adds another 1-2 RTTs. This overhead happens on every new connection — that's why connection reuse matters.

---

## 10. Interview Preparation

### Q1: (Must-know) Explain the TCP 3-way handshake and why it's needed.

**Answer:** SYN → SYN-ACK → ACK. Both sides exchange initial sequence numbers and confirm they can receive from each other. Three packets because: (1) client announces its ISN, (2) server acknowledges AND announces its ISN, (3) client acknowledges the server's ISN. Two packets would leave the server unsure if the client received its SYN-ACK.

### Q2: A production service under high load starts getting "connection reset by peer." What do you check first?

**Answer:**
1. `ss -s` — Check for TIME_WAIT exhaustion (port running out means the OS sends RST on new connections)
2. Check ALB idle timeout — is the ALB closing connections that the app thinks are still alive?
3. Check app logs for crashes — a crashing backend sends RST to all its connections
4. `tcpdump` on the destination — confirm RST is coming from the expected host (not from a firewall in between)
5. Check NAT Gateway limits — 55,000 concurrent connections per destination

### Q3: Explain TIME_WAIT. Why does it exist? When is it a problem?

**Answer:** TIME_WAIT is a state where a closed connection holds its port for 2×MSL (60s on Linux). It exists to handle late-arriving duplicate packets that could corrupt a new connection using the same 5-tuple. It's a problem under high connection churn — creating and closing thousands of connections per second exhausts ephemeral ports. Fix with connection reuse, `tcp_tw_reuse`, or expanding the ephemeral port range.

### Q4: What's the difference between CLOSE_WAIT and TIME_WAIT?

**Answer:**
- **TIME_WAIT:** Normal — the side that initiated the close waits before releasing the port. Clears automatically after 60s.
- **CLOSE_WAIT:** Abnormal — the remote side closed, but the local application hasn't closed its end. Stays forever until the application calls close(). Almost always indicates a bug (socket/connection leak).

### Q5: You see 50,000 TIME_WAIT connections on a Kubernetes node. What's happening and what do you do?

**Answer:** High TIME_WAIT count means massive connection churn. Likely causes: (1) kube-proxy making non-persistent connections, (2) pods making short-lived HTTP/1.0 calls without keep-alive, (3) health check probes creating/closing connections rapidly. Solutions: enable HTTP keep-alive, use connection pools, consider `net.ipv4.tcp_tw_reuse=1`, check the source (`ss -tnp state time-wait | awk '{print $5}' | sort | uniq -c | sort -rn`).

### Q6: Why do NACLs need to allow ephemeral ports? Security Groups don't have this problem.

**Answer:** Security Groups are stateful — they track connections automatically. When you allow inbound TCP 443, the SG automatically allows the response. NACLs are stateless — each packet is evaluated independently. The response to an inbound TCP 443 request comes back from a random ephemeral port (32768-60999). If outbound rules don't allow that port range, the response is dropped, and the client sees a timeout.

### Q7: Explain TCP slow start. Why does it matter for microservice architectures?

**Answer:** TCP slow start begins with a small congestion window (typically 10 segments ≈ 14KB) and doubles it each RTT until packet loss is detected. This ramp-up means a fresh TCP connection starts slow and gets faster. For microservices making many short requests, creating new connections for each request means every request pays the slow start penalty. Connection pooling avoids this — established connections already have a warmed-up congestion window.

### Q8: A pod's liveness probe is a TCP check on port 8080. The pod is marked healthy but requests fail with 500. Explain.

**Answer:** TCP liveness probes only check that the port is accepting connections (SYN → SYN-ACK). They don't check if the application can actually handle requests correctly. A deadlocked application, an OOM-killed but not yet reaped process, or a misconfigured app can all accept TCP connections but fail at L7. Use HTTP liveness probes instead — they verify the application logic returns a 200.

---

## 11. Advanced Insights (Senior Level)

### TCP Tuning for Production

| Parameter | Default | When to Change | What It Does |
|-----------|---------|----------------|-------------|
| `net.ipv4.tcp_tw_reuse` | 0 | High connection churn | Allow reusing TIME_WAIT sockets for new outbound connections |
| `net.ipv4.ip_local_port_range` | 32768-60999 | Running out of ephemeral ports | Expand the range of ports available for outbound connections |
| `net.core.somaxconn` | 4096 | High-traffic servers | Max queue length for incoming connections waiting to be accepted |
| `net.ipv4.tcp_max_syn_backlog` | 1024 | SYN flood mitigation | Max queue for half-open connections (SYN_RECEIVED) |
| `net.ipv4.tcp_syncookies` | 1 | SYN flood attack | When SYN backlog is full, use SYN cookies instead of dropping |
| `net.ipv4.tcp_keepalive_time` | 7200 (2hrs) | Load balancer timeout alignment | How long before TCP sends a keepalive probe on an idle connection |
| `net.netfilter.nf_conntrack_max` | varies | K8s nodes with many Services | Max connections tracked by conntrack. Exhaustion = random connection drops |

### Connection Tracking (conntrack) — The Hidden Problem

Every TCP connection on a Linux machine is tracked in the conntrack table (used by iptables for stateful firewalling and NAT). On a Kubernetes node with hundreds of pods and Services, conntrack can fill up:

```bash
# Check current usage
sysctl net.netfilter.nf_conntrack_max      # Max entries
sysctl net.netfilter.nf_conntrack_count    # Current entries

# If count approaches max → random packet drops with "nf_conntrack: table full, dropping packet" in dmesg
```

### Why ALB Idle Timeout Matters

```
Client ←→ ALB ←→ Backend

ALB idle timeout: 60s (default)
Backend keepalive timeout: 30s (wrong!)
```

If the backend closes the connection after 30s of idle, but the ALB thinks it's still valid for 60s, the ALB will try to send traffic on a closed connection → **502 Bad Gateway**.

**Rule:** Backend idle timeout MUST be greater than ALB idle timeout.

---

## 12. Common Mistakes

### Mistake 1: "Timeout = the server is down"
**Wrong.** Timeout means the packet never got a response. Could be: firewall dropping, wrong route, server too slow to respond. A truly down server often can't even send a RST, so timeout makes sense, but it's one of many causes.

### Mistake 2: "Increase the timeout to fix timeout errors"
**Wrong.** Increasing the timeout just makes the user wait longer before seeing the same error. The fix is to find *why* the packet is dropped or the server isn't responding.

### Mistake 3: Confusing TCP keepalive with HTTP keep-alive
- **TCP keepalive** = The kernel sends tiny probe packets on idle connections to check if the other side is still alive. Default: 2 hours. Has nothing to do with HTTP.
- **HTTP keep-alive** = Reuse the same TCP connection for multiple HTTP requests instead of creating a new one each time. Completely different concept at a different layer.

### Mistake 4: Thinking Security Groups need outbound rules for responses
**Wrong.** Security Groups are stateful. If you allow inbound TCP 443, the response is automatically allowed outbound. You only need outbound rules for *new* outbound connections.

### Mistake 5: Not understanding that TCP is bidirectional
A TCP connection is full-duplex — both sides can send simultaneously. FIN only closes one direction. The other side can still send data until it also sends FIN. This is called a "half-close."

---

## 13. Cheat Sheet

### TCP States Quick Reference

| State | Meaning | Normal? | Action |
|-------|---------|---------|--------|
| LISTEN | Waiting for connections | ✅ | Expected on servers |
| ESTABLISHED | Active connection | ✅ | Normal operation |
| TIME_WAIT | Closed, holding port for 60s | ⚠️ in excess | Enable connection reuse |
| CLOSE_WAIT | Remote closed, local hasn't | ❌ | Fix app code — socket leak |
| SYN_SENT | Connecting to remote | ⚠️ if stuck | Firewall or host down |
| SYN_RECEIVED | Received SYN, sent SYN-ACK | ⚠️ if many | Possible SYN flood |

### Key Port Numbers

| Port | Service | Why You Care |
|------|---------|-------------|
| 22 | SSH | Remote access |
| 53 | DNS | Name resolution |
| 80 | HTTP | Web traffic |
| 443 | HTTPS | Encrypted web traffic |
| 3306 | MySQL | Database |
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache |
| 6443 | K8s API Server | `kubectl` talks here |
| 2379-2380 | etcd | K8s data store |
| 10250 | kubelet | Node agent API |
| 30000-32767 | K8s NodePort | External service access |

### Essential Commands

```bash
ss -tnlp                    # What's listening?
ss -tnp                     # Current connections
ss -s                       # State summary
ss -tnp state time-wait     # Time-wait count
ss -tnp state close-wait    # Close-wait (find leaks!)
tcpdump -i any port 443 -nn # See actual packets
```

### The TCP Rules

1. **3 packets before data** — SYN, SYN-ACK, ACK
2. **Connection refused = RST = L4 problem (on the host)**
3. **Connection timeout = DROP = L3/L4 problem (between you and the host)**
4. **TIME_WAIT = normal, but too many = port exhaustion**
5. **CLOSE_WAIT = almost always a bug — the app isn't closing connections**
6. **Ephemeral ports (32768-60999)** — NACLs MUST allow these for responses

---

## 🔗 Connections

### How This Connects to Previous Topics
**OSI Model (Topic 1.1):** TCP lives at **Layer 4 (Transport)**. In the previous topic, you learned that L4 is where ports, connections, and handshakes live. Now you've seen exactly how TCP implements all of that — sequence numbers, connection states, and the handshake mechanism. When someone says "this is an L4 problem," they mean something in TCP is going wrong.

The diagnostic ladder from Topic 1.1 (`ping` → `telnet` → `curl`) — the `telnet` step specifically tests if a TCP handshake can complete. Now you understand *exactly* what those 3 packets are doing.

### What This Prepares You For Next
**Next topic: IP Addressing, Subnets, and CIDR (Layer 3)**

TCP (L4) delivers data reliably between two endpoints identified by `IP:Port`. But how does the packet find the right IP in the first place? That's Layer 3 — IP addressing and routing. The next topic covers how IPs work, how subnets divide networks, how CIDR notation works, and why VPC design and Kubernetes pod CIDRs require you to do subnet math in your head. Every route table entry, every Security Group rule, every NACL rule references IP addresses — and you need to understand them.
