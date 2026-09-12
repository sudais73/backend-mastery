# Week 4B: Caching Strategy Mastery

## Overview

**Goal:** Master caching as a strategic tool to accelerate performance without serving stale data.

**Why it matters:** The fastest query is the one you never run. Caching turns 200ms database hits into sub-millisecond lookups. But caching is a deal with the devil — every cache is a copy of the truth that can drift out of sync.

**The Real Skill:** Not making things fast, but making them fast AND correct.

---

## The Mental Model (Foundation)

### **Cache is a Notepad, Database is a Filing Cabinet**

**Filing Cabinet (PostgreSQL)**
```
┌─────────────────────────────────┐
│ System of Record                │
│ - Authoritative                 │
│ - Durable (permanent)           │
│ - Complete (all data)           │
│ - Slow (200ms per access)       │
└─────────────────────────────────┘
```

**Notepad (Redis/Memcached)**
```
┌─────────────────────────────────┐
│ Cache                           │
│ - Copy of "hot" data            │
│ - Fast (1ms per access)         │
│ - Incomplete (not everything)   │
│ - Temporary (expires)           │
└─────────────────────────────────┘
```

### **The Two Problems with Caches**

**Problem 1: Cache Misses (Normal)**
```
User requests user:42
Cache: MISS (not there)
→ Query database (200ms)
→ Return result
→ Store in cache
Next request: HIT (1ms)

This is fine! Misses are cheap.
```

**Problem 2: Stale Hits (DANGEROUS)**
```
User updates profile → database changes
Cache still has old data (hasn't expired)
Next request: HIT (1ms) but returns OLD data
→ User sees wrong information
→ Bugs that only appear under load
→ THIS IS THE REAL DANGER

Solution: Invalidate immediately on writes
```

### **The Three Cache Promises**

1. **Fast:** Sub-millisecond reads ✅
2. **Correct:** Never serve stale data ✅
3. **Safe:** Invalidate when data changes ✅

Design for invalidation first, speed second.

---

## Core Concept 1: Cache Layers (Browser → CDN → Proxy → App → DB)

### **The Stack**

Every request passes through multiple caches, from browser to database:

```
┌──────────────────────────────────────────┐
│ Browser Cache                            │ (Fastest: disk cache)
│ Cache-Control header                     │
│ Avoid network entirely                   │
└──────────────────────────────────────────┘
           ↓ (if not cached locally)
┌──────────────────────────────────────────┐
│ CDN (Content Delivery Network)           │ (Edge cache)
│ Cloudflare, AWS CloudFront               │
│ Caches static assets, cacheable responses│
│ Edge server near user                    │
└──────────────────────────────────────────┘
           ↓ (if not cached at edge)
┌──────────────────────────────────────────┐
│ Reverse Proxy (NGINX, Varnish)           │ (In-front of app)
│ Caches rendered pages, API responses     │
│ Between CDN and application              │
└──────────────────────────────────────────┘
           ↓ (if not cached by proxy)
┌──────────────────────────────────────────┐
│ Application Cache (Redis, Memcached)     │ (In-app)
│ Caches computed results, database rows   │
│ Within your application                  │
└──────────────────────────────────────────┘
           ↓ (if not cached by app)
┌──────────────────────────────────────────┐
│ Database Cache                           │ (Internal)
│ Buffer pool, query cache                 │
│ Inside PostgreSQL                        │
└──────────────────────────────────────────┘
```

### **Latency Comparison**

```
Browser Cache:        ~0.001ms (disk, no network)
CDN:                  ~10ms (closest edge server)
Reverse Proxy:        ~20ms (in-front of app)
Application Cache:    ~1ms (Redis in-memory)
Database:             ~200ms (disk I/O)
```

**Rule:** Cache closer to the user = faster response. BUT harder to invalidate.

### **Example: Product Page**

```
Request: GET /products/42

1. Browser checks Cache-Control header
   → If fresh (not expired), return from disk ✅ 1ms

2. Browser checks CDN
   → If fresh, return from edge ✅ 10ms

3. Browser checks reverse proxy
   → If fresh, return from proxy ✅ 20ms

4. Browser checks application cache
   → If fresh, return from Redis ✅ 1ms

5. Query database
   → Slow path ❌ 200ms

If cached at layer 1, request takes 1ms.
If cached at layer 5 only, request takes 200ms + 1ms cache store.
```

### **When to Cache at Each Layer**

**Browser Cache (Cache-Control: public, max-age=3600)**
- Static files (CSS, JS, images)
- Data unlikely to change (product catalog, pricing)

**CDN Cache**
- Static content (images, CSS, JS)
- API responses that are globally stable
- Homepage, blog posts

**Reverse Proxy Cache**
- Rendered HTML pages
- API responses cacheable for 10-60s
- Save application server cycles

**Application Cache (Redis)**
- Database rows (users, products)
- Computed results
- Sessions
- Rate limit counters

**Database Cache**
- Query results
- Indexes
- No action needed (automatic)

---

## Core Concept 2: Cache-Aside (Lazy Loading)

### **The Pattern**

Application owns the cache logic:

```
READ REQUEST:

1. Check cache (Redis)
   GET user:42:profile
   
2a. CACHE HIT
    → Return immediately (1ms)
    
2b. CACHE MISS
    → Query database (200ms)
    → Store in cache: SET user:42:profile <json> EX 300
    → Return to user

WRITE REQUEST:

1. Update database
   UPDATE users SET name='Alice' WHERE id=42
   
2. Invalidate cache
   DEL user:42:profile
   
3. Return to user
```

### **Code Example**

```javascript
async function getUser(userId) {
  const cacheKey = `user:${userId}:profile`;
  
  // 1. Try cache first
  let user = await redis.get(cacheKey);
  if (user) {
    console.log("CACHE HIT");
    return JSON.parse(user);
  }
  
  // 2. Cache miss → query database
  console.log("CACHE MISS");
  user = await db.query(
    "SELECT * FROM users WHERE id = $1",
    [userId]
  );
  
  // 3. Store in cache (5-minute TTL)
  await redis.set(
    cacheKey,
    JSON.stringify(user),
    "EX",
    300
  );
  
  return user;
}

async function updateUser(userId, updates) {
  // 1. Update database
  const user = await db.query(
    "UPDATE users SET $1 WHERE id = $2 RETURNING *",
    [updates, userId]
  );
  
  // 2. Invalidate cache
  await redis.del(`user:${userId}:profile`);
  
  return user;
}
```

### **Why Cache-Aside is Good**

✅ **Simple:** Application controls logic
✅ **Cheap:** Only caches data actually requested (no waste on cold keys)
✅ **Flexible:** Can cache anything (not just simple key-value)
✅ **Safe:** Full control over invalidation

### **Why Cache-Aside is Hard**

❌ **Application complexity:** Must write cache logic in app
❌ **Stampede risk:** Many concurrent misses hit database together
❌ **Invalidation bugs:** Forget to invalidate = stale data forever

---

## Core Concept 3: Write-Through & Write-Behind

### **Write-Through (Synchronous)**

```
WRITE REQUEST:

1. Write to cache
   SET user:42:profile <json>
   
2. Write to database (waits for this)
   UPDATE users SET ...
   
3. Return to user

Problems:
- Slower write (must write to two places)
- Both must succeed (if DB fails, cache is inconsistent)
```

**When to use:**
- Critical data (financial records)
- Small datasets
- Writes are infrequent

### **Write-Behind (Write-Back) (Asynchronous)**

```
WRITE REQUEST:

1. Write to cache immediately
   SET user:42:profile <json>
   
2. Queue database write (async, in background)
   QUEUE: UPDATE users SET ...
   
3. Return to user immediately (fast!)

4. Eventually, database write happens

Benefits:
- Blazing fast writes (return immediately)
- Cache is always fresh
- Database gets batched updates

Problems:
- Risk of data loss (if cache dies before flush)
- Complexity (must track pending writes)
```

**When to use:**
- Non-critical data (sessions, analytics)
- Write-heavy workloads
- Can afford to lose a few seconds of data

### **Hybrid (Recommended)**

```
READ:  Use cache-aside (lazy loading)
WRITE: Use write-through or explicit invalidation
       (write to DB, then delete cache key)

Best balance of safety and speed
```

---

## Core Concept 4: TTL (Time To Live)

### **What TTL Does**

```
SET user:42 "alice" EX 300
         ↓
At T+0:   user:42 = "alice" (fresh)
At T+100: user:42 = "alice" (fresh)
At T+300: user:42 = nil     (expired, gone!)
At T+301: Query database, refresh cache
```

### **TTL = Staleness Bound**

Choose TTL based on how old data can safely be:

```
Static data (product catalog):      24 hours
Semi-static (user profiles):        5 minutes
Dynamic (scores, inventory):        10-30 seconds
Hot/volatile (real-time rankings):  1 second
```

### **TTL Trade-Offs**

**Long TTL (1 hour)**
- Fewer misses (cache stays warm)
- Risk: Stale data lingers for an hour
- Good for: Non-critical, rarely-changing data

**Short TTL (10 seconds)**
- More misses (cache expires often)
- Benefit: Staleness bounded to 10 seconds
- Good for: Dynamic, important data

**No TTL**
- Cache never expires
- **DANGER:** If data changes and you forget to invalidate, serves stale forever
- Only use if you ALWAYS invalidate on writes

### **Best Practice**

Always combine:
1. **Short TTL** (auto-cleanup, self-healing)
2. **Explicit Invalidation** (immediate freshness on writes)

```redis
# Set with TTL (self-heals if invalidation forgotten)
SET user:42 "alice" EX 300

# On write, invalidate immediately
DEL user:42

# If DB update succeeds but invalidation fails (rare),
# stale data self-heals after 300 seconds
```

---

## Core Concept 5: Eviction (LRU vs LFU)

### **When Does Eviction Happen?**

Redis hits `maxmemory` limit:

```
maxmemory: 2GB
Current usage: 1.99GB
New write: SET huge_value (10MB)
→ 1.99GB + 10MB > 2GB (over limit!)
→ Eviction policy decides what to delete
```

### **LRU (Least Recently Used)**

```
Evict the key accessed longest ago

Example:
Key1: accessed 30 minutes ago  ← DELETE ME
Key2: accessed 2 minutes ago
Key3: accessed 1 minute ago
Key4: accessed 10 seconds ago

Delete Key1, make room for new write
```

**Good for:** Stable hot sets (product catalog, user profiles)
**Why:** Recently accessed = probably still needed

### **LFU (Least Frequently Used)**

```
Evict the key accessed least often

Example:
Key1: accessed 1,000 times  ← Keep me
Key2: accessed 50 times
Key3: accessed 3 times      ← DELETE ME
Key4: accessed 1 time

Delete Key3, make room for new write
```

**Good for:** Very stable access patterns
**Why:** Frequently accessed = valuable to keep

### **Configuration**

```redis
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru    # Default
CONFIG SET maxmemory-policy allkeys-lfu    # Alternative
CONFIG SET maxmemory-policy volatile-lru   # Only evict keys with TTL
```

### **LRU vs LFU Trade-off**

| Scenario | LRU | LFU |
|----------|-----|-----|
| "Keep recently accessed" | ✅ | ❌ |
| "Keep frequently accessed" | ❌ | ✅ |
| "Stable hot set" | ✅ | ✅ |
| "Bursty traffic" | ✅ | ✅ |
| "Cold data with new hits" | ❌ | ✅ |

**Default to LRU** unless you have a reason.

---

## Core Concept 6: Cache Keys (The Naming Problem)

### **The Problem**

Sloppy keys silently serve wrong data:

```redis
# BAD: Key omits user_id
SET product:42 <json>

# User 1 requests product 42
GET product:42 → Returns cached data

# User 2 requests product 42
GET product:42 → Returns SAME cached data ✅ (correct by luck)

# BUT if key also includes locale...

# BAD: Key omits locale
SET product:42 <json in English>

# User 1 (English) requests product 42
GET product:42 → English version ✅

# User 2 (Spanish) requests product 42
GET product:42 → English version ❌ (WRONG! Should be Spanish)
```

### **Rule: Cache Key Must Include Every Input That Affects Output**

```javascript
// Product page shows price in user's currency
async function getProductPrice(productId, currency) {
  // WRONG KEY (omits currency)
  // const key = `product:${productId}:price`;
  
  // CORRECT KEY (includes currency)
  const key = `product:${productId}:price:${currency}`;
  
  let price = await redis.get(key);
  if (!price) {
    price = await db.query(
      "SELECT price FROM products WHERE id = $1",
      [productId]
    );
    // Convert to user's currency
    price = convertCurrency(price, currency);
    await redis.set(key, price, "EX", 300);
  }
  
  return price;
}

// Calling it
getProductPrice(42, "USD")   // Caches to product:42:price:USD
getProductPrice(42, "EUR")   // Caches to product:42:price:EUR
```

### **Cache Key Checklist**

Every input that changes the result:
- ✅ User ID (different users, different data)
- ✅ Locale/Language (affects output format)
- ✅ Filter parameters (changes query result)
- ✅ API version (different response structure)
- ✅ Feature flags (different behavior)
- ✅ Time-based (if result changes by time)

### **Key Naming Convention**

```javascript
// Descriptive, hierarchical, includes all inputs
`entity:id:field:modifier`

Examples:
`user:42:profile`
`user:42:profile:v2`
`product:42:price:USD`
`post:1:comments:page:1`
`leaderboard:game:day:2024-09-04`
`user:42:subscriptions:active`
```

---

## Core Concept 7: Invalidation (The Hard Part)

### **The Problem**

Invalidation is the hardest part of caching:

```
User updates profile:
1. Database updated ✅
2. Cache invalidated ✅
→ Next request: fresh data ✅

BUT what if:
1. Database updated ✅
2. Cache invalidation FAILS ❌
→ Next request: stale data (forever!)

OR:

1. Database updated ✅
2. Response returned to user ✅
3. Cache invalidation FAILS (async job crashes) ❌
→ Cache never updated!
```

### **Invalidation Strategies**

**Strategy 1: TTL Only (Risky)**

```redis
SET user:42 "alice" EX 300
# Relies on TTL expiry
# If data changes at T+1, stale until T+300
```

**Pros:** Simple
**Cons:** Stale data lingers for up to TTL duration

---

**Strategy 2: TTL + Explicit Invalidation (Good)**

```javascript
// On write
async function updateUser(userId, updates) {
  // Write to DB
  const user = await db.query(
    "UPDATE users SET ... WHERE id = $1 RETURNING *",
    [userId]
  );
  
  // Invalidate cache
  await redis.del(`user:${userId}:profile`);
  
  return user;
}
```

**Pros:** Fresh data immediately after write
**Cons:** Must invalidate every write path

---

**Strategy 3: Versioning (Advanced)**

```javascript
// Don't delete key, increment version

// Old way (delete)
// DEL user:42:profile

// New way (version)
SET `user:42:version` 5

// Cache key includes version
GET `user:42:profile:v5`
// On update: increment version
INCR `user:42:version`  // Now v6
// Automatically invalidated (different key)
```

**Pros:** No cache deletion (just new version)
**Cons:** More complex, uses more keys

---

**Strategy 4: Event-Based (Most Robust)**

```javascript
// On every write, publish event
async function updateUser(userId, updates) {
  await db.query(...);
  
  // Publish invalidation event
  await redis.publish(`events:user:${userId}`, "updated");
}

// Subscriber listens for events
redis.subscribe(`events:user:*`);
redis.on("message", (channel, message) => {
  const userId = channel.split(":")[2];
  // Invalidate cache for this user
  redis.del(`user:${userId}:profile`);
});
```

**Pros:** Decoupled, robust
**Cons:** Complexity, requires event system

---

### **Invalidation Checklist**

Every write must invalidate relevant caches:

```javascript
// Create user
async function createUser(name) {
  await db.query("INSERT INTO users ...");
  // Invalidate list of users (if cached)
  await redis.del("users:list");
}

// Update user
async function updateUser(userId, updates) {
  await db.query("UPDATE users ...");
  // Invalidate user profile
  await redis.del(`user:${userId}:profile`);
  // Invalidate list of users (if filtered by this user)
  await redis.del("users:list");
}

// Delete user
async function deleteUser(userId) {
  await db.query("DELETE FROM users ...");
  // Invalidate user profile
  await redis.del(`user:${userId}:profile`);
  // Invalidate list
  await redis.del("users:list");
}
```

---

## Core Concept 8: Cache Stampede (Thundering Herd)

### **The Problem**

When a hot key expires, all concurrent requests miss at once:

```
Time: 10:00:00
Key expires: user:42:profile (100 concurrent requests)

All 100 requests hit cache simultaneously:
10:00:00.001:  Request 1: GET user:42 → MISS → Query DB
10:00:00.002:  Request 2: GET user:42 → MISS → Query DB
10:00:00.003:  Request 3: GET user:42 → MISS → Query DB
...
10:00:00.100:  Request 100: GET user:42 → MISS → Query DB

Database sees 100 simultaneous queries for same data!
Database CPU spikes, query takes 500ms instead of 200ms
All 100 requests wait 500ms
User experiences 500ms latency (instead of 1ms cache hit)
```

### **Fix 1: Rebuild Lock (Single-Flight)**

```javascript
async function getUser(userId) {
  const cacheKey = `user:${userId}:profile`;
  const lockKey = `lock:${cacheKey}`;
  
  // Try cache
  let user = await redis.get(cacheKey);
  if (user) return JSON.parse(user);
  
  // Cache miss → try to acquire lock
  const acquired = await redis.set(
    lockKey,
    "1",
    "NX",      // Only set if absent
    "EX",      // With TTL
    5          // 5 seconds
  );
  
  if (acquired === "OK") {
    // I got the lock → rebuild
    try {
      user = await db.query(...);  // Only one query!
      await redis.set(cacheKey, JSON.stringify(user), "EX", 300);
      return user;
    } finally {
      await redis.del(lockKey);  // Release lock
    }
  } else {
    // Someone else has lock → wait and retry
    // Or return stale value if available
    await sleep(100);  // Wait 100ms
    return await getUser(userId);  // Retry (now cache is warm)
  }
}
```

**Result:**
```
10:00:00.001:  Request 1: Acquires lock → Queries DB (1 query)
10:00:00.002:  Request 2: Can't get lock → Waits
10:00:00.003:  Request 3: Can't get lock → Waits
...
10:00:00.200:  Request 1: Cache populated → Others now hit
10:00:00.201:  Request 2: Cache HIT ✅
10:00:00.202:  Request 3: Cache HIT ✅

Database: 1 query (not 100!)
Users: ~200ms total (not 500ms)
```

---

### **Fix 2: Request Coalescing (Deduplication)**

```javascript
// Track in-flight requests globally
const inflightRequests = new Map();

async function getUser(userId) {
  const cacheKey = `user:${userId}:profile`;
  
  // Try cache
  let user = await redis.get(cacheKey);
  if (user) return JSON.parse(user);
  
  // Check if another request is already loading this
  if (inflightRequests.has(cacheKey)) {
    // Wait for that request to finish
    return await inflightRequests.get(cacheKey);
  }
  
  // Start loading
  const promise = (async () => {
    const user = await db.query(...);
    await redis.set(cacheKey, JSON.stringify(user), "EX", 300);
    return user;
  })();
  
  inflightRequests.set(cacheKey, promise);
  
  try {
    return await promise;
  } finally {
    inflightRequests.delete(cacheKey);
  }
}
```

**Result:** Same as lock — 1 database query, not 100.

---

### **Fix 3: Jittered TTL (Desynchronized Expiry)**

```javascript
async function setUserCache(user) {
  // Instead of all keys expiring at same time...
  // Add random jitter (0-30 seconds)
  
  const baseTTL = 300;  // 5 minutes
  const jitter = Math.random() * 30;  // 0-30 sec random
  const ttl = baseTTL + jitter;  // 300-330 seconds
  
  await redis.set(
    `user:${user.id}:profile`,
    JSON.stringify(user),
    "EX",
    ttl
  );
}
```

**Result:**
```
10:00:00: 100 keys set with TTLs:
  Key 1: expires at 10:05:07
  Key 2: expires at 10:05:12
  Key 3: expires at 10:05:03
  ...
  Key 100: expires at 10:05:25

No synchronized expiry → No synchronized stampede!
Misses are spread out → Database load stays low
```

---

### **Fix 4: Stale-While-Revalidate (Best UX)**

```javascript
async function getUser(userId) {
  const cacheKey = `user:${userId}:profile`;
  
  // Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    // Check if stale
    const staleKey = `${cacheKey}:stale`;
    const isStale = await redis.exists(staleKey);
    
    if (!isStale) {
      // Fresh, return immediately
      return JSON.parse(cached);
    }
    
    // Stale, but return anyway (users don't wait)
    returnImmediately(JSON.parse(cached));
    
    // Refresh in background (async)
    refreshInBackground();
  }
}

async function refreshInBackground() {
  const user = await db.query(...);
  await redis.set(cacheKey, JSON.stringify(user), "EX", 300);
  await redis.del(`${cacheKey}:stale`);  // No longer stale
}
```

**Result:**
```
User 1: Gets cached data IMMEDIATELY (even if slightly stale)
        + Background refresh happens
User 2: Gets fresh data from cache (from User 1's refresh)

Users never wait for cache rebuild!
Best latency + eventual consistency
```

---

## Build This: Four Practice Projects

### **Project 1: Cache-Aside with Measurement**

```javascript
// Simulate slow database query
async function slowQuery() {
  // Add 200ms delay
  await new Promise(r => setTimeout(r, 200));
  return { id: 42, name: "Alice", email: "alice@ex.com" };
}

async function getUserWithCache(userId) {
  const key = `user:${userId}`;
  
  // Try cache
  const cached = await redis.get(key);
  if (cached) {
    console.log("CACHE HIT");
    return JSON.parse(cached);
  }
  
  // Cache miss
  console.log("CACHE MISS");
  const user = await slowQuery();
  await redis.set(key, JSON.stringify(user), "EX", 60);
  return user;
}

// Test
console.time("First call (miss)");
await getUserWithCache(42);
console.timeEnd("First call (miss)");
// Output: ~200ms

console.time("Second call (hit)");
await getUserWithCache(42);
console.timeEnd("Second call (hit)");
// Output: ~1ms (200x faster!)
```

---

### **Project 2: Invalidation on Write**

```javascript
async function updateUser(userId, updates) {
  // Update database
  const user = await db.query(
    "UPDATE users SET ... WHERE id = $1 RETURNING *",
    [userId]
  );
  
  // Invalidate cache
  await redis.del(`user:${userId}:profile`);
  
  return user;
}

async function test() {
  // Set user in cache
  const key = `user:42:profile`;
  await redis.set(key, '{"name":"Alice"}', "EX", 3600);
  
  // Read (cache hit)
  console.log(await redis.get(key));  // {"name":"Alice"}
  
  // Update (should invalidate)
  await updateUser(42, { name: "Bob" });
  
  // Read (should be miss, re-queries DB)
  const cached = await redis.get(key);
  console.log(cached);  // nil (invalidated!)
}
```

---

### **Project 3: Simulate Stampede**

```javascript
async function getHotUser(userId) {
  const key = `user:${userId}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  // Simulate slow query (200ms)
  await new Promise(r => setTimeout(r, 200));
  const user = { id: userId, name: "Alice" };
  await redis.set(key, JSON.stringify(user), "EX", 1);
  return user;
}

async function test() {
  // Warm cache
  await getHotUser(42);
  
  // Wait for key to expire
  await new Promise(r => setTimeout(r, 1100));
  
  // Hammer it with 100 concurrent requests right at expiry
  console.time("Stampede");
  const promises = [];
  for (let i = 0; i < 100; i++) {
    promises.push(getHotUser(42));
  }
  await Promise.all(promises);
  console.timeEnd("Stampede");
  // Output: ~200ms (all 100 requests hit DB simultaneously!)
}
```

---

### **Project 4: Fix Stampede with Lock**

```javascript
async function getHotUserWithLock(userId) {
  const key = `user:${userId}`;
  const lockKey = `lock:${key}`;
  
  // Try cache
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  // Try to acquire lock
  const acquired = await redis.set(lockKey, "1", "NX", "EX", 5);
  
  if (acquired) {
    // I own the lock → rebuild
    try {
      console.log("Rebuilding...");
      await new Promise(r => setTimeout(r, 200));  // Slow query
      const user = { id: userId, name: "Alice" };
      await redis.set(key, JSON.stringify(user), "EX", 60);
      return user;
    } finally {
      await redis.del(lockKey);
    }
  } else {
    // Someone else rebuilding → wait and retry
    console.log("Waiting for lock...");
    await new Promise(r => setTimeout(r, 250));
    return await getHotUserWithLock(userId);  // Retry
  }
}

async function test() {
  // Warm cache
  await getHotUserWithLock(42);
  
  // Wait for expiry
  await new Promise(r => setTimeout(r, 1100));
  
  // Hammer with 100 concurrent requests
  console.time("Stampede fixed");
  const promises = [];
  for (let i = 0; i < 100; i++) {
    promises.push(getHotUserWithLock(42));
  }
  await Promise.all(promises);
  console.timeEnd("Stampede fixed");
  // Output: ~250ms (only 1 rebuild!)
}
```

---

## Mastery Checklist (0/7 → 7/7)

### **1. ✅ I can explain the cache layers from browser to CDN to app to database**

**What this means:**
- Browser cache (cache-control, local storage)
- CDN cache (edge servers, static content)
- Reverse proxy cache (NGINX, rendered pages)
- Application cache (Redis, hot data)
- Database cache (internal buffers)

**Practice:**
- Trace a request through all layers
- Identify where each request type should be cached
- Explain latency at each layer

---

### **2. ✅ I can implement cache-aside and know when to prefer write-through or write-behind**

**What this means:**
- Cache-aside: Check cache, miss → query DB, populate cache
- Write-through: Write to both cache and DB synchronously
- Write-behind: Write to cache, async flush to DB

**Practice:**
- Implement cache-aside (with TTL)
- Measure before/after latency
- Implement write-through (slow writes but safe)
- Implement write-behind (fast writes but risky)

---

### **3. ✅ I choose TTLs from acceptable staleness and understand LRU vs LFU eviction**

**What this means:**
- TTL = How old data can safely be
- LRU = Evict least recently used (default, good)
- LFU = Evict least frequently used (alternative)

**Practice:**
- Decide TTL for different data types
- Explain trade-off (long = fewer misses, short = fresher)
- Configure Redis eviction policy

---

### **4. ✅ I design cache keys that include every input affecting the result**

**What this means:**
- Key = includes user_id, locale, filters, version, etc.
- Omitting input = silent data leak (wrong user sees wrong data)

**Practice:**
- Design cache keys for: user profile, product price, filtered list
- Identify what inputs affect output
- Verify key prevents data leaks

---

### **5. ✅ I invalidate cached entries on writes instead of relying on TTL alone**

**What this means:**
- TTL = passive (key expires eventually)
- Invalidation = active (delete on write)
- Combine both for safety

**Practice:**
- Write code: update DB, delete cache key
- Test: verify next read repopulates cache
- Check all write paths invalidate correct keys

---

### **6. ✅ I can describe a cache stampede and fix it with locks, coalescing, and jittered TTLs**

**What this means:**
- Stampede: Hot key expires, all concurrent requests miss
- Fix 1: Lock (only one rebuilds, others wait)
- Fix 2: Coalesce (dedupe identical in-flight loads)
- Fix 3: Jitter (randomize TTLs so not synchronized)
- Fix 4: Stale-while-revalidate (serve stale, refresh async)

**Practice:**
- Simulate stampede (100 concurrent on expired key)
- Measure DB load (should spike without fix)
- Add lock, re-run (DB load stays low)
- Add jitter, re-run (misses spread out)

---

### **7. ✅ I know when stale-while-revalidate is the right tradeoff for a hot key**

**What this means:**
- Serve cached value immediately (even if stale)
- Refresh in background
- Users never wait for cache rebuild
- Best for: hot keys, acceptable staleness

**Practice:**
- Mark key as "stale" (TTL approaching expiry)
- Return stale value immediately
- Refresh in background
- Verify user sees instant response

---

## Cheat Sheet (Quick Reference)

### **Cache-Aside Pattern**

```javascript
async function getItem(id) {
  const key = `item:${id}`;
  
  // Check cache
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  // Miss → load from DB
  const item = await db.get(id);
  
  // Store with TTL
  await redis.set(key, JSON.stringify(item), "EX", 300);
  
  return item;
}

async function updateItem(id, data) {
  // Write to DB
  const item = await db.update(id, data);
  
  // Invalidate cache
  await redis.del(`item:${id}`);
  
  return item;
}
```

### **TTL & Eviction**

```redis
SET key value EX 300              # 5-minute TTL
TTL key                           # Check remaining seconds
CONFIG SET maxmemory-policy allkeys-lru  # Eviction on overflow
```

### **Stampede Protection**

```javascript
// Lock
const lock = await redis.set(`lock:key`, "1", "NX", "EX", 5);

// Jitter
const ttl = 300 + Math.random() * 30;  // 300-330 seconds

// Stale-while-revalidate
if (stale) {
  returnImmediately(cachedValue);
  refreshInBackground();
}
```

---

## Common Pitfalls (Don't Do This)

### **Pitfall 1: No TTL + No Invalidation**

❌ **BAD:**
```redis
SET user:1 "alice"      # No TTL, forever cached!
# If data changes, never invalidated
# Serves stale forever
```

✅ **GOOD:**
```redis
SET user:1 "alice" EX 300   # TTL (self-heals)
# On update: DEL user:1     # Invalidate
```

---

### **Pitfall 2: Sloppy Cache Keys**

❌ **BAD:**
```javascript
// Omits locale
SET `product:42:price` "100"

// User 1 (USD): gets $100
// User 2 (EUR): also gets $100 (wrong!)
```

✅ **GOOD:**
```javascript
SET `product:42:price:USD` "100"
SET `product:42:price:EUR` "90"

// User 1: product:42:price:USD → $100 ✅
// User 2: product:42:price:EUR → €90 ✅
```

---

### **Pitfall 3: Forgetting to Invalidate**

❌ **BAD:**
```javascript
// Update DB
await db.query("UPDATE users ...");
// Forgot to invalidate! ❌
// Cache serves old data forever
```

✅ **GOOD:**
```javascript
// Update DB
await db.query("UPDATE users ...");
// Invalidate cache
await redis.del(`user:${id}`);
```

---

### **Pitfall 4: All Keys Expiring Together (Synchronized Stampede)**

❌ **BAD:**
```redis
SET key1 val EX 300
SET key2 val EX 300
SET key3 val EX 300
# All expire at same time
# → Synchronized stampede at T+300
```

✅ **GOOD:**
```javascript
const jitter = Math.random() * 30;
SET key1 val EX (300 + jitter)
SET key2 val EX (300 + jitter)
SET key3 val EX (300 + jitter)
# Staggered expiry → spread out misses
```

---

### **Pitfall 5: Caching Write-Heavy Data**

❌ **BAD:**
```
Cache data that changes constantly
Invalidate constantly
Hit rate: 10% (mostly misses)
Cost: Cache overhead without benefit
```

✅ **GOOD:**
```
Cache data that's:
- Read frequently
- Written infrequently
Hit rate: 80%+
Benefit: Massive speedup
```

---

## Interview Signals & Mastery

### **Signal of Mastery**

You can:
- ✅ Design cache keys that prevent data leaks
- ✅ Explain stampede and its 4 fixes
- ✅ Implement cache-aside with TTL
- ✅ Invalidate on every write path
- ✅ Choose TTL from staleness requirements
- ✅ Trade off between staleness and performance
- ✅ Know when to use each pattern (cache-aside, write-through, etc.)

### **Common Interview Questions**

1. "Design a caching strategy for a user profile" → Cache-aside + TTL + invalidate on write
2. "How do we avoid cache stampede?" → Lock + coalescing + jittered TTL
3. "What should the TTL be?" → Based on acceptable staleness
4. "How to prevent data leaks?" → Include all inputs in cache key
5. "Should we cache this?" → Only if read-heavy + acceptable staleness

---

## Summary

**Caching is fast but dangerous:**
- ✅ Speed: 200ms → 1ms (200x faster)
- ❌ Risk: Serves stale data if not invalidated

**Design for invalidation first, speed second:**
1. TTL (passive, self-healing)
2. Explicit invalidation on writes (active, immediate)
3. Cache keys that include all inputs (prevent leaks)
4. Stampede protection (locks, coalescing, jitter)
5. Measure cache hit rate (only cache if worthwhile)

**Master these 7 items and you're a caching expert:**
1. Explain cache layers
2. Implement cache-aside
3. Choose TTL + eviction policy
4. Design safe cache keys
5. Invalidate on writes
6. Prevent stampede
7. Use stale-while-revalidate

**Senior engineers don't just cache things — they cache strategically, with deep understanding of trade-offs.**

---

## Next Steps

- **Days 1-2:** Implement cache-aside (measure before/after)
- **Day 3:** Add TTL + invalidation (write updates)
- **Day 4:** Design cache keys (prevent leaks)
- **Day 5:** Simulate + fix stampede
- **Days 6-7:** Integration test (real queries, real cache)

By end of week: You can design caching strategies for production systems.

---

## You've Now Completed Month 1! 🎉

**Week 1:** Linux, networking, shells
**Week 2:** Docker containerization
**Week 3:** PostgreSQL databases
**Week 4:** Redis + Caching

Next 7 days: **REVISION & PRACTICE**
- Re-read all 4 README documents
- Build the practice projects
- Integrate into your getedutrack project

**September 20:** **WEEK 5 STARTS**
- APIs (REST, routing, error handling)
- Authentication (JWT, OAuth)
- Real-time (WebSockets)
- Rate limiting in production

You're building a production backend system piece by piece.
