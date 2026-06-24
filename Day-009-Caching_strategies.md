# Day 009 — Caching Strategies

---

Every time data is updated, the system faces a question: **update the cache first, or the database first?** And when reading, does the app manage the cache itself, or does the cache handle it automatically?

The answer depends on your consistency, performance, and reliability requirements. These five strategies cover the full picture.

---

## Part 1 — Write Strategies

---

## 1. Write-Through Cache

**Write to cache AND database at the same time. Both are always in sync.**

```
Application
     │
     ▼
  Cache ──────────────┐
     │                │  (same operation)
     ▼                ▼
Database         ← both updated
     │
Return Success
```

**Example — user updates their username:**
```
User → "name = Rahul"
  1. Write to Cache   ✅
  2. Write to Database ✅
  3. Return success
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Strong consistency — cache and DB always match | Higher write latency — two writes per operation |
| Fast reads — data is already cached after writing | Cache pollution — data written but never read again wastes memory |
| No cold-miss after a write | |

**Best for:** Banking, user profiles, config data — anywhere data correctness is non-negotiable.

---

## 2. Write-Back Cache (Write-Behind)

**Write to cache immediately. Update the database later in the background.**

```
Application
     │
     ▼
  Cache ──→ Return Success (immediately)
     │
     │  (async, later)
     ▼
Database
```

**Example — social media likes:**
```
User likes a post
  → Cache: like count +1 (instant)
  → Return success to user
  → Background worker batches 1000 likes → DB update once
```

Instead of 1000 individual DB writes, you do one bulk write. Massive reduction in DB load.

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Extremely fast writes — no DB wait | Data loss risk — if cache crashes before flushing, updates are lost |
| Reduced DB load — writes can be batched | Eventual consistency — DB temporarily has stale data |

**Best for:** Like counters, view counts, analytics, logging — high-volume writes where minor data loss is acceptable.

---

## 3. Write-Around Cache

**Writes go directly to the database. Cache is completely bypassed on writes.**

The cache only gets populated when data is actually read.

```
Write:  Application → Database (cache skipped)

Read:   Application → Cache
                        │ Miss
                        ▼
                    Database → store in Cache → return
```

**Example — order history:**
```
Order created → write directly to DB (cache untouched)

Later, user views order:
  → Cache miss → read from DB → store in cache → return
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| No cache pollution — only read data enters cache | First read after write = cache miss (slower) |
| Saves memory — rarely read data stays out | Write-heavy workloads still hit DB hard |

**Best for:** Product catalogs, historical records, audit logs — data that is written once and rarely read immediately after.

---

## Write Strategies Comparison

| Feature | Write-Through | Write-Back | Write-Around |
|---|---|---|---|
| Write speed | Medium | ⚡ Fastest | Medium |
| Read speed | Fast | Fast | First read is slow |
| Consistency | Strong | Eventual | Strong |
| Data loss risk | Low | ⚠️ High | Low |
| Cache pollution | High | High | Low |
| DB write load | Medium | Low | High |
| Best for | Banking, profiles | Counters, analytics | Catalogs, logs |

---
---

## Part 2 — Read Strategies

---

## 4. Cache-Aside (Lazy Loading)

**The application manages the cache manually. Check cache first — on a miss, load from DB and store in cache.**

The most widely used caching strategy.

```
Request → Check Cache
               │
         ┌─────┴─────┐
      Hit ✅       Miss ❌
         │             │
    Return data     Query DB
                        │
                   Store in Cache
                        │
                   Return data
```

**Example — product page:**
```
1st request: Product 101
  → Cache miss → DB query → store in cache → return

2nd request: Product 101
  → Cache hit → return immediately (fast)
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Only popular data gets cached — saves memory | First request is always slow (cold start) |
| Simple to implement — most apps use this | Cache stampede risk — many users miss at once |
| Reduces DB load over time | |

> **Cache Stampede:** If a popular item expires and 10,000 users request it simultaneously, all miss and all hit the DB at once. Fix with request locking or background refresh (see Day 004).

**Best for:** E-commerce, social media, most web applications.

---

## 5. Read-Through Cache

**The application only ever talks to the cache. The cache is responsible for loading from the DB on a miss.**

```
Application → Cache → (on miss) → DB → store → return
```

Looks similar to Cache-Aside, but the key difference is **who manages the DB fetch**:

| | Cache-Aside | Read-Through |
|---|---|---|
| Who fetches from DB on miss? | The application | The cache itself |
| Application complexity | Higher | Lower |
| Cache infrastructure complexity | Lower | Higher |
| Control over loading logic | Full control | Less control |

**Example:**
```
App asks Redis for user:101
Redis: not found → Redis fetches from DB → stores → returns to app
App never touched the DB directly
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Simpler application code | Cache layer is more complex to set up |
| Cache handles all DB interaction | Less control over how/when data is loaded |

**Best for:** Enterprise systems, managed caching solutions where you want the cache to be a transparent layer.

---
---

## How Big Systems Combine These Strategies

| System | Read Strategy | Write Strategy | Why |
|---|---|---|---|
| **Facebook Feed** | Cache-Aside | Write-Back | Massive read traffic + enormous write volume; minor staleness acceptable |
| **Amazon Product Catalog** | Cache-Aside | Write-Around | Millions of products; most are rarely read right after being updated |
| **Banking System** | Cache-Aside | Write-Through | Strong consistency required; data loss is not acceptable |

---

## All Five Strategies — Quick Reference

| Strategy | Type | Core Idea | Key Trade-off |
|---|---|---|---|
| Write-Through | Write | Cache + DB updated together | Consistent but slower writes |
| Write-Back | Write | Cache first, DB async later | Fastest writes but risk of data loss |
| Write-Around | Write | Skip cache on write, DB only | No pollution but cold miss on first read |
| Cache-Aside | Read | App checks cache, loads DB on miss | Simple but app manages everything |
| Read-Through | Read | Cache auto-loads from DB on miss | Simpler app but complex cache layer |

---

## Memory Trick

```
Write-Through  → Cache + DB together       (consistency first)
Write-Back     → Cache now, DB later       (speed first)
Write-Around   → DB only on write          (avoid pollution)

Cache-Aside    → App checks cache first    (most common read)
Read-Through   → Cache handles DB access   (transparent read)
```

---

## One-Line Formula

```
Consistency needed?  → Write-Through
Speed needed?        → Write-Back
Avoid stale cache?   → Write-Around
Standard reads?      → Cache-Aside
Transparent caching? → Read-Through
```
