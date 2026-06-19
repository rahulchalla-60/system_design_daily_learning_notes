# Day 004 — Caching & CDN

---

## Part 1 — Caching

---

## What is Caching?

Caching is the process of **storing frequently accessed data in a faster storage layer** so future requests can be served without hitting the database every time.

**Without cache:**
```
Application → Database (every request)
```

**With cache:**
```
Application → Cache → (if miss) → Database
```

The first request fetches from the DB and stores in cache. Every subsequent request is served from cache — much faster.

---

## Why Do We Need It?

Every database query has a cost — time, CPU, memory. At scale this adds up fast:

```
1 DB query = 50ms
1M users   = DB overwhelmed
```

**Problems without caching:**
- High latency for users
- Database becomes a bottleneck
- More DB servers needed = higher cost
- Poor user experience under load

---

## Cache Hit vs Cache Miss

```
Request → Check Cache
               │
       ┌───────┴───────┐
    Hit ✅           Miss ❌
       │               │
  Return data      Query DB
  (fast ~2ms)     Store in Cache
                  Return data
                  (slow ~50ms)
```

**Cache Hit Ratio** — the most important cache metric:
```
Hit Ratio = Hits / Total Requests

Example: 900 hits + 100 misses = 900/1000 = 90%
```

> Aim for 90%+ in production. A low hit ratio means your cache isn't helping much.

---

## What Should (and Shouldn't) Be Cached?

| ✅ Good to Cache | ❌ Avoid Caching |
|---|---|
| User profiles | Real-time financial balances |
| Product listings | Frequently changing transactions |
| Home feeds | Highly personalised per-request data |
| Search results | |
| Config / feature flags | |
| Popular posts | |

---

## Cache Types

### 1. Client Cache
Stored on the user's device — browser cache, mobile app cache.
```
Browser → Local Storage / HTTP Cache
```
Fast, but only useful for that one user.

---

### 2. Application Cache
Stored in application memory (in-process).
```
Node.js App → In-Memory HashMap / LRU Cache
```
Fastest possible access, but not shared across app instances.

---

### 3. Distributed Cache
Shared across all application servers. The standard for production.
```
App1 ┐
App2 ├──► Redis Cluster (Node1, Node2, Node3)
App3 ┘
```
**Tools:** Redis (most common), Memcached

---

## Cache Levels

| Level | Location | Speed | Example |
|---|---|---|---|
| L1 | Inside application (in-memory) | Fastest | Local HashMap |
| L2 | Shared distributed cache | Fast | Redis |
| L3 | Database query cache | Slower | DB-level cache |

---

## Cache Eviction Policies

Cache has limited storage — old entries must be removed to make room for new ones.

### 1. FIFO — First In, First Out
Removes the oldest entry regardless of usage.
```
Inserted order: A → B → C → D
Cache full → remove A
```

---

### 2. LRU — Least Recently Used ⭐ Most Common
Removes the item that hasn't been accessed for the longest time.
```
Access order: A B C D → then access A
New order:    B C D A  ← B is now evicted next
```
**Best for:** General-purpose caching where recent access predicts future use.

---

### 3. LFU — Least Frequently Used
Removes the item with the lowest total access count.
```
A → accessed 100 times
B → accessed 5 times   ← evicted first
```
**Best for:** When popularity (not recency) predicts future use.

---

### 4. TTL — Time To Live
Every cache entry has an expiry time. Removed automatically after it expires.
```
Cache entry set at 10:00 with TTL = 1 hour
Entry auto-deleted at 11:00
```
**Best for:** Data that goes stale after a known period (e.g., session tokens, API responses).

---

## Cache Patterns

### 1. Cache Aside (Lazy Loading) — Most Common

The application manages the cache explicitly. Data is only loaded into cache on a miss.

```
Request → Check Cache
              │
           Miss?
              │
          Query DB → Store in Cache → Return
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Simple to implement | First request is always slow (cold start) |
| Cache only what's actually needed | Cache and DB can go out of sync |

**Best for:** Read-heavy workloads. Default choice for most systems.

---

### 2. Read Through

Application always reads from cache. Cache is responsible for fetching from DB on a miss.

```
Application → Cache → (on miss) → DB
```

Similar to Cache Aside, but the cache library handles the DB fetch — less code in the app.

---

### 3. Write Through

Every write goes to cache and DB simultaneously.

```
Application → Write to Cache + Write to DB (together)
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Cache always consistent with DB | Slower writes (two operations) |
| No stale data | Caches data that may never be read |

**Best for:** Systems where read consistency is critical.

---

### 4. Write Back (Write Behind)

Write to cache first, DB updated asynchronously in the background.

```
Application → Cache (immediate) → DB (async, later)
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Very fast writes | Risk of data loss if cache crashes before DB sync |
| Reduces DB write pressure | More complex to implement |

**Best for:** High write-throughput systems where small data loss is acceptable (e.g., metrics, counters).

---

## Cache Pattern Summary

| Pattern | Read From | Write To | Consistency | Speed |
|---|---|---|---|---|
| Cache Aside | Cache, then DB | DB only | Eventual | Fast reads |
| Read Through | Cache (auto-fetches) | DB only | Eventual | Fast reads |
| Write Through | Cache | Cache + DB | Strong | Slower writes |
| Write Back | Cache | Cache first, DB async | Eventual | Fastest writes |

---

## Cache Problems

### Cache Stampede (Thundering Herd)
A popular cache entry expires. Thousands of requests all miss simultaneously and hammer the DB.

```
TTL expires → 100K requests → all hit DB → DB crashes
```

**Fix:** Use request locking (only one request fetches from DB, others wait) or background refresh before TTL expires.

---

### Cache Penetration
Requests for data that doesn't exist in cache OR database (e.g., made-up user IDs). Every request hits the DB uselessly.

```
GET user/99999999 → cache miss → DB miss → repeat forever
```

**Fix:**
- **Bloom Filter** — probabilistic check before hitting DB
- **Null Caching** — cache the "not found" result with a short TTL

---

### Cache Avalanche
Many cache entries expire at the same time, causing a sudden spike of DB queries.

```
1000 entries all set with TTL = 1 hour
→ All expire at same time → DB overwhelmed
```

**Fix:** Add random jitter to TTL values so expiry is spread out.

```
TTL = 3600 + random(0, 300)  ← entries expire at slightly different times
```

---

## Distributed Caching

A single cache server eventually becomes a bottleneck. Distribute across a cluster:

```
App1 ┐
App2 ├──► Redis Cluster
App3 ┘       │
         ┌───┼───┐
       Node1 Node2 Node3
```

**Data partitioning example:**
```
Users A–F → Node 1
Users G–M → Node 2
Users N–Z → Node 3
```

---

## Cache Consistency

When the DB is updated, the cache may still hold stale data. Three main solutions:

| Strategy | How It Works |
|---|---|
| **TTL** | Cache expires automatically after set time |
| **Cache Invalidation** | Explicitly delete cache entry when DB is updated |
| **Write Through** | Always write to cache and DB together |

---

## Cache Monitoring Metrics

| Metric | What to Watch For |
|---|---|
| Hit Ratio | Should be > 90% |
| Miss Ratio | High miss = poor cache design |
| Latency | Cache reads should be < 5ms |
| Memory Usage | Watch for eviction pressure |
| Eviction Rate | High rate = cache too small |

---
---

## Part 2 — CDN (Content Delivery Network)

---

## What is a CDN?

A CDN is a **globally distributed network of servers** that caches content physically closer to users to reduce latency.

**Without CDN:**
```
User (India) → Request → Server (USA) → Response
Distance = ~13,000 km → High latency
```

**With CDN:**
```
User (India) → Request → CDN Edge (Mumbai) → Response
Distance = ~30 km → Low latency
```

---

## CDN Components

| Component | Role |
|---|---|
| **Origin Server** | The source of truth — where your actual content lives |
| **Edge Server (PoP)** | CDN server deployed close to users worldwide |
| **CDN Provider** | The company operating the edge network |

**Popular CDN Providers:** Cloudflare, AWS CloudFront, Akamai, Fastly

---

## CDN Architecture

```
              Origin Server (USA)
                      │
         ┌────────────┼────────────┐
         │            │            │
    USA Edge      India Edge    EU Edge
         │            │            │
      US Users    India Users   EU Users
```

---

## How CDN Works

```
1. User requests logo.png
         ↓
2. DNS routes to nearest Edge Server
         ↓
3. Edge checks its cache
         ↓
   ┌──────┴──────┐
 Hit ✅        Miss ❌
   │              │
Return file    Fetch from Origin
               Store at Edge
               Return file
```

---

## CDN Delivery Strategies

### Pull CDN — Most Common
CDN fetches content from origin automatically on first request. Edge caches it for future users.

```
User → CDN → (if not cached) → Origin → cache at CDN → serve user
```
**Best for:** Most web applications. Zero setup on origin side.

---

### Push CDN
You (the origin) proactively upload content to the CDN before users request it.

```
Origin → push content → CDN → User
```
**Best for:** Large files (videos, software downloads) that you know will be requested heavily.

---

## What CDN Caches Well

| ✅ Great for CDN | ❌ Hard to cache with CDN |
|---|---|
| Images, videos, audio | Real-time data |
| CSS, JavaScript, fonts | Shopping carts |
| HTML pages | Banking dashboards |
| PDFs, downloads | User-specific dynamic content |

---

## TTL and Cache Invalidation in CDN

**TTL** — defines how long the edge server keeps the cached copy before re-fetching from origin.
```
TTL = 24 hours → edge serves cached file for 24h before checking origin
```

**Cache Invalidation** — forcing the CDN to drop stale content before TTL expires:

| Method | How |
|---|---|
| **Purge** | Explicitly delete a specific file from all edges |
| **Versioning** | Change the filename (`logo-v2.png`) — CDN treats it as new |
| **Expire headers** | Set short TTL, let it naturally refresh |

---

## CDN Security Features

| Feature | What It Does |
|---|---|
| **DDoS Protection** | Absorbs attack traffic at the edge before it reaches origin |
| **WAF (Web App Firewall)** | Blocks SQL injection, XSS, bot traffic |
| **SSL/TLS Termination** | Handles HTTPS at the edge, reducing origin load |
| **Anycast Routing** | Multiple edges share one IP; user automatically hits nearest one |

---

## CDN Benefits & Limitations

| ✅ Benefits | ❌ Limitations |
|---|---|
| Dramatically lower latency for global users | Dynamic content is hard to cache |
| Reduces origin server load | Cache invalidation can be complex |
| Handles traffic spikes gracefully | Adds cost |
| Built-in DDoS protection | Stale content if TTL misconfigured |
| High availability — reroutes if edge fails | |

---
---

## Cache vs CDN — Side by Side

| Feature | Cache (e.g., Redis) | CDN (e.g., CloudFront) |
|---|---|---|
| Purpose | Reduce database/backend load | Reduce network latency |
| Location | Near the application server | Near the end user |
| What it stores | Application data (JSON, objects) | Static assets (images, CSS, JS) |
| Scope | Internal (your infrastructure) | Global (distributed worldwide) |
| Main benefit | Faster data access | Faster content delivery |

> They solve different problems — most production systems use **both**.

---

## Complete Production Architecture

```
User
  │
CDN (edge cache — static assets)
  │
Load Balancer
  │
Application Servers
  │
Redis Cache (app data cache)
  │
Database (primary + replicas)
```

---

## Full Request Flow Examples

**Static asset (image/video):**
```
User → CDN Edge → Hit? → Serve immediately
                → Miss? → Fetch Origin → Cache at Edge → Serve
```

**Application data (user profile, feed):**
```
User → App → Redis Cache → Hit? → Return data
                         → Miss? → Query DB → Store in Redis → Return
```

---

## Quick Reference Cheat Sheet

### Caching
| Concept | One-Line Summary |
|---|---|
| Cache Hit | Data found in cache — fast response |
| Cache Miss | Data not in cache — must query DB |
| LRU | Evict least recently used item |
| LFU | Evict least frequently used item |
| TTL | Auto-expire entries after a set time |
| Cache Aside | App checks cache, falls back to DB on miss |
| Write Through | Write to cache and DB together |
| Write Back | Write to cache first, DB updated async |
| Cache Stampede | Mass cache miss → DB overload |
| Cache Avalanche | Mass TTL expiry → DB overload |
| Cache Penetration | Repeated requests for non-existent data |

### CDN
| Concept | One-Line Summary |
|---|---|
| Origin Server | Source of content |
| Edge Server / PoP | CDN server near the user |
| Pull CDN | Edge fetches from origin on first request |
| Push CDN | You upload content to CDN proactively |
| TTL | How long edge caches content before refreshing |
| Purge | Force-delete cached content before TTL expires |
| Anycast | Multiple edges share one IP; routes to nearest |
| WAF | Firewall at the edge blocking malicious traffic |

---

## One-Line Formulas

```
Cache = Faster Data Access + Reduced DB Load + Lower Latency + Higher Throughput

CDN   = Global Content Distribution + Lower Network Latency + Reduced Origin Load + DDoS Protection
```

---

## The Internet-Scale Formula

```
CDN
  + Load Balancer
  + Application Servers
  + Redis Cache
  + DB Replication
  + DB Sharding
= Internet-Scale System
```
