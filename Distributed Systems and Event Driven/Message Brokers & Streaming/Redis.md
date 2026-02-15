https://www.geeksforgeeks.org/system-design/introduction-to-redis-server/
https://www.digitalocean.com/community/tutorials/how-to-implement-caching-in-node-js-using-redis

https://docs.nestjs.com/microservices/redis
https://redis.io/

# 🔴 Redis – Complete High-Level Guide (Node.js + System Design Perspective)

Redis = **In-memory data structure store**
Used for: caching, sessions, queues, rate limiting, real-time systems.

Official: Redis

---

# 1️⃣ Redis Data Types

## 🔹 1. String

* Most basic type
* Max 512MB per value
* Used for: cache, tokens, OTP

```bash
SET user:1 "Surya"
GET user:1
```

Used for:

* JWT storage
* OTP codes
* API response caching

---

## 🔹 2. Hash

Key-value pairs inside a key (like object)

```bash
HSET user:1 name "Surya" age 30
HGET user:1 name
```

Used for:

* User profile
* Config objects
* Metadata

---

## 🔹 3. List

Ordered collection (like array)

```bash
LPUSH tasks "job1"
RPOP tasks
```

Used for:

* Queues (basic)
* Background jobs
* Activity logs

---

## 🔹 4. Set

Unordered unique values

```bash
SADD users "101"
SISMEMBER users "101"
```

Used for:

* Unique visitors
* Permission groups
* Deduplication

---

## 🔹 5. Sorted Set (ZSET)

Like Set + score (ranking)

```bash
ZADD leaderboard 100 "user1"
ZRANGE leaderboard 0 -1
```

Used for:

* Leaderboards
* Ranking systems
* Rate limiting
* Priority queues

---

## 🔹 6. Streams

Append-only log (event streaming)

```bash
XADD mystream * field value
```

Used for:

* Event-driven systems
* Microservice messaging
* Kafka alternative (lightweight)

---

## 🔹 7. Bitmaps

Bit-level operations

Used for:

* Feature flags
* Daily login tracking
* Analytics

---

# 2️⃣ Redis Expiration & TTL

```bash
SET session:123 data EX 300
TTL session:123
```

## Used For:

* Session expiry
* OTP expiration
* Cache invalidation
* Temporary tokens

---

## 🔥 Real Examples

### Session

```
session:userId → TTL 30 min
```

### OTP

```
otp:userId → TTL 120 sec
```

### Cache

```
user_profile → TTL 5 min
```

---

# 3️⃣ Redis as Cache

Pattern:

```
1. Check Redis
2. If hit → return
3. If miss → DB → store in Redis
```

Best Practices:

* Always set TTL
* Avoid caching highly dynamic data
* Use cache invalidation strategy

---

# 4️⃣ Redis Pub/Sub

Real-time messaging

Publisher:

```bash
PUBLISH channel message
```

Subscriber:

```bash
SUBSCRIBE channel
```

Used for:

* Chat apps
* Notification systems
* WebSocket scaling

⚠ Not persistent (if subscriber offline → message lost)

---

# 5️⃣ Redis Streams (Reliable Messaging)

Unlike Pub/Sub:

* Persistent
* Consumer groups
* Message replay

Used for:

* Order processing
* Event sourcing
* Distributed microservices

---

# 6️⃣ Redis as Queue

Using:

* Lists
* Streams

Example:

```
LPUSH queue job
BRPOP queue
```

Used for:

* Background workers
* Email sending
* Payment processing

---

# 7️⃣ Redis Transactions

```bash
MULTI
SET key value
INCR counter
EXEC
```

Ensures:

* Atomic execution
* No interleaving commands

⚠ Not full ACID like DB

---

# 8️⃣ Redis Lua Scripting

```bash
EVAL "return redis.call('GET', KEYS[1])" 1 key
```

Used for:

* Atomic operations
* Complex rate limiting
* Distributed locks

Why?
Lua executes atomically inside Redis.

---

# 9️⃣ Redis Locks (Distributed Lock)

Used in:

* Leader election
* Avoid duplicate processing

Basic:

```
SET lock:payment 1 NX EX 10
```

NX → only if not exists
EX → expiry

Prevents:

* Double payment execution

---

# 🔟 Redis Rate Limiting

Using:

* INCR + EXPIRE
* Sorted Set
* Lua scripts

Example:

```
User can call API 100 times per minute
```

Common in:

* Public APIs
* Fintech systems

---

# 1️⃣1️⃣ Redis Clustering

Redis Cluster:

* Sharding
* Horizontal scaling
* High availability

Architecture:

* Multiple master nodes
* Replica nodes

Used when:

* Large datasets
* High throughput systems

---

# 1️⃣2️⃣ Redis Filters

### Bloom Filter

* Check existence
* Avoid DB hits

Used for:

* Fraud detection
* Duplicate check

False positive possible.

---

# 1️⃣3️⃣ Redis Persistence

Two types:

### RDB

* Snapshot based
* Faster restart
* Less durable

### AOF

* Append only log
* More durable
* Slower

Production:

* Use both for safety

---

# 1️⃣4️⃣ Redis Client Libraries (Node.js)

Most used:

```
ioredis
redis (official)
```

Example:

```js
import Redis from 'ioredis';
const redis = new Redis();
```

---

# 1️⃣5️⃣ Redis Use Cases

| Use Case      | Redis Feature   |
| ------------- | --------------- |
| Session Store | Strings + TTL   |
| OTP           | String + Expire |
| Cache         | String + TTL    |
| Rate Limiting | Sorted Set      |
| Chat          | Pub/Sub         |
| Queue         | List / Streams  |
| Leaderboard   | Sorted Set      |
| Locks         | SET NX EX       |
| Feature Flags | Hash            |
| Analytics     | Bitmaps         |

---

# 🔥 Production Best Practices

## ✅ Always set TTL for cache

## ✅ Avoid large keys

## ✅ Monitor memory usage

## ✅ Use connection pooling

## ✅ Enable persistence if needed

## ✅ Use cluster for scaling

## ✅ Avoid blocking commands (KEYS)

## ✅ Use Lua for atomic complex logic

---

# 🧠 Interview-Ready Summary

If interviewer asks:

> “How have you used Redis?”

You can say:

> I’ve used Redis as a distributed cache, session store, rate limiter, and message queue. I leverage TTL for expiration, Lua scripting for atomic operations, sorted sets for rate limiting, and Redis locks to prevent duplicate processing in distributed systems. In production, I prefer clustered Redis with AOF persistence and proper memory monitoring.

---

---

# 🔴 First: “Redis is in RAM — Which RAM?”

Redis runs inside a server (VM, bare metal, or container).

It uses:

> The **RAM of the machine where Redis is installed**

Example:

```
AWS EC2 instance
16GB RAM
Redis configured to use 12GB
```

Redis stores data inside that RAM.

---

# ❓ But RAM data disappears after restart?

Correct. By default, RAM is volatile.

That’s why Redis provides **Persistence** mechanisms.

---

# 🔐 Redis Persistence Options

## 1️⃣ RDB (Snapshot)

* Saves snapshot periodically
* Example: every 5 minutes
* Faster recovery
* May lose few minutes of data

## 2️⃣ AOF (Append Only File)

* Logs every write operation
* More durable
* Slower but safer

## 3️⃣ Production Best Practice

Use:

```
RDB + AOF together
```

For critical systems like BFSI.

---

# 🎯 Why Use Redis in Real-Time Applications?

Because:

| Feature                | Why It Matters      |
| ---------------------- | ------------------- |
| In-memory              | Microsecond latency |
| Simple data structures | Easy modeling       |
| Atomic operations      | Safe concurrency    |
| TTL support            | Auto expiry         |
| Pub/Sub                | Real-time updates   |
| High throughput        | 100k+ ops/sec       |

---

# 🏦 Redis in BFSI Payment Architecture

Imagine a payment system:

User → Payment API → Risk Engine → Bank → Ledger

Where Redis fits:

---

## 1️⃣ Idempotency

```
SET payment:txnId result NX EX 300
```

Prevents duplicate charges.

---

## 2️⃣ Rate Limiting

Prevent fraud or API abuse.

```
User → Max 5 payment attempts per minute
```

---

## 3️⃣ Distributed Lock

Settlement batch job:

```
Only one node runs settlement
```

---

## 4️⃣ OTP Expiry

```
otp:userId → TTL 120 sec
```

---

## 5️⃣ Session Store

For internal dashboards.

---

## 6️⃣ Risk Scoring Cache

Risk engine frequently checks:

* Blacklist
* Fraud rules

Cache reduces DB pressure.

---

# 🛒 Redis in E-commerce

* Cart storage
* Flash sale inventory locking
* Product page caching
* Real-time stock count
* Session store

Example:

```
DECR stock:product123
```

Atomic inventory deduction.

---

# 🔥 Rate Limiting Deep Dive (Node.js Code)

### Sliding Window using Sorted Set

```js
import Redis from "ioredis";
const redis = new Redis();

async function rateLimit(userId) {
  const key = `rate:${userId}`;
  const now = Date.now();
  const window = 60000; // 1 min
  const limit = 5;

  await redis.zadd(key, now, now);
  await redis.zremrangebyscore(key, 0, now - window);
  const count = await redis.zcard(key);
  await redis.expire(key, 60);

  return count <= limit;
}
```

Used in:

* Login attempts
* Payment retry limit
* Public APIs

---

# 🧠 Redis vs Kafka vs RabbitMQ

| Feature        | Redis               | Kafka                     | RabbitMQ       |
| -------------- | ------------------- | ------------------------- | -------------- |
| Speed          | Very fast           | Fast                      | Medium         |
| Persistence    | Optional            | Strong                    | Strong         |
| Ordering       | Basic               | Strong partition ordering | Yes            |
| Message Replay | Limited             | Yes                       | Limited        |
| Pub/Sub        | Yes                 | Yes                       | Yes            |
| Streaming      | Basic               | Enterprise-grade          | Moderate       |
| Best For       | Cache + light queue | Event streaming           | Reliable queue |

---

## When to Use What?

* Redis → Cache, rate limit, lightweight queue
* Kafka → Large-scale event streaming
* RabbitMQ → Complex routing queue

---

# ⚙ Complete Node.js Redis Production Blueprint

## Architecture

```
Client
   ↓
Load Balancer
   ↓
Node.js API
   ↓
Redis Cluster
   ↓
Postgres / DB
```

---

## Best Practices

### 1️⃣ Use Redis Cluster in Production

* Sharding
* High availability
* Replicas

---

### 2️⃣ Always Set Memory Limit

```
maxmemory 12gb
maxmemory-policy allkeys-lru
```

Eviction strategy:

* LRU (least recently used)

---

### 3️⃣ Use Connection Pooling

Don’t create new Redis connection per request.

---

### 4️⃣ Use Lua for Atomic Logic

Example:

* Rate limiting
* Wallet debit
* Lock acquisition

---

### 5️⃣ Avoid Blocking Commands

❌ KEYS *
Use:

```
SCAN
```

---

### 6️⃣ Monitor

Use:

* Redis Insight
* Prometheus
* CloudWatch

Monitor:

* Memory
* Hit ratio
* Evictions
* CPU

---

# 🎯 Senior-Level Interview Answer

If interviewer asks:

> Why Redis in real-time systems?

You answer:

> Redis provides ultra-low latency access because it operates in-memory. It supports atomic operations, TTL-based expiration, distributed locking, and rate limiting. In BFSI systems, we use Redis for idempotency control, fraud prevention rate limiting, OTP expiry, and distributed job locking. In e-commerce, it helps with inventory management, cart storage, and flash sale handling. It reduces database load significantly while maintaining high throughput.

---

# 🚀 Final Concept Clarity

Redis is not a replacement for database.

It is:

> A performance accelerator + coordination engine + real-time processor.

Database = source of truth
Redis = speed layer

---

Perfect 👌 this is **senior-level distributed systems knowledge**.

Let’s break down **Redis Cluster Internals** in a clean, system-design way.

---

# 🔴 What is Redis Cluster?

Redis Cluster is Redis’s built-in mechanism for:

* Horizontal scaling (sharding)
* High availability
* Fault tolerance
* Automatic failover

It removes the need for manual sharding.

---

# 🧠 1️⃣ Core Concept: Hash Slots (16384 Slots)

Redis Cluster does **not** shard by key range directly.

Instead, it divides the keyspace into:

> **16384 hash slots**

When you store a key:

```
hashSlot = CRC16(key) % 16384
```

Each slot is assigned to a node.

---

## Example

Suppose we have 3 master nodes:

| Node   | Slot Range    |
| ------ | ------------- |
| Node A | 0 – 5000      |
| Node B | 5001 – 10000  |
| Node C | 10001 – 16383 |

If:

```
SET user:1
```

Redis calculates slot and routes to correct node.

---

# 🔄 2️⃣ Sharding (Data Distribution)

Each master node owns some slots.

So:

```
user:1 → slot 4321 → Node A
cart:10 → slot 9000 → Node B
order:22 → slot 15000 → Node C
```

This gives:

* Automatic horizontal scaling
* Data distributed across cluster

---

# 🔁 3️⃣ Replication (High Availability)

Each master node can have replicas.

Example:

```
Master A → Replica A1
Master B → Replica B1
Master C → Replica C1
```

If Master A fails:

* Replica A1 is promoted to master
* Slots remain intact

---

# 🧠 4️⃣ Cluster Bus (Node Communication)

Redis nodes talk to each other using:

> Cluster bus protocol (separate TCP port)

Used for:

* Heartbeats (PING/PONG)
* Failure detection
* Slot ownership updates
* Failover coordination

---

# ⚡ 5️⃣ Failure Detection & Automatic Failover

Every node periodically:

* Pings other nodes
* If no response → marks as PFAIL
* If majority agrees → marks as FAIL

Then:

Replica election happens.

Replica with highest replication offset becomes new master.

This prevents split brain.

---

# 🔄 6️⃣ Redirection (Client Perspective)

If client connects to wrong node:

Redis replies:

```
-MOVED 9000 192.168.1.10:6379
```

Client must reconnect to correct node.

Modern clients (like ioredis) handle this automatically.

---

# 🔒 7️⃣ Multi-Key Limitation

Because keys are sharded:

You cannot run multi-key commands across different slots.

❌ This will fail:

```
MSET key1 key2
```

If they belong to different slots.

---

# ✅ Solution: Hash Tags

Use `{}` to force keys into same slot.

Example:

```
SET user:{100}:profile
SET user:{100}:cart
```

Everything inside `{}` is hashed.

Both go to same slot.

Critical for:

* Transactions
* Lua scripts
* Multi-key operations

---

# 🔥 8️⃣ Rebalancing

When adding new node:

* Cluster redistributes slots
* Data migrates gradually
* No full downtime

Used for scaling.

---

# 💾 9️⃣ Memory Management in Cluster

Each node:

* Has its own RAM
* Own eviction policy
* Own persistence config

Cluster is just a collection of independent Redis instances.

---

# 🔐 1️⃣0️⃣ Consistency Model

Redis Cluster is:

> Eventually consistent (replication async)

If master crashes before replica sync:

Some data loss possible.

For stronger durability:

* Use AOF
* Use WAIT command
* Or use Redis Enterprise

---

# 🏦 Redis Cluster in BFSI Architecture

Example:

Payment System:

```
Redis Cluster (6 masters + 6 replicas)
```

Used for:

* Idempotency keys
* Rate limiting
* Session storage
* OTP
* Risk cache

Design:

* Use hash tags for txn-based keys
* Use AOF persistence
* Use monitoring for slot imbalance
* Use multi-AZ deployment

---

# 🧠 Internal Flow Example (Payment Request)

User → API
→ Redis Cluster
→ Hash slot calculated
→ Correct master selected
→ Replica asynchronously updated
→ Response returned

Latency: <1 ms

---

# ⚠ Important Production Concerns

## 1️⃣ Don’t use KEYS

Use SCAN.

## 2️⃣ Avoid huge keys (>1MB)

## 3️⃣ Monitor:

* Memory
* Slot distribution
* Replication lag
* Failover frequency

## 4️⃣ Use Proper Client

Node.js:

```
ioredis (cluster mode)
```

---

# 🧠 Redis Cluster vs Sentinel

| Feature    | Cluster    | Sentinel |
| ---------- | ---------- | -------- |
| Sharding   | Yes        | No       |
| HA         | Yes        | Yes      |
| Scaling    | Horizontal | Vertical |
| Complexity | Higher     | Lower    |

Use:

* Sentinel → small systems
* Cluster → large scale

---

# 🎯 Interview-Ready Summary

If interviewer asks:

> Explain Redis Cluster internals.

You answer:

> Redis Cluster divides the keyspace into 16384 hash slots and distributes them across master nodes. Each master can have replicas for high availability. Nodes communicate using a cluster bus for heartbeat and failover detection. On failure, replicas are promoted automatically. Clients are redirected using MOVED responses. It provides horizontal scaling with eventual consistency.

---

# 🚀 Advanced Insight (Senior Level)

* Uses gossip protocol for node communication
* Failover requires quorum
* Supports online rebalancing
* Multi-key ops require hash tags
* Replication is asynchronous
* Cluster-aware clients are mandatory

---

