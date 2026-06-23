# Day 008 — Communication & Storage Buzzwords

---

## Part 1 — API Communication

---

## 1. REST API

**The standard way most web applications communicate between client and server.**

REST (Representational State Transfer) is an architectural style that uses HTTP to exchange data. It's the default choice for public APIs and most backend systems.

**Analogy:** A restaurant. You (client) place an order through the waiter (API). The kitchen (server) prepares it and the waiter brings it back (response).

```
Client
  │  HTTP Request (GET /users/1)
  ▼
REST API → Server
  │  HTTP Response { "id": 1, "name": "Rahul" }
  ▼
Client
```

### HTTP Methods

| Method | Purpose | Example |
|---|---|---|
| GET | Read data | `GET /users/1` |
| POST | Create data | `POST /users` |
| PUT | Replace entire resource | `PUT /users/1` |
| PATCH | Partial update | `PATCH /users/1` |
| DELETE | Remove resource | `DELETE /users/1` |

### Key Property — Stateless

Every request is independent. The server doesn't remember previous requests. This makes REST easy to scale — any server instance can handle any request.

### Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Simple — uses standard HTTP | **Over-fetching** — you get fields you don't need |
| Stateless — easy to scale | **Under-fetching** — one request may not be enough |
| Every developer knows it | Multiple round trips for related data |

**Over-fetching example:**
```json
GET /users/1
→ { "id": 1, "name": "Rahul", "address": "XYZ", "phone": "1234" }
  You only needed "name" but got everything
```

**Under-fetching example:**
```
Need user + orders + payments?
→ GET /users/1
→ GET /orders/1
→ GET /payments/1
  Three separate network calls
```

**Best for:** Public APIs, e-commerce backends, social media apps, most general-purpose systems.

---

## 2. GraphQL

**The client asks for exactly the data it needs — nothing more, nothing less.**

GraphQL is a query language for APIs developed by Meta. Instead of the server deciding what to return, the client specifies the exact fields it wants.

**Analogy:**
```
REST:     "Give me the combo meal"
          → Gets burger + fries + drink (even if you only wanted the burger)

GraphQL:  "Give me only the burger"
          → Gets exactly the burger
```

**REST vs GraphQL in practice:**
```
REST:
  GET /users/1
  → { "id": 1, "name": "Rahul", "email": "...", "address": "..." }

GraphQL:
  query { user(id: 1) { name } }
  → { "name": "Rahul" }
```

**Fetching multiple resources in one call:**
```graphql
query {
  user(id: 1) { name }
  orders(userId: 1) { id, total }
}
```
One request, two resources — no under-fetching.

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| No over-fetching — get only what you ask for | More complex server implementation |
| No under-fetching — fetch multiple resources in one call | Poorly written queries can overload the DB |
| Great for mobile — saves bandwidth | Harder to cache than REST |

**Best for:** Mobile apps, frontend-heavy apps, multiple client types (web + mobile + TV), microservice aggregation.

---

## 3. gRPC

**Ultra-fast service-to-service communication — built for internal microservice calls.**

gRPC (Google Remote Procedure Call) is a high-performance protocol developed by Google. Instead of HTTP + JSON, it uses HTTP/2 + Protocol Buffers (binary format) — much smaller and faster.

**Analogy:**
```
REST:  Two colleagues writing formal emails back and forth (verbose, slow)
gRPC:  Two colleagues calling each other directly (fast, direct)
```

**How it works:**
```protobuf
// Define the contract (proto file)
service UserService {
  rpc GetUser(UserRequest) returns (UserResponse);
}
```
```
Client calls:  GetUser(id: 1)
Server returns: UserResponse immediately
```

Both sides know the contract upfront — strongly typed, no ambiguity.

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Very fast — binary serialization | Hard to debug — binary data isn't human-readable |
| Small payloads — less bandwidth | Limited browser support |
| Supports streaming (client/server/bidirectional) | Steeper learning curve than REST |
| Strongly typed contracts | |

**Best for:** Microservice-to-microservice communication, real-time systems, high-throughput internal APIs.

---

## REST vs GraphQL vs gRPC — Full Comparison

| Feature | REST | GraphQL | gRPC |
|---|---|---|---|
| Transport | HTTP/1.1 | HTTP/1.1 | HTTP/2 |
| Format | JSON | JSON | Binary (Protobuf) |
| Human Readable | ✅ Yes | ✅ Yes | ❌ No |
| Speed | Medium | Medium | ⚡ Very Fast |
| Over/Under-fetching | ⚠️ Yes | ✅ No | ✅ No |
| Browser Support | ✅ Excellent | ✅ Excellent | ⚠️ Limited |
| Streaming | ⚠️ Limited | ⚠️ Limited | ✅ Excellent |
| Best For | Public APIs | Flexible frontends | Internal microservices |

---
---

## Part 2 — Storage

---

## 4. SQL Databases

**Structured data in rows and columns with strong consistency guarantees.**

SQL (Structured Query Language) databases, also called relational databases, store data in tables with a fixed schema. Every row must conform to the table structure.

```
Users Table
┌────┬────────┬─────┐
│ ID │  Name  │ Age │
├────┼────────┼─────┤
│  1 │ Rahul  │  22 │
│  2 │ Ram    │  24 │
└────┴────────┴─────┘
```

**Popular SQL databases:** MySQL, PostgreSQL, Oracle, Microsoft SQL Server

### ACID Properties

SQL databases guarantee ACID — this is what makes them trustworthy for financial data:

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | All or nothing | Transfer ₹100: debit AND credit both succeed, or neither does |
| **Consistency** | DB stays in valid state | Can't have negative balance |
| **Isolation** | Transactions don't interfere | Two transfers at same time don't corrupt each other |
| **Durability** | Committed data survives crashes | Power cut can't lose a completed transaction |

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Strong consistency — always accurate | Rigid schema — hard to change structure later |
| Powerful JOINs for related data | Horizontal scaling is complex |
| Mature ecosystem, great tooling | Can become a bottleneck at massive scale |

**Best for:** Banking, payments, ERP, inventory, anything requiring transactions.

---

## 5. NoSQL Databases

**Flexible schema, built for horizontal scale and high-volume workloads.**

NoSQL databases ditch the rigid table structure. Different records can have different fields — perfect for unstructured or rapidly changing data.

**Analogy:**
```
SQL:    Every user row must have the same columns
NoSQL:  User A → { name, hobby }
        User B → { name, vehicle, location }  ← completely different shape, fine
```

### Types of NoSQL Databases

**Document Store** — stores JSON-like documents:
```json
{ "id": 1, "name": "Rahul", "hobby": "Cricket" }
```
**Examples:** MongoDB, Couchbase — great for user profiles, content, catalogs

**Key-Value Store** — simplest form, a giant dictionary:
```
user:1  →  "Rahul"
session:abc123  →  { userId: 1, expires: ... }
```
**Examples:** Redis, DynamoDB — great for caching, sessions, leaderboards

**Column-Family Store** — optimised for reading/writing large amounts of specific columns:
```
Examples: Apache Cassandra, HBase
```
Great for time-series data, IoT, write-heavy workloads

**Graph Database** — stores relationships between entities:
```
(Rahul) --[follows]--> (Ram) --[follows]--> (Priya)
```
**Examples:** Neo4j — great for social graphs, recommendations, fraud detection

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Flexible schema — easy to evolve | Eventual consistency — data may lag briefly |
| Built for horizontal scaling | JOIN-like operations are complex or impossible |
| High performance at scale | Weaker transaction support |

**Best for:** Social media, analytics, IoT, real-time apps, caching, recommendation engines.

---

## SQL vs NoSQL — Side by Side

| Feature | SQL | NoSQL |
|---|---|---|
| Data structure | Tables (rows + columns) | Documents, key-value, columns, graphs |
| Schema | Fixed — defined upfront | Flexible — each record can differ |
| Consistency | Strong (ACID) | Eventual (configurable) |
| Scaling | Vertical (primarily) | Horizontal (designed for it) |
| JOIN support | ✅ Excellent | ⚠️ Limited or none |
| Transactions | ✅ Excellent | ⚠️ Limited |
| Big data | Moderate | ✅ Excellent |
| Best for | Banking, payments | Social, analytics, IoT |

---

## 6. Object Storage

**The right way to store large files at scale — not in your database.**

Object storage stores data as discrete objects (files + metadata + a unique ID) rather than in a file system hierarchy or database table.

**Structure of an object:**
```
Object
 ├── Data       (the actual file — photo.jpg, video.mp4, etc.)
 ├── Metadata   (uploadedBy, date, contentType, tags...)
 └── Object ID  (abc123xyz — globally unique identifier)
```

**Popular object storage services:**

| Provider | Service |
|---|---|
| AWS | Amazon S3 |
| Google Cloud | Google Cloud Storage |
| Microsoft Azure | Azure Blob Storage |

### Why Not Store Files in a Database?

```
❌ Wrong:
  Database
   ├── image.jpg  (10MB)
   ├── video.mp4  (500MB)
   └── report.pdf (5MB)
  → DB grows huge, queries slow, expensive

✅ Right:
  Database
   └── file_url: "https://s3.amazonaws.com/bucket/abc123"

  Object Storage
   ├── image.jpg
   ├── video.mp4
   └── report.pdf
  → DB stays lean, files stored cheaply at scale
```

| ✅ Advantages | Use Case |
|---|---|
| Massive scalability (petabytes) | User-uploaded images and videos |
| Very cheap per GB | Application backups |
| Durable — replicated automatically | Log archives |
| CDN-friendly for fast delivery | Data lakes, ML training data |

---

## When to Use What — Interview Decision Guide

| Technology | Use When |
|---|---|
| **REST** | External client ↔ server, public APIs, standard web apps |
| **GraphQL** | Frontend needs flexible queries, mobile apps, multiple client types |
| **gRPC** | Microservice ↔ microservice, high-throughput internal calls |
| **SQL** | Strong consistency required, transactions, relational data |
| **NoSQL** | Massive scale, flexible schema, high read/write throughput |
| **Object Storage** | Large files — images, videos, PDFs, backups |

---

## Typical Large-Scale Architecture

```
User
  │
Frontend
  │
REST / GraphQL  ← external-facing APIs
  │
API Gateway
  │
Microservices ← gRPC between services
  │
┌─────────────────────────────────┐
│                                 │
SQL Database                 NoSQL Database
(Orders, Payments)           (Feeds, Sessions, Cache)

Object Storage
(Images, Videos, PDFs, Backups)
```

---

## Quick Reference — All Terms

| Term | One-Line Definition |
|---|---|
| REST | HTTP-based API using standard methods (GET, POST, etc.) |
| GraphQL | Query language — client specifies exactly what data it needs |
| gRPC | Binary protocol for fast internal service-to-service calls |
| SQL | Relational database with fixed schema and ACID transactions |
| NoSQL | Flexible, scalable database for unstructured or high-volume data |
| Document Store | NoSQL storing JSON-like documents (MongoDB) |
| Key-Value Store | NoSQL storing simple key → value pairs (Redis, DynamoDB) |
| Column-Family | NoSQL optimised for column reads (Cassandra) |
| Graph DB | NoSQL storing relationships between entities (Neo4j) |
| Object Storage | Scalable file storage for large assets (S3, GCS) |
| ACID | Atomicity, Consistency, Isolation, Durability — SQL guarantees |

---

## One-Line Formula

```
REST     → standard external APIs
GraphQL  → flexible frontend data fetching
gRPC     → fast internal microservice calls
SQL      → transactions and relational data
NoSQL    → scale and flexibility
Objects  → files and media
```
