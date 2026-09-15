# Month 1 Quick Reference Card
## One-Page Cheat Sheet for Backend Mastery

---

## Week 1: Linux & Networking in 60 Seconds

**File Permissions:** `rwxr-xr-x` = owner/group/others, read/write/execute
**Process Management:** `ps aux` (find) → `kill PID` (stop) → `journalctl -u service` (logs)
**Pipes:** `command1 | command2` chains stdout → stdin
**OSI Model:** Layer 3 (IP) → Layer 4 (TCP/UDP) → Layer 7 (HTTP/TLS)
**Tools:** `dig` (DNS), `ss` (ports), `curl` (HTTP), `tcpdump` (packets)

✅ **Master Test:** Run `trace.sh google.com` and explain each step

---

## Week 2: Docker in 60 Seconds

**Image = Blueprint** (Dockerfile → docker build → tagged image)
**Container = Instance** (docker run → running process)
**Layer Caching:** Stable first (base, deps) → changing last (code) = 3x faster
**Multi-Stage:** Builder stage (build) → Runtime stage (production only) = 40% smaller
**Compose:** One YAML file orchestrates multiple containers + networks

```dockerfile
# Production Dockerfile structure
FROM node:20-alpine AS builder
# Install deps
FROM node:20-alpine
# Copy only production artifacts
```

✅ **Master Test:** Docker build getedutrack in <3 minutes

---

## Week 3: PostgreSQL in 60 Seconds

**3NF:** Each fact once, no duplicates
**Constraints:** NOT NULL, UNIQUE, CHECK, FOREIGN KEY (database enforces rules)
**Indexes:** B-tree (default), composite (leftmost prefix), partial (WHERE), covering (INCLUDE)
**EXPLAIN:** Read plan inside-out, spot Seq Scan (bad), seek Index Scan (good)
**N+1 Problem:** Loop in app = N queries. Solution: JOIN = 1 query.

```sql
-- Bad: N+1
for each customer: SELECT * FROM orders WHERE customer_id = X

-- Good: Single join
SELECT * FROM customers c JOIN orders o ON c.id = o.customer_id
```

✅ **Master Test:** Optimize a slow query with EXPLAIN ANALYZE (100x faster)

---

## Week 4A: Redis in 60 Seconds

**Data Types:** Strings (INCR), Hashes (HSET), Lists (LPUSH), Sets (SADD), Sorted Sets (ZADD), Streams
**Rate Limit:** INCR key, if count > limit: reject, EXPIRE key to reset window
**Leaderboard:** ZADD board score player, ZREVRANGE board 0 9 (top 10)
**Lock:** SET lock token NX EX 10 (acquire), DEL if token matches (release)
**TTL:** EXPIRE key 60, TTL key, SET key val EX 300

✅ **Master Test:** Implement rate limiter + leaderboard in 30 minutes

---

## Week 4B: Caching in 60 Seconds

**Layers:** Browser → CDN → Proxy → App → Database (each closer = faster + harder to invalidate)
**Cache-Aside:** Check cache → miss → load DB → SET cache → return
**Invalidation:** DEL key on every write (not just TTL)
**Stampede:** When hot key expires, all concurrent requests miss. Fix: Lock (one rebuild) + Jitter (staggered TTL) + Coalesce (dedupe)
**Keys:** MUST include all inputs (user_id, locale, filters) or data leaks

```javascript
// Safe key: includes all inputs
`product:${productId}:price:${currency}`

// Unsafe key: omits locale
`product:${productId}:price`  // ❌ Wrong user gets wrong data
```

✅ **Master Test:** Design complete caching strategy (measure 100x speedup)

---

## The Full Stack

```
         Browser (Cache-Control)
              ↓ (10ms hit)
         CDN (Edge servers)
              ↓ (20ms hit)
         Reverse Proxy (NGINX)
              ↓ (1ms hit)
         Redis Cache
              ↓ (200ms miss)
         PostgreSQL Database
         ✅ Correct data
         ✅ Durable storage
```

---

## Performance Checklist

- [ ] Database indexed? (use EXPLAIN ANALYZE)
- [ ] Queries using indexes? (Index Scan not Seq Scan)
- [ ] Hot data cached? (Redis cache-aside)
- [ ] Cache invalidated on writes? (DEL key)
- [ ] TTL reasonable? (based on staleness tolerance)
- [ ] Stampede protected? (lock + jitter)
- [ ] Connections pooled? (PgBouncer in prod)

---

## Interview Traps to Avoid

❌ **"Database is slow"** → Actually, your queries are slow. Use EXPLAIN ANALYZE.
❌ **"Let's add a cache"** → What are you caching? How will you invalidate?
❌ **"Cache keys are fine"** → Do they include all inputs? Test with different users.
❌ **"TTL only"** → Combine TTL + explicit invalidation on writes.
❌ **"Redis is our database"** → No. PostgreSQL = source of truth. Redis = accelerator.
❌ **"Run KEYS * to debug"** → No. Blocks entire server. Use SCAN instead.

---

## 7-Day Revision Schedule

**Day 1 (Sep 13):** Linux - run trace.sh on 3 hosts
**Day 2 (Sep 14):** Docker - rebuild getedutrack, measure image size
**Day 3 (Sep 15):** PostgreSQL - optimize 3 slow queries
**Day 4 (Sep 16):** Redis - build rate limiter + leaderboard
**Day 5 (Sep 17):** Caching - implement cache-aside + invalidation
**Day 6 (Sep 18):** Integration - add Redis to getedutrack
**Day 7 (Sep 19):** Full test - trace request through all layers

---

## Measurement Checklist

```
Database Query:         200-300ms → 2-5ms (100x with index)
Cache Hit:              200ms    → 1-2ms (200x faster)
Docker Build:           6min     → 2-3min (3x with caching)
N+1 to Join:            100 hits → 1 hit  (100x faster)
Overall API Response:   250ms    → 5ms   (50x faster)
```

---

## Key Definitions

**ACID:** Atomic (all/nothing) + Consistent (constraints) + Isolated (no interference) + Durable (survives crash)

**MVCC:** Multi-Version Concurrency Control (readers don't block writers, no locks needed)

**3NF:** 3rd Normal Form (every non-key column depends on key, whole key, nothing but the key)

**Stampede:** Hot key expires → all concurrent requests miss → database slammed

**Cache-Aside:** Check cache → miss → load source → store in cache → return

**Invalidation:** Remove cached value when source changes (hard part of caching)

---

## Commands You'll Use Daily

```bash
# PostgreSQL
psql -h localhost -U user -d app
EXPLAIN ANALYZE SELECT ...
CREATE INDEX idx_name ON table (column);
ANALYZE table;

# Redis
redis-cli
PING
INCR counter
SET key val EX 300
TTL key
ZADD leaderboard score player
ZREVRANGE leaderboard 0 9 WITHSCORES

# Docker
docker build -t app:1.0 .
docker images
docker-compose up -d
docker-compose logs -f
docker exec -it container bash

# Linux
ps aux | grep process
kill -9 PID
grep pattern file | wc -l
dig domain.com
curl -v https://api.com
```

---

## What You Should Be Thinking

**Database question:** "What index would help?" → Check EXPLAIN ANALYZE
**Performance issue:** "Is it cached?" → Check Redis first
**Caching question:** "What's the invalidation strategy?" → This is the hard part
**Production concern:** "Can we scale?" → Measure first, then optimize what matters

---

## Success = These Statements

✅ "I can read a query plan and know exactly which index to add"
✅ "I can design cache keys that prevent data leaks"
✅ "I understand why cache invalidation is hard"
✅ "I can trace a request from browser to database"
✅ "I know when to cache and when not to"
✅ "I can fix a cache stampede in my head"
✅ "I choose data structures strategically (not just 'key-value')"

---

## September 20 Readiness

- [ ] All 5 README files read twice
- [ ] All 4 practice projects completed
- [ ] Redis cache added to getedutrack
- [ ] Query optimization measured (100x faster)
- [ ] Docker build optimized (<500MB)
- [ ] Can explain all 38 concepts in depth

**If all checked: You're ready for Week 5** ✅

---

## Your Competitive Edge

You understand what 80% of junior developers don't:
- Databases aren't just storage (they enforce correctness)
- Caching isn't just speed (it's about staleness tradeoff)
- Performance isn't magic (it's systematic measurement)
- Containers aren't just Docker (they're reproducibility)

**Use this knowledge ruthlessly in Week 5.**

---

**Print this. Reference daily. Master it.**

**Week 5 (Sep 20): APIs, Auth, Real-time — same depth, same mastery.**

🚀
