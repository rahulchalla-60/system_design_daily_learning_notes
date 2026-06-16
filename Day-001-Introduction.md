# Day 001 — Introduction to System Design

---

## What is System Design?

System Design is the process of defining the **architecture, components, modules, interfaces, and data flow** of a software system to meet specific business and technical requirements.

The goal is to build systems that are **scalable, reliable, maintainable, and efficient** — capable of handling real-world workloads.

---

## Core Objectives

| Objective | Description |
|---|---|
| **Scalability** | Handle increasing users and data |
| **Reliability** | Operate continuously with minimal failures |
| **Availability** | Keep services accessible at all times |
| **Performance** | Achieve low latency and high throughput |
| **Maintainability** | Make updates and bug fixes simple |
| **Security** | Protect data and resources |
| **Cost Efficiency** | Optimize infrastructure costs |

---

## System Design Framework

Use this framework whenever you're designing a system — in interviews or real-world scenarios.

---

### Step 1 — Understand Requirements

#### Functional Requirements
What the system **must do**.

- User registration / login
- Upload photos
- Send messages
- Search products

#### Non-Functional Requirements
How well the system **must perform**.

- Support 10 million users
- 99.99% availability
- Response time < 200ms
- Secure authentication

#### Clarifying Questions to Ask
- Who are the users?
- What is the expected traffic?
- Is it read-heavy or write-heavy?
- Are there real-time requirements?
- What is the data retention policy?

---

### Step 2 — Estimate Scale

Do rough calculations to guide technology choices.

**Example:**
```
10 million daily users × 100 requests/day = 1 Billion requests/day
1B / 86,400 seconds ≈ 11,500 RPS (Requests Per Second)
```

**Storage Estimation:**
```
1 MB per user × 10M users = 10 TB
```

> Estimations help you choose the right databases, caches, and infrastructure.

---

### Step 3 — High-Level Design

Sketch the major components and how they connect.

```
Client
  ↓
Load Balancer
  ↓
API Gateway
  ↓
Application Servers
  ↓
Cache (Redis)
  ↓
Database
```

**Key Components:**
- Client
- Load Balancer
- API Gateway
- Application Servers
- Cache
- Database
- Message Queue
- Storage System

---

### Step 4 — Database Design

#### Choose the Right Database Type

**SQL (Relational)** — MySQL, PostgreSQL
- Use when you need strong consistency, complex relationships, or transactions.

**NoSQL** — MongoDB, Cassandra, DynamoDB
- Use when you need massive scale, flexible schema, or high throughput.

#### Design Entities (Example)

```
User              Post
────────────      ────────────────
user_id           post_id
name              user_id
email             content
                  timestamp
```

---

### Step 5 — API Design

Define clear interfaces for your system.

| Action | Method | Endpoint |
|---|---|---|
| Create User | `POST` | `/users` |
| Get User | `GET` | `/users/{id}` |
| Create Post | `POST` | `/posts` |

---

### Step 6 — Data Flow Design

Trace how a request moves through the system.

```
Client
  ↓
Load Balancer
  ↓
Application Server
  ↓
Cache Check  →  Cache Hit → Return Response
  ↓ (Cache Miss)
Database
```

---

### Step 7 — Scaling Strategies

**Horizontal Scaling** — Add more servers
```
Server 1 | Server 2 | Server 3
```

**Vertical Scaling** — Increase CPU/RAM of one machine

> Large-scale systems generally favor **horizontal scaling** for flexibility and fault tolerance.

---

### Step 8 — Caching

Store frequently accessed data closer to the application.

```
Client → Cache (Redis) → Database
```

**Benefits:**
- Reduced latency
- Lower database load

**Common Tools:** Redis, Memcached

---

### Step 9 — Load Balancing

Distribute incoming traffic across multiple servers.

**Methods:**
- Round Robin
- Least Connections
- Weighted Routing

**Benefits:** High availability, better performance

---

### Step 10 — Database Scaling

**Replication** — Copy data to multiple nodes
```
       Master
      /       \
Replica 1   Replica 2
```
- Increases read capacity
- Improves fault tolerance

**Sharding** — Split data across nodes
```
Shard 1 → Users A–F
Shard 2 → Users G–M
Shard 3 → Users N–Z
```
- Handles large datasets efficiently

---

### Step 11 — Asynchronous Processing

Use message queues to decouple services and handle tasks asynchronously.

```
Producer → Message Queue → Consumer
```

**Tools:** Apache Kafka, RabbitMQ, Amazon SQS

**Common Use Cases:**
- Notifications
- Email delivery
- Log processing

---

### Step 12 — Reliability & Availability

| Concept | Description |
|---|---|
| **Redundancy** | Run multiple instances of each service |
| **Failover** | Backup system takes over when primary fails |
| **Health Checks** | Continuously monitor service status |

---

### Step 13 — Security Considerations

- Authentication & Authorization
- HTTPS everywhere
- Data Encryption (at rest and in transit)
- Rate Limiting
- Input Validation
- DDoS Protection

---

### Step 14 — Monitoring & Logging

**Monitoring Tools:** Prometheus, Grafana, Datadog

**Logging Tools:** ELK Stack, Splunk

**Track:**
- CPU & Memory usage
- Latency
- Error rates

---

### Step 15 — Identify & Fix Bottlenecks

**Common Bottlenecks:**
- Slow database queries
- Network latency
- Cache misses
- Unoptimized APIs
- Large file transfers

**Optimization Techniques:**
- Caching
- Database indexing
- CDN for static assets
- Query optimization

---

## Quick System Design Checklist

Use this in interviews or when starting a new design:

- [ ] Step 1 — Gather Requirements (Functional + Non-Functional)
- [ ] Step 2 — Capacity Estimation
- [ ] Step 3 — High-Level Architecture
- [ ] Step 4 — Database Design
- [ ] Step 5 — API Design
- [ ] Step 6 — Scaling Strategy
- [ ] Step 7 — Caching
- [ ] Step 8 — Load Balancing
- [ ] Step 9 — Message Queues
- [ ] Step 10 — Reliability & Security
- [ ] Step 11 — Monitoring
- [ ] Step 12 — Bottleneck Analysis

---

## One-Line Formula

```
Requirements → Estimation → Architecture → Database → APIs → Scaling
     → Caching → Reliability → Security → Monitoring → Optimization
```

---

## Systems You Can Design With This Framework

- URL Shortener
- Chat Application
- Social Media Platform
- Ride-Sharing System
- Video Streaming Platform
- E-Commerce Website
