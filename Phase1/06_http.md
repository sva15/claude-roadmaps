# HTTP/1.1, HTTP/2, and How Load Balancers Use Them

> 🔹 **Phase:** 1 — Network Foundations  
> 🔹 **Topic:** HTTP Protocols and Load Balancers  
> 🔹 **Priority:** HIGH

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

You have a TCP connection. You have TLS encryption. Now you need a language for clients and servers to actually **request and respond to things**. That language is HTTP.

HTTP defines: *"How do I ask for a webpage? How do I send data? How does the server tell me what happened?"*

Think of it as the conversation protocol: TCP sets up the phone line, TLS makes it private, and HTTP is the actual words you speak.

### The Analogy: Ordering at a Restaurant

**HTTP/1.1:** You order one dish at a time. Wait for it to arrive. Then order the next one. The waiter makes one trip per dish. If you're ordering 20 dishes, you wait a long time.

**HTTP/2:** You hand the waiter a list of all 20 dishes at once. The kitchen works on them simultaneously. Dishes arrive as they're ready. Same waiter, same table (connection), but massively faster.

**HTTP/3 (QUIC):** You order over the phone. Even if one dish is delayed, the others still arrive on time (no head-of-line blocking). And if you move tables, the order follows you (connection migration).

### Why Does It Exist?

HTTP/1.0 (1996) created a new TCP connection for every single request—catastrophically slow. HTTP/1.1 (1997) added `Connection: keep-alive` to reuse connections, but still only one request at a time per connection. HTTP/2 (2015) introduced multiplexing—multiple simultaneous requests on one connection.

> 💡 **Key Insight:** ALBs operate at HTTP (Layer 7). They read HTTP headers, inspect paths, and route based on content. NLBs operate at TCP (Layer 4). They don't understand HTTP at all. Choosing the wrong one is a common architecture mistake.

---

## 2. Why This Matters in DevOps

| Scenario | HTTP Concept |
|----------|-------------|
| ALB routing `/api/*` to backend A, `/*` to backend B | L7 path-based routing using HTTP path |
| gRPC calls between microservices | Requires HTTP/2 — HTTP/1.1 won't work |
| Client gets 504 from ALB | ALB timed out waiting for backend HTTP response |
| Debugging slow APIs | HTTP timing: DNS → TCP → TLS → TTFB → Transfer |
| Health checks returning 200 vs 503 | HTTP status codes drive LB decisions |
| Source IP logging behind ALB | X-Forwarded-For header — ALB adds it |
| WebSocket connections through ALB | ALB upgrades HTTP to WebSocket |

---

## 3. Core Concept (Deep Dive)

### HTTP Request/Response Structure

```
Request:
┌────────────────────────────────────────────┐
│ GET /api/users HTTP/1.1                    │ ← Method + Path + Version
│ Host: api.example.com                      │ ← Required in HTTP/1.1
│ Accept: application/json                   │ ← Preferred response format
│ Authorization: Bearer eyJhbGci...          │ ← Authentication token
│ X-Request-ID: abc-123                      │ ← Tracing header
│ User-Agent: curl/7.68.0                    │ ← Client identifier
│                                            │ ← Empty line = end of headers
│ (optional body for POST/PUT)               │
└────────────────────────────────────────────┘

Response:
┌────────────────────────────────────────────┐
│ HTTP/1.1 200 OK                            │ ← Version + Status Code + Reason
│ Content-Type: application/json             │ ← Body format
│ Content-Length: 1234                        │ ← Body size
│ X-Request-ID: abc-123                      │ ← Echo back for tracing
│ Set-Cookie: session=xyz; Secure; HttpOnly  │ ← Session management
│                                            │
│ {"users": [...]}                           │ ← Response body
└────────────────────────────────────────────┘
```

### HTTP Methods

| Method | Purpose | Idempotent? | Body? | Real Use |
|--------|---------|-------------|-------|----------|
| **GET** | Retrieve data | Yes | No | Fetch records, read APIs |
| **POST** | Create/submit data | No | Yes | Create user, submit form, trigger action |
| **PUT** | Replace resource entirely | Yes | Yes | Update entire record |
| **PATCH** | Partial update | No | Yes | Update specific fields |
| **DELETE** | Remove resource | Yes | Sometimes | Delete record |
| **HEAD** | GET without body | Yes | No | Health checks, check if resource exists |
| **OPTIONS** | Describe allowed methods | Yes | No | CORS preflight |

### HTTP Status Codes (Memorize These)

```
2xx — Success
  200 OK              ← Standard success
  201 Created         ← Resource created (POST)
  204 No Content      ← Success, no body (DELETE)

3xx — Redirect
  301 Moved Permanently  ← Use new URL forever
  302 Found              ← Temporary redirect
  304 Not Modified       ← Cached version is fine

4xx — Client Error
  400 Bad Request        ← Malformed request
  401 Unauthorized       ← Not authenticated (need login)
  403 Forbidden          ← Authenticated but not allowed
  404 Not Found          ← Resource doesn't exist
  405 Method Not Allowed ← Wrong HTTP method
  429 Too Many Requests  ← Rate limited

5xx — Server Error
  500 Internal Server Error  ← Server crashed
  502 Bad Gateway            ← Upstream server gave invalid response
  503 Service Unavailable    ← Server overloaded or maintenance
  504 Gateway Timeout        ← Upstream server didn't respond in time
```

> 🚨 **Critical for LB debugging:**
> - **502** from ALB = backend responded with something invalid or closed the connection
> - **503** from ALB = no healthy targets in the target group
> - **504** from ALB = backend didn't respond before ALB timeout (default 60s)

### HTTP/1.1 vs HTTP/2 vs HTTP/3

```
HTTP/1.1 (1997):
  ┌──────────────────────────────────────┐
  │ Connection 1: Request 1 → Response 1 │
  │                Request 2 → Response 2 │  Sequential (one at a time)
  │                Request 3 → Response 3 │
  └──────────────────────────────────────┘
  Problem: Head-of-line blocking — Request 2 must wait for Response 1

HTTP/2 (2015):
  ┌──────────────────────────────────────┐
  │ Connection 1: Request 1 ─────► Resp 1│
  │               Request 2 ───► Resp 2  │  Multiplexed (parallel)
  │               Request 3 ──► Resp 3   │
  └──────────────────────────────────────┘
  Binary protocol (not text). Headers compressed (HPACK).
  Single connection carries all streams.

HTTP/3 (2022):
  ┌──────────────────────────────────────┐
  │ QUIC (UDP): Stream 1 ─────► Resp 1  │
  │             Stream 2 ───► Resp 2    │  No TCP head-of-line
  │             Stream 3 ──► Resp 3     │  Connection migration
  └──────────────────────────────────────┘
  Uses QUIC (UDP-based). Lost packet in stream 1 doesn't block stream 2.
```

### Key Headers for DevOps

| Header | Direction | Purpose | DevOps Relevance |
|--------|-----------|---------|-----------------|
| `Host` | Request | Which hostname is being requested | ALB routes based on this (host-based routing) |
| `X-Forwarded-For` | Added by LB | Original client IP | ALB adds this. App uses it for logging/rate limiting. |
| `X-Forwarded-Proto` | Added by LB | Original protocol (http/https) | App uses to determine if request was HTTPS |
| `Content-Type` | Both | Media type of body | `application/json`, `text/html`, `multipart/form-data` |
| `Authorization` | Request | Authentication token | `Bearer <JWT>`, `Basic <base64>` |
| `Cache-Control` | Response | Caching instructions | `max-age=3600`, `no-cache`, `no-store` |
| `Connection` | Both | `keep-alive` or `close` | HTTP/1.1 connection reuse |
| `X-Request-ID` | Both | Distributed tracing ID | Correlate logs across services |

### ALB vs NLB: The Architecture Decision

```
                    ┌─────── ALB (Layer 7) ──────┐
                    │ • Reads HTTP headers/path   │
                    │ • Routes: /api → TG-A       │
Use ALB when:       │ • Routes: /web → TG-B       │
• HTTP/HTTPS traffic│ • TLS termination (ACM)     │
• Path-based routing│ • WebSocket support         │
• Host-based routing│ • WAF integration           │
• gRPC              │ • Adds X-Forwarded-For      │
• WebSockets        │ • Access logs (HTTP-level)  │
                    └─────────────────────────────┘

                    ┌─────── NLB (Layer 4) ──────┐
                    │ • Routes by IP + port       │
                    │ • Does NOT read HTTP        │
Use NLB when:       │ • Preserves source IP       │
• TCP passthrough   │ • Static IPs (Elastic IPs)  │
• Non-HTTP protocols│ • Ultra-low latency         │
• Need static IPs   │ • Millions of connections   │
• Extreme throughput│ • No header modification    │
• Database proxying │ • TLS passthrough option    │
                    └─────────────────────────────┘
```

---

## 4. Internal Working (Under the Hood)

### How ALB Processes an HTTP Request

```
1. Client sends HTTPS request to ALB
2. ALB terminates TLS (using ACM certificate)
3. ALB reads the HTTP request:
   - Extracts Host header → host-based routing
   - Extracts path → path-based routing  
   - Reads query parameters → query-based routing (if configured)
4. ALB matches against listener rules (evaluated in priority order)
5. Rule matches → selects target group
6. ALB selects a healthy target (round-robin or least-outstanding-requests)
7. ALB creates a NEW connection to the target:
   - Adds X-Forwarded-For: <client-ip>
   - Adds X-Forwarded-Proto: https
   - Adds X-Forwarded-Port: 443
8. Forwards the HTTP request to the target
9. Target responds → ALB forwards response to client
```

**Key timing:**
- ALB idle timeout: **60 seconds** (configurable). If no data flows for 60s, ALB closes the connection.
- Backend must respond before this timeout, or client gets a **504**.
- Backend keepalive timeout must be GREATER than ALB idle timeout to avoid 502s.

### HTTP/2 and gRPC on ALB

```
Client ── HTTP/2 ──► ALB ── HTTP/1.1 ──► Backend (default)
Client ── HTTP/2 ──► ALB ── HTTP/2 ────► Backend (with gRPC target group)

For gRPC:
  - ALB target group must be "gRPC" protocol type
  - Health checks must use gRPC health checking protocol
  - ALB handles the HTTP/2 framing
```

---

## 5. Mental Models

### Mental Model 1: The Status Code Decision Tree

```
Got a response?
├── 2xx → Success! Application handled it.
├── 3xx → Redirect. Follow the Location header.
├── 4xx → YOUR fault (client). Fix the request.
│    ├── 401 → Not logged in
│    ├── 403 → Logged in, but not allowed
│    └── 404 → Wrong URL
└── 5xx → SERVER'S fault. Check server logs.
     ├── 502 → Server gave bad response (crash, protocol mismatch)
     ├── 503 → Server overloaded or no targets
     └── 504 → Server took too long
```

### Mental Model 2: ALB as a Smart Receptionist

```
ALB = Smart receptionist at a building entrance

Visitor arrives: "I need api.example.com/users"

Receptionist:
1. Reads the badge (TLS check)
2. Reads the department (Host header) → Engineering floor
3. Reads the room (URL path) → /users → Room 301
4. Writes down visitor's name for the log (X-Forwarded-For)
5. Directs visitor to Room 301 (target group)

NLB = Automatic revolving door. Doesn't ask anything. Just pushes you through to a floor based on the number you pressed. Fast but dumb.
```

---

## 6. Real-World Mapping

### Linux

```bash
# Full HTTP debug
curl -v https://api.example.com/users

# Measure each phase
curl -w "\ndns:%{time_namelookup}s\ntcp:%{time_connect}s\ntls:%{time_appconnect}s\nttfb:%{time_starttransfer}s\ntotal:%{time_total}s\nhttp_code:%{http_code}\n" -o /dev/null -s https://api.example.com

# Send POST with JSON body
curl -X POST -H "Content-Type: application/json" -d '{"name":"test"}' https://api.example.com/users

# See only response headers
curl -I https://api.example.com

# Follow redirects
curl -L https://example.com

# Check X-Forwarded-For (behind LB)
curl -H "X-Forwarded-For: 1.2.3.4" http://backend:8080/

# Test HTTP/2
curl --http2 -v https://api.example.com
```

### AWS ALB Configuration

| ALB Feature | What It Does | DevOps Relevance |
|------------|-------------|-----------------|
| **Listener rules** | Route by host, path, query, header, method | Implement microservice routing |
| **Target groups** | Group of targets (instances, IPs, Lambda) | Separate TGs per service |
| **Health checks** | HTTP GET to path, expect 200 | Unhealthy targets get no traffic |
| **Idle timeout** | Close connection after N seconds of inactivity | Must be < backend keepalive timeout |
| **Deregistration delay** | Time to drain connections during deploys | Lower (30s) for faster deploys |
| **Sticky sessions** | Route same client to same target (via cookie) | Use for stateful apps (avoid if possible) |
| **WAF integration** | Attach AWS WAF rules | Block SQL injection, XSS, rate limit |
| **Access logs** | Log every HTTP request to S3 | Essential for debugging 502/504/latency |

### Kubernetes Ingress

```yaml
# ALB Ingress Controller example:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip           # Direct to pod IPs
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 3000
```

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: 504 Gateway Timeout from ALB

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Client receives HTTP 504 from ALB |
| **Root Cause** | Backend didn't respond within ALB idle timeout (60s default) |
| **Debug Steps** | 1. Check ALB access logs — `target_processing_time` will be near the timeout value. 2. Check backend logs — is the request processing slowly? 3. Check backend health — is it overloaded? 4. Check target group health — any unhealthy targets? |
| **Fix** | 1. Optimize backend processing time. 2. Increase ALB idle timeout (if the long processing is expected — e.g., report generation). 3. Use async processing — return 202 Accepted and process in the background. |

### Scenario 2: 502 Bad Gateway from ALB

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Client receives HTTP 502 from ALB |
| **Root Cause** | Backend either (1) closed the connection before responding, (2) sent an invalid response, or (3) protocol mismatch (HTTPS target but HTTP backend). |
| **Debug Steps** | 1. Check ALB error reason in access logs. 2. Check target group protocol matches actual backend protocol. 3. Check backend keepalive timeout — must be GREATER than ALB idle timeout. 4. `curl -v http://target-ip:port` directly. |
| **Fix** | Set backend keepalive timeout > ALB idle timeout (e.g., ALB=60s, backend=65s). Fix protocol mismatch. Check backend stability. |

### Scenario 3: gRPC Calls Failing Through ALB

| Attribute | Detail |
|-----------|--------|
| **Symptom** | gRPC calls return errors or fail to connect through ALB |
| **Root Cause** | ALB target group not configured for gRPC, or HTTP/2 not enabled |
| **Debug Steps** | 1. Check target group protocol — must be "gRPC." 2. Check ALB listener — must be HTTPS (gRPC requires HTTP/2, which requires TLS on ALB). 3. Check health check — must use gRPC health check protocol. |

### Scenario 4: Source IP Lost Behind ALB

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Application logs show the ALB's IP as the client IP, not the real user's IP |
| **Root Cause** | ALB is a reverse proxy — it makes a new TCP connection to the backend. The TCP source IP is the ALB's IP. |
| **Debug Steps** | Check if the application is reading the `X-Forwarded-For` header. |
| **Fix** | Configure the application or web server to read client IP from `X-Forwarded-For` header (e.g., nginx `set_real_ip_from`, Express.js `trust proxy`). |

---

## 8. Debugging Playbook

```
HTTP Request Failing? Follow this:

Step 1: What status code?
─────────────────────────
$ curl -v https://target 2>&1 | grep "< HTTP"
  → 2xx = working
  → 4xx = client error (fix request)
  → 5xx = server error (check server)

Step 2: Measure each phase
──────────────────────────
$ curl -w "dns:%{time_namelookup}s tcp:%{time_connect}s tls:%{time_appconnect}s ttfb:%{time_starttransfer}s total:%{time_total}s\n" -o /dev/null -s https://target
  → High DNS? → DNS problem (Topic 1.4)
  → High TCP? → Network/routing problem
  → High TLS? → Certificate/crypto issue
  → High TTFB? → Server processing slow
  → High total but low TTFB? → Large response body

Step 3: Is it the LB or the backend?
──────────────────────────────────────
$ curl -v http://BACKEND_IP:PORT directly
  → If it works directly but fails through ALB → ALB config issue
  → If it fails directly → backend problem

Step 4: Check ALB access logs
──────────────────────────────
→ target_processing_time: how long the backend took
→ elb_status_code vs target_status_code: who generated the error?
→ If elb=502 and target=- → backend didn't respond at all
```

---

## 9. Hands-On Practice

### Lab 1: Read HTTP Headers

```bash
curl -v https://google.com 2>&1 | grep -E "^[<>]"

# > lines are your request headers
# < lines are the response headers
# Find: Host, status code, Content-Type, Set-Cookie
```

### Lab 2: HTTP Timing Breakdown

```bash
curl -w "\ndns: %{time_namelookup}s\ntcp: %{time_connect}s\ntls: %{time_appconnect}s\nttfb: %{time_starttransfer}s\ntotal: %{time_total}s\nsize: %{size_download} bytes\ncode: %{http_code}\n" -o /dev/null -s https://google.com

# Run against multiple endpoints. Compare across:
# - Same region vs. cross-region
# - HTTP vs. HTTPS (see TLS overhead)
```

### Lab 3: Test Path-Based Routing

```bash
# If you have an ALB with path rules:
curl -v https://your-alb.com/api/health    # Should route to API service
curl -v https://your-alb.com/              # Should route to web service

# Check X-Forwarded-For from inside the backend:
# In your app, log the X-Forwarded-For header — should show your real IP
```

---

## 10. Interview Preparation

### Q1: A client gets 504 from the ALB. Is this an ALB problem, a target problem, or a network problem?

**Answer:** 504 means the ALB timed out waiting for the backend. It's most likely a **target problem** (backend is too slow). Check ALB access logs: `target_processing_time` will show how long the backend took. If it's near the ALB idle timeout (60s), the backend is processing too slowly. Could also be a target that's reachable by health check but overloaded for actual requests. Fix: optimize backend, increase timeout, or use async processing.

### Q2: When would you choose NLB over ALB?

**Answer:** Two production scenarios: (1) **Database proxy** — need TCP passthrough for MySQL/PostgreSQL with source IP preservation. ALB doesn't understand database protocols. (2) **Static IPs** — a partner requires you to whitelist specific IPs. ALB IPs change; NLB supports Elastic IPs. Additional: extreme throughput (millions of connections), non-HTTP protocols (Kafka, MQTT), or when latency below 1ms matters.

### Q3: What is X-Forwarded-For and why does it matter?

**Answer:** When ALB proxies a request, the backend sees the ALB's IP as the TCP source IP, not the real client. ALB adds `X-Forwarded-For: <client-ip>` header so the backend can log the real client IP, apply rate limiting, and make access control decisions. The app must be configured to trust and read this header. Important: validate the header — in multi-proxy chains, X-Forwarded-For can contain multiple IPs.

### Q4: Why does gRPC require HTTP/2?

**Answer:** gRPC uses HTTP/2 features: multiplexing (multiple concurrent RPCs on one connection), bidirectional streaming (server and client send messages simultaneously), binary framing (efficient serialization), and header compression. None of these exist in HTTP/1.1. ALB supports gRPC but requires HTTPS listeners (HTTP/2 requires TLS on ALB) and gRPC-type target groups.

### Q5: Explain the difference between 401 and 403.

**Answer:** **401 Unauthorized** = you haven't authenticated (missing or invalid credentials). Action: log in or provide a valid token. **403 Forbidden** = you ARE authenticated but don't have permission for this resource. Action: request higher permissions. The distinction matters for debugging: 401 = broken auth flow, 403 = RBAC/IAM policy issue.

---

## 11. Advanced Insights (Senior Level)

### ALB Idle Timeout vs Backend Keepalive

```
WRONG:
  ALB idle timeout: 60s
  Backend HTTP keepalive: 30s
  → Backend closes connection after 30s, ALB tries to reuse → 502

CORRECT:
  ALB idle timeout: 60s
  Backend HTTP keepalive: 65s (or higher)
  → Backend always keeps connection alive longer than ALB expects
```

### Connection Draining and Zero-Downtime Deploys

```
Rolling Deploy:
1. New pod starts, passes readiness probe
2. New pod added to target group
3. Old pod removed from target group
4. ALB starts draining old pod:
   - No NEW connections to old pod
   - EXISTING connections allowed to complete
   - Deregistration delay timer starts (default 300s)
5. After delay or all connections closed → old pod terminated

For faster deploys: reduce deregistration delay to 30s
For long-lived connections: keep it at 300s
```

### HTTP/2 Performance Considerations

- HTTP/2 multiplexing means 1 TCP connection handles all requests → fewer connections
- But one TCP loss affects ALL streams (TCP head-of-line blocking)
- HTTP/3 (QUIC over UDP) eliminates this — each stream is independent
- ALB supports HTTP/2 to clients but may use HTTP/1.1 to backends (default)

---

## 12. Common Mistakes

### Mistake 1: Using NLB for HTTP traffic
NLB can't route by path, can't terminate TLS with ACM, can't add X-Forwarded-For. If you need any HTTP feature, use ALB.

### Mistake 2: Backend keepalive shorter than ALB idle timeout
This is the #1 cause of intermittent 502 errors from ALB. Always set backend keepalive > ALB idle timeout.

### Mistake 3: Treating 502 and 504 the same
502 = backend gave a bad response (or closed the connection). 504 = backend didn't respond at all before timeout. Very different root causes.

### Mistake 4: Not logging X-Forwarded-For
Without it, all requests appear to come from the ALB's IP. You lose all client IP information for security, rate limiting, and debugging.

---

## 13. Cheat Sheet

### Status Codes Quick Reference

```
200 = OK          301 = Permanent redirect   400 = Bad request
201 = Created     302 = Temporary redirect   401 = Not authenticated
204 = No Content  304 = Not Modified         403 = No permission
                                             404 = Not found
502 = Bad gateway (backend broke)            429 = Rate limited
503 = No targets / overloaded
504 = Backend timeout                        500 = Server error
```

### ALB vs NLB Decision

```
HTTP routing needed? → ALB
Static IPs needed?   → NLB
TCP passthrough?     → NLB
gRPC?                → ALB (HTTPS + gRPC target group)
WebSockets?          → ALB
Extreme throughput?  → NLB
```

### Essential curl Commands

```bash
curl -v URL                      # Full debug
curl -I URL                      # Headers only
curl -w "TIMING_FORMAT" URL      # Measure phases
curl -X POST -d '{}' URL         # POST request
curl -H "Header: value" URL      # Custom header
curl -L URL                      # Follow redirects
curl --http2 URL                 # Force HTTP/2
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**OSI Model (Topic 1.1):** HTTP is the quintessential **Layer 7** protocol. ALB operates at L7 because it reads HTTP. NLB operates at L4 because it only sees TCP.

**TCP (Topic 1.2):** HTTP runs on TCP connections. Connection reuse (keep-alive), idle timeouts, and the relationship between TCP and HTTP connections directly impact performance and reliability.

**DNS (Topic 1.4):** Before HTTP can send a request, DNS resolves the hostname. The `Host` header in HTTP tells the server which site you want — SNI (TLS) and Host header (HTTP) both carry the hostname.

**TLS (Topic 1.5):** HTTPS = HTTP + TLS. TLS handshake completes before the first HTTP byte. ALB terminates TLS and forwards plain HTTP — understanding both layers is essential.

### What This Prepares You For Next
**Phase 2: Linux Networking** — Now that you understand all the protocols (IP, TCP, DNS, TLS, HTTP), Phase 2 shows you how Linux actually implements them. Network interfaces, routing tables, iptables rules, and network namespaces are the Linux kernel mechanisms that make all these protocols work. Every concept in Phase 2 will build on the protocol knowledge from Phase 1.
