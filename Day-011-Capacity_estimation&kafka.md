# Day 011 — Capacity Estimation & Apache Kafka

---

## Part 1 — Capacity Estimation

---

## Why Estimate Capacity First?

Before designing any system, you need to know its scale. The architecture for 1,000 users looks completely different from the one for 1 billion users.

Capacity estimation answers:
- How many servers do we need?
- How much storage is required?
- How much network bandwidth will we consume?
- Do we need caching, sharding, or replication?

Without these numbers, every architecture decision is just a guess.

---

## What We Estimate

### 1. Users

Start with user numbers — everything else derives from them.

| Metric | Description |
|---|---|
| **DAU** (Daily Active Users) | Users active on a given day |
| **MAU** (Monthly Active Users) | Users active in a given month |
| **Concurrent Users** | Users on the system at the same time |

> Concurrent users matter most for server capacity — they represent simultaneous load.

---

### 2. Read vs Write Ratio

Understanding traffic shape drives caching and database decisions:

| System | Read : Write | Why |
|---|---|---|
| Instagram Feed | 100 : 1 | Millions read posts, few create them |
| WhatsApp | ~1 : 1 | Every message sent is also received |
| Wikipedia | 1000 : 1 | Rarely written, constantly read |

**High read ratio** → invest in caching and read replicas
**High write ratio** → invest in write-optimised DBs and partitioning

---

### 3. Queries Per Second (QPS)

The most common estimation in interviews.

```
QPS = Total Daily Requests / 86,400 seconds

Example:
  864 million requests/day
  864,000,000 / 86,400 = 10,000 QPS
```

---

### 4. Peak Traffic

**Always design for peak, not average.**

Traffic is never constant. Plan for the worst case:

```
Average:  10,000 QPS
Peak:     50,000 QPS  ← design for this

Real examples:
  Amazon on Prime Day
  BookMyShow during movie releases
  FIFA streaming during World Cup finals
```

A common rule of thumb: peak ≈ 2–5× average.

---

### 5. Storage Estimation

```
Storage/day = Uploads per day × Size per item

Example (photo sharing app):
  5 million uploads/day × 2 MB = 10 TB/day
  10 TB/day × 365 = 3.65 PB/year
```

This tells you: need object storage (S3), CDN, and data lifecycle policies.

---

### 6. Bandwidth Estimation

```
Bandwidth = QPS × Average response size

Example:
  20,000 QPS × 500 KB = 10 GB/sec outbound
```

This tells you: need CDN, compression, and caching to reduce bandwidth costs.

---

### 7. Database Size

```
Example (user profiles):
  100 million users × 2 KB = 200 GB  → single DB may be fine

Example (messages):
  2 billion messages/day × 500 bytes = ~1 TB/day → sharding is necessary
```

---

## Capacity Estimation Flow

```
Define Users (DAU/MAU/Concurrent)
          ↓
Calculate Total Daily Requests
          ↓
Derive QPS + Peak QPS
          ↓
Estimate Storage/day and Storage/year
          ↓
Estimate Bandwidth
          ↓
Choose: Servers, DB type, Caching, Sharding, CDN
```

---

## Worked Example — Design Instagram

| Step | Calculation | Result |
|---|---|---|
| DAU | Given | 50 million |
| Posts per day | 2 posts/user/day | 100 million posts |
| Image storage/day | 100M × 3 MB | 300 TB/day |
| Feed requests/day | 50M users × 40 feeds | 2 billion/day |
| Average QPS | 2B / 86,400 | ~23,000 QPS |
| Peak QPS | 4× average | ~100,000 QPS |

**Architecture conclusions from these numbers:**
- 300 TB/day → Object storage + CDN + compression essential
- 100K peak QPS → Load balancer + horizontal scaling + caching
- 2B feed reads/day → Read replicas + cache feed results
- Rapid data growth → Sharding for DB

---

## Key Numbers to Memorise

| Constant | Value |
|---|---|
| Seconds per day | 86,400 |
| 1 KB | 1,000 bytes |
| 1 MB | 1,000 KB |
| 1 GB | 1,000 MB |
| 1 TB | 1,000 GB |
| 1 PB | 1,000 TB |

---
---

## Part 2 — Apache Kafka

---

## What is Kafka?

Apache Kafka is a **distributed event streaming platform**. Instead of services calling each other directly, they communicate by publishing and consuming events through Kafka's durable log.

---

## The Problem Kafka Solves

**Without Kafka — tight coupling:**
```
User Places Order
      ↓
Order Service
      ├──→ Payment Service
      ├──→ Inventory Service
      ├──→ Email Service
      └──→ Analytics Service
```

Problems:
- One slow service slows everything down
- One crashed service can cascade failures
- Adding a new consumer requires changing Order Service code

**With Kafka — loose coupling:**
```
User Places Order
      ↓
Order Service → publishes "OrderCreated" → Kafka Topic
                                                 │
                    ┌────────────────────────────┤
                    ↓          ↓          ↓      ↓
              Payment    Inventory    Email   Analytics
              Service     Service    Service   Service
```

Order Service doesn't know or care who's listening. New consumers can be added without touching Order Service at all.

---

## What is an Event?

An event is a **record of something that happened**.

```json
{
  "event": "OrderCreated",
  "orderId": 101,
  "userId": 25,
  "amount": 500,
  "timestamp": "2026-06-25T10:30:00Z"
}
```

Examples: `UserRegistered`, `OrderCreated`, `PaymentCompleted`, `ItemAddedToCart`, `MessageSent`

---

## Core Components

```
Producer → Topic → Partition → Broker → Consumer Group → Consumer
```

### Producer
Publishes events to a Kafka topic.
```
Order Service → publishes "OrderCreated" to topic: orders
```

### Topic
A named, logical stream of related events. Like a named channel.
```
Topics: orders | payments | users | notifications
```

### Partition
Each topic is split into partitions for parallel processing:
```
orders topic
  ├── Partition 0
  ├── Partition 1
  ├── Partition 2
  └── Partition 3
```
Different partitions are processed simultaneously by different consumers.

### Offset
Every event in a partition gets a unique sequential number — its **offset**:
```
Partition 0:  [0] [1] [2] [3] [4] [5] [6] ...
```
Consumers track their offset to know where they left off. If a consumer restarts, it resumes from where it stopped.

### Broker
A Kafka server that stores and serves events. Kafka clusters run multiple brokers for fault tolerance:
```
Broker 1 | Broker 2 | Broker 3
```

### Consumer Group
Multiple consumers cooperating to process a topic in parallel:
```
orders topic
  Partition 0 → Consumer A
  Partition 1 → Consumer B
  Partition 2 → Consumer C
  Partition 3 → Consumer A (shared load)
```
Each partition is handled by exactly one consumer within the group — preserving order per partition while enabling parallelism.

---

## Kafka Architecture Overview

```
PRODUCERS
Order Service   Payment Service   User Service
       \               |               /
        └──────────────┼──────────────┘
                       ↓
            ┌─────────────────────┐
            │     Kafka Cluster   │
            │  Broker1  Broker2   │
            │                     │
            │  orders topic       │
            │  P0  P1  P2  P3     │
            └──────────┬──────────┘
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
   Inventory      Analytics     Notification
    Service        Service        Service
```

---

## Replication — Fault Tolerance

Each partition is replicated across brokers:

```
Broker 1: Partition 0 (Leader)  ← producers/consumers use this
Broker 2: Partition 0 (Replica) ← takes over if Broker 1 fails
Broker 3: Partition 0 (Replica)
```

If Broker 1 goes down, Broker 2 is automatically elected leader. No data loss.

---

## Event Retention — Replay Capability

Unlike traditional queues, Kafka **keeps events after they're consumed**:

```
Producer → Kafka → Consumer A reads at offset 50
                 → Consumer B reads at offset 30 (behind, catching up)
                 → Consumer C reads from offset 0 (full replay)
```

This allows:
- New services to replay historical events and rebuild state
- Reprocessing events when there's a bug
- Auditing exactly what happened and when

---

## Why Kafka is Fast

| Technique | Why it helps |
|---|---|
| Sequential disk writes | Much faster than random writes |
| Partitioning | Horizontal scaling — more partitions = more parallelism |
| Consumer pull model | Consumers read at their own pace, no backpressure |
| Batching | Multiple events sent together, reducing network overhead |
| Zero-copy I/O | Data goes from disk to network without copying through CPU |

---

## Kafka vs Traditional Message Queue

| Feature | Traditional Queue | Kafka |
|---|---|---|
| Message kept after consumption | ❌ Usually deleted | ✅ Retained (configurable) |
| Multiple independent consumers | ⚠️ Limited | ✅ Yes |
| Replay old events | ❌ Usually not | ✅ Yes |
| Throughput | Medium | Very high |
| Ordering guarantee | Queue order | Per-partition order |
| Horizontal scaling | Moderate | Excellent |
| Best for | Task queues | Event streaming, audit logs |

---

## Real-World Example — Food Delivery App

```
Customer Places Order
         ↓
   Order Service
         │ publishes "OrderCreated"
         ↓
  ┌──────────────┐
  │  Kafka Topic │
  │   "orders"   │
  └──────┬───────┘
         │
  ┌──────┼──────────────┐
  ↓      ↓              ↓
Payment  Email      Analytics
Service  Service     Service
  ↓
Inventory
Service
```

Each consumer works independently:
- **Payment Service** → charges the customer
- **Inventory Service** → reserves the items
- **Email Service** → sends confirmation
- **Analytics Service** → updates dashboards

If Analytics is down temporarily, the others keep running. When it recovers, it replays missed events from Kafka's log.

---

## When to Use Kafka

| Use Case | Why Kafka fits |
|---|---|
| Microservice communication | Decouples producers from consumers |
| Real-time analytics | Stream processing at high throughput |
| Event sourcing | Full event history with replay capability |
| Log aggregation | Centralised, durable log store |
| Fraud detection | Process every transaction event in real time |
| IoT data ingestion | Handle millions of sensor events per second |

---

## Quick Reference Cheat Sheet

| Term | One-Line Summary |
|---|---|
| Producer | Publishes events to a Kafka topic |
| Consumer | Reads events from a Kafka topic |
| Topic | Named stream of related events |
| Partition | Sub-division of a topic for parallelism |
| Offset | Sequential position of an event within a partition |
| Broker | Kafka server that stores and serves events |
| Consumer Group | Set of consumers sharing a topic's partitions |
| Replication | Copies of partitions on other brokers for fault tolerance |
| Event Retention | Events stay in Kafka after consumption (configurable) |

---

## One-Line Formulas

```
Capacity Estimation:
  QPS = Daily requests / 86,400
  Storage = Uploads/day × size per item
  Bandwidth = QPS × response size
  Always design for peak, not average

Kafka:
  Producer → Topic → Partition → Broker → Consumer
  More partitions = more parallelism
  Offsets = replay from any point in history
  Kafka keeps events; queues delete them
```
