# System Design Interview Questions & Answers

---

## How to Use This File

Each question follows this format:
- **The question** — what the interviewer is actually testing
- **The answer** — the key insight, not just a definition
- **The trap** — the common mistake that costs candidates points

Read one section per day. For each answer, try to say it out loud in your own words before reading — that's how you know if you actually own it.

---

## Section 1 — Foundations & Trade-offs

---

### Q1. What's the difference between scalability and performance?

**Performance** is how fast a system responds for a single user/request — low latency, fast response time.

**Scalability** is how well the system *maintains* that performance as load grows.

A system can be fast for one user but not scalable — e.g., a single-threaded server that's blazing fast alone but has no concurrency model and falls over at 10k concurrent users.

> **Trap:** Treating these as the same thing. An interviewer asking about "performance at scale" wants both addressed separately.

---

### Q2. Latency vs throughput — why can't you always maximise both?

- **Latency** = time for a single request to complete
- **Throughput** = number of requests processed per unit time

Batching increases throughput (fewer round trips, better resource utilisation) but increases latency for each individual request because it must wait in the batch.

**Rule of thumb:**
- Batch ETL jobs → optimise for throughput
- APIs serving a live UI → optimise for latency

> **Trap:** Designing a low-latency API with batching "for efficiency" — you've accidentally built the wrong thing.

---

### Q3. Explain the CAP theorem, and why "choose 2 of 3" is misleading.

CAP says a distributed system can't simultaneously guarantee **C**onsistency, **A**vailability, and **P**artition tolerance during a network partition.

But partition tolerance isn't optional — networks *will* partition. So the real choice is: **during a partition, do you sacrifice C or A?**

| Choice | Behaviour | Example |
|---|---|---|
| CP | Rejects requests rather than return stale data | Traditional RDBMS with majority-quorum replicas |
| AP | Returns possibly-stale data rather than error | DynamoDB, Cassandra |

> **Trap:** Saying "I'll pick C and A" — that's not a valid choice in a distributed system.

---

### Q4. What is the PACELC theorem and how does it extend CAP?

PACELC adds a second dimension: **even when there's no partition**, you're making a trade-off.

```
If Partition:   choose Availability or Consistency  (= CAP)
Else (normal):  choose Latency or Consistency
```

Strongly consistent systems (waiting for all replicas to ack) pay a latency tax on every write, not just during partitions. PACELC is the more honest framing of the trade-off you're always making.

---

### Q5. Horizontal vs vertical scaling — what breaks each?

| | Vertical (Scale Up) | Horizontal (Scale Out) |
|---|---|---|
| Method | Bigger machine | More machines |
| Code changes needed | None | Requires stateless app + load balancer |
| Ceiling | Hardware limit | Near-unlimited |
| Fault tolerance | Single point of failure | High |

> **Trap:** Teams vertically scale for too long because it's "free" architecturally, then hit a cliff and need to re-architect under production pressure.

---

### Q6. Why is "stateless services" a core scalability principle?

If a service holds session state in local memory, every request from the same user must hit the same server (sticky sessions). This:
- Defeats load balancing
- Makes that server a SPOF for that user
- Makes rolling deployments painful

Externalising state (Redis, DB, JWT tokens) lets any instance handle any request — true horizontal scaling.

---

### Q7. What is a Single Point of Failure (SPOF), and how do you eliminate one?

Any component whose failure takes down the entire system.

You eliminate it via **redundancy at every layer**: multiple instances behind a load balancer, multi-AZ databases, replicated queues.

> **Trap:** Adding a second instance doesn't remove the SPOF if both share a dependency (same DB, same rack power). You've just moved the SPOF one level down.

---

## Section 2 — Networking, Load Balancing, DNS

---

### Q8. Layer 4 vs Layer 7 load balancing — when do you pick each?

| | L4 (Transport) | L7 (Application) |
|---|---|---|
| Routing based on | IP + port | URL path, headers, cookies |
| SSL termination | No | Yes |
| CPU overhead | Low | Higher (parses payload) |
| Use case | Raw throughput (game servers) | Microservices routing |

L7 can route `/api/v1` to one service and `/static` to another. L4 can't inspect the request at all — it just forwards TCP packets.

---

### Q9. Round robin vs least connections vs consistent hashing — what fails with plain round robin?

**Round robin** rotates requests in strict order, ignoring actual server load. A slow or expensive request piles up on a server, which then immediately gets the next request.

**Least connections** tracks active connections and routes to the least-loaded server — better for requests with uneven cost.

**Consistent hashing** routes based on a hash of the request (e.g., user ID) so the same client always hits the same backend — critical for cache locality. Requires virtual nodes to avoid uneven distribution when servers are added/removed.

---

### Q10. What is DNS round robin and why is it a weak load-balancing mechanism?

DNS returns multiple IPs in rotating order, clients pick one (usually the first). Problems:
- DNS responses are cached at OS/browser/ISP level — a dead server keeps getting traffic until caches expire
- No health awareness — DNS doesn't know if a backend is actually up
- Client behaviour is uncontrolled — some clients always pick the first IP

Fine as a first layer in front of real load balancers. Terrible as the only mechanism.

---

### Q11. What does a CDN do, and why is "just cache everything" naive?

A CDN caches content at edge nodes close to users, cutting latency and offloading origin servers.

The naive mistake is caching personalised or frequently-changing content (e.g., a user's dashboard). This either:
- Serves stale/wrong data to the wrong user, or
- Requires TTLs so short the cache barely helps

Good CDN design separates static assets (images, JS) from dynamic responses, and uses cache-control headers, surrogate keys, and edge-side includes for partial page caching.

---

### Q12. What's the difference between a forward proxy and a reverse proxy?

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| Sits in front of | Clients | Servers |
| Hides | Client identity | Server identity |
| Used for | Corporate filtering, VPN | Load balancing, SSL termination |
| Example | Squid | Nginx, AWS ALB |

> **Trap:** Confusing the two in an interview is a red flag — they solve opposite problems.

---

### Q13. What happens if you don't set connection timeouts between services?

A slow downstream service causes calling threads to pile up waiting indefinitely. In a thread-per-request model, this exhausts the thread pool and the failure cascades upstream — one slow service takes down the whole call chain.

The fix requires **both** timeouts AND circuit breakers. A timeout alone just means you retry into the same broken service repeatedly.

---

## Section 3 — Databases

---

### Q14. SQL vs NoSQL — what's the real decision criterion?

The real driver is not "structured vs unstructured data" (you can store JSON in Postgres). It's:

| Need | Choice |
|---|---|
| ACID transactions, complex joins, strong consistency | SQL |
| Horizontal write scale, flexible schema, eventual consistency acceptable | NoSQL |

> **Trap:** Saying "NoSQL because our data is flexible." That's not a technical reason.

---

### Q15. What is database indexing, and why do naive engineers over-index?

An index (typically a B+ tree) lets the DB find rows without a full table scan, at the cost of extra storage and slower writes (every insert/update must also update the index).

Over-indexing every column "just in case" slows writes and bloats storage — especially bad on high-write tables.

The senior move: index based on **actual query patterns** (WHERE, JOIN, ORDER BY columns), not defensively.

---

### Q16. Normalisation vs denormalisation — when is denormalising correct, not a shortcut?

**Normalisation** eliminates redundancy — single source of truth, but requires joins to reassemble data (expensive at scale).

**Denormalisation** duplicates data to avoid joins — trades storage and write complexity for read speed.

It's the right call for read-heavy systems: storing a user's display name inline on every comment row instead of joining the users table on every feed read. Not laziness — a deliberate consistency-for-speed trade-off.

---

### Q17. Synchronous vs asynchronous replication — what does each sacrifice?

| | Synchronous | Asynchronous | Semi-synchronous |
|---|---|---|---|
| Waits for replica ACK? | Yes | No | 1 replica |
| Data loss on crash? | None | Possible | Minimal |
| Write latency | Higher | Lower | Middle ground |

Most production systems use async replication with semi-sync for critical tables.

---

### Q18. What is replication lag and why can it break your application silently?

Async replicas can fall behind the primary. If your app writes to the primary then reads from a replica, you get stale data — the classic "read-your-own-writes" bug.

Example: user updates profile → refreshes → sees old value.

**Fixes:**
- Route the immediate follow-up read to the primary
- Use session-consistency tokens
- Accept eventual consistency in the UX where appropriate

---

### Q19. Database sharding — what's the hardest part people underestimate?

The hard part isn't the initial split. It's:
1. **Cross-shard queries** — a query needing data from two shards requires application-level scatter-gather
2. **Resharding** — when a shard gets too big, live data migration is expensive and risky
3. **Hot shards** — skewed key distribution (celebrity user overloading one shard)

> **Trap:** Saying "just add more shards" as if it's free. Each shard adds operational overhead.

---

### Q20. Range-based vs hash-based sharding — what does each get wrong?

| | Range-based | Hash-based |
|---|---|---|
| Range queries | Fast | Expensive (all shards) |
| Distribution | Uneven (hotspot on latest shard) | Even |
| Resharding cost | Lower | Higher (most keys remap) |

Real systems use **consistent hashing with virtual nodes** — even distribution + minimal data movement when adding/removing shards.

---

### Q21. What is a database connection pool, and what happens without one?

Opening a DB connection is expensive (TCP handshake, auth, TLS). Without pooling, every request opens a fresh connection → slow + can exhaust the DB's max-connections limit under load.

> **Trap:** Oversizing the pool. Too many connections can itself overwhelm the DB. Pool size should map to DB capacity, not just app concurrency.

---

### Q22. What's the N+1 query problem and why does it survive code review?

Fetch a list (1 query) then loop over it querying related data individually (N queries). Example: fetch 50 orders, then query each order's customer separately.

It survives review because it works fine in dev with small datasets and only blows up at production scale.

**Fix:** Eager loading, JOINs, or batched `WHERE id IN (...)` queries.

---

## Section 4 — Caching

---

### Q23. Where can you cache, and why does caching at every layer create a coherence problem?

Cache layers (outermost to innermost):
1. Client (browser cache)
2. CDN (edge cache)
3. API Gateway cache
4. Application (in-memory)
5. Distributed cache (Redis)
6. DB query cache

Every layer speeds up reads but is also a place where **stale data hides**. When you update the source of truth, you must invalidate at every layer — or users see inconsistent results depending on which cache path served them.

---

### Q24. Cache-aside vs write-through vs write-back — what does each optimise for?

| Strategy | Write goes to | Optimises for | Risk |
|---|---|---|---|
| **Cache-aside** | DB only (cache populated on read miss) | Cache memory efficiency | First read is slow; stale on missed invalidation |
| **Write-through** | Cache + DB together (sync) | Read freshness | Slower writes; caches data that may never be read |
| **Write-back** | Cache first, DB async later | Write speed | Data loss if cache crashes before flushing |

> **Trap:** Picking write-back purely for speed without mentioning the data loss risk.

---

### Q25. What is cache stampede, and how do you prevent it?

When a popular cache key expires, many concurrent requests all miss at once and hammer the DB simultaneously — potentially crashing it from that single spike.

**Prevention:**
- **Request coalescing / mutex lock** — only one request recomputes, others wait
- **Jittered TTLs** — spread expiry times so keys don't all expire together
- **Stale-while-revalidate** — serve the old value while refreshing asynchronously

---

### Q26. Cache eviction policies — when is LRU the wrong choice?

| Policy | Evicts | Best for |
|---|---|---|
| LRU | Least recently used | General-purpose hot data |
| LFU | Least frequently used | Access-frequency-heavy workloads |
| FIFO | Oldest inserted | Simple queue-style caches |
| TTL | Expired entries | Time-sensitive data |

**LRU fails** when access patterns are scan-heavy — a batch job scanning millions of records once floods "recently used" with one-time data, evicting your actual hot working set. LFU handles this better since scanned items don't build frequency.

---

### Q27. Distributed cache vs local in-process cache — why use both together?

| | Local (in-process) | Distributed (Redis) |
|---|---|---|
| Latency | Zero network cost | One network hop |
| Scope | Per-instance (inconsistent across servers) | Shared across all instances |
| Size | Limited to one machine's RAM | Large cluster |
| Survives restart | No | Yes |

Many high-performance systems use a **two-tier cache**: local for ultra-hot data (short TTL to bound staleness) + Redis as the shared source + DB as the origin.

---

## Section 5 — Messaging, Queues, Async Systems

---

### Q28. Why use a message queue instead of direct synchronous calls?

Direct calls couple the caller's availability to the callee's:
- If downstream is slow, caller blocks
- If downstream is down, caller fails
- Traffic spikes crash the consumer directly

A queue decouples them: producer fires-and-forgets, consumer processes at its own pace, absorbing spikes as a buffer.

**The cost:** you lose the immediate response. You need acknowledgement, retries, and eventual result delivery (polling, webhook, or WebSocket).

---

### Q29. At-most-once vs at-least-once vs exactly-once — why is "exactly-once" mostly a marketing term?

| Guarantee | Behaviour | Risk |
|---|---|---|
| At-most-once | Message may be lost, never duplicated | Data loss |
| At-least-once | Always delivered, may be duplicated | Duplicate processing |
| Exactly-once | Delivered exactly once | Extremely hard to guarantee |

Most systems that claim exactly-once actually implement **at-least-once + idempotent processing** (dedup by message ID). True transport-level exactly-once requires distributed transactions across two systems, which is expensive and rarely worth it.

---

### Q30. What is idempotency and why is it critical in distributed systems?

An idempotent operation produces the same result no matter how many times it's applied.

- Safe: `SET balance = 100` (idempotent)
- Unsafe: `ADD 10 TO balance` (not idempotent — retrying double-charges)

Networks fail and retries are unavoidable. Any non-idempotent operation risks corruption on retry.

**Standard fix:** idempotency keys — a client-generated unique request ID the server stores and deduplicates on.

---

### Q31. What is a Dead Letter Queue (DLQ) and why is skipping one a production incident waiting to happen?

When a message repeatedly fails processing, a DLQ routes it aside after N retries instead of retrying forever.

Without a DLQ:
- Poison messages (bad data, bugs) retry infinitely, wasting resources
- The queue may appear fine while silently dropping real data
- You discover the problem only when "why hasn't X appeared in 3 days?"

---

### Q32. Pub/Sub vs point-to-point queues — how does delivery differ?

| | Point-to-point | Pub/Sub |
|---|---|---|
| Each message consumed by | Exactly one consumer | All subscribers |
| Good for | Work distribution, load balancing | Broadcasting an event to many systems |
| Example | SQS FIFO queue | Kafka topic with multiple consumer groups |

> **Trap:** Using pub/sub for task queues causes **duplicate task execution** across all consumers instead of splitting the work.

---

### Q33. What is backpressure, and what breaks without it?

Backpressure is a mechanism for a slow consumer to signal a fast producer to slow down.

Without it, the producer keeps pushing faster than can be consumed. Buffers grow unbounded until memory is exhausted → OOM crash → outage.

**Solutions:**
- Bounded queues that block or reject when full
- Rate limiting the producer
- Reactive streams that propagate demand signals upstream

---

### Q34. What is the Saga pattern and when do you need it?

The Saga pattern manages distributed transactions across multiple services without a two-phase commit.

**Why needed:** In microservices, a "place order" action might span Order Service + Payment Service + Inventory Service. You can't use a SQL `BEGIN...COMMIT` across three databases.

**Two approaches:**
- **Choreography** — each service publishes events and reacts to others (decoupled, harder to debug)
- **Orchestration** — a central coordinator tells each service what to do and handles failures (easier to trace, single point of control)

Each step has a compensating action: if Payment fails, the Order is cancelled and Inventory is released.

---

## Section 6 — Consistency, Availability & Distributed Coordination

---

### Q35. Strong consistency vs eventual consistency — give a concrete example of each.

| | Strong Consistency | Eventual Consistency |
|---|---|---|
| Every read sees | Latest write immediately | Latest write eventually |
| Example | Bank balance after transfer | Instagram like count |
| Cost | Higher latency, lower availability | Lower latency, temporary staleness |

Choosing strong consistency everywhere "to be safe" kills performance for data where staleness genuinely doesn't matter.

---

### Q36. What is a quorum, and how does W + R > N guarantee consistency?

With N replicas:
- **W** = number of nodes that must acknowledge a write
- **R** = number of nodes read from

If W + R > N, every read set overlaps with every write set by at least one node → the read is guaranteed to see the latest write.

**Example:** N=3, W=2, R=2 → 2+2=4 > 3 ✅

Trade-off: higher W = safer writes but slower; higher R = safer reads but slower.

---

### Q37. What is a distributed lock, and why is a plain Redis SET NX dangerous?

A naive Redis lock (`SET key val NX EX ttl`) can fail because:

1. If the process pauses (GC, network blip) longer than the TTL, the lock expires while it's still "held" — another process acquires it → two processes think they hold the lock (split brain)
2. If Redis fails over to a replica that hadn't replicated the lock key, the new primary doesn't know the lock exists

**Proper solution:** fencing tokens — a monotonically increasing number given to the lock holder, checked by the resource being protected. Even if a stale lock holder acts, the resource rejects its request based on the token.

---

### Q38. What is leader election and why is it needed even in an "all replicas equal" system?

Some operations need exactly one active decision-maker to avoid conflicting writes. Leader election (Raft, Paxos, etcd) picks that node and detects failure to promote a new one.

Without it, two nodes can independently believe they're in charge and issue conflicting writes — **split brain**.

---

### Q39. Explain the Raft algorithm at a high level.

Raft gets a cluster to agree on a single ordered sequence of operations (a replicated log) despite partial failures:

1. **Leader election** — randomised timeouts avoid split votes; one leader is elected
2. **Log replication** — leader replicates entries to followers
3. **Commit** — entry is committed only once a majority of nodes acknowledge it

The problem it solves isn't storage — it's **agreement despite partial failure**: all nodes end up with the same ordered history even if some crash mid-way.

---

### Q40. What are vector clocks and what problem do they solve over timestamps?

Wall-clock timestamps aren't reliable across machines (clock drift, no global clock). Vector clocks track **causality** — a counter per node — letting the system determine:
- Did event A happen before event B?
- Or were they concurrent (conflicting)?

This is how DynamoDB detects and surfaces conflicting concurrent writes to the same key, instead of silently discarding one update with "last write wins."

---

### Q41. Optimistic vs pessimistic concurrency control — when does each break down?

| | Pessimistic | Optimistic |
|---|---|---|
| Method | Lock before read (`SELECT FOR UPDATE`) | Check version at commit, retry if changed |
| Safe | Always | Under low contention |
| Problem | Reduces concurrency, risk of deadlocks | High contention → constant retries (thrashing) |
| Best for | High-conflict data (financial ledgers) | Low-conflict data (user profiles) |

---

## Section 7 — API & Service Design

---

### Q42. REST vs gRPC — what's the actual trade-off?

| | REST + JSON | gRPC + Protobuf |
|---|---|---|
| Format | Human-readable | Binary (smaller, faster) |
| Browser support | Universal | Limited |
| Debugging | curl, browser | Needs tooling |
| Codegen | Optional | Built-in, strongly typed |
| Best for | Public APIs, diverse clients | Internal service-to-service |

> **Trap:** Picking gRPC for a public API (alienates simple integrations) or REST for a latency-critical internal hot path (leaves performance on the table).

---

### Q43. Rate limiting — token bucket vs fixed window, what's the flaw in fixed window?

**Fixed window** (e.g., 100 req/min, resets at :00):
- A client sends 100 requests at 0:59 and another 100 at 1:00 = **200 requests in 2 seconds** — twice the limit, right at the window boundary

**Token bucket** (tokens refill continuously, each request consumes one):
- No hard reset boundary
- Allows brief bursts up to the bucket size
- Enforces steady average rate
- Used by AWS, Stripe, most production systems

---

### Q44. What's an API gateway and what breaks when every microservice handles its own cross-cutting concerns?

An API gateway centralises: auth, rate limiting, logging, routing, SSL termination.

Without it, every service reimplements these independently → inconsistent behaviour, duplicated effort, larger attack surface (auth implemented differently in 20 places).

---

### Q45. Why does API versioning matter?

Once clients depend on a response shape, changing it in place breaks every consumer not updated simultaneously — impossible to coordinate across external clients.

Versioning (`/v1/`, header-based) lets you introduce breaking changes while old clients keep working, giving a migration window.

---

### Q46. Synchronous vs long polling vs webhooks vs WebSockets — when to use each?

| Mechanism | How | Best for | Drawback |
|---|---|---|---|
| Sync request-response | Client asks, waits | Simple request/reply | Wasteful for infrequent events |
| Long polling | Request held open until data arrives | Infrequent push, simple clients | One connection per waiting client |
| Webhooks | Server POSTs to client URL | Server → client push | Client must expose a public endpoint |
| WebSockets | Persistent bidirectional connection | Chat, live dashboards, games | More server-side connection state |

---

## Section 8 — Reliability & Fault Tolerance

---

### Q47. Circuit breaker vs retries — why are retries alone dangerous?

Retries alone multiply load on an already-struggling service ("retry storm"). A circuit breaker monitors the failure rate and:

1. **Closed** — requests pass through normally
2. **Open** (threshold crossed) — fails fast, stops sending requests entirely for a cooldown
3. **Half-open** — sends a test request to see if the service recovered

Without a circuit breaker, retries kick a service while it's down. Used by: Netflix Hystrix, Resilience4j.

---

### Q48. Graceful degradation vs failing hard — how do you decide which per feature?

**Graceful degradation:** keep core functionality working, fail non-critical features silently.
- Recommendations service down → hide the section, checkout still works ✅

**Fail hard:** block the entire request on failure.
- Payment service down → hard-fail the checkout (you can't silently skip charging) ✅

**The senior instinct:** classify every dependency as critical or optional at design time, not when the outage happens.

---

### Q49. What is chaos engineering?

Deliberately injecting failures (killing instances, adding latency, dropping packets) in production or production-like environments to verify the system actually handles failure the way it was designed to.

Most failure-handling code is untested until a real outage reveals an undiscovered shared dependency. Chaos engineering converts **assumed resilience** into **verified resilience**.

---

### Q50. Health check vs readiness check — what breaks if you conflate them?

| | Liveness (Health) Check | Readiness Check |
|---|---|---|
| Question | Is this process alive? | Is this process ready for traffic? |
| Action on failure | Restart the container | Remove from load balancer rotation |
| Example failure | Deadlock, OOM crash | Still warming cache, DB not connected |

If you route traffic based only on liveness, a container that's alive-but-not-ready (still starting up) gets real user requests and errors immediately.

---

### Q51. Explain the bulkhead pattern.

Ships are divided into watertight compartments — a breach floods one section, not the whole ship.

Applied to systems: isolate resource pools (thread pools, connection pools) per dependency. If calls to Service A hang, they exhaust only Service A's dedicated pool, not the shared pool used by Service B.

Without bulkheads, one misbehaving dependency exhausts a shared resource pool and takes down calls to completely unrelated, healthy services.

---

### Q52. What is a pre-signed URL and why does it matter for media uploads?

A pre-signed URL is a temporary, signed URL that allows a client to upload directly to object storage (S3/GCS) without routing through your application servers.

**Without it:**
```
Client → (500 MB file) → API Server → Object Storage
         ↑ API bandwidth bottleneck
```

**With it:**
```
Client → POST /upload/initiate → API Server → returns signed URL
Client → (500 MB file) → Object Storage directly
API server never touches the file bytes
```

The URL contains a cryptographic signature, expiry time, and scoped permissions — it can't be used to overwrite other objects or remain valid indefinitely.

---
## Section 9 — Applied System Design Scenarios

---

### Q53. Design a URL shortener — what's the key decision and what fails at scale?

**Naive approach:** auto-increment ID in a single SQL DB, base62-encode it.

This breaks at scale because a single auto-increment counter becomes a contention bottleneck across multiple app servers.

**Better approach:**
- Pre-allocate ID ranges per server (each instance claims a block without coordinating every request)
- Or use a distributed ID generator (Snowflake: timestamp + machine ID + sequence)

**Base62 vs Hash:**
- Base62 of an auto-ID: unique by construction, no collision checks
- Hash first 7 chars of URL: simple but needs a collision check + retry loop

> **Trap:** Using `GET` with a 301 redirect instead of 302 — 301 is permanent and browsers cache it forever, so you lose click tracking and can't change the destination.

---

### Q54. Design a distributed rate limiter — what's wrong with per-server counting?

If each server tracks counts locally, a client hitting N servers behind a load balancer gets N times the allowed quota. The limit is enforced per-instance, not globally.

**Fix:** centralize counters in Redis, using atomic `INCR` + `EXPIRE` operations so the limit is enforced globally regardless of which server handles the request.

**The atomicity trap:** `GET` then `SET` is a race condition — two requests can both read 99, both increment to 100, and both get allowed. Use `INCR` (atomic) or a Lua script for check-and-increment.

---

### Q55. Design a "top K trending" feature — why is exact counting infeasible?

Exact real-time counting across high-cardinality, high-throughput streams (every hashtag, every second) requires memory proportional to the number of unique items.

**What real systems use:**
- **Count-Min Sketch** — approximate frequency counter in bounded memory (fixed-size hash grid)
- **Heavy hitter algorithms** — track only the top K candidates, not every item
- Trade-off: small, bounded inaccuracy for massive space savings

Perfect precision on "trending" isn't required — nobody notices if the 7th trending topic was actually 9th.

---

### Q56. Design a notification system — what's the naive "loop and send" mistake?

Naively looping through millions of users and calling the push API synchronously is:
- Slow (minutes to hours)
- Fragile (mid-loop crash = you don't know who was notified, retrying duplicates for early users)

**Correct design:**
1. Publish a "send notification" event to a queue
2. Pool of stateless worker consumers pull and send in parallel
3. Idempotency keys prevent double-sends on retry
4. Track delivery status in a persistent store so a crash is recoverable from exactly where it stopped

---

### Q57. Design a news feed — fan-out on write vs fan-out on read, and where each breaks.

**Fan-out on write:** when a user posts, push the post into every follower's precomputed feed cache immediately.
- Feed reads are instant
- Breaks for celebrities: one post by a user with 50M followers triggers 50M cache writes

**Fan-out on read:** compute the feed at request time by merging posts from all followed accounts.
- No write-storm on post creation
- Breaks at read scale: must query and merge across all follows on every feed load

**Hybrid (Twitter/Instagram approach):**
- Fan-out on write for normal users
- Fan-out on read for celebrity accounts (merged in at request time)
- Best of both, at the cost of slightly more complex read-path logic

---

### Q58. Design a web crawler — what are the key system design challenges?

Beyond "fetch pages and store links," the hard problems are:

1. **Politeness** — don't hammer a single domain. Rate-limit per domain, respect `robots.txt`
2. **Deduplication** — avoid crawling the same URL twice. Use a distributed bloom filter or URL fingerprint store
3. **Prioritisation** — some pages (news sites, home pages) should be re-crawled more frequently
4. **Scale** — billions of URLs require a distributed URL frontier (priority queue sharded across nodes)
5. **Trap URLs** — spider traps generate infinite unique URLs (e.g., calendar with infinite next-month links). Detect and skip them

---

## Section 10 — Quick-Fire Concept Checks

---

### Q59. Why is premature sharding an anti-pattern?

Sharding adds real, immediate complexity (no cross-shard transactions, rebalancing pain, operational overhead) before you've hit the scale that requires it.

Most systems never reach single-node DB limits. Premature sharding trades concrete pain for a hypothetical future benefit.

**The senior approach:** vertically scale and optimise first (indexing, caching, read replicas). Shard only when you have concrete evidence of hitting a ceiling.

---

### Q60. Why do interviewers ask for scale estimates before you design?

Because the "best" architecture is entirely scale-dependent:
- 100 req/day → a single well-indexed Postgres is fine, no queues or caching needed
- 100K req/sec → sharding, caching, and async workers are non-optional

Over-engineering for scale you'll never hit adds failure points and wastes time. Under-engineering for real scale means a crisis redesign under production fire.

Estimating scale first is what determines whether you even need half the patterns in this doc.

---

### Q61. What is the difference between Kafka and a traditional message queue (e.g., SQS)?

| | Kafka | Traditional Queue (SQS/RabbitMQ) |
|---|---|---|
| Message retention | Kept after consumption (configurable) | Deleted after consumption |
| Multiple consumers | Yes — each consumer group reads independently | Limited |
| Replay | Yes — read from any offset | No |
| Throughput | Very high (sequential disk writes) | Medium |
| Ordering | Per-partition | Queue order |
| Best for | Event streaming, audit logs, replay | Task distribution, simple async work |

---

### Q62. What's the difference between monolith and microservices — when is a monolith actually the right choice?

| | Monolith | Microservices |
|---|---|---|
| Deployment | Single unit | Independent per service |
| Teams | One team | Multiple teams, one per service |
| Complexity | Low initially | High (networking, distributed tracing, etc.) |
| Scaling | Scale whole app | Scale individual services |
| Best for | MVPs, small teams, early stage | Large teams, different scaling needs per service |

> **The senior answer:** start with a monolith, migrate to microservices when you have a proven reason (team independence, different scaling needs) — not because microservices "feel more professional."

---

### Q63. What is the difference between availability and durability?

**Availability:** the system is accessible and responds to requests — uptime.

**Durability:** data that was successfully written is never lost — data persistence.

A system can be:
- Available but not durable: accepts writes but loses them on crash (no persistent storage)
- Durable but not available: data is safe on disk but the service is down for maintenance

Both matter, but they're measured and ensured through different mechanisms (replication for availability, fsync/journaling/backups for durability).

---

## Quick Reference — All Topics at a Glance

| Section | Questions | Key Concepts |
|---|---|---|
| Foundations | Q1–Q7 | Scalability, latency/throughput, CAP, PACELC, SPOF |
| Networking | Q8–Q13 | L4/L7 LB, DNS, CDN, proxies, timeouts |
| Databases | Q14–Q22 | SQL vs NoSQL, indexing, sharding, replication, N+1 |
| Caching | Q23–Q27 | Cache layers, strategies, stampede, eviction |
| Messaging | Q28–Q34 | Queues, delivery guarantees, idempotency, Saga |
| Distributed Systems | Q35–Q41 | Consistency, quorum, Raft, vector clocks, locking |
| APIs | Q42–Q46 | REST vs gRPC, rate limiting, gateway, versioning |
| Reliability | Q47–Q52 | Circuit breaker, bulkhead, chaos, health checks |
| Scenarios | Q53–Q58 | URL shortener, rate limiter, feed, notifications, crawler |
| Quick-fire | Q59–Q63 | Sharding, scale estimation, Kafka, monolith vs micro |
