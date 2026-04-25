# System Design Fundamentals
### A Complete Reference for DevOps Engineers

> **How to use this document:** System design is the skill that separates engineers who "just do tasks" from engineers who understand *why* things are built the way they are. This document teaches you to think at the architecture level — how systems are structured, how they scale, how they fail, and how you design them to survive all of it.

---

## Table of Contents

1. [What is System Design?](#1-what-is-system-design)
2. [Monolith vs Microservices](#2-monolith-vs-microservices)
3. [Horizontal vs Vertical Scaling](#3-horizontal-vs-vertical-scaling)
4. [Stateless vs Stateful Services](#4-stateless-vs-stateful-services)
5. [Load Balancing — Deep Dive](#5-load-balancing--deep-dive)
6. [Caching Strategies](#6-caching-strategies)
7. [Message Queues & Event-Driven Architecture](#7-message-queues--event-driven-architecture)
8. [Databases in System Design](#8-databases-in-system-design)
9. [Single Point of Failure — SPOF](#9-single-point-of-failure--spof)
10. [High Availability & Fault Tolerance](#10-high-availability--fault-tolerance)
11. [CDN — Content Delivery Networks](#11-cdn--content-delivery-networks)
12. [API Design & API Gateway](#12-api-design--api-gateway)
13. [Service Discovery & Configuration](#13-service-discovery--configuration)
14. [Rate Limiting & Throttling](#14-rate-limiting--throttling)
15. [Observability — Logs, Metrics, Traces](#15-observability--logs-metrics-traces)
16. [SLI, SLO, SLA — Reliability Targets](#16-sli-slo-sla--reliability-targets)
17. [Disaster Recovery](#17-disaster-recovery)
18. [Designing for Failure — Resilience Patterns](#18-designing-for-failure--resilience-patterns)
19. [Capacity Planning & Back-of-Envelope Estimation](#19-capacity-planning--back-of-envelope-estimation)
20. [Real-World Architecture Walkthroughs](#20-real-world-architecture-walkthroughs)
21. [System Design Interview Framework](#21-system-design-interview-framework)
22. [Quick Reference Cheat Sheet](#22-quick-reference-cheat-sheet)

---

## 1. What is System Design?

**System design** is the process of defining the architecture, components, and interactions of a system to meet specific requirements — particularly around scale, reliability, and performance.

When someone asks "how would you design Twitter?" they're not asking you to write code. They're asking you to think through:
- How many users? How many requests per second?
- Where is data stored? How is it organized?
- What happens when one server fails?
- How does it stay fast under load?
- How do the pieces talk to each other?

### Why DevOps Engineers Must Understand System Design

As a DevOps engineer you are not just running what developers built — you are making architectural decisions constantly:

| Decision | System Design Concept |
|---|---|
| "Should we use RDS or self-managed Postgres?" | Database selection, managed vs self-hosted |
| "How many servers do we need?" | Capacity planning, horizontal scaling |
| "What happens if this service goes down?" | SPOF, fault tolerance, redundancy |
| "Why is the app slow under load?" | Bottleneck analysis, caching, load balancing |
| "How should we deploy without downtime?" | Blue-green, canary, rolling deployments |
| "Should we use Kafka or SQS?" | Message queue selection |
| "How do we ensure 99.9% uptime?" | SLOs, HA design |

Every infrastructure decision you make is a system design decision.

### Key Properties of Well-Designed Systems

```
Scalability   → Can handle more load by adding resources
Reliability   → Works correctly even when parts fail
Availability  → Accessible when users need it
Performance   → Responds fast under expected load
Maintainability → Easy to operate, debug, and change
Security      → Protects data and resists attacks
Cost Efficiency → Doesn't waste resources
```

---

## 2. Monolith vs Microservices

This is the most fundamental architectural decision. Every application starts somewhere on this spectrum.

### Monolith — One Big Application

All functionality lives in a single deployable unit. The entire application — user auth, orders, payments, notifications, reporting — is one codebase deployed as one binary or process.

```
┌─────────────────────────────────────────┐
│            Monolith Application         │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │   Auth   │  │  Orders  │  │ Email │ │
│  └──────────┘  └──────────┘  └───────┘ │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ Payments │  │ Products │  │ Admin │ │
│  └──────────┘  └──────────┘  └───────┘ │
│                                         │
│          One Database                   │
└─────────────────────────────────────────┘
           │
     Deploy everything together
```

**Advantages:**
- Simple to develop, deploy, and debug (one thing to run)
- No network latency between components (all in-process function calls)
- Transactions are easy (everything in one database)
- Easy to test end-to-end
- Lower operational complexity — one service to monitor, log, and scale

**Disadvantages:**
- Scaling: must scale the whole app even if only one part is under load
- Deployment: changing one line requires redeploying everything
- Technology lock-in: entire app must use the same language/framework
- Team scaling: large teams working on one codebase have merge conflicts and slow release cycles
- Reliability: one bug can crash the whole application

**When monolith is the right choice:**
- Early-stage startups — speed of development matters more than scale
- Small teams (fewer than ~8 engineers)
- Well-understood, stable domains
- When you don't yet know how to draw the boundaries

### Microservices — Many Small Services

The application is split into multiple independent services, each responsible for a specific business capability, deployed and scaled independently.

```
                    ┌──────────────┐
     Client ──────→ │  API Gateway │
                    └──────┬───────┘
                           │ routes requests
          ┌────────────────┼────────────────────┐
          ↓                ↓                     ↓
   ┌────────────┐   ┌────────────┐       ┌────────────┐
   │Auth Service│   │Order Service│      │Product Svc │
   │  (Node.js) │   │  (Python)  │       │   (Go)     │
   └─────┬──────┘   └─────┬──────┘       └─────┬──────┘
         │                │                     │
   ┌─────▼──────┐   ┌─────▼──────┐       ┌─────▼──────┐
   │  Auth DB   │   │  Orders DB │       │ Product DB │
   └────────────┘   └────────────┘       └────────────┘
                           │
                    ┌──────▼──────┐
                    │   Message   │
                    │    Queue    │  → Email Service
                    └─────────────┘  → Analytics Service
```

**Advantages:**
- Independent scaling: scale only the services under load
- Independent deployment: deploy one service without touching others
- Technology freedom: each service can use the best language/framework
- Fault isolation: one service crashing doesn't bring down others
- Team autonomy: each team owns one service end-to-end

**Disadvantages:**
- Network latency between services (in-process call vs HTTP call)
- Distributed systems complexity: partial failures, eventual consistency
- Much higher operational overhead (many services to deploy, monitor, debug)
- Distributed tracing is hard — one user request touches 5 services
- Database transactions across services require saga patterns
- Much harder to test end-to-end

**When microservices make sense:**
- Large teams where independent deployment velocity is critical
- Wildly different scaling requirements per function (image processing vs auth)
- Clear domain boundaries that rarely change
- Mature DevOps capabilities — CI/CD, containerization, service mesh

### The Spectrum: Modular Monolith

The smartest approach for most growing companies:

**Modular Monolith:** A well-structured monolith where different domains are cleanly separated by module boundaries — but still deployed as one unit. Easy to split into microservices later when the boundaries are proven and scale demands it.

```
Monolith → Modular Monolith → Microservices (when justified)
```

> **Rule of thumb:** Don't start with microservices. Start as a monolith. Extract services when you have clear evidence that specific parts need independent scaling or deployment.

### Service Granularity

When designing microservices, the hardest question is how to draw the boundaries.

**Too coarse-grained (not enough services):**
- `UserService` handles auth, profiles, settings, preferences, social connections — too many responsibilities

**Too fine-grained (too many services):**
- `GetUsernameService` just returns usernames — wasteful overhead

**Right-sized: Domain-Driven Design (DDD)**
Services should align with **bounded contexts** — the natural boundaries of your business domain:
- `AuthService` — login, logout, tokens
- `UserProfileService` — user data, preferences
- `OrderService` — order lifecycle
- `PaymentService` — payment processing
- `NotificationService` — emails, SMS, push

---

## 3. Horizontal vs Vertical Scaling

When your system can't handle the load, you scale. Two fundamental approaches.

### Vertical Scaling (Scale Up)

Make the existing machine bigger — more CPU cores, more RAM, faster disk.

```
Before:       After:
┌────────┐    ┌──────────────┐
│ 4 CPU  │ →  │   32 CPU     │
│ 16 GB  │    │   256 GB     │
│ SSD    │    │   NVMe SSD   │
└────────┘    └──────────────┘
```

**Advantages:**
- Simple — no application changes required
- No distributed systems complexity
- Better for stateful workloads (databases)

**Disadvantages:**
- Hard limit: you can only get so big (the biggest AWS instance has limits)
- Expensive at the high end (cost doesn't scale linearly — doubles in price for 20% more power)
- Single point of failure: still one machine — if it fails, you're down
- Requires downtime to upgrade (usually)

**Best for:** Databases (early/mid stage), stateful services that are hard to distribute.

### Horizontal Scaling (Scale Out)

Add more machines running the same software. Put a load balancer in front to distribute traffic.

```
Before:                   After:
                          ┌──────────┐
                   ┌────→ │ Server 1 │
┌────────┐         │      └──────────┘
│ Server │  →  LB ─┼────→ │ Server 2 │
└────────┘         │      └──────────┘
                   └────→ │ Server 3 │
                          └──────────┘
```

**Advantages:**
- No hard ceiling — keep adding machines as load grows
- High availability — if one server dies, others handle traffic
- Commodity hardware — no need for expensive specialized machines
- Can scale down (remove servers) to save cost

**Disadvantages:**
- Application must be stateless (or state must be externalized) — can't store session in local memory if requests hit different servers
- More complex infrastructure (load balancer, multiple servers, shared storage)
- Distributed systems challenges emerge

**Best for:** Stateless web/application servers, API servers, workers.

### The Combination

In practice, most systems use both:

```
Application tier:   Horizontal scaling (many small app servers behind LB)
Database tier:      Vertical scaling first → then read replicas → then sharding
Cache tier:         Horizontal (Redis Cluster)
```

### Auto-Scaling

Modern cloud infrastructure auto-scales horizontally based on load:

```
CPU > 70% for 5 minutes → add 2 servers
CPU < 30% for 10 minutes → remove 1 server
```

```yaml
# AWS Auto Scaling Group concept
min_instances: 2          # Never go below 2 (for HA)
max_instances: 20         # Never exceed 20 (cost control)
desired_capacity: 4       # Start with 4
scale_up_threshold: 70%   # CPU > 70% → add instances
scale_down_threshold: 30% # CPU < 30% → remove instances
```

---

## 4. Stateless vs Stateful Services

This is one of the most critical distinctions in system design.

### Stateless Services

A **stateless** service does not store any data about previous requests. Every request contains all the information needed to process it. The server has no memory of past interactions.

```
Request 1 → Server A: "Get user 1001's profile"  → reads from DB → responds
Request 2 → Server B: "Get user 1001's profile"  → reads from DB → responds
(Requests can go to any server — same result)
```

**Benefits:**
- Any server can handle any request → trivially horizontally scalable
- A server can die and no data is lost (it holds no state)
- Easy to add and remove servers
- No session stickiness needed

**How to stay stateless:** Push state into external systems:
- User sessions → Redis / JWT tokens in the request
- Uploaded files → S3 / object storage (not local disk)
- Caches → Redis (not in-process memory)
- Configuration → environment variables or config service

### Stateful Services

A **stateful** service remembers previous interactions or holds data that is not easily replicated.

```
Connection 1 → Server A: game session data stored in Server A's memory
Connection 2 → Server B: no knowledge of the session on Server A
(Must use sticky sessions or session is lost)
```

Examples: databases, video game servers with in-memory game state, streaming connections, WebSocket servers.

**Challenges:**
- Must route requests to the right server (sticky sessions / consistent hashing)
- Server failure = potential state loss
- Much harder to scale horizontally

### The Golden Rule

> **Make your application tier stateless. Push state to the data tier.**

```
Stateless application servers (scale freely)
           ↕
     Load Balancer
           ↕
┌──────────────────────────────────┐
│          Stateful Data Tier       │
│  Database   Redis   Object Store  │
│  (managed separately, carefully)  │
└──────────────────────────────────┘
```

---

## 5. Load Balancing — Deep Dive

You covered load balancing in the networking document. Here we go deeper on the system design implications.

### Where Load Balancers Live

```
Internet
    ↓
DNS Load Balancing (GeoDNS — route to nearest data center)
    ↓
Global Load Balancer / CDN Edge (Cloudflare, AWS Global Accelerator)
    ↓
L4/L7 Load Balancer (AWS ALB, nginx, HAProxy)
    ↓
Application Server Pool
    ↓
Internal Load Balancer (service-to-service traffic)
    ↓
Database (read replica load balancing — PgBouncer, ProxySQL)
```

### Load Balancer Algorithms — Which to Use When

**Round Robin:**
Requests go to each server in turn (1→2→3→1→2→3...).
Best when: All requests have similar processing time and all servers have equal capacity.

**Least Connections:**
Next request goes to the server with the fewest active connections.
Best when: Requests have varying processing times (some take 100ms, some take 5 seconds) — prevents one server accumulating many long-running connections.

**IP Hash / Consistent Hashing:**
The same client IP always goes to the same server (sticky sessions without a cookie).
Best when: Session data is stored server-side (stateful applications, though you should avoid this).

**Weighted Round Robin:**
Servers with more capacity get proportionally more requests.
Best when: Servers have different hardware specs — a 32-core server gets 4× more traffic than an 8-core server.

**Least Response Time:**
Routes to the server with lowest average response time AND fewest connections.
Best when: Latency is the primary concern, servers are heterogeneous.

### Health Checks

Load balancers continuously probe backends:

```yaml
# nginx upstream health check
upstream app_servers {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

# HAProxy health check config
backend app_servers
    option httpchk GET /health
    http-check expect status 200
    server app1 10.0.0.1:8080 check inter 5s fall 3 rise 2
    # check every 5 seconds, mark down after 3 failures, mark up after 2 successes
```

The health check endpoint (`/health`, `/healthz`, `/ping`) should:
- Return 200 OK when the server is healthy
- Return 5xx when the server should not receive traffic (DB connection down, etc.)
- Respond quickly — don't make it do database queries

### Connection Draining / Graceful Shutdown

When removing a server from rotation (deployment, scaling down), you don't want to abruptly cut connections:

```
1. Mark server as "draining" in load balancer
   (Stop sending new connections, but existing ones continue)
2. Wait for existing connections to complete (30-60 seconds)
3. If still connections after timeout, force close
4. Remove server from pool
```

```bash
# AWS: deregister target with connection draining (300 second delay)
aws elbv2 deregister-targets --target-group-arn arn:... --targets Id=i-1234567890

# nginx: graceful reload (no connection drops)
nginx -s reload

# systemd: SIGTERM triggers graceful shutdown in well-written apps
systemctl stop myapp   # Sends SIGTERM, waits for graceful shutdown
```

---

## 6. Caching Strategies

You covered Redis in the database document. Here we look at caching as a system design pattern — where in the architecture to cache and what.

### The Caching Decision Framework

Ask these questions before adding a cache:

1. Is this data read far more often than it's written? (**read-heavy**)
2. Is computing or fetching it expensive? (complex query, external API call)
3. Is it OK if users see slightly stale data for a few seconds/minutes?
4. Is the data the same (or similar) for many users?

If most answers are yes → caching is appropriate.

### Where to Cache

```
Browser         → HTTP cache headers (Cache-Control, ETag) — user's browser stores responses
CDN             → static assets, entire page HTML for anonymous users
Application     → Redis/Memcached — frequently accessed DB results, computed data
Database        → PostgreSQL's shared_buffers, MySQL's InnoDB buffer pool
Hardware        → CPU L1/L2/L3 cache, disk read cache — managed by OS/hardware
```

### Cache Placement Patterns

**In-Process Cache (Local Cache):**
Cache inside the application process memory. Fastest possible — no network hop.
```python
# Simple in-process dict cache (Python)
_cache = {}
def get_config(key):
    if key in _cache:
        return _cache[key]
    value = db.get(key)
    _cache[key] = value
    return value
```
Problem: Each app server has its own cache. If you have 10 app servers, you have 10 caches — each server might cache different versions of the same data.

**Distributed Cache (Shared Cache):**
A separate caching service (Redis, Memcached) shared by all app servers.

```
App Server 1 ──┐
App Server 2 ──┼──→ Redis Cluster → same cached data for all servers
App Server 3 ──┘
```

All servers see the same cache. One write invalidates for everyone.

### Cache-Aside vs Read-Through vs Write-Through

**Cache-Aside (Lazy Loading):**
Application manages the cache explicitly. Most common and flexible.
```
Read: Check cache → HIT: return | MISS: read DB → write to cache → return
Write: Write to DB → invalidate/update cache
```

**Read-Through:**
Cache sits in front of the DB. Cache handles the miss automatically.
```
Read: Application asks cache → MISS: cache reads DB → cache stores → returns to app
Application only ever talks to the cache
```

**Write-Through:**
Writes go to cache and database simultaneously.
```
Write: App writes to cache → cache writes to DB → both updated
Pros: Cache always in sync
Cons: Write latency, cache filled with data that may never be read
```

**Write-Behind (Write-Back):**
Write to cache only. DB is updated asynchronously.
```
Write: App writes to cache → cache returns OK immediately → DB updated later
Pros: Fastest write performance
Cons: Risk of data loss if cache fails before DB write
```

### What to Cache — And What Not To

**Good candidates for caching:**
- User profile data (read millions of times, written rarely)
- Product catalog / pricing (same for all users, changes infrequently)
- Computed aggregates (total orders today — expensive to recompute)
- Authentication tokens / session data
- Configuration and feature flags
- Rendered HTML fragments
- API responses from external services

**Poor candidates for caching:**
- Highly personalized data (different per user, low reuse)
- Rapidly changing data (stock prices, real-time scores)
- Security-sensitive data that must always be current (account balances, permissions)
- Data that is already fast to retrieve

### Cache Stampede / Thundering Herd

A popular cache key expires. Simultaneously, hundreds of requests arrive, all miss the cache, all hit the database at once → DB is overwhelmed.

**Solutions:**

**1. Probabilistic Early Expiration:**
Slightly before TTL expires, probabilistically refresh the cache (each request has a small chance of proactively refreshing).

**2. Locking / Mutex:**
Only one process rebuilds the cache key. Others wait or return stale data.
```python
import redis

def get_user(user_id):
    key = f"user:{user_id}"
    lock_key = f"lock:{key}"

    cached = redis.get(key)
    if cached:
        return json.loads(cached)

    # Try to acquire lock
    if redis.set(lock_key, 1, nx=True, ex=30):   # nx = only set if not exists
        # We got the lock — rebuild cache
        user = db.query(...)
        redis.setex(key, 3600, json.dumps(user))
        redis.delete(lock_key)
        return user
    else:
        # Another process is rebuilding — wait briefly and retry
        time.sleep(0.1)
        return get_user(user_id)   # Retry
```

**3. Staggered TTLs:**
Add random jitter to TTLs so not everything expires at the same moment.
```python
base_ttl = 3600
jitter = random.randint(0, 300)  # ±5 minutes variation
redis.setex(key, base_ttl + jitter, value)
```

---

## 7. Message Queues & Event-Driven Architecture

Message queues are one of the most powerful tools in system design. Understanding them transforms how you think about system coupling and reliability.

### Why Message Queues Exist

**Direct / Synchronous coupling (without a queue):**
```
User places order
    → OrderService calls PaymentService (HTTP) — must wait
    → OrderService calls InventoryService (HTTP) — must wait
    → OrderService calls EmailService (HTTP) — must wait
    → OrderService calls AnalyticsService (HTTP) — must wait
    → Return success to user (only after ALL steps complete)

Problems:
- If EmailService is down → order fails entirely
- High latency (sequential calls or parallel but all must complete)
- OrderService must know about every downstream service
- Adding a new service requires changing OrderService
```

**With a message queue:**
```
User places order
    → OrderService publishes "OrderPlaced" event to queue
    → Return success to user immediately

Queue delivers to subscribers asynchronously:
    PaymentService     → processes payment
    InventoryService   → reserves stock
    EmailService       → sends confirmation email
    AnalyticsService   → records the event

Benefits:
- If EmailService is down → message stays in queue, retried when it recovers
- Order completes in milliseconds (queue write is fast)
- OrderService knows nothing about downstream services
- Add a new subscriber without changing OrderService
```

### Core Concepts

**Producer:** Sends messages to the queue.
**Consumer:** Reads messages from the queue and processes them.
**Queue / Topic:** The channel that stores and delivers messages.
**Message:** The data payload being sent (JSON, Protobuf, etc.).
**Consumer Group:** Multiple consumers that share the work of processing messages from a topic (each message goes to one consumer in the group).
**Acknowledgment (ACK):** Consumer tells the queue "I processed this successfully — you can delete it."

```
Producer → [Message Queue] → Consumer 1
                           → Consumer 2
                           → Consumer 3
(Each message delivered to exactly one consumer in the group)
```

### Message Queue Guarantees

**At-Most-Once Delivery:**
Message delivered zero or one time. May be lost if consumer crashes before ACK. Fastest.

**At-Least-Once Delivery:**
Message delivered one or more times. If consumer crashes before ACK, message is re-queued and redelivered. **Your consumer must be idempotent** (processing the same message twice produces the same result as processing it once).

**Exactly-Once Delivery:**
Message delivered exactly once. Very hard to achieve at scale. Kafka supports it with transactions, but with performance cost.

**Most systems use at-least-once + idempotent consumers.** Design your consumers to handle duplicate messages:
```python
def process_order(message):
    order_id = message['order_id']

    # Idempotency check — already processed?
    if db.exists(f"processed:order:{order_id}"):
        return  # Skip duplicate

    # Process the order
    payment.charge(message)
    inventory.reserve(message)

    # Mark as processed
    db.set(f"processed:order:{order_id}", True, ex=86400)
```

### Kafka vs SQS/RabbitMQ — Key Differences

**Apache Kafka:**
```
- Distributed, append-only log (not a traditional queue)
- Messages are retained for a configurable period (hours to forever)
- Consumers can replay messages from any point in history
- Extremely high throughput (millions of messages/second)
- Multiple consumer groups — each gets its own independent view of the log
- Pull-based — consumers fetch messages at their own pace
- Use when: event streaming, audit logs, real-time analytics, event sourcing
```

**AWS SQS (Simple Queue Service):**
```
- Managed queue — no infrastructure to operate
- Messages deleted after successful consumption
- Dead Letter Queue (DLQ) for failed messages
- Standard: at-least-once, out-of-order | FIFO: exactly-once, in-order (lower throughput)
- Push to Lambda, or poll with workers
- Use when: decoupling services, task queues, async processing
```

**RabbitMQ:**
```
- Traditional message broker — rich routing capabilities
- Exchange types: direct, fanout, topic, headers
- Messages can be routed to different queues based on routing keys
- Push-based — broker pushes to consumers
- Use when: complex routing rules, request/reply patterns, task distribution
```

### Event-Driven Architecture Patterns

**Fan-out (Pub/Sub):**
One event delivered to **all** subscribers.
```
"UserRegistered" event
    → WelcomeEmailConsumer
    → OnboardingConsumer
    → AnalyticsConsumer
    → FreemiumSetupConsumer
(Everyone gets it — broadcast)
```

**Work Queue (Competing Consumers):**
Multiple workers compete to process messages from the same queue. Each message goes to exactly one worker.
```
10,000 image processing jobs in queue
    → Worker 1 takes job 1
    → Worker 2 takes job 2
    → Worker 3 takes job 3
    (Scale workers up/down based on queue depth)
```

**Request/Reply:**
Synchronous-style communication over async messaging. Requester publishes a message with a reply-to address and waits for a response.

**Event Sourcing:**
Store all state changes as a sequence of events. Current state is derived by replaying events.
```
UserCreated → EmailVerified → ProfileUpdated → PlanUpgraded → AccountClosed
(The event log IS the source of truth — current state is computed from it)
```

### Dead Letter Queues (DLQ)

When a message fails processing repeatedly, it's moved to a **Dead Letter Queue** (DLQ) so it doesn't block the main queue.

```
Main Queue → Consumer fails 3 times → DLQ
                                        ↓
                                  Alert DevOps team
                                  Inspect failed messages
                                  Fix the bug
                                  Replay messages back to main queue
```

Always configure DLQs in production. Without them, failing messages are lost silently.

---

## 8. Databases in System Design

Choosing the right database and structuring data correctly is a central system design skill.

### Read-Heavy vs Write-Heavy Workloads

**Read-Heavy (most web applications):**
- Add read replicas to distribute read load
- Add caching layer (Redis) in front of database
- Optimize with indexes for read query patterns

**Write-Heavy:**
- Consider write-optimized databases (Cassandra writes to memory first)
- Batch writes where possible
- Consider write sharding
- Use async writes (queue → batch → database)

### OLTP vs OLAP

**OLTP (Online Transaction Processing):**
- Many concurrent users making small, fast transactions
- INSERT, UPDATE, DELETE individual rows
- Optimized for low latency per query
- Example: your production web app database
- Database: PostgreSQL, MySQL

**OLAP (Online Analytical Processing):**
- Few users running complex queries on large datasets
- Aggregate queries scanning millions of rows (SUM, GROUP BY, JOIN)
- Optimized for throughput on large scans
- Example: business intelligence, data warehouse
- Database: Redshift, BigQuery, Snowflake, ClickHouse

**Never run heavy analytics on your OLTP database.** It starves the production app of resources. Use a separate read replica or a dedicated analytics database (data warehouse).

### The Read Replica Pattern

```
Application ──── Writes ────→ Primary PostgreSQL
                                     │ Async replication
                         ┌───────────┴───────────┐
                         ↓                       ↓
                    Replica 1               Replica 2
                         ↑                       ↑
                         └──────── Reads ─────────┘
                              (for most queries)
```

Route reads to replicas using your application's DB connection logic or a proxy (PgBouncer, ProxySQL).

### Database Selection Guide

| If you need... | Use |
|---|---|
| General purpose relational + advanced SQL | PostgreSQL |
| Legacy web app compatibility | MySQL |
| Simple, serverless, embedded | SQLite |
| Massive write scale, IoT | Cassandra |
| Fast key lookups, caching | Redis |
| Flexible document schema | MongoDB |
| Full-text search | Elasticsearch |
| Time series metrics | InfluxDB or TimescaleDB |
| Graph traversal | Neo4j |
| Petabyte analytics | BigQuery, Redshift, Snowflake |
| Globally distributed SQL | CockroachDB, Spanner |

---

## 9. Single Point of Failure — SPOF

A **Single Point of Failure (SPOF)** is any component whose failure causes the entire system to fail.

Eliminating SPOFs is the primary goal of high availability design.

### Common SPOFs and Their Solutions

**Single server:**
```
SPOF:    App → Server → DB
Fix:     App → LB → [Server1, Server2] → [Primary DB + Replica]
```

**Single load balancer:**
```
SPOF:    Internet → Single LB → Servers
Fix:     Internet → DNS (round-robin) → [LB1, LB2] → Servers
         Or: Use cloud-managed LBs which are inherently HA (AWS ALB)
```

**Single database:**
```
SPOF:    App → Single DB (if it dies, everything dies)
Fix:     App → Primary DB (sync replication) → Standby DB
         Auto-failover via Patroni, RDS Multi-AZ
```

**Single availability zone:**
```
SPOF:    All servers in us-east-1a (entire AZ goes down → outage)
Fix:     Spread across us-east-1a, us-east-1b, us-east-1c
         Use AZ-aware load balancing and auto-scaling
```

**Single region:**
```
SPOF:    All infrastructure in us-east-1 (region outage = global outage)
Fix:     Multi-region deployment with global load balancing
         Active-active or active-passive cross-region
```

**Single DNS provider:**
```
SPOF:    All DNS records at one provider (DDoS against them = your site is unreachable)
Fix:     Use multiple DNS providers or a DNS provider with Anycast + DDoS protection
```

**Single third-party dependency:**
```
SPOF:    Payment processor goes down → can't take payments
Fix:     Circuit breaker, fallback to secondary processor, graceful degradation
```

### How to Identify SPOFs

Draw your architecture. For every component, ask:
> "If this component went down right now, what breaks?"

If the answer is "the whole system," it's a SPOF. Add redundancy.

---

## 10. High Availability & Fault Tolerance

### Availability — The Nines

Availability is typically expressed in "nines":

| Availability | Downtime per year | Downtime per month | What it means |
|---|---|---|---|
| 99% (2 nines) | 3.65 days | 7.3 hours | Basic — probably not good enough for production |
| 99.9% (3 nines) | 8.76 hours | 43.8 minutes | Standard — acceptable for most internal tools |
| 99.95% | 4.38 hours | 21.9 minutes | — |
| 99.99% (4 nines) | 52.6 minutes | 4.4 minutes | High — most SaaS products aim here |
| 99.999% (5 nines) | 5.26 minutes | 26 seconds | Very high — telecoms, banking |
| 99.9999% (6 nines) | 31.5 seconds | 2.6 seconds | Extreme — almost impossible to sustain |

**Going from 3 nines to 4 nines is not 10% harder — it's orders of magnitude harder** (and more expensive). Every additional nine requires massive redundancy, automated failover, chaos engineering, and rigorous operational processes.

### Redundancy Patterns

**Active-Active:**
Multiple instances all handling production traffic simultaneously.
```
            ┌── Server 1 (handling traffic)
LB ─────────┼── Server 2 (handling traffic)
            └── Server 3 (handling traffic)
```
If one fails → load redistributes to remaining. Zero failover time.
All capacity is used, but you still need enough headroom if one fails (N+1 or N+2).

**Active-Passive:**
One instance handles traffic. Others are standby, ready to take over.
```
LB → Primary Server (active)
     Standby Server (passive — ready to activate)
```
If primary fails → standby activates. Small failover delay (seconds to minutes).
Standby capacity is "wasted" during normal operation but reduces cost compared to active-active if instances are expensive.

**N+1 Redundancy:**
Run N instances to handle the load, plus 1 extra. If one fails, you still have N instances (no degradation).

**Geographic Redundancy:**
```
Users in US   → us-east-1 region (primary)
Users in EU   → eu-west-1 region (replica)
If us-east-1 fails → all traffic routes to eu-west-1
```

### Graceful Degradation

When a component fails, the system continues operating in a reduced capacity rather than failing completely.

```
Normal:     User sees personalized homepage with recommendations
            ↓ Recommendations service fails
Degraded:   User sees generic homepage (recommendations omitted)
            ↓ Also, user DB is struggling
Degraded:   User sees static cached homepage
            (system still works — just without full features)
```

Design each feature as independent, optional enrichment rather than a hard dependency.

### Chaos Engineering

Intentionally injecting failures into production (or staging) to verify the system handles them correctly.

**Chaos Monkey** (Netflix): Randomly terminates production instances to ensure services recover automatically.

```bash
# Simulate a server failure
# On a random app server:
sudo systemctl stop myapp

# Or in Kubernetes:
kubectl delete pod $(kubectl get pods -l app=myapp -o name | shuf -n 1)

# Observe: Does traffic shift to healthy pods? Do alerts fire?
# Did any requests fail or was there a graceful failover?
```

---

## 11. CDN — Content Delivery Networks

A **CDN** is a globally distributed network of servers that caches and serves content from locations geographically close to the user.

### How a CDN Works

```
Without CDN:
User in Mumbai → Origin Server in US-East → 200ms latency

With CDN:
User in Mumbai → CDN Edge in Mumbai → 10ms latency
                 (content cached at the edge)
                 Only on first request (cache miss):
                 CDN Edge in Mumbai → Origin Server in US-East → caches response
```

### What CDNs Cache

**Static assets (always cache):**
- Images, CSS, JavaScript, fonts
- Videos
- PDF documents
- Any file that rarely changes

**Dynamic content (cache carefully):**
- API responses (with short TTL)
- HTML for anonymous users
- Personalized content usually should NOT be cached at CDN

### CDN Cache Control

You control what CDNs cache via HTTP headers:

```
# Cache forever (use content-addressed filenames like main.a3f5c9.js)
Cache-Control: public, max-age=31536000, immutable

# Cache for 1 hour, CDN can reuse for all users
Cache-Control: public, max-age=3600, s-maxage=86400

# Don't cache at CDN, but browser can cache privately
Cache-Control: private, max-age=3600

# Never cache (e.g., user dashboard)
Cache-Control: no-store
```

### CDN Benefits Beyond Speed

- **DDoS mitigation:** CDN edges absorb attack traffic before it reaches your origin
- **Origin offload:** 90%+ of requests served from cache → your servers handle only cache misses
- **SSL termination:** CDN handles TLS, reducing load on your origin
- **Edge computing:** Run logic at the CDN edge (Cloudflare Workers, Lambda@Edge)

### CDN Providers

| Provider | Strengths |
|---|---|
| **Cloudflare** | DDoS protection, edge computing, free tier, DNS |
| **AWS CloudFront** | Deep AWS integration, Lambda@Edge |
| **Fastly** | Real-time purging, highly configurable, streaming |
| **Akamai** | Largest network, enterprise, complex pricing |

---

## 12. API Design & API Gateway

### REST API Design Principles

You covered HTTP in the networking document. Here is how to design REST APIs well.

**Resource-based URLs:**
```
# Bad — verb in URL
GET /getUser?id=1001
POST /createOrder
DELETE /deleteProduct?id=5

# Good — nouns, actions expressed via HTTP methods
GET    /users/1001          # Get user
POST   /orders              # Create order
DELETE /products/5          # Delete product
GET    /users/1001/orders   # Get orders for user 1001
```

**HTTP methods map to CRUD:**
```
GET    → Read (safe, idempotent)
POST   → Create (not idempotent — calling twice creates two records)
PUT    → Replace entirely (idempotent)
PATCH  → Partial update (idempotent if designed correctly)
DELETE → Delete (idempotent)
```

**Consistent response format:**
```json
// Success
{
    "data": { "id": 1001, "name": "Alice" },
    "meta": { "request_id": "abc123", "timestamp": "2024-01-15T10:00:00Z" }
}

// Error
{
    "error": {
        "code": "USER_NOT_FOUND",
        "message": "User with id 1001 not found",
        "request_id": "abc123"
    }
}
```

**Versioning:**
```
# URL versioning (most common)
/api/v1/users
/api/v2/users

# Header versioning
Accept: application/vnd.myapi.v2+json
```

**Pagination:**
```json
GET /orders?page=2&per_page=20
{
    "data": [...],
    "pagination": {
        "page": 2,
        "per_page": 20,
        "total": 543,
        "next": "/orders?page=3&per_page=20",
        "prev": "/orders?page=1&per_page=20"
    }
}
```

### API Gateway

An **API Gateway** is a single entry point for all client requests. It sits in front of all your backend services.

```
                              ┌─────────────────┐
                              │   API Gateway   │
Mobile App  ──────────────────│                 │────→ User Service
Web Browser ──────────────────│  - Auth         │────→ Order Service
Third Party ──────────────────│  - Rate Limit   │────→ Product Service
                              │  - Routing      │────→ Payment Service
                              │  - Logging      │
                              │  - SSL Term.    │
                              └─────────────────┘
```

**What an API Gateway does:**
- **Authentication:** Verify JWTs or API keys before passing to backends
- **Rate limiting:** Enforce per-client limits
- **Request routing:** Route `/api/orders/*` to Order Service, `/api/users/*` to User Service
- **SSL termination:** Handle HTTPS at the gateway, pass HTTP to internal services
- **Request/response transformation:** Translate between protocols, reshape JSON
- **Logging and tracing:** Centralized request logging across all services
- **Circuit breaking:** Stop sending requests to unhealthy services
- **API composition:** Aggregate multiple backend calls into one response

**API Gateway examples:**
- **AWS API Gateway** — serverless, deep AWS integration
- **Kong** — open source, plugin ecosystem, high performance
- **nginx** — widely used as both load balancer and API gateway
- **Traefik** — Kubernetes-native, automatic service discovery
- **Envoy** — used as the data plane in service meshes (Istio, Consul Connect)

---

## 13. Service Discovery & Configuration

In a dynamic environment where servers come and go, services need to find each other automatically.

### The Problem

```
Static config (bad in dynamic environments):
OrderService config:
  payment_service_host: 10.0.0.5
  payment_service_port: 8080

But: Auto-scaling changes IPs. Rolling deployments create new IPs.
     If PaymentService moves, you must manually update OrderService config
```

### Service Discovery Solutions

**DNS-Based Service Discovery:**
Services register with DNS. Other services resolve by name.
```bash
payment-service.internal → 10.0.0.5, 10.0.0.6, 10.0.0.7  (round-robin DNS)
```
Kubernetes uses this pattern — every Service gets a DNS name automatically.

**Client-Side Discovery (with a Registry):**
Services register themselves in a registry (Consul, etcd, Eureka). Clients query the registry to find available instances.
```
PaymentService starts → registers "payment-service" at 10.0.0.5:8080 in Consul
OrderService needs PaymentService → queries Consul → gets 10.0.0.5:8080 → calls it directly
```

**Server-Side Discovery:**
Client sends request to a load balancer/router. The router handles service discovery.
```
OrderService → API Gateway → Gateway queries Consul → routes to PaymentService
(OrderService knows nothing about service discovery)
```

Kubernetes Services are a server-side discovery mechanism — your app talks to `payment-service:8080` and kube-proxy handles routing to the actual pods.

### Configuration Management

**The 12-Factor App** principle: Store config in environment variables, not code.

```bash
# Never in code:
DATABASE_URL = "postgresql://user:password@10.0.0.5:5432/mydb"

# In environment variables:
export DATABASE_URL="postgresql://user:password@10.0.0.5:5432/mydb"
```

**Configuration patterns:**

**Environment Variables:** Simple, universally supported. Best for simple values.

**Config files** (not checked into Git): `/etc/myapp/config.yaml` — loaded at startup.

**Consul / etcd:** Distributed key-value stores for dynamic configuration.
```bash
# Store config in Consul
consul kv put config/myapp/db_pool_size 20

# Retrieve in application
curl http://localhost:8500/v1/kv/config/myapp/db_pool_size
```

**AWS Systems Manager Parameter Store / Secrets Manager:** Cloud-native config and secrets management.

**Feature Flags:** Enable/disable features without deployment using a flag service (LaunchDarkly, Unleash, Flipt).

---

## 14. Rate Limiting & Throttling

**Rate limiting** controls how many requests a client can make in a given time window. Essential for:
- Preventing abuse and DDoS
- Enforcing fair usage (one user can't consume all server capacity)
- Protecting downstream services from overload
- Enforcing business rules (free tier: 100 API calls/day)

### Rate Limiting Algorithms

**Token Bucket:**
A bucket holds tokens (up to a max). Tokens are added at a fixed rate. Each request consumes one token. If the bucket is empty, the request is rejected.
```
Max capacity: 100 tokens
Refill rate: 10 tokens/second

10:00:00 - Bucket has 100 tokens
10:00:00 - 80 requests arrive → 80 tokens consumed, 20 remain
10:00:01 - 10 tokens added, 10 requests arrive → 20 remain
10:00:05 - 40 tokens added → 60 available
```
Allows bursting (use up to max capacity at once). Good for allowing occasional spikes.

**Leaky Bucket:**
Requests enter a queue (bucket). The queue drains at a fixed rate. If queue is full, request is dropped.
Smooths out traffic — constant output rate regardless of input bursts. Good for rate-limiting traffic to a downstream service.

**Fixed Window:**
Count requests in fixed time windows (e.g., 1-minute blocks).
```
10:00:00 - 10:00:59: 100 requests allowed
11:01:00 - 11:01:59: 100 requests allowed
```
Problem: A client can send 100 at 10:00:59 and 100 at 11:01:00 — 200 requests in 2 seconds (boundary attack).

**Sliding Window:**
More accurate — count requests in a rolling window. Prevents boundary attack.

### Implementing Rate Limiting in Redis

```python
import redis
import time

def is_rate_limited(user_id, limit=100, window=60):
    """
    Sliding window rate limiting with Redis.
    Returns True if request should be rejected.
    """
    now = time.time()
    key = f"rate_limit:{user_id}"

    pipe = redis.pipeline()
    # Remove requests older than the window
    pipe.zremrangebyscore(key, 0, now - window)
    # Count remaining requests
    pipe.zcard(key)
    # Add this request
    pipe.zadd(key, {str(now): now})
    # Set expiry on the key
    pipe.expire(key, window)
    results = pipe.execute()

    request_count = results[1]
    return request_count >= limit   # True = reject this request
```

### Rate Limit Response

When a client is rate limited, return:
```
HTTP 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1609459200
```

---

## 15. Observability — Logs, Metrics, Traces

**Observability** is the ability to understand what is happening inside a system from the outside — from its outputs. The three pillars are logs, metrics, and traces.

### The Three Pillars

#### Logs — "What happened?"
A timestamped record of discrete events.

```json
{
    "timestamp": "2024-01-15T10:00:01.234Z",
    "level": "ERROR",
    "service": "order-service",
    "trace_id": "abc123",
    "user_id": 1001,
    "message": "Failed to charge payment",
    "error": "Card declined",
    "order_id": 5678
}
```

**Structured logging** (JSON) is better than unstructured text — machines can parse and query it.

Use log levels appropriately:
- **DEBUG:** Detailed diagnostic info — for development only
- **INFO:** Normal operations — service started, order placed
- **WARN:** Something unexpected that didn't cause failure — retry succeeded
- **ERROR:** A failure that needs investigation — payment failed
- **FATAL/CRITICAL:** System is going down

#### Metrics — "How many times? How fast? How much?"
Numeric measurements collected over time. Types:
- **Counter:** Ever-increasing count (requests served, errors, orders placed)
- **Gauge:** A value that goes up and down (current connections, memory usage, queue depth)
- **Histogram:** Distribution of values (request duration buckets: <10ms, <50ms, <100ms, <500ms, >500ms)

```
http_requests_total{method="POST", endpoint="/orders", status="200"} 1523
http_request_duration_seconds_bucket{le="0.1"} 1200
http_request_duration_seconds_bucket{le="0.5"} 1500
http_request_duration_seconds_bucket{le="1.0"} 1520
active_db_connections{host="postgres-primary"} 45
```

Key metrics to track (the **RED method** for services):
- **Rate:** Requests per second
- **Errors:** Error rate (errors / total requests)
- **Duration:** Request latency (p50, p95, p99)

And the **USE method** for resources:
- **Utilization:** How busy the resource is (CPU %, disk %)
- **Saturation:** How much work is queued waiting (load average, disk I/O queue)
- **Errors:** Error events on the resource

#### Distributed Traces — "What happened end-to-end?"
A trace follows a single request as it moves through multiple services. Each unit of work is a **span**.

```
User request trace (trace_id: abc123):

→ API Gateway       [span: 0ms - 5ms]
  → OrderService    [span: 5ms - 50ms]
    → DB query      [span: 10ms - 40ms]   ← slow!
    → PaymentSvc    [span: 41ms - 80ms]
      → Stripe API  [span: 45ms - 78ms]
    → Queue publish [span: 81ms - 85ms]
Total: 85ms
```

Without distributed tracing, you see the user experienced 85ms — but you can't tell where the time went. With tracing, you can pinpoint the slow DB query.

**Tools:** Jaeger, Zipkin, AWS X-Ray, Datadog APM, Honeycomb, Tempo (Grafana)

### The Observability Stack

**Open Source (most common):**
```
Logs:    Fluentd/Fluent Bit → Elasticsearch → Kibana  (ELK Stack)
         or Promtail → Loki → Grafana
Metrics: Prometheus → Grafana
Traces:  OpenTelemetry → Jaeger or Tempo → Grafana
```

**Cloud Managed:**
```
AWS:   CloudWatch Logs + CloudWatch Metrics + X-Ray
GCP:   Cloud Logging + Cloud Monitoring + Cloud Trace
Azure: Log Analytics + Azure Monitor + Application Insights
```

**SaaS:**
```
Datadog, New Relic, Dynatrace, Honeycomb, Grafana Cloud
```

### Alerting

Alerts tell you when something is wrong before users report it.

**Good alerts:**
- Fire on symptoms (high error rate, high latency) — things users experience
- Have clear runbooks (what to do when this fires)
- Are actionable (if you can't do anything, don't alert)
- Have appropriate urgency (PagerDuty at 3am only for true emergencies)

**Bad alerts:**
- Alert on causes instead of symptoms (CPU > 80% — does it matter if users aren't affected?)
- Too many false positives → alert fatigue → team ignores all alerts

```yaml
# Prometheus alerting rule example
groups:
- name: api_alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) /
          rate(http_requests_total[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Error rate above 5%"
      runbook: "https://wiki.internal/runbooks/high-error-rate"
```

---

## 16. SLI, SLO, SLA — Reliability Targets

These are the formal framework for defining and measuring reliability. Google popularized them in the **Site Reliability Engineering** book. You will encounter these in any serious organization.

### SLI — Service Level Indicator

An **SLI** is a specific metric that measures some aspect of the level of service. It's a number.

```
SLI examples:
- HTTP success rate: (successful requests / total requests) × 100%
- API latency p99: 99th percentile response time = 250ms
- Availability: (time system was reachable / total time) × 100%
- Error rate: (errors / total requests) × 100%
- Throughput: requests per second = 5000 rps
```

Choose SLIs that reflect user experience — not internal system metrics.

### SLO — Service Level Objective

An **SLO** is the target value for an SLI. It's a goal you commit to internally.

```
SLO examples:
- Success rate SLO: ≥ 99.9% of requests succeed
- Latency SLO: p99 response time ≤ 500ms
- Availability SLO: ≥ 99.95% uptime measured monthly
```

**Error budget** = 100% - SLO target

If your availability SLO is 99.9%, your monthly error budget is:
- 0.1% of 30 days = 43.2 minutes of downtime per month
- If you've used 30 minutes already this month → only 13 minutes left → be very cautious with risky changes

**Error budget drives decision-making:**
- Error budget healthy → you can take risks (new features, big deployments)
- Error budget nearly exhausted → freeze risky changes, focus on reliability

### SLA — Service Level Agreement

An **SLA** is a contract with external customers (or internal with penalties). It's the legal commitment.

```
SLA example (AWS S3):
"AWS will use commercially reasonable efforts to make Amazon S3
available with a Monthly Uptime Percentage of at least 99.9% during any monthly billing cycle."

If S3 fails to meet 99.9%:
- 99.0% to < 99.9% → 10% service credit
- 95.0% to < 99.0% → 25% service credit
- < 95.0% → 100% service credit
```

**The hierarchy:**
```
SLO (internal goal) must be STRICTER than SLA (customer commitment)
You need a buffer — if SLO = 99.95% and SLA = 99.9%,
you have 0.05% buffer before you start breaking your customer commitment.
```

### How to Set SLOs

1. Measure current SLIs (you can't set targets without baseline data)
2. Consider what users actually need (what makes them unhappy?)
3. Start with something achievable (99.9% is better to hit consistently than 99.99% you always miss)
4. Tighten over time as your systems mature

---

## 17. Disaster Recovery

**Disaster Recovery (DR)** is your plan for restoring service after a catastrophic failure — data center fire, region outage, ransomware attack, massive accidental deletion.

### Key Metrics

**RTO (Recovery Time Objective):** Maximum acceptable time to restore service after a disaster.
**RPO (Recovery Point Objective):** Maximum acceptable data loss — how old can the data be when you recover?

```
Disaster strikes at 10:00 AM
Last backup at 9:00 AM

RPO = 1 hour (you're willing to lose 1 hour of data)
RTO = 4 hours (you'll be back online by 2:00 PM)
```

Lower RTO and RPO = more expensive solution. Choose based on business impact:
- Losing 1 day of orders from a $10M/day business = $10M of RTO cost justification
- Losing 1 hour of blog posts = not worth spending $100K/year on real-time replication

### DR Strategies (Cheapest to Most Expensive)

**Backup and Restore:**
- RTO: Hours to days | RPO: Hours
- Take regular backups, store offsite, restore when disaster strikes
- Cheapest — no standby infrastructure
- Acceptable for: dev/test, non-critical systems, RPO/RTO in hours is OK

**Pilot Light:**
- RTO: 10-30 minutes | RPO: Minutes (continuous replication)
- Minimal DR infrastructure always running (just DB replication to another region)
- On disaster: quickly provision app servers in DR region, cut over DNS
- Cost: just the DR database + storage

```
Primary Region (active):          DR Region (standby):
App Servers (4×)                  No app servers (provision on demand)
Primary DB ──── continuous ────→ DR DB Replica
(Full traffic)                    (Ready to promote)
```

**Warm Standby:**
- RTO: Minutes | RPO: Seconds
- Scaled-down version of production always running in DR region
- On disaster: scale up DR environment, cut over DNS

**Active-Active (Multi-Region):**
- RTO: Near zero | RPO: Near zero
- Both regions serve traffic simultaneously
- On region failure: remove failed region from DNS rotation
- Most expensive — full production capacity in 2+ regions

```
Users in US ──→ us-east-1 (50% traffic)
Users in EU ──→ eu-west-1 (50% traffic)
               Both regions have full production infrastructure
               Global load balancer (Route 53, Cloudflare) routes by geography
               Databases replicated bidirectionally (conflict resolution needed)
```

### DR Testing

A DR plan that has never been tested is not a DR plan.

- **Tabletop exercise:** Walk through the runbook on paper without executing it
- **Data restore test:** Actually restore from backups to a test environment monthly
- **Failover drill:** Manually trigger failover to DR environment (scheduled maintenance window)
- **Chaos engineering:** Randomly trigger failures in production to test automatic recovery

```bash
# Example: Monthly DR drill
# 1. Stand up environment in DR region from Terraform
terraform apply -var="region=eu-west-1"

# 2. Restore from backup
pgbackrest --stanza=mydb restore --target-timeline=latest

# 3. Run smoke tests against DR environment
./run_smoke_tests.sh https://dr.example.com

# 4. Measure: how long did it take? Was RTO met?
# 5. Document gaps and fix them
```

---

## 18. Designing for Failure — Resilience Patterns

Systems don't fail all at once — they degrade. Designing for failure means making sure degradation is graceful.

### Circuit Breaker Pattern

Named after electrical circuit breakers. When calls to a service start failing, "open" the circuit to stop making calls — fail fast instead of waiting for timeouts.

```
States:
CLOSED (normal):    Requests flow through. Failure count tracked.
OPEN (failing):     Requests fail immediately (no waiting). Error returned fast.
HALF-OPEN (testing): Allow a few requests through. If they succeed → CLOSED.
                      If they fail → back to OPEN.

┌──────────┐  failure threshold     ┌──────────┐
│  CLOSED  │ ──────────────────────→│   OPEN   │
│ (normal) │                        │ (failing)│
└──────────┘                        └──────────┘
     ↑                                   │
     │         timeout expires           │
     │ success ┌────────────┐ ←──────────┘
     └─────────│ HALF-OPEN  │
               └────────────┘
```

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.state = "CLOSED"
        self.last_failure_time = None
        self.timeout = timeout

    def call(self, func, *args):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF-OPEN"
            else:
                raise Exception("Circuit is OPEN — failing fast")

        try:
            result = func(*args)
            if self.state == "HALF-OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

Libraries: **Resilience4j** (Java), **Polly** (.NET), **pybreaker** (Python), **Hystrix** (Netflix, now deprecated in favor of Resilience4j).

### Retry Pattern

Automatically retry failed requests — with backoff.

**Naive retry (bad):**
```
Request fails → retry immediately → retry immediately → retry immediately
If 10,000 clients all do this → thundering herd → server collapses further
```

**Exponential backoff with jitter (good):**
```python
import time
import random

def retry_with_backoff(func, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            # Exponential backoff: 1s, 2s, 4s, 8s, 16s
            delay = base_delay * (2 ** attempt)
            # Add jitter: ±25% random variation
            jitter = random.uniform(0.75, 1.25)
            time.sleep(delay * jitter)
```

**When NOT to retry:** Don't retry non-idempotent operations (POST creating a resource) unless you have idempotency keys. Retrying "create payment" twice charges the customer twice.

### Timeout Pattern

Every external call must have a timeout. A slow dependency must not make your service slow or exhaust your connection pool.

```python
import requests

# Bad — no timeout (can hang forever)
response = requests.get("http://payment-service/charge")

# Good — explicit timeout
try:
    response = requests.get(
        "http://payment-service/charge",
        timeout=(3.05, 10)  # (connect timeout, read timeout) in seconds
    )
except requests.Timeout:
    # Handle timeout gracefully
    raise ServiceUnavailableError("Payment service timed out")
```

Set timeouts based on your SLO. If your API latency SLO is p99 < 500ms, your service's timeout for downstream calls should be much less (e.g., 200ms) to leave room for your own processing time.

### Bulkhead Pattern

Isolate failures so one failing component doesn't exhaust resources for everything else.

Named after ship bulkheads — compartments that prevent one flooded section from sinking the whole ship.

```
Without bulkhead:
All traffic shares one connection pool:
[Normal Requests]──┐
[Slow Requests]────┼──→ [10 DB connections] ← slow requests fill all connections
[Critical Paths]───┘     Normal + critical paths are starved

With bulkhead:
[Normal Requests]──→ [6 DB connections]
[Slow Reports]─────→ [2 DB connections]  ← slow reports can't affect normal requests
[Critical Paths]───→ [2 DB connections]  ← always have dedicated capacity
```

In practice: separate thread pools, connection pools, or queues per category of work.

### Idempotency

An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

```
GET /users/1001        → Always idempotent (read)
DELETE /users/1001     → Idempotent (delete twice = same result: user deleted)
PUT /users/1001 {...}  → Idempotent (replace twice = same result)
POST /orders           → NOT idempotent by default (creates a new order each time)
```

**Making POST idempotent with idempotency keys:**
```
POST /orders
Idempotency-Key: client-generated-uuid-abc123

Server logic:
  - Check if key "abc123" was already processed
  - If yes → return the same response as before (don't create another order)
  - If no → process, store result with key, return result

Client can safely retry the POST with the same key — guaranteed at-most-one order created
```

Stripe, AWS, and most modern payment APIs implement idempotency keys.

---

## 19. Capacity Planning & Back-of-Envelope Estimation

Knowing how to estimate system requirements quickly is a core system design skill — both for real planning and in interviews.

### Key Numbers to Know

**Latency numbers every engineer should memorize:**

| Operation | Approximate Time |
|---|---|
| L1 cache reference | 1 ns |
| L2 cache reference | 4 ns |
| RAM access | 100 ns |
| SSD read (4KB) | 150 μs (150,000 ns) |
| Read 1MB from RAM | 250 μs |
| Network round-trip in same datacenter | 500 μs |
| Read 1MB from SSD | 1 ms |
| HDD seek | 10 ms |
| Network round-trip US coast to coast | 40 ms |
| Network round-trip US to Europe | 80 ms |
| Read 1MB from HDD | 20 ms |

Key insight: **RAM is 1000× faster than SSD, SSD is 100× faster than HDD.** Network in the same datacenter is faster than disk.

**Useful multipliers:**
```
1 KB = 10^3 bytes
1 MB = 10^6 bytes
1 GB = 10^9 bytes
1 TB = 10^12 bytes

1 million = 10^6
1 billion = 10^9

Seconds in a day: 86,400 ≈ 10^5
Seconds in a year: ≈ 3.2 × 10^7 ≈ 31.5 million
```

### Example: Capacity Planning for a URL Shortener

**Requirements:**
- 100 million new URLs shortened per day
- 10× more reads than writes (1 billion reads per day)
- Stored for 5 years

**Step 1: Request rates**
```
Writes: 100M / day = 100M / 86,400s ≈ 1,200 writes/second
Reads:  1B / day  = 1B / 86,400s   ≈ 11,600 reads/second
Peak is typically 2-3× average, so design for:
  Writes: 3,600/s
  Reads: 35,000/s
```

**Step 2: Storage**
```
Each URL mapping: ~500 bytes (original URL 200B + short code 10B + metadata)
Per day: 100M × 500B = 50 GB/day
Per year: 50GB × 365 = ~18 TB/year
5 years: ~90 TB total
```

**Step 3: Bandwidth**
```
Read response: ~500 bytes
Reads: 35,000/s × 500B = 17.5 MB/s = 140 Mbps   (very manageable)
```

**Step 4: Servers needed**
```
Assume each app server handles 5,000 requests/second (stateless, in-memory cache)
35,000 reads/second → 7 servers minimum
Add 100% buffer for HA → 14 app servers
Use auto-scaling: min=4, max=20
```

**Step 5: Caching**
```
80% of reads go to 20% of URLs (Pareto principle)
Cache the hot 20% in Redis
20% of total URLs = 100M × 5 years × 20% = 100M URLs
100M × 500B = 50GB of Redis — doable on one Redis server (or small cluster)
Cache hit rate: 80% → only 20% of reads reach the database
DB reads: 35,000 × 20% = 7,000/s (manageable with a primary + 2 read replicas)
```

This is the kind of thinking that separates engineers who understand systems from those who just configure them.

---

## 20. Real-World Architecture Walkthroughs

### Architecture 1: Simple Web Application (Starting Point)

What most small applications look like:

```
Users
  ↓
[nginx] (reverse proxy + static file serving)
  ↓
[Application Server] (Django/Rails/Node.js)
  ↓
[PostgreSQL] (single instance)
```

Problems at scale: Single server SPOF, can't handle more traffic, DB is the bottleneck.

### Architecture 2: Scaled Web Application

After the first growing pains:

```
Users
  ↓
[CloudFlare CDN] (static assets, DDoS protection)
  ↓
[AWS ALB] (Application Load Balancer — managed, HA)
  ↓
[App Server Pool] (Auto Scaling Group: 2-10 instances)
  ↓
[Redis] (session storage + caching — ElastiCache)
  ↓
[PostgreSQL Primary] ─── replication ──→ [PostgreSQL Replica]
                                           (read queries)
  ↓
[S3] (file storage — images, uploads)
```

This handles 10-100× more traffic than Architecture 1.

### Architecture 3: High-Scale Microservices

After the business has grown significantly:

```
Users
  ↓
[Cloudflare] (global CDN + WAF + DDoS)
  ↓
[AWS Global Accelerator] (GeoDNS + Anycast)
  ↓              ↓
[US Region]     [EU Region]   (active-active multi-region)
     ↓
[AWS ALB]
     ↓
[API Gateway] (Kong / AWS API GW)
     │ authentication, rate limiting, routing
     ├─────────────────────────────────────────┐
     ↓                                         ↓
[User Service]                          [Order Service]
[Auth Service]                          [Payment Service]
[Product Service]                       [Notification Service]
     │                                         │
[Redis Cache]                         [Redis Cache]
     │                                         │
[User DB]                              [Orders DB]
(PostgreSQL)                          (PostgreSQL)
                                             │
                                      [Kafka] (event bus)
                                        ├── [Analytics Consumer]
                                        ├── [Email Consumer]
                                        └── [Search Indexer]
                                              ↓
                                       [Elasticsearch]
```

Each service:
- Deployed independently via CI/CD
- Has its own database (no shared databases)
- Communicates via REST API (sync) or Kafka (async)
- Observed via Prometheus + Grafana + Jaeger
- Scaled independently based on its own metrics

---

## 21. System Design Interview Framework

Even though you're a DevOps engineer, system design thinking helps you in infrastructure discussions, architecture reviews, and job interviews at senior levels.

### The RESHADED Framework

A structured approach to any system design question:

**R — Requirements Clarification**
Before designing anything, ask:
```
"How many users? Daily active vs total?"
"Read-heavy or write-heavy?"
"What consistency is required? (strong vs eventual)"
"What's the latency requirement?"
"Any specific regulations? (GDPR, HIPAA)"
"Do we need to design mobile clients too?"
```

**E — Estimation**
Back-of-envelope numbers:
```
QPS (Queries Per Second)
Storage requirements
Bandwidth requirements
Server count
```

**S — System Interface / API Design**
Define the APIs the system exposes.

**H — High-Level Design**
Draw the major components:
```
Client → CDN → Load Balancer → App Servers → Database
                    ↓
               Message Queue → Workers
```

**A — Detailed Component Design**
Go deep on each component. Database schema, API endpoints, caching strategy.

**D — Data Flow**
Walk through how data moves for the most important use cases.

**E — Explain Bottlenecks & Scale**
Identify where the system will fail under load. How do you fix each one?

**D — Deep Dives**
Interviewer picks a component to explore further — reliability, security, specific algorithm.

### Example: "Design a URL Shortener" (Brief Answer)

**Requirements:** 100M URLs/day, 1B reads/day, 5-year retention.

**Core design:**
- **API:** `POST /shorten {url}` → `{short_code}`, `GET /{code}` → `301 redirect`
- **Short code:** 6-character base62 (62^6 = 56 billion — enough for 5 years)
- **Database:** PostgreSQL — table: `{short_code, original_url, created_at, user_id}`
- **Caching:** Redis — cache hot short_code → URL mappings (80% cache hit rate target)
- **Read path:** Client → CDN (static 301 cached) → App → Redis → DB
- **Write path:** App → DB insert → cache invalidate
- **Scale:** 12K reads/s → 3 app servers + Redis + 1 DB + 1 read replica

---

## 22. Quick Reference Cheat Sheet

### Architecture Decision Framework
```
Monolith first → extract services when you have proof
Stateless apps → stateful data tier
Horizontal scale apps → vertical scale DBs (until you can't)
Cache reads → queue writes → partition when needed
Add redundancy at every tier to eliminate SPOFs
```

### Scaling Ladder
```
1. Optimize queries + add indexes
2. Add caching (Redis)
3. Add read replicas (for read-heavy)
4. Vertical scale the database
5. Add CDN for static assets
6. Horizontal scale app servers (ensure stateless)
7. Add message queue for async processing
8. Database partitioning / sharding
9. Microservices decomposition
10. Multi-region deployment
```

### When to Use What
```
REST API       → standard request/response between services
GraphQL        → flexible queries, multiple clients with different data needs
Message Queue  → async decoupling, work distribution, event fan-out
WebSocket      → real-time bidirectional (chat, live data)
gRPC           → high-performance service-to-service (protobuf, streaming)
Redis          → cache, session, rate limiting, pub/sub, leaderboards
Kafka          → event streaming, audit log, replay, analytics pipeline
Elasticsearch  → full-text search, log analysis
CDN            → static assets, global distribution, DDoS protection
```

### SLI/SLO/SLA
```
SLI = what you measure (success rate, p99 latency)
SLO = your internal target (99.9% success rate)
SLA = your customer commitment (99.5% with penalties)
Error budget = 100% - SLO target (spend wisely)
```

### Reliability Quick Math
```
99.9%   = 8.7 hrs downtime/year   = 43.8 min/month
99.95%  = 4.4 hrs downtime/year   = 21.9 min/month
99.99%  = 52.6 min downtime/year  = 4.4 min/month
99.999% = 5.3 min downtime/year   = 26 sec/month

Serial dependency: if A=99.9% and B=99.9%, A+B = 99.9% × 99.9% = 99.8%
(availability degrades as you chain dependencies!)
```

### Resilience Patterns
```
Circuit Breaker → fail fast when dependency is down
Retry + Backoff → retry with exponential delay + jitter
Timeout         → every external call must have a timeout
Bulkhead        → isolate failure domains, separate resource pools
Idempotency     → safe to retry without side effects
Graceful Degrade → partial service beats no service
```

---

## System Design Concepts Map

```
FUNDAMENTALS
  Scalability → handle more load
  Reliability → works when parts fail
  Availability → up when users need it
  Performance → fast under load

ARCHITECTURE
  Monolith vs Microservices → start monolith, extract when justified
  Stateless Apps + Stateful Data Tier → horizontal scale app, careful with DB
  Vertical vs Horizontal Scaling → vertical for DB, horizontal for app

DATA FLOW
  Synchronous (REST/gRPC) → direct service calls, immediate response
  Asynchronous (Queue/Kafka) → decouple services, absorb load spikes

PERFORMANCE
  CDN → static assets, geographic distribution
  Caching → Redis at every layer, right TTL, right invalidation strategy
  Load Balancing → distribute traffic, remove SPOFs, enable horizontal scale
  Connection Pooling → PgBouncer, don't starve your database

RELIABILITY
  Eliminate SPOFs → redundancy at every tier
  Active-Active vs Active-Passive → based on RTO/cost tradeoff
  Graceful Degradation → partial service beats failure
  Chaos Engineering → test failure handling before it happens
  Backups + DR → 3-2-1 rule, test your backups, know your RTO/RPO

RESILIENCE PATTERNS
  Circuit Breaker → stop hammering failing services
  Retry + Backoff → retry carefully, with exponential backoff + jitter
  Timeout → always set timeouts on external calls
  Idempotency → safe retries, idempotency keys for mutations

OBSERVABILITY
  Logs (what happened) + Metrics (how much) + Traces (end-to-end path)
  RED method → Rate, Errors, Duration (for services)
  USE method → Utilization, Saturation, Errors (for resources)
  SLI → SLO → SLA → Error Budget → Release decision

CAPACITY PLANNING
  Back-of-envelope estimation → QPS, storage, bandwidth, servers
  Know your latency numbers → RAM 100ns, SSD 150μs, Network 500μs
  Cache hit rate matters → 80% cache hit = 5× less DB load
  Design for peak, not average → 2-3× headroom minimum
```

---

*Document version 1.0 — System Design Fundamentals for DevOps Engineers. This is the capstone document — it connects everything from OS, Networking, Security, and Databases into how real systems are built and operated.*
