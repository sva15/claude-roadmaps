# TLS/HTTPS — What Happens Before the First HTTP Byte

> 🔹 **Phase:** 1 — Network Foundations  
> 🔹 **Topic:** TLS/HTTPS  
> 🔹 **Priority:** MUST-KNOW — Interview Blocker

---

## 1. First Principles (Start Simple)

### What Problem Does This Solve?

When you send data over the internet, it travels through many routers, switches, and networks you don't control. Without encryption, anyone along that path can:

1. **Read** your data (passwords, credit cards, API keys)
2. **Modify** your data (inject malware, change bank transfers)
3. **Impersonate** a server (fake bank website that steals your credentials)

**TLS (Transport Layer Security)** solves all three:
- **Confidentiality** — Data is encrypted; only the intended receiver can read it
- **Integrity** — Data can't be modified in transit without detection
- **Authentication** — The server proves its identity via certificates

### The Analogy: The Sealed, Signed Envelope

- **HTTP** = Sending a postcard — anyone handling it can read the message
- **HTTPS** = Sending a sealed, signed envelope — nobody can read it, nobody can tamper with it, and the recipient can verify who sent it

But before you can seal the envelope, you need to:
1. Verify the other person IS who they claim to be (authentication)
2. Agree on a secret code only you two know (key exchange)
3. Then start sending sealed messages (encrypted data)

This "setup" process is the **TLS handshake** — it happens AFTER the TCP handshake but BEFORE any HTTP data flows.

### Why Does It Exist?

SSL (Secure Sockets Layer) was created by Netscape in 1994 for secure web commerce. It evolved through SSL 2.0, SSL 3.0, then was renamed to TLS:
- TLS 1.0 (1999), TLS 1.1 (2006) — both deprecated, insecure
- **TLS 1.2 (2008)** — current standard, minimum acceptable version
- **TLS 1.3 (2018)** — faster (1 fewer RTT), more secure, growing adoption

> 💡 **Key Insight:** HTTPS = HTTP + TLS. Every `kubectl` command, every Helm install, every service mesh sidecar connection, every HTTPS endpoint — all use TLS. Certificate errors in production are as common as DNS errors. Understanding TLS is not optional.

---

## 2. Why This Matters in DevOps

### Where This Appears in Real Systems

| Scenario | TLS Component |
|----------|--------------|
| HTTPS on ALB/Ingress | ACM certificate, TLS termination |
| `kubectl` talking to the API server | mTLS — client cert + server cert |
| Service mesh (Istio/Linkerd) | Automatic mTLS between all pods |
| cert-manager in K8s | Automatic certificate issuance + renewal |
| "Certificate expired" production alert | Missing renewal automation |
| Private CA for internal services | Self-signed CA chain, trust store management |
| "SSL certificate problem: unable to get local issuer certificate" | Missing intermediate CA cert |

### Why DevOps Engineers Must Know This

1. **Certificate lifecycle management** — Certificates expire. If renewal isn't automated, you get a 2 AM production incident. cert-manager, ACM, and Let's Encrypt all solve this differently.

2. **TLS termination decisions** — Where to terminate TLS (ALB? Ingress? Application?) has security, performance, and operational trade-offs.

3. **mTLS in service meshes** — Istio automatically encrypts all pod-to-pod traffic with mTLS. Understanding how mTLS works (both sides present certs) is essential for zero-trust architectures.

4. **Debugging certificate errors** — "CERTIFICATE_VERIFY_FAILED" is one of the most cryptic and common production errors. Understanding the certificate chain tells you exactly where the break is.

---

## 3. Core Concept (Deep Dive)

### The TLS 1.2 Handshake

After the TCP 3-way handshake completes, TLS begins:

```
Client                                 Server
  │                                       │
  │── ClientHello ───────────────────────►│
  │   • TLS version (1.2)                 │
  │   • Random bytes (client_random)      │
  │   • Supported cipher suites           │
  │   • SNI: "api.example.com"            │
  │                                       │
  │◄── ServerHello ──────────────────────│
  │   • Selected cipher suite             │
  │   • Random bytes (server_random)      │
  │   • Session ID                        │
  │                                       │
  │◄── Certificate ──────────────────────│
  │   • Server's X.509 certificate        │
  │   • Intermediate CA certificate(s)    │
  │                                       │
  │◄── ServerKeyExchange ────────────────│
  │   • ECDHE public key (for key exchange)│
  │                                       │
  │◄── ServerHelloDone ──────────────────│
  │                                       │
  │── ClientKeyExchange ─────────────────►│
  │   • Client's ECDHE public key         │
  │                                       │
  │   [Both sides compute the shared      │
  │    secret using ECDHE + randoms]      │
  │                                       │
  │── ChangeCipherSpec ─────────────────►│
  │── Finished (encrypted) ──────────────►│
  │                                       │
  │◄── ChangeCipherSpec ─────────────────│
  │◄── Finished (encrypted) ─────────────│
  │                                       │
  │      ═══ ENCRYPTED DATA FLOWS ═══    │
  │── GET / HTTP/1.1 (encrypted) ────────►│
```

**Key points:**
- Takes **2 additional RTTs** after TCP handshake (TLS 1.3 reduces this to 1)
- **SNI (Server Name Indication):** Client tells the server which hostname it's connecting to — critical for virtual hosting (multiple certs on same IP)
- **Cipher suite:** Agreement on encryption algorithm (e.g., `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`)
- **ECDHE:** Ephemeral Diffie-Hellman key exchange — even if the private key is later compromised, past sessions can't be decrypted (Forward Secrecy)

### TLS 1.3 Improvements

```
TLS 1.2: TCP (1 RTT) + TLS (2 RTT) = 3 RTTs before first data
TLS 1.3: TCP (1 RTT) + TLS (1 RTT) = 2 RTTs before first data

Key differences:
- Fewer cipher suites (only strong ones)
- Key exchange happens in the first message
- 0-RTT resumption (for repeat connections — controversial for replay attacks)
- No RSA key exchange (ECDHE only — forward secrecy mandatory)
```

### Certificates — The Chain of Trust

A certificate is a file that contains:
- **Subject:** Who the certificate identifies (e.g., `CN=api.example.com`)
- **Public key:** Used for encryption/key exchange
- **Issuer:** Who signed this certificate (the CA)
- **Validity:** Not Before / Not After dates
- **SAN (Subject Alternative Names):** Additional hostnames this cert is valid for
- **Signature:** The CA's digital signature proving this cert is legitimate

**The certificate chain:**

```
Root CA (trusted by OS/browser — pre-installed)
  │
  └── Intermediate CA ("I vouch for api.example.com")
        │
        └── Leaf Certificate (api.example.com's actual cert)
              │
              └── Contains the server's public key
```

The client verifies the chain:
1. Check the leaf cert — is it for the hostname I'm connecting to? Check SAN.
2. Check the intermediate cert — did it sign the leaf? Is it valid?
3. Check the root CA — is it in my trust store? Is it valid?
4. If the entire chain validates → **trusted**

> 🚨 **The #1 production TLS error:** A missing intermediate certificate. The server sends only the leaf cert. The client can't verify it because it doesn't have the intermediate. The root CA is trusted, but the chain is broken.

### mTLS (Mutual TLS)

In normal TLS, only the **server** presents a certificate. The client verifies the server's identity, but the server doesn't verify the client.

In **mTLS**, both sides present certificates:

```
Standard TLS:
  Client → "Show me your certificate" → Server shows cert
  Client verifies → Connection established

mTLS:
  Client → "Show me your certificate" → Server shows cert
  Client verifies → OK
  Server → "Now show me YOUR certificate" → Client shows cert
  Server verifies → OK → Connection established
```

**Where mTLS is used:**
- **Kubernetes API server** — `kubectl` uses a client certificate to authenticate
- **Service meshes (Istio)** — Every pod gets a certificate. Every connection is mTLS.
- **etcd** — Peer and client communication is mTLS
- **Zero-trust architectures** — Every service must prove its identity

---

## 4. Internal Working (Under the Hood)

### What the Kernel Does During TLS

TLS runs in **userspace**, not in the kernel (unlike TCP):

```
Application (nginx, curl, kubectl)
       │
  OpenSSL / BoringSSL / rustls (TLS library)
       │
  Socket API (read/write encrypted data)
       │
  Kernel TCP stack (handles TCP segments)
       │
  Network interface
```

The TLS library:
1. Generates random bytes for key material
2. Performs the handshake (cryptographic operations)
3. Encrypts each outgoing byte with the session key
4. Decrypts each incoming byte
5. Verifies message integrity with HMAC/AEAD

**CPU cost:** TLS handshake is the expensive part (asymmetric crypto: RSA/ECDHE). Data encryption is cheap (symmetric AES-GCM). This is why connection reuse is important — one handshake, many requests.

### Certificate Verification Process

```
openssl s_client -connect api.example.com:443

The TLS library does:
1. Receive server's certificate chain
2. For each certificate in the chain:
   a. Check validity dates (not before / not after)
   b. Check signature using the issuer's public key
   c. Check revocation status (OCSP/CRL if configured)
3. Check the leaf certificate's SAN:
   a. Does it include the hostname we connected to?
   b. Wildcard matching: *.example.com matches api.example.com
      but NOT sub.api.example.com (only one level)
4. Check if the root CA is in the trust store:
   a. System trust store: /etc/ssl/certs/ (Linux)
   b. JVM trust store: $JAVA_HOME/lib/security/cacerts
   c. Custom trust store: --cacert (curl), ca: (node.js)
5. If ALL checks pass → connection proceeds
   If ANY check fails → TLS error
```

### SNI (Server Name Indication)

When multiple domains share the same IP (common in cloud/K8s):

```
ALB IP: 52.1.2.3
  ├── api.example.com     → Certificate A
  ├── www.example.com     → Certificate B
  └── admin.example.com   → Certificate C

ClientHello includes SNI: "api.example.com"
→ ALB selects Certificate A
```

Without SNI, the server wouldn't know which certificate to present. SNI is sent in **plaintext** in TLS 1.2 (visible to network observers). TLS 1.3 introduced **Encrypted Client Hello (ECH)** to fix this.

---

## 5. Mental Models

### Mental Model 1: The ID Check at the Door

```
You arrive at a building (server):

Standard TLS:
  Guard: "Show me your building ID badge" (server cert)
  You check: Is it a real badge? From a real company? Not expired?
  ✓ Valid → You walk in

mTLS:
  Guard: "Show me your building badge"
  You: ✓ Valid
  Guard: "Now show ME your visitor pass" (client cert)
  Guard checks: Real pass? From authorized company? Not expired?
  ✓ Both valid → You walk in
```

### Mental Model 2: The Trust Chain

```
"Why should I trust this server's certificate?"

Because it was signed by DigiCert (intermediate CA)
  "Why should I trust DigiCert?"
  
Because DigiCert was signed by DigiCert Global Root
  "Why should I trust DigiCert Global Root?"
  
Because it's pre-installed in my operating system's trust store
  "Why should I trust my OS?"
  
...that's where the chain stops. Root CAs are the axioms of TLS trust.
```

### Mental Model 3: The Session Key Lifecycle

```
Handshake (expensive — asymmetric crypto):
  Client + Server negotiate → derive a shared session key
  This key exists ONLY for this connection
  
Data Transfer (cheap — symmetric crypto):  
  All data encrypted with the session key (AES-GCM)
  Very fast, hardware-accelerated on modern CPUs
  
Connection Close:
  Session key is destroyed
  Even if someone recorded all the traffic, they can't decrypt
  without the ephemeral key (Forward Secrecy)
```

---

## 6. Real-World Mapping

### Linux

```bash
# View a server's certificate
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -text -noout

# Check specific fields
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# Check certificate expiry
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -enddate

# Check the full certificate chain
openssl s_client -connect google.com:443 -showcerts

# Test specific TLS version
openssl s_client -connect google.com:443 -tls1_2
openssl s_client -connect google.com:443 -tls1_3

# Generate a self-signed CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt -subj "/CN=My CA"

# Generate a server cert signed by your CA
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=my-server.local"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365

# Test with curl using custom CA
curl --cacert ca.crt https://my-server.local:443

# View system trust store
ls /etc/ssl/certs/
```

### AWS

| AWS Service | TLS Role |
|------------|---------|
| **ACM (Certificate Manager)** | Free TLS certificates for ALB, CloudFront, API Gateway. Auto-renewal. Can't export private key. |
| **ALB TLS Termination** | ALB decrypts HTTPS, forwards HTTP to targets. Cert from ACM. Supports SNI (multiple certs per ALB). |
| **NLB TLS Passthrough** | NLB passes TLS traffic directly to targets. Server handles TLS. Use for TCP passthrough. |
| **NLB TLS Termination** | NLB can also terminate TLS (added later). Uses ACM certs. |
| **CloudFront** | CDN terminates TLS at edge. ACM cert (must be in us-east-1). |
| **API Gateway** | Automatically provides TLS. Custom domains use ACM certs. |
| **Private CA (ACM PCA)** | Run your own Certificate Authority in AWS. Issue private certs for internal services. Expensive ($400/month/CA). |

### Kubernetes

| K8s Component | TLS Role |
|--------------|---------|
| **API Server** | Serves on port 6443 with TLS. Verifies client certs (mTLS). |
| **kubelet** | mTLS with API server. Certificate rotation. |
| **etcd** | mTLS for peer and client connections. |
| **Ingress + cert-manager** | cert-manager watches Ingress annotations → requests cert from Let's Encrypt → mounts as TLS secret → Ingress uses it. |
| **Service Mesh (Istio)** | Istiod is the CA. Issues certs to all pods. Auto-rotates. All pod-to-pod traffic is mTLS. |
| **Secrets (TLS type)** | `kubectl create secret tls my-cert --cert=cert.pem --key=key.pem`. Referenced by Ingress. |

---

## 7. Failure Scenarios (VERY IMPORTANT)

### Scenario 1: "CERTIFICATE_VERIFY_FAILED" / "unable to get local issuer certificate"

| Attribute | Detail |
|-----------|--------|
| **Symptom** | `curl: (60) SSL certificate problem: unable to get local issuer certificate` |
| **Root Cause** | The server's certificate chain is incomplete — missing an intermediate CA certificate. |
| **OSI Layer** | L6/L7 |
| **Debug Steps** | 1. `openssl s_client -connect host:443 -showcerts` — count certificates returned. 2. Should see: leaf + intermediate(s). If only leaf → incomplete chain. 3. Check: who issued the leaf cert? Is that issuer's cert included? |
| **Fix** | On the server, configure the full chain: `ssl_certificate /path/to/fullchain.pem` (not just the leaf cert). The fullchain file should contain: leaf cert + intermediate cert(s). |

### Scenario 2: "certificate has expired"

| Attribute | Detail |
|-----------|--------|
| **Symptom** | `SSL certificate problem: certificate has expired`. All HTTPS connections fail. |
| **Root Cause** | Server certificate passed its "Not After" date. |
| **OSI Layer** | L6/L7 |
| **Debug Steps** | 1. `echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -enddate` 2. Compare with current date. |
| **Fix** | 1. Renew the certificate (ACM does this automatically). 2. If using cert-manager, check the Certificate resource: `kubectl describe certificate`. 3. If manual: regenerate, resign with CA, deploy. |
| **Prevention** | ACM auto-renews. cert-manager auto-renews. For manual certs: monitoring alerts when cert expires in < 30 days. |

### Scenario 3: "SSL: HANDSHAKE_FAILURE"

| Attribute | Detail |
|-----------|--------|
| **Symptom** | TLS connection fails immediately. No certificate error — the handshake itself fails. |
| **Root Cause** | Client and server don't share a common cipher suite or TLS version. |
| **OSI Layer** | L6/L7 |
| **Possible Causes** | 1. Server requires TLS 1.3 but client only supports TLS 1.2. 2. Server restricts to specific cipher suites that the client doesn't support. 3. Old client library with outdated cipher support. |
| **Debug Steps** | 1. `openssl s_client -connect host:443 -tls1_2` — try different TLS versions. 2. Check server config for cipher restrictions. 3. Compare client's supported ciphers with server's allowed list. |

### Scenario 4: ALB 502 Due to TLS Mismatch with Backend

| Attribute | Detail |
|-----------|--------|
| **Symptom** | ALB returns HTTP 502 to clients. Backend appears healthy. |
| **Root Cause** | ALB target group is configured for HTTPS (port 443 to backend), but the backend is serving plain HTTP, or vice versa. |
| **OSI Layer** | L7 |
| **Debug Steps** | 1. Check target group protocol (HTTP vs HTTPS). 2. Check what the backend actually serves. 3. `curl -v http://backend-ip:port` directly — does it respond? 4. Check ALB access logs for error reason. |
| **Fix** | Match the target group protocol with what the backend serves. Most common setup: ALB terminates TLS (HTTPS listeners), forwards HTTP to backend targets. |

### Scenario 5: cert-manager Not Issuing Certificate

| Attribute | Detail |
|-----------|--------|
| **Symptom** | Ingress shows "fake certificate" or browser shows "not secure." Certificate resource shows pending/failed. |
| **Root Cause** | cert-manager can't complete the ACME challenge (Let's Encrypt). |
| **OSI Layer** | L7 |
| **Debug Steps** | 1. `kubectl describe certificate <name>` — check conditions. 2. `kubectl describe certificaterequest` — check status. 3. `kubectl describe challenge` — did the HTTP-01 or DNS-01 challenge fail? 4. For HTTP-01: is port 80 accessible from the internet? Is the /.well-known/acme-challenge path reachable? 5. For DNS-01: does cert-manager have permission to create TXT records? |

---

## 8. Debugging Playbook

### TLS Debug Steps

```
Step 1: Can you connect at all?
───────────────────────────────
$ openssl s_client -connect host:443
  ✓ "Verify return code: 0 (ok)" → Certificate valid
  ✗ "Verify return code: 20" → unable to get local issuer certificate
  ✗ "Verify return code: 10" → certificate has expired

Step 2: Check the certificate details
──────────────────────────────────────
$ echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -text -noout
→ Check Subject, SAN, Issuer, Validity dates

Step 3: Check the chain
────────────────────────
$ openssl s_client -connect host:443 -showcerts
→ Should show multiple certs (leaf + intermediate)
→ If only one cert → incomplete chain

Step 4: Check specific TLS version
───────────────────────────────────
$ openssl s_client -connect host:443 -tls1_2
$ openssl s_client -connect host:443 -tls1_3
→ Does it connect with the required version?

Step 5: Check certificate expiry
────────────────────────────────
$ echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -enddate
→ Is it expired? Expiring soon?

Step 6: Measure TLS handshake time
──────────────────────────────────
$ curl -w "\ntcp:%{time_connect}s\ntls:%{time_appconnect}s\n" -o /dev/null -s https://host
→ TLS time = appconnect - connect
→ If TLS time is high → slow cryptographic operations or high RTT
```

---

## 9. Hands-On Practice

### Lab 1: Read a Real Certificate

```bash
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -text -noout

# Find:
# 1. Subject (CN and O)
# 2. Issuer (who signed it)
# 3. Validity dates
# 4. Subject Alternative Names (SAN) — all domains this cert covers
# 5. Public Key algorithm and size
# 6. Signature algorithm
```

### Lab 2: Create Your Own CA and Sign a Certificate

```bash
# 1. Create a CA
openssl genrsa -out myCA.key 4096
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 365 -out myCA.crt \
  -subj "/C=US/ST=Test/L=Test/O=MyOrg/CN=My Test CA"

# 2. Create a server key and CSR
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
  -subj "/CN=my-server.local"

# 3. Sign the server cert with your CA
openssl x509 -req -in server.csr -CA myCA.crt -CAkey myCA.key \
  -CAcreateserial -out server.crt -days 365 -sha256

# 4. Start a test server (nginx or openssl)
openssl s_server -cert server.crt -key server.key -port 8443

# 5. In another terminal, connect WITHOUT trusting the CA:
curl https://localhost:8443
# → Error: unable to get local issuer certificate

# 6. Connect WITH custom CA:
curl --cacert myCA.crt https://localhost:8443
# → Success!
```

### Lab 3: Check Certificate Expiry Across Services

```bash
# Create a script to check multiple endpoints:
for host in google.com github.com api.example.com; do
  expiry=$(echo | openssl s_client -connect $host:443 2>/dev/null | openssl x509 -noout -enddate 2>/dev/null)
  echo "$host: $expiry"
done
```

### Lab 4: Measure TLS Impact on Latency

```bash
# HTTP (no TLS):
curl -w "tcp:%{time_connect}s total:%{time_total}s\n" -o /dev/null -s http://example.com

# HTTPS (with TLS):
curl -w "tcp:%{time_connect}s tls:%{time_appconnect}s total:%{time_total}s\n" -o /dev/null -s https://example.com

# TLS overhead = time_appconnect - time_connect
```

---

## 10. Interview Preparation

### Q1: (Must-know) What is the difference between a certificate and a private key? What happens if the private key leaks?

**Answer:** A certificate contains the public key + identity + CA signature. It's meant to be shared publicly. The private key is the secret that only the server should have — it proves ownership of the certificate. If the private key leaks, an attacker can impersonate the server (MITM attack). You must immediately revoke the certificate and issue a new one with a new key pair.

### Q2: Walk through the TLS handshake.

**Answer:** After TCP handshake: ClientHello (supported ciphers, TLS version, SNI) → ServerHello (selected cipher) + Certificate (server's cert chain) + KeyExchange (ECDHE parameters) → Client verifies cert chain against trust store → Client sends its KeyExchange → Both derive session keys → ChangeCipherSpec → Finished → Data flows encrypted.

### Q3: What is mTLS and where is it used?

**Answer:** Mutual TLS — both client and server present certificates and verify each other. Standard TLS only verifies the server. mTLS is used in: K8s API server (kubectl uses client cert), service meshes (Istio — every pod has a cert, every connection is mTLS), etcd cluster communication, zero-trust architectures.

### Q4: A service returns "certificate has expired." How do you handle it?

**Answer:** 1. Verify: `openssl s_client -connect host:443` — check expiry date. 2. If ACM: check certificate status — ACM should auto-renew (requires DNS validation to work). 3. If cert-manager: `kubectl describe certificate` — check renewal status, check challenge status. 4. If manual: renew with CA, deploy new cert. 5. Prevention: monitoring alerts for certs expiring within 30 days.

### Q5: Where should TLS be terminated in an EKS architecture?

**Answer:** Most common: terminate at the ALB. ACM provides the cert (free, auto-renew). ALB forwards plain HTTP to pods. Benefits: simplified cert management, ALB handles TLS CPU overhead, ACM auto-renews.

Alternative for mTLS requirements: terminate at the Ingress controller (nginx) or at the application. Istio: terminate at the sidecar — pod-to-pod is mTLS. The trade-off: more operational complexity but end-to-end encryption.

### Q6: Explain why a missing intermediate certificate causes failures.

**Answer:** The client verifies the cert chain: leaf → intermediate → root. The root is in the trust store. But the client can't jump from leaf to root — it needs the intermediate to verify the leaf's signature. If the server only sends the leaf cert, the client says "I can't verify who signed this" → CERTIFICATE_VERIFY_FAILED. Fix: configure the server to send the full chain (leaf + intermediate).

---

## 11. Advanced Insights (Senior Level)

### TLS Termination Architecture Decisions

```
Option A: Terminate at ALB (most common)
Client ── HTTPS ──► ALB ── HTTP ──► Pods
✓ Simple. ACM auto-renew. No cert management in K8s.
✗ Traffic between ALB and pods is unencrypted (within VPC).

Option B: Terminate at Ingress (nginx)
Client ── HTTPS ──► ALB(passthrough) ──► nginx ── HTTP ──► Pods
✓ More control over TLS config. cert-manager for automation.
✗ More complex. Must manage cert renewal in K8s.

Option C: Terminate at Application
Client ── HTTPS ──► ALB(passthrough) ──► Pod(TLS)
✓ End-to-end encryption.
✗ Every app needs TLS config. Certificate management per service.

Option D: Service Mesh (Istio)
Client ── HTTPS ──► ALB ── HTTP ──► Sidecar ── mTLS ──► Sidecar ── HTTP ──► Pod
✓ Automatic mTLS everywhere. Zero-trust. No app changes.
✗ Added latency (~1-3ms), complexity, operational overhead.
```

### Certificate Rotation Strategies

| Approach | How It Works | Downtime? |
|----------|-------------|-----------|
| **ACM** | AWS auto-renews. Updates ALB/CloudFront transparently. | No |
| **cert-manager** | Watches Certificate CRD. Renews before expiry. Updates K8s Secret. Ingress reloads. | No |
| **Manual** | Generate new cert, replace file/secret, restart/reload server. | Possible |

### Forward Secrecy (Why ECDHE Matters)

If the server's RSA private key is compromised, an attacker with recorded traffic can decrypt ALL past sessions (RSA key exchange). With ECDHE (Ephemeral Diffie-Hellman), each connection uses a new temporary key pair. Even if the private key leaks, past sessions remain secure. This is called **Perfect Forward Secrecy (PFS)**.

> TLS 1.3 mandates forward secrecy — RSA key exchange is removed entirely.

---

## 12. Common Mistakes

### Mistake 1: Using self-signed certs in production without a trust store
Self-signed certificates are not trusted by default. Adding `--insecure` or `VERIFY_NONE` to skip verification defeats the purpose of TLS. Use ACM, Let's Encrypt, or a private CA with proper trust store distribution.

### Mistake 2: Not including intermediate CA certificates
Only deploying the leaf certificate. Browsers might work (they cache intermediates) but `curl`, APIs, and other non-browser clients fail with "unable to get local issuer certificate."

### Mistake 3: Letting certificates expire
Manual renewal is fragile. Use automated solutions: ACM (AWS), cert-manager (K8s), Certbot (standalone). Monitor expiry with alerting.

### Mistake 4: Confusing TLS termination with end-to-end encryption
Terminating TLS at the ALB means traffic between ALB and pods is unencrypted. This is fine if the VPC is trusted, but not acceptable for compliance-heavy environments (PCI-DSS, HIPAA). Use mTLS (service mesh) for true end-to-end encryption.

### Mistake 5: Thinking "SSL" and "TLS" are different things
SSL is the old name (deprecated). TLS is the current standard. When someone says "SSL certificate," they mean a TLS certificate. The protocols SSL 2.0 and SSL 3.0 are insecure and must never be enabled.

---

## 13. Cheat Sheet

### Certificate Verification Commands

```bash
# View certificate
openssl s_client -connect host:443 | openssl x509 -text -noout

# Check expiry
openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -enddate

# View chain
openssl s_client -connect host:443 -showcerts

# Test specific TLS version
openssl s_client -connect host:443 -tls1_2
openssl s_client -connect host:443 -tls1_3

# curl with timing
curl -w "tcp:%{time_connect}s tls:%{time_appconnect}s\n" -o /dev/null -s https://host
```

### Common TLS Errors

| Error | Meaning | Fix |
|-------|---------|-----|
| `unable to get local issuer certificate` | Missing intermediate cert | Add intermediate to cert chain |
| `certificate has expired` | Cert past validity | Renew certificate |
| `HANDSHAKE_FAILURE` | Cipher/version mismatch | Check TLS version and cipher compatibility |
| `hostname mismatch` | Cert doesn't match hostname | Check SAN field in certificate |
| `self signed certificate` | Not signed by trusted CA | Add CA to trust store or use a public CA |

### TLS in AWS

```
ACM → Free certs, auto-renew, use with ALB/CloudFront
ALB → Terminates TLS, listeners on 443, forwards HTTP to targets
NLB → Can terminate TLS or passthrough
CloudFront → TLS at edge, cert must be in us-east-1
```

### TLS in K8s

```
cert-manager → Automates cert issuance from Let's Encrypt
Ingress → References a TLS Secret (cert + key)
Service Mesh → Automatic mTLS between all pods
API Server → mTLS on port 6443
```

---

## 🔗 Connections

### How This Connects to Previous Topics
**OSI Model (Topic 1.1):** TLS operates at layers 5-6 (Session/Presentation) of the OSI model, though in practice it's treated as part of L7. When you see a TLS error, you're looking at a layer above TCP but below HTTP.

**TCP (Topic 1.2):** TLS runs ON TOP of TCP. The TCP 3-way handshake must complete first, then TLS adds its own handshake (2 more RTTs for TLS 1.2, 1 for TLS 1.3). Connection reuse is even more valuable with TLS because you avoid both the TCP and TLS handshake overhead.

**IP/CIDR (Topic 1.3):** Certificates contain SANs (hostnames) not IPs. TLS connects names to identity. IP routing gets the packet there; TLS ensures you're talking to the right entity.

**DNS (Topic 1.4):** DNS resolves the name, then TLS verifies that name matches the certificate. If DNS is poisoned (returns a fake IP), TLS catches it — the server at the fake IP won't have a valid certificate for the real domain. This is why DNS + TLS together provide security.

### What This Prepares You For Next
**Next topic: HTTP/1.1, HTTP/2, and How Load Balancers Use Them**

Now you understand everything that happens BEFORE HTTP data flows: DNS resolves the name (Topic 4), IP routing gets the packet to the server (Topic 3), TCP establishes a reliable connection (Topic 2), and TLS encrypts the channel (this topic). The next topic covers what happens on that encrypted channel — HTTP methods, status codes, headers, and how ALBs use HTTP features for intelligent routing.
