# Day 005 — Monolith, Microservices & Distributed Systems

---

## Part 1 — Monolithic Architecture

---

## What is a Monolith?

A monolithic architecture packages **all parts of an application into a single deployable unit**. One codebase, one process, one database.

```
┌─────────────────────────────┐
│        E-Commerce App       │
│─────────────────────────────│
│  User Module                │
│  Product Module             │
│  Order Module               │
│  Payment Module             │
│  Notification Module        │
└─────────────────────────────┘
              │
          Database
```

Everything — auth, orders, payments, notifications — lives and deploys together.

---

## Core Characteristics

**Single Codebase**
```
Application/
 ├── User Module
 ├── Product Module
 ├── Order Module
 ├── Payment Module
 └── Notification Module
```

**Single Deployment** — even a one-line payment fix requires redeploying the entire app.

**Shared Database** — all modules read and write to the same DB. No separation of concerns at the data layer.

---

## Advantages

| Advantage | Why It Matters |
|---|---|
| Simple to develop | One repo, one run command, everything works locally |
| Easy testing | `localhost:8080` — the full app runs in one place |
| Easier debugging | One log stream, one process to trace |
| Fast to start | No service coordination overhead at the beginning |

Monoliths are the right starting point for most projects. Don't over-engineer early.

---

## Disadvantages

| Disadvantage | What Goes Wrong |
|---|---|
| Scaling is wasteful | Heavy Product Search traffic? You must scale the entire app, not just search |
| Slow deployments | A payment bug fix requires redeploying everything |
| Grows unwieldy | Codebases hitting 1M+ lines become hard to navigate and change safely |
| Technology lock-in | Chose Java in 2015? The whole system is Java forever |
| Single point of failure | One crash = entire application down |

---

## When to Use a Monolith

- Startup or early-stage product (MVP)
- Small team (1–5 engineers)
- Internal tools or low-traffic apps
- When you need to move fast and validate quickly

> Start with a monolith. Split into services only when you have a clear, proven reason to.

---

## Scaling a Monolith

**Vertical** — upgrade the server:
```
Before: CPU=4,  RAM=8GB
After:  CPU=32, RAM=64GB
```

**Horizontal** — run multiple copies behind a load balancer:
```
Load Balancer
      │
┌─────┼─────┐
App1  App2  App3   ← identical copies of the whole app
```

The downside: you can't scale one module independently. Wasteful when only one part is under load.

---
---

## Part 2 — Microservices Architecture

---

## What is Microservices?

Microservices breaks a large application into **small, independent services** — each with its own codebase, deployment pipeline, and often its own database.

```
              Users
                │
           API Gateway
                │
   ┌────────────┼────────────┐
   │            │            │
User Svc   Product Svc   Order Svc   Payment Svc
   │            │            │            │
UserDB    ProductDB      OrderDB    PaymentDB
```

No shared database. No shared deployment. Each service is fully autonomous.

---

## Core Characteristics

**Independent Services** — separate codebases, separate repos, separate teams.

**Independent Deployment:**
```
Payment Service updated → deploy only Payment Service
Everything else keeps running untouched
```

**Independent Databases** — each service owns its data. No direct cross-service DB queries.

---

## Advantages

**Independent Scaling** — only scale what's under pressure:
```
Product Search is slow → scale Product Service only
User Service → 1 instance (fine)
Product Service → 4 instances (under load)
```

**Fault Isolation** — one service crashing doesn't bring down the system:
```
Payment Service ❌ crashes
Users ✅ | Products ✅ | Orders ✅  ← still working
```

**Technology Flexibility:**
```
User Service      → Java
Product Service   → Go
Analytics Service → Python
Chat Service      → Node.js
```

**Team Independence** — teams own their services end to end. Ship without coordinating with other teams.

**Faster Deployments** — no full-app redeploy for a single service change.

---

## Disadvantages

| Disadvantage | Why It's Hard |
|---|---|
| Network complexity | Services call each other over HTTP/gRPC — network failures are now a thing |
| More infrastructure | Need API Gateway, service discovery, distributed tracing, centralised logging |
| Distributed transactions | A single "place order" action spans Order + Payment + Inventory services |
| Hard to debug | A request touches 5 services — where did it fail? |

**Distributed transaction example:**

In a monolith:
```sql
BEGIN
  UPDATE orders SET status = 'placed'
  UPDATE payments SET status = 'charged'
COMMIT
```

In microservices — Order and Payment are separate services. You can't use a simple DB transaction. Solutions: **Saga Pattern** or **Event-Driven Architecture**.

**Debugging example:**
```
API Gateway → User Service → Order Service → Payment Service → Notification Service
                                    ↑
                              Failed here?
```
You need distributed tracing tools (Jaeger, Zipkin, Datadog) to trace a request across services.

---

## When to Use Microservices

- Large application with multiple distinct domains
- Multiple teams working in parallel
- High scalability requirements (millions of users)
- Frequent independent deployments needed
- Different parts have very different scaling needs

**Real-world examples:** Netflix, Amazon, Uber, Spotify

---
---

## Monolith vs Microservices — Full Comparison

| Feature | Monolith | Microservices |
|---|---|---|
| Codebase | Single | One per service |
| Deployment | Entire app at once | Per service independently |
| Database | Shared | Separate per service |
| Scalability | Scale whole app | Scale individual services |
| Fault Isolation | Poor — one crash = all down | Good — failures are contained |
| Team Independence | Low | High |
| Technology Choice | One stack | Any stack per service |
| Complexity | Low | High |
| Debugging | Easy | Hard (needs tracing tools) |
| Best For | Small teams, early stage | Large teams, large scale |

---
---

## Part 3 — Distributed Systems

---

## What is a Distributed System?

A distributed system is a **collection of independent machines that cooperate to appear as one system** to the end user.

```
User sees:    google.com  (one thing)

Reality:      Server1
              Server2       (thousands of machines)
              Server3
              ...
```

Both a distributed monolith (multiple copies of one app) and microservices (multiple services) are distributed systems.

---

## Why Do We Need Them?

A single server has hard limits — CPU, RAM, disk, network bandwidth. Once you hit those limits, you need more machines.

```
Single Server Limits:
  CPU      → exhausted
  RAM      → exhausted
  Disk     → full
  Network  → saturated
       ↓
   Add more servers
```

---

## How Machines Communicate

Nodes in a distributed system talk to each other over the network using:

| Protocol | Used For |
|---|---|
| HTTP / REST | Standard service-to-service calls |
| gRPC | High-performance internal calls |
| TCP | Low-level reliable communication |
| Message Queues | Async communication (Kafka, RabbitMQ) |

---

## Advantages

| Advantage | Description |
|---|---|
| Scalability | Add more nodes instead of upgrading one machine |
| High Availability | One node down? Others take over |
| Fault Tolerance | System survives partial failures |
| Performance | Parallel processing across many nodes |

---

## Challenges

| Challenge | Why It's Hard |
|---|---|
| Network failures | Messages between nodes can be lost or delayed |
| Latency | A remote call is always slower than a local function call |
| Data consistency | Multiple copies of data can go out of sync |
| Clock synchronisation | Each server has its own clock — they drift |
| Debugging | A failure might span 10 machines — hard to trace |

---

## CAP Theorem

Every distributed system must make a tradeoff between three properties — but can only fully guarantee **two of the three**:

```
        Consistency
            /\
           /  \
          /    \
         /      \
Availability ── Partition Tolerance
```

| Property | Meaning | Example |
|---|---|---|
| **Consistency (C)** | Every read returns the most recent write | Banking — you always see your real balance |
| **Availability (A)** | Every request gets a response (may be stale) | Social feeds — always loads, might be slightly old |
| **Partition Tolerance (P)** | System keeps working if nodes can't talk to each other | Required for any real-world distributed system |

> Partition Tolerance is non-negotiable in practice — networks fail. So the real choice is between **CP** (consistent but may be unavailable) and **AP** (always available but may be stale).

---

## Monolith → Distributed System → Microservices Evolution

Most systems follow this path as they grow:

```
Stage 1 — Startup
  Single server, monolith app, one DB
         ↓
Stage 2 — Growing Traffic
  Distributed monolith: same codebase, multiple instances behind a load balancer
         ↓
Stage 3 — Team & Scale Problems
  Split into microservices: independent services, independent deployments
         ↓
Stage 4 — Internet Scale
  Full microservices + sharding + replication + CDN + caching
```

**Important:** a monolith running on multiple servers IS a distributed system. Microservices are almost always distributed. The terms are related but not the same.

---

## Quick Reference Cheat Sheet

| Concept | One-Line Summary |
|---|---|
| Monolith | One codebase, one deployment, one DB |
| Microservices | Independent services, each with its own deploy and DB |
| Distributed System | Multiple machines working together as one |
| API Gateway | Single entry point that routes requests to the right service |
| Service Discovery | How services find each other's addresses dynamically |
| Saga Pattern | Manages distributed transactions across services |
| CAP Theorem | Can only guarantee 2 of: Consistency, Availability, Partition Tolerance |
| Distributed Tracing | Tracks a request as it flows through multiple services |

---

## One-Line Formulas

```
Monolith      = Simple to build + Hard to scale at size

Microservices = Easy to scale + Hard to manage

Distributed   = Multiple machines + Looks like one system to users
```

---

## The Golden Evolution Rule

```
Startup      →  Monolith
Growth       →  Distributed Monolith
Scale        →  Microservices
Internet Scale → Microservices + Sharding + Replication + Cache + CDN
```
