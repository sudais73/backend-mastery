# Week 4: Redis & Caching Mastery

## Overview

**Goal:** Master Redis as an in-memory data structure server. Understand when to use it, which data structures solve which problems, and how to avoid common pitfalls.

**Why it matters:** PostgreSQL is where your data lives; Redis is where your data goes to be fast. Every serious backend needs Redis.

---

## The Mental Model (Foundation)

### **Redis is a Whiteboard, Not a Filing Cabinet**

**PostgreSQL = Filing Cabinet**
```
┌─────────────────────────────┐
│ Authoritative records       │
│ (customers, orders, etc)    │
│ Durable, searchable         │
│ On disk, slower to access   │
└─────────────────────────────┘
```

**Redis = Whiteboard**
```
┌─────────────────────────────┐
│ Hot/temporary data          │
│ (cache, sessions, counters) │
│ Fast, in RAM                │
│ Gone if power fails         │
└─────────────────────────────┘
```

### **Key Differences**

| Aspect | PostgreSQL (Cabinet) | Redis (Whiteboard) |
|--------|----------------------|-------------------|
| **Speed** | Milliseconds | Microseconds |
| **Storage** | Disk | RAM |
| **Durability** | Permanent | Volatile (unless configured) |
| **Use Case** | System of record | Cache, sessions, counters |
| **Data Loss** | Never (ACID) | Acceptable (ephemeral data) |
| **Complexity** | SQL queries, joins | Simple key-value, data structures |

### **Why Redis is Fast**

1. **In-memory:** No disk I/O
2. **Single-threaded:** No lock contention, no context switching
3. **Specialized data structures:** Counter, list, set, sorted set — all optimized
4. **Atomic commands:** `INCR` is thread-safe by definition

### **What You Should NOT Do**

❌ Use Redis as your primary database
❌ Expect data to survive a crash (unless you configure persistence)
❌ Store data you can't afford to lose
❌ Treat it as a replacement for PostgreSQL

✅ Use Redis for:
- Caching
- Sessions
- Rate limiting
- Counters
- Leaderboards
- Pub/sub messaging
- Distributed locks
- Queues

---

## Core Data Types (Pick the Right Tool)

### **1. Strings (The Basic Building Block)**

**What it is:**
- Simple key-value: `key → "value"`
- Also used for counters, raw bytes, anything

**Commands:**
```redis
SET user:1:name "Alice"           # Set a string
GET user:1:name                   # Get it back → "Alice"
SET counter 10 EX 60              # With TTL (expires in 60 seconds)
GETSET counter 100                # Get old value, set new one
APPEND user:1:name " Smith"       # Append to string
```

**Use Cases:**
- Cache simple values
- Configuration
- Session data (if small)

**Example:**
```redis
# Cache a user profile
SET user:42 '{"name":"Alice","email":"alice@ex.com"}' EX 300
GET user:42
# Returns: '{"name":"Alice","email":"alice@ex.com"}'
```

---

### **2. Hashes (Objects/Maps)**

**What it is:**
- A hash map: `key → {field1: value1, field2: value2, ...}`
- Perfect for objects

**Commands:**
```redis
HSET session:abc user_id 42 role admin  # Set multiple fields at once
HGET session:abc user_id                # Get one field → 42
HGETALL session:abc                     # Get all fields → {user_id: 42, role: admin}
HEXISTS session:abc user_id             # Check if field exists → 1
HDEL session:abc role                   # Delete a field
```

**Use Cases:**
- Store objects (user sessions, profiles)
- Store configuration
- Better than storing JSON strings (queryable fields)

**Example:**
```redis
# Store a user session
HSET session:xyz user_id 42 username alice role admin ip 192.168.1.1

# Get entire session
HGETALL session:xyz
# Returns: {user_id: 42, username: alice, role: admin, ip: 192.168.1.1}

# Get one field
HGET session:xyz user_id
# Returns: 42

# Check if logged in
HEXISTS session:xyz user_id
# Returns: 1 (true)
```

**Advantage over String:**
```redis
# BAD: String (must parse JSON every time)
GET session:xyz
# Returns: '{"user_id":42,"username":"alice"}'
# Must parse this JSON in app code

# GOOD: Hash (query individual fields)
HGET session:xyz user_id
# Returns: 42 (already parsed!)
```

---

### **3. Lists (Queues/Stacks)**

**What it is:**
- Linked list: ordered, can be FIFO queue or LIFO stack
- Elements can repeat

**Commands:**
```redis
LPUSH queue job1 job2 job3        # Push to left (head)
RPOP queue                         # Pop from right (tail) → job1 (FIFO)
LPOP queue                         # Pop from left → job3 (LIFO stack)
LRANGE queue 0 -1                  # View all elements
LLEN queue                          # Length → 2
LTRIM queue 0 99                   # Keep only first 100 elements
```

**Use Cases:**
- Job queues (FIFO)
- Stacks (LIFO)
- Recent items (sliding window)

**Example: Job Queue**
```redis
# Add jobs to queue
LPUSH jobs:high priority_task other_task
LPUSH jobs:low minor_task

# Worker processes job
RPOP jobs:high
# Returns: priority_task

# Another worker
RPOP jobs:high
# Returns: other_task
```

**FIFO vs LIFO:**
```redis
# FIFO (Queue) — first in, first out
LPUSH queue A B C        # Queue: [C, B, A]
RPOP queue               # A (first in, first out)

# LIFO (Stack) — last in, first out
LPUSH stack A B C        # Stack: [C, B, A]
LPOP stack               # C (last in, first out)
```

---

### **4. Sets (Unique Members, Set Math)**

**What it is:**
- Unordered collection of unique members
- No duplicates
- Fast membership check

**Commands:**
```redis
SADD tags redis cache python      # Add members
SMEMBERS tags                     # Get all members → {redis, cache, python}
SISMEMBER tags redis              # Is "redis" in the set? → 1 (true)
SCARD tags                         # Cardinality (count) → 3
SREM tags python                  # Remove member
SINTER set1 set2                  # Intersection (members in both)
SUNION set1 set2                  # Union (members in either)
```

**Use Cases:**
- Unique tracking (unique visitors, unique tags)
- Set math (find common interests, overlap)
- Fast membership checks

**Example: Unique Visitors**
```redis
# Track unique visitors today
SADD visitors:2024-09-04 user:1 user:2 user:3 user:1  # user:1 added twice, stored once!

# Count unique visitors
SCARD visitors:2024-09-04
# Returns: 3

# Is user:2 a visitor today?
SISMEMBER visitors:2024-09-04 user:2
# Returns: 1 (true)

# Find common viewers between day 1 and day 2
SINTER visitors:2024-09-04 visitors:2024-09-05
# Returns: users who visited both days
```

---

### **5. Sorted Sets (Leaderboards, Rankings)**

**What it is:**
- Set of unique members, each with a score
- Automatically sorted by score
- Perfect for rankings

**Commands:**
```redis
ZADD leaderboard 100 alice 200 bob 150 charlie  # Add with scores
ZREVRANGE leaderboard 0 9 WITHSCORES            # Top 10 (highest scores first)
ZRANGE leaderboard 0 9 WITHSCORES               # Bottom 10 (lowest scores first)
ZREVRANK leaderboard alice                      # Rank of alice (1 = top)
ZSCORE leaderboard alice                        # alice's score → 100
ZCOUNT leaderboard 100 150                      # Count members with score 100-150
ZINCRBY leaderboard 50 alice                    # Increase alice's score by 50
```

**Use Cases:**
- Leaderboards (games, contests)
- Top N rankings
- Range queries (top 10, score between X and Y)
- Real-time rankings

**Example: Game Leaderboard**
```redis
# Players submit scores
ZADD leaderboard 100 alice
ZADD leaderboard 200 bob
ZADD leaderboard 150 charlie

# Get top 3 players
ZREVRANGE leaderboard 0 2 WITHSCORES
# Returns:
# 1) bob        2) 200
# 2) charlie    3) 150
# 3) alice      4) 100

# Check alice's rank
ZREVRANK leaderboard alice
# Returns: 3 (3rd place)

# alice scores 100 more points!
ZINCRBY leaderboard 100 alice

# Check rank again
ZREVRANK leaderboard alice
# Returns: 1 (1st place!)
```

---

### **6. Streams (Append-Only Logs with Consumer Groups)**

**What it is:**
- Append-only log (like Kafka but simpler)
- Each message has an ID, timestamp, key-value pairs
- Consumer groups track which consumers processed which messages

**Commands:**
```redis
XADD events * user alice action login   # Append message (auto-generate ID)
XRANGE events - +                        # Read all messages (from oldest to newest)
XREAD COUNT 10 STREAMS events 0         # Read 10 messages from the beginning
XGROUP CREATE events group1 0           # Create consumer group
XREADGROUP GROUP group1 consumer1 STREAMS events >  # Read unprocessed messages
```

**Use Cases:**
- Real-time logs (analytics, audit trails)
- Kafka-lite (event sourcing)
- Replay-able messages (unlike pub/sub which loses offline messages)

**Difference from Pub/Sub:**
```redis
# PUB/SUB (fire-and-forget, no storage)
PUBLISH news "Breaking: Redis is fast"
SUBSCRIBE news
# If you're offline, you miss the message

# Streams (persistent, replayable)
XADD events * message "Breaking: Redis is fast"
XREAD STREAMS events 0
# Message is stored forever (or until trimmed)
```

---

## Core Concept: TTL & Expiry

### **Why TTL Matters**

Without TTL:
```redis
SET user:1 "alice" 
# Stays forever!
# Memory fills up with stale data
```

With TTL:
```redis
SET user:1 "alice" EX 300
# Auto-deletes after 300 seconds
# Memory stays clean
```

### **Set Expiry**

**At set time:**
```redis
SET key value EX 60              # 60 seconds
SET key value PX 60000           # 60,000 milliseconds
SET key value EXAT 1735689600    # Expire at Unix timestamp
```

**After set:**
```redis
EXPIRE key 60                    # Set TTL to 60 seconds
PEXPIRE key 60000                # Set TTL to 60,000 milliseconds
TTL key                          # Check remaining seconds → 45
PTTL key                         # Check remaining milliseconds → 45000
PERSIST key                      # Remove TTL (keep forever)
```

### **Cache Example**

```redis
# Cache a user profile for 5 minutes
SET user:42 '{"name":"Alice","email":"alice@ex.com"}' EX 300

# Check how long until it expires
TTL user:42
# Returns: 285 (expires in 285 seconds)

# After 5 minutes...
GET user:42
# Returns: nil (key expired, deleted)
```

### **Lazy Expiry vs Active Expiry**

Redis uses both:
1. **Lazy:** On access, if key is expired, delete it
2. **Active:** Background sampler periodically deletes expired keys

**Result:** Expiry is prompt but not exact-to-the-millisecond. You might see an expired key for a few more seconds.

---

## Core Concept: Persistence (RDB vs AOF)

### **The Problem**

Redis is in-memory — power fails, data is gone!

**Scenario:**
```
3:00 PM: SET user:1 "alice"
3:05 PM: Server crashes
3:06 PM: Restart Redis
        GET user:1 → nil (lost!)
```

### **Solution: Persistence**

Redis offers two durability modes:

### **1. RDB (Redis Database Snapshots)**

**What it does:**
- Periodically saves entire database to disk
- Point-in-time snapshot

**Configuration:**
```
# /etc/redis/redis.conf
save 900 1       # Save if 1 key changed in 900 seconds
save 300 10      # Save if 10 keys changed in 300 seconds
save 60 10000    # Save if 10,000 keys changed in 60 seconds
```

**Pros:**
- Compact file
- Fast restoration
- Good for backups

**Cons:**
- Snapshot-based (you lose data since last snapshot)
- If server crashes at 3:05, and last snapshot was 3:00, you lose 5 minutes

**Example:**
```
3:00:00 PM: Save snapshot to disk (dump.rdb)
3:02:30 PM: SET user:1 "alice"
3:03:00 PM: SET user:2 "bob"
3:03:15 PM: Server crashes!
3:03:45 PM: Redis restarts, loads dump.rdb from 3:00:00
            GET user:1 → nil (lost!)
            GET user:2 → nil (lost!)
            Lost 3 minutes of data
```

---

### **2. AOF (Append-Only File)**

**What it does:**
- Logs every write command to a file
- Far more durable

**Configuration:**
```
# /etc/redis/redis.conf
appendonly yes                  # Enable AOF
appendfsync everysec            # Fsync every second (or: always, no)
```

**Pros:**
- Very durable (can fsync every command)
- Only loses data since last fsync

**Cons:**
- Larger file
- Slower restarts (must replay every command)

**Example:**
```
3:00:00 PM: SET user:1 "alice"     → Logged to AOF
3:02:30 PM: SET user:2 "bob"       → Logged to AOF
3:03:15 PM: Server crashes!
3:03:45 PM: Redis restarts, replays AOF
            GET user:1 → "alice" (restored!)
            GET user:2 → "bob" (restored!)
            Lost 0 seconds of data (or up to 1 second if appendfsync=everysec)
```

---

### **RDB vs AOF Trade-off**

| Aspect | RDB | AOF |
|--------|-----|-----|
| **Speed** | Fast | Slower |
| **File Size** | Compact | Larger |
| **Data Loss** | Minutes | Seconds/none |
| **Restart Time** | Fast | Slow (replays) |
| **Best For** | Backups, non-critical cache | Critical data |

### **Production Best Practice**

Use both:
```
# /etc/redis/redis.conf
save 60 10000           # RDB every minute (backup)
appendonly yes          # AOF for durability
appendfsync everysec    # Fsync every second
```

---

## Core Concept: Single-Threaded Model & Atomicity

### **Why Single-Threaded is Fast**

**Multi-threaded database:**
```
Thread 1: READ balance  (locks balance)
Thread 2: WAIT...       (blocked, waiting for lock)
Context switch overhead, lock contention, slow.
```

**Redis (single-threaded):**
```
Command 1: READ balance    (executes to completion)
Command 2: WRITE balance   (executes after command 1)
No locks, no context switches, microseconds.
```

### **Atomicity for Free**

In PostgreSQL, you need transactions:
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

In Redis, atomic operations are built-in:
```redis
INCR account:1:balance   # Atomic increment (can't race)
DECR account:2:balance   # Atomic decrement (can't race)
```

Why? Single thread runs each command to completion. No other thread can interrupt.

### **Atomic Commands**

**These are always safe:**
```redis
INCR counter                    # Increment atomically
LPUSH queue item                # Push to list atomically
SADD set member                 # Add to set atomically
ZADD leaderboard score player   # Add to sorted set atomically
SET key value                   # Set atomically (overwrites old)
```

**No race conditions, no locks needed!**

---

### **The Flip Side: One Slow Command Blocks Everyone**

❌ **BAD: Blocking everyone**

```redis
KEYS *                          # O(n) — scans all keys, blocks single thread!
```

If Redis has 1 million keys, this blocks for seconds. All other requests wait.

✅ **GOOD: Non-blocking iteration**

```redis
SCAN 0                          # O(1) per iteration, non-blocking
SCAN 0 MATCH user:* COUNT 100   # Scan with pattern
```

**Rule:** Don't run expensive commands (`KEYS *`, big `SORT`) in production. Use `SCAN` for iteration.

---

## Core Concept: Eviction Policies

### **The Problem**

Redis hits `maxmemory` limit (e.g., 2GB):

```
Memory: [████████████████████] 2GB (full!)
Next write: SET new_key value
```

**What happens?**

Depends on eviction policy:

### **1. noeviction (Default)**

```
SET new_key value
→ (error) OOM command not allowed when used memory > 'maxmemory'
```

❌ **Bad for cache:** Writes start failing when full instead of evicting old data.

### **2. allkeys-lru (Best for Cache)**

```
Evict least-recently-used key (accessed least recently)
SET new_key value → OK (after evicting old key)
```

✅ **How it works:**
- Track last access time for each key
- When full, delete key accessed longest ago
- Repeat until there's room

**Example:**
```
Memory: [User1 (2min ago), Product2 (30sec ago), Config (never), ...]
maxmemory reached!
Evict User1 (least recently used — 2min ago)
SET new_key value → OK
```

### **3. allkeys-lfu (Best for Frequently-Accessed Cache)**

```
Evict least-frequently-used key (accessed least often)
```

**How it works:**
- Track access frequency for each key
- When full, delete key accessed least often
- Good if some keys are always hot

**Example:**
```
Memory: [ProductA (accessed 1000x), ProductB (accessed 2x), ...]
maxmemory reached!
Evict ProductB (least frequently used — only 2 accesses)
SET new_key value → OK
```

### **4. volatile-lru / volatile-lfu / volatile-ttl**

```
Only evict keys that have a TTL set
Keys without TTL are never evicted (pinned data)
```

**Use case:**
```redis
# This is evicted if needed
SET session:abc user_id 42 EX 3600

# This NEVER evicted (critical data)
SET config:version "1.0"
# (no TTL)
```

---

### **Configuration**

```
# /etc/redis/redis.conf
maxmemory 2gb                     # 2GB limit
maxmemory-policy allkeys-lru      # Evict LRU on overflow
```

**Common mistake:**
```
# Default: noeviction (writes fail when full)
# Change to allkeys-lru (for cache)
CONFIG SET maxmemory-policy allkeys-lru
```

---

## Core Concept: Pub/Sub (Real-Time Messaging)

### **What It Is**

Fire-and-forget messaging for real-time fan-out (broadcasts).

```
┌─────────────┐
│ Publisher   │ PUBLISH news "Breaking news!"
└─────────────┘
        ↓
    ┌─────────────────────────────┐
    │ Redis (message broker)      │
    └─────────────────────────────┘
        ↓          ↓          ↓
    ┌──────┐  ┌──────┐  ┌──────┐
    │ Sub1 │  │ Sub2 │  │ Sub3 │ (all receive message)
    └──────┘  └──────┘  └──────┘
```

### **Commands**

```redis
# Subscriber (listening)
SUBSCRIBE news updates      # Listen to channels "news" and "updates"
# Waits for messages...

# Publisher (sends message)
PUBLISH news "Breaking: Redis is fast!"    # Sends to all subscribers
# Returns: 1 (number of subscribers received)
```

### **Use Cases**

- Chat (real-time messages)
- Notifications (user activity, alerts)
- Live updates (score changes, new posts)
- Coordination (workers getting tasks)

### **Example: Chat**

```redis
# User 1 joins chat
SUBSCRIBE room:lobby

# User 2 joins chat
SUBSCRIBE room:lobby

# User 1 sends message
PUBLISH room:lobby "User1: Hello!"

# Both users receive:
# Message from room:lobby: "User1: Hello!"
```

---

### **IMPORTANT: Pub/Sub Loses Messages**

❌ **Message not stored:**

```
3:00:00 PM: PUBLISH room:lobby "Message 1"
3:00:05 PM: SUBSCRIBE room:lobby  ← Joins late, misses message!
# Receives nothing (message is gone)
```

✅ **Solution: Use Streams if you need durability**

```redis
# Stores message
XADD room:lobby * user user1 message "Hello!"

# Late subscriber can still read
XRANGE room:lobby - +
# Returns: Message 1 (stored permanently)
```

---

## Core Concept: When NOT to Use Redis as Primary Database

### **The Fundamental Problem**

Redis is memory-bound:

```
Dataset must fit in RAM
┌──────────────────────┐
│ Redis: 16GB RAM      │
│ Max dataset: ~16GB   │
└──────────────────────┘

PostgreSQL: Unlimited
┌──────────────────────┐
│ Disk storage: 10TB+  │
│ Query anything       │
└──────────────────────┘
```

### **Why Not Use Redis as Primary?**

**1. Memory Limit**
- Redis: 16GB max (on 16GB server)
- PostgreSQL: Unlimited (can grow to terabytes)

**2. Persistence is Best-Effort**
- RDB: Loses data since last snapshot
- AOF: Slower, still some loss possible
- Not ACID like PostgreSQL

**3. No Complex Queries**
```redis
# Can't do this in Redis:
SELECT * FROM orders 
WHERE customer_id = 42 AND total > 100 
  AND created_at BETWEEN '2024-01-01' AND '2024-09-04'
ORDER BY created_at DESC
```

**PostgreSQL handles it easily.**

4. **Data Loss is Risky**
- Server crash → lose recent writes
- Not acceptable for financial records, user data

### **The Right Architecture**

```
┌──────────────────────────────┐
│ System of Record             │
│ PostgreSQL (durable, queryable)
├──────────────────────────────┤
│ Primary: orders, customers, |
│ products, accounts           │
└──────────────────────────────┘
        ↓ (cache/accelerate)
┌──────────────────────────────┐
│ Acceleration Layer           │
│ Redis (fast, ephemeral)      │
├──────────────────────────────┤
│ Cache: user profiles,        │
│ sessions, rate limits        │
└──────────────────────────────┘
```

**Flow:**
```
Read request:
1. Check Redis cache → Cache hit? Return immediately
2. Cache miss → Query PostgreSQL
3. Store result in Redis (TTL 5 min)
4. Return to user

Write request:
1. Write to PostgreSQL (durable)
2. Invalidate Redis cache (delete key)
3. Return to user
```

---

## Build This: Four Practice Projects

### **Project 1: Rate Limiter (INCR + EXPIRE)**

**Goal:** Allow 100 requests/minute per user.

```redis
# User makes a request at 3:00:00 PM
INCR ratelimit:user:42                   # Count: 1
EXPIRE ratelimit:user:42 60              # Expires at 3:01:00 PM

# User makes 99 more requests in the same minute
INCR ratelimit:user:42                   # Count: 100
# Still within limit ✅

# User makes 101st request
INCR ratelimit:user:42                   # Count: 101
# Check: 101 > 100? → REJECT ❌

# At 3:01:00 PM, key expires
TTL ratelimit:user:42                    # Returns: -2 (key expired)
INCR ratelimit:user:42                   # Count: 1 (fresh window)
```

**Code:**

```javascript
async function checkRateLimit(userId, limit = 100, windowSeconds = 60) {
  const key = `ratelimit:${userId}`;
  const count = await redis.incr(key);
  
  if (count === 1) {
    // First request in window, set expiry
    await redis.expire(key, windowSeconds);
  }
  
  if (count > limit) {
    return false;  // Rate limited
  }
  
  return true;  // Allowed
}

// Usage
if (await checkRateLimit(42)) {
  console.log("Request allowed");
} else {
  console.log("Rate limited!");
}
```

---

### **Project 2: Leaderboard (Sorted Sets)**

**Goal:** Track top 10 players by score.

```redis
# Players submit scores
ZADD leaderboard 100 alice
ZADD leaderboard 200 bob
ZADD leaderboard 150 charlie
ZADD leaderboard 180 david

# Get top 10 (highest scores first)
ZREVRANGE leaderboard 0 9 WITHSCORES
# Returns:
# 1) "bob"      2) "200"
# 2) "david"    3) "180"
# 3) "charlie"  4) "150"
# 4) "alice"    5) "100"

# Check alice's rank
ZREVRANK leaderboard alice
# Returns: 4

# alice scores 100 more!
ZINCRBY leaderboard 100 alice

# Check new rank
ZREVRANK leaderboard alice
# Returns: 1 (now 1st place!)
```

**Code:**

```javascript
async function leaderboard() {
  // Add scores
  await redis.zadd("leaderboard", 100, "alice");
  await redis.zadd("leaderboard", 200, "bob");
  
  // Get top 10
  const top10 = await redis.zrevrange("leaderboard", 0, 9, "WITHSCORES");
  console.log(top10);
  
  // Get one player's rank
  const rank = await redis.zrevrank("leaderboard", "alice");
  console.log(`alice is rank: ${rank + 1}`);  // +1 because Redis is 0-indexed
  
  // Increase score
  await redis.zincrby("leaderboard", 50, "alice");
}
```

---

### **Project 3: Cache-Aside Pattern (TTL)**

**Goal:** Cache expensive database queries.

```javascript
async function getUser(userId) {
  const cacheKey = `user:${userId}`;
  
  // Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    console.log("Cache HIT");
    return JSON.parse(cached);
  }
  
  // Cache miss → query database
  console.log("Cache MISS");
  const user = await db.query(
    "SELECT * FROM users WHERE id = $1",
    [userId]
  );
  
  // Store in cache for 5 minutes
  await redis.set(
    cacheKey,
    JSON.stringify(user),
    "EX",
    300  // 5 minutes
  );
  
  return user;
}

// Measure latency
console.time("First call (cache miss)");
await getUser(42);
console.timeEnd("First call (cache miss)");
// Output: ~50ms (database query)

console.time("Second call (cache hit)");
await getUser(42);
console.timeEnd("Second call (cache hit)");
// Output: ~1ms (Redis read)
```

**Speed comparison:**
- First call (database): 50ms
- Second call (Redis): 1ms
- **50x faster!**

---

### **Project 4: Distributed Lock (SET NX EX + Token)**

**Goal:** Ensure only one worker processes a job at a time.

```javascript
async function acquireLock(resource, workerToken, ttl = 10) {
  // SET only if key doesn't exist, with TTL
  const acquired = await redis.set(
    `lock:${resource}`,
    workerToken,
    "NX",      // Only set if doesn't exist
    "EX",      // With expiry
    ttl        // 10 seconds
  );
  
  return acquired === "OK";  // true if acquired
}

async function releaseLock(resource, workerToken) {
  // Delete only if token matches (prevent deleting others' locks)
  const script = `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
    else
      return 0
    end
  `;
  
  await redis.eval(script, 1, `lock:${resource}`, workerToken);
}

// Usage
const workerId = "worker-1";
const token = `${workerId}-${Date.now()}`;

if (await acquireLock("resource-x", token)) {
  try {
    console.log("Processing job...");
    // Do work...
  } finally {
    await releaseLock("resource-x", token);
  }
} else {
  console.log("Another worker has the lock, skipping");
}
```

**How it works:**

```
Worker 1:
  SET lock:resource worker-1-token NX EX 10
  → OK (acquired lock)
  → Processing...

Worker 2 (concurrent):
  SET lock:resource worker-2-token NX EX 10
  → nil (lock exists, can't acquire)
  → Wait or skip

Worker 1 finishes:
  DELETE lock:resource (only if token matches)
  → OK

Worker 2:
  SET lock:resource worker-2-token NX EX 10
  → OK (lock now available)
  → Processing...
```

---

## Mastery Checklist (0/8 → 8/8)

### **1. ✅ I can name the core data types and pick the right one for a given problem**

**What this means:**
- Strings (simple values, counters)
- Hashes (objects, sessions)
- Lists (queues, stacks)
- Sets (unique members, set math)
- Sorted Sets (leaderboards, rankings)
- Streams (durable event logs)

**Practice:**
- "I need to track top 10 players" → Sorted Set
- "I need a task queue" → List
- "I need to find common tags" → Set intersection
- "I need user session" → Hash
- "I need to count unique visitors" → Set

---

### **2. ✅ I can implement a rate limiter with INCR and EXPIRE**

**What this means:**
- Use INCR to count requests
- Use EXPIRE to reset counter after time window
- Reject when count exceeds limit

**Practice:**
- Implement 100 requests/minute limiter
- Test with loop (hammer server, watch limit kick in)
- Verify counter resets after window

---

### **3. ✅ I can build a leaderboard with sorted sets and query ranks**

**What this means:**
- ZADD to add scores
- ZREVRANGE to get top N
- ZREVRANK to get one player's rank
- ZINCRBY to update scores

**Practice:**
- Add 100 players with scores
- Get top 10
- Update scores, verify rankings change
- Query one player's rank

---

### **4. ✅ I can apply the cache-aside pattern with TTLs in front of a slow call**

**What this means:**
- Check Redis first
- On miss, query database
- Store result with TTL
- Return from cache on hit

**Practice:**
- Cache expensive query (e.g., user profile)
- Measure latency: database vs Redis
- Verify hit returns immediately
- Verify key expires after TTL

---

### **5. ✅ I understand why Redis is single-threaded and why that makes commands atomic**

**What this means:**
- Single thread runs each command to completion
- No race conditions
- INCR, LPUSH, ZADD are all atomic for free
- Avoid expensive commands (KEYS *, big SORT)

**Practice:**
- Explain atomicity of INCR without locks
- Trace through concurrent access (only one thread)
- Compare to multi-threaded database (need transactions)

---

### **6. ✅ I can explain RDB vs AOF and choose an eviction policy for a cache**

**What this means:**
- RDB: Snapshots (compact, fast restore, data loss on crash)
- AOF: Append-only file (durable, slower, larger)
- Eviction: allkeys-lru (good for cache), noeviction (bad for cache)

**Practice:**
- Configure RDB (save every minute)
- Configure AOF (fsync every second)
- Change eviction to allkeys-lru
- Fill Redis, watch eviction work

---

### **7. ✅ I know when NOT to use Redis as a primary database**

**What this means:**
- Redis is memory-bound (dataset must fit in RAM)
- Persistence is best-effort (not ACID)
- No complex queries
- Data loss is risky

**Use PostgreSQL for:**
- System of record (durable, queryable)
- Large datasets (>RAM)
- Complex transactions

**Use Redis for:**
- Cache (ephemeral)
- Sessions (temporary)
- Counters (fast updates)
- Leaderboards (ranking)

---

### **8. ✅ I can implement a safe distributed lock with SET NX EX and a matching token**

**What this means:**
- SET key token NX EX ttl to acquire
- DELETE only if token matches (prevent deleting others' locks)
- Use token to prevent cross-worker conflicts

**Practice:**
- Implement acquire + release
- Run two workers competing for lock
- Verify only one acquires
- Verify token prevents accidental deletion

---

## Cheat Sheet (Quick Reference)

### **Data Types**

```redis
# Strings
SET key value EX 60           # String with TTL
GET key
INCR counter                  # Atomic increment

# Hashes
HSET session:1 user_id 42 role admin
HGETALL session:1
HGET session:1 user_id

# Lists
LPUSH queue job1 job2         # Push (head)
RPOP queue                    # Pop (tail) → FIFO
LPOP queue                    # Pop (head) → LIFO

# Sets
SADD tags redis cache
SMEMBERS tags
SINTER set1 set2              # Intersection

# Sorted Sets
ZADD leaderboard 100 alice 200 bob
ZREVRANGE leaderboard 0 9 WITHSCORES
ZREVRANK leaderboard alice
ZINCRBY leaderboard 50 alice

# Streams
XADD events * user alice action login
XRANGE events - +
```

### **Operations**

```redis
# Expiry
EXPIRE key 60
TTL key
PERSIST key

# Eviction (configuration)
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru

# Pub/Sub
SUBSCRIBE channel
PUBLISH channel "message"

# Persistence (configuration)
BGSAVE                        # Trigger RDB
BGREWRITEAOF                  # Trigger AOF rewrite

# Locks
SET lock:resource token NX EX 10    # Acquire
DEL lock:resource  (if token matches) # Release
```

---

## Common Pitfalls (Don't Do This)

### **Pitfall 1: Running KEYS * in Production**

❌ **BAD:**
```redis
KEYS *                        # Scans all keys, blocks everyone!
# With 1M keys, blocks for seconds
```

✅ **GOOD:**
```redis
SCAN 0                        # Non-blocking iteration
SCAN 0 MATCH user:* COUNT 100 # With pattern
```

---

### **Pitfall 2: Treating Redis as Your Primary Database**

❌ **BAD:**
```
Store all customer data in Redis
Server crashes
Lost all data! 😱
```

✅ **GOOD:**
```
PostgreSQL: System of record (durable)
Redis: Accelerator layer (ephemeral)
```

---

### **Pitfall 3: Caching Without TTL**

❌ **BAD:**
```redis
SET user:1 "alice"            # No TTL
# Forever cached (even after user updates!)
# Memory fills with stale data
```

✅ **GOOD:**
```redis
SET user:1 "alice" EX 300     # 5-minute TTL
# Auto-deletes, memory stays clean
```

---

### **Pitfall 4: Leaving maxmemory-policy on noeviction**

❌ **BAD:**
```
maxmemory: 2GB
maxmemory-policy: noeviction (default)

→ When full, writes start failing
→ Cache becomes useless!
```

✅ **GOOD:**
```
CONFIG SET maxmemory-policy allkeys-lru
→ When full, evicts old keys
→ New writes succeed
```

---

### **Pitfall 5: Losing Pub/Sub Messages**

❌ **BAD:**
```redis
PUBLISH room:1 "Message"
# Late subscriber misses it
```

✅ **GOOD:**
```redis
# Use Streams for durability
XADD room:1 * message "text"
# Message stored, replayable
```

---

## Interview Signals & Mastery

### **Signal of Mastery**

You can:
- ✅ Name the right data structure for a problem instantly
- ✅ Explain why INCR is atomic (single-threaded)
- ✅ Spot rate limiting with INCR + EXPIRE
- ✅ Explain leaderboard with sorted sets
- ✅ Design cache-aside pattern with TTLs
- ✅ Explain RDB vs AOF trade-offs
- ✅ Know when Redis is NOT the right tool
- ✅ Implement distributed lock safely

### **Common Interview Questions**

1. "Design a rate limiter" → INCR + EXPIRE
2. "Design a leaderboard" → Sorted sets
3. "Should we cache this?" → Yes, with TTL
4. "How do we avoid stale data?" → TTL + invalidation
5. "Can we use Redis as our database?" → No, PostgreSQL is the source
6. "What if Redis crashes?" → Persistence (RDB/AOF) + replication
7. "How to do distributed locking?" → SET NX EX + token
8. "Why is Redis faster than PostgreSQL?" → Single-threaded, in-memory, simple ops

---

## Quick Start: Redis Commands Cheat Sheet

```bash
# Start Redis
redis-cli

# Health check
PING                          # → PONG

# Flush (danger!)
FLUSHALL                      # Delete everything!
FLUSHDB                       # Delete current database only

# Monitor
MONITOR                       # See all commands in real-time
DBSIZE                        # Total keys

# Persistence
SAVE                          # Synchronous save (blocks!)
BGSAVE                        # Async save in background
LASTSAVE                      # Last save time
```

---

## Next Steps

- **This Week:** Practice all 4 projects (rate limiter, leaderboard, cache, lock)
- **Next Week:** Application layer (APIs, auth, real-time)

By end of week: You understand when and how to use Redis, and which problem each data structure solves.

---

## Summary

**Redis is a whiteboard, not a filing cabinet:**
- ✅ Fast (microseconds)
- ✅ Specialized data structures (counter, list, set, sorted set, stream)
- ✅ Perfect for cache, sessions, rate limiting, leaderboards
- ❌ Don't use as primary database
- ❌ Data loss is acceptable
- ❌ Must fit in RAM

**Master these 8 items and you're a Redis expert:**
1. Pick right data type
2. Rate limiter (INCR + EXPIRE)
3. Leaderboard (sorted sets)
4. Cache-aside pattern (TTL)
5. Single-threaded atomicity
6. RDB vs AOF, eviction policy
7. Know when NOT to use Redis
8. Distributed locks (SET NX EX + token)

**Senior engineers use Redis strategically — not as a silver bullet, but as a precision tool for specific problems.**
