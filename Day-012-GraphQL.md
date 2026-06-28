# Day 012 — GraphQL

---

## What is GraphQL?

GraphQL is a **query language for APIs** that lets clients ask for exactly the data they need — nothing more, nothing less. Developed by Facebook in 2015, it was designed to fix the over-fetching and under-fetching problems of REST.

**The key difference from REST:**

```
REST:     Many endpoints, fixed responses
          GET /users
          GET /users/1
          GET /users/1/posts
          GET /posts

GraphQL:  One endpoint, client defines the response
          POST /graphql  ← everything goes here
```

**Analogy:**
```
REST:     "Give me the combo meal"
          → gets burger + fries + coke + sauce + spoon (even if you only wanted the burger)

GraphQL:  "Give me burger and coke"
          → gets exactly burger and coke
```

---

## Why GraphQL Was Created — The REST Problems

### Problem 1: Over-fetching

You ask for one thing, you get everything:

```
GET /users/1
→ { name, age, phone, address, salary, hobbies, company, posts, friends }
```

Your app only needed `name`. You received 10 fields. Wasted bandwidth, wasted processing.

### Problem 2: Under-fetching

One endpoint isn't enough, so you make multiple calls:

```
REST (3 separate requests):
  GET /users/1
  GET /users/1/posts
  GET /users/1/followers

GraphQL (1 request):
  query { user(id:1) { name  posts { title }  followers { name } } }
```

---

## REST vs GraphQL

| Feature | REST | GraphQL |
|---|---|---|
| Endpoints | Multiple | Single (`/graphql`) |
| Response shape | Fixed by server | Defined by client |
| Over-fetching | ⚠️ Common | ✅ None |
| Under-fetching | ⚠️ Common | ✅ None |
| Caching | ✅ Easy (HTTP cache) | ⚠️ More complex |
| Learning curve | Low | Medium |
| Best for | Simple CRUD, public APIs | Complex data, mobile, microservices |

---

## The Four Core Concepts

```
GraphQL
 ├── Schema     → defines what data exists and its types
 ├── Query      → reads data (like GET)
 ├── Mutation   → writes/changes data (like POST/PUT/DELETE)
 └── Resolver   → the function that actually fetches the data
```

---

## 1. Schema

The schema is a **contract between client and server** — it defines every type, field, and operation available.

```graphql
type User {
  id:    ID!       # required (! = non-null)
  name:  String!
  age:   Int
  email: String
}

type Book {
  id:     ID!
  title:  String!
  author: String!
  price:  Float
}
```

**Built-in scalar types:** `String`, `Int`, `Float`, `Boolean`, `ID`

The schema serves as self-documentation — any client can introspect it to discover what's available.

---

## 2. Query (Reading Data)

Queries are equivalent to `GET` in REST. The client specifies exactly which fields to return.

**Basic query:**
```graphql
query {
  user(id: 10) {
    name
    email
  }
}
```
```json
Response: { "name": "Rahul", "email": "abc@gmail.com" }
```

The database may have 20 columns. Only the 2 requested are returned.

**Nested query — the real power:**
```graphql
query {
  user(id: 1) {
    name
    posts {
      title
      comments {
        text
      }
    }
  }
}
```

This fetches user + posts + comments in **one single request**. In REST this would take 3 separate API calls.

---

## 3. Mutation (Writing Data)

Mutations are equivalent to `POST`, `PUT`, `PATCH`, and `DELETE` in REST.

**Create:**
```graphql
mutation {
  createUser(name: "Rahul", age: 22) {
    id
    name
  }
}
```
```json
Response: { "id": 1, "name": "Rahul" }
```

**Update:**
```graphql
mutation {
  updateUser(id: 5, age: 25) {
    id
    age
  }
}
```

**Delete:**
```graphql
mutation {
  deleteUser(id: 5) {
    id
  }
}
```

---

## 4. Resolver

A resolver is the **function that executes when a query or mutation is received**. It contains the actual business logic — fetching from a database, calling another service, etc.

```
Client → Query → Schema Validation → Resolver → Database → Response
```

**Example:**
```javascript
// Query: user(id: 5)
const resolvers = {
  Query: {
    user: (_, { id }) => database.users.find(id)
  }
}
```

Resolvers can fetch from any source:
- SQL (MySQL, PostgreSQL)
- NoSQL (MongoDB, Redis)
- External REST APIs
- Other microservices

---

## Complete Request Lifecycle

```
1. Client writes a GraphQL query
         ↓
2. Sent as POST /graphql
         ↓
3. Server validates against schema
         ↓
4. Resolver function executes
         ↓
5. Database / service is queried
         ↓
6. Only requested fields are returned as JSON
         ↓
7. Client receives exactly what it asked for
```

---

## GraphQL vs SQL — Common Confusion

Many beginners confuse these. They operate at completely different layers:

| | GraphQL | SQL |
|---|---|---|
| What it queries | Your API | Your database |
| Runs on | Application server | Database server |
| Read | `query { }` | `SELECT` |
| Create | `mutation { create... }` | `INSERT` |
| Update | `mutation { update... }` | `UPDATE` |
| Delete | `mutation { delete... }` | `DELETE` |

**GraphQL does not replace SQL.** It sits above it:
```
Client → GraphQL → Resolver → SQL Query → Database
```

---

## GraphQL in Microservices

GraphQL acts as an **API Gateway** that aggregates multiple services into one response:

**Without GraphQL — multiple round trips:**
```
Frontend
  ├──→ User API
  ├──→ Order API
  ├──→ Product API
  └──→ Payment API
```

**With GraphQL Gateway — one request:**
```
Frontend
    ↓
GraphQL Gateway
  ├── User Service
  ├── Order Service
  ├── Product Service
  └── Payment Service
    ↓
Single combined response
```

This is the **Backend for Frontend (BFF)** pattern — very common in large systems.

---

## Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| No over-fetching or under-fetching | More complex backend (resolvers take effort) |
| Client controls response shape | Harder to cache (every query is unique) |
| Single endpoint — easier API management | Deep nested queries can overload the DB |
| Great for mobile (smaller responses) | Harder to monitor (everything goes to `/graphql`) |
| Self-documenting schema | Requires query depth limits and complexity controls |
| Aggregates multiple services easily | |

**The caching problem explained:**
```
REST:     GET /users/5  → easily cacheable by URL
GraphQL:  POST /graphql → query body varies each time → HTTP cache can't help
Solution: Use persisted queries or DataLoader for batching
```

**The N+1 query problem:**
```graphql
query { users { posts { comments { } } } }
```
Without optimisation, fetching comments for 100 posts triggers 100 DB queries.
Fix with: **DataLoader** (batches and deduplicates DB calls).

---

## When to Use GraphQL

| ✅ Use GraphQL | ❌ Avoid GraphQL |
|---|---|
| Mobile apps (bandwidth matters) | Simple CRUD applications |
| Single Page Apps (React/Vue/Angular) | Small internal tools |
| Microservices aggregation | Public APIs where HTTP caching is critical |
| Multiple frontend teams with different needs | Systems with simple, stable data requirements |
| Complex nested data relationships | |
| APIs with frequently evolving requirements | |

---

## Typical System Design with GraphQL

```
React App (Web)    Mobile App
       \                /
        └──────┬────────┘
          POST /graphql
               ↓
      GraphQL Gateway
    ┌──────┬──────┬──────┐
    ↓      ↓      ↓      ↓
  User  Order  Product  Payment
  Svc    Svc    Svc      Svc
    ↓      ↓      ↓      ↓
  MySQL Postgres MongoDB  MySQL
```

---

## Quick Reference Cheat Sheet

| Concept | One-Line Summary |
|---|---|
| GraphQL | Query language — client specifies exactly what it needs |
| Schema | Contract defining available types and operations |
| Query | Reads data (equivalent to GET) |
| Mutation | Creates, updates, or deletes data (equivalent to POST/PUT/DELETE) |
| Resolver | Function that fetches the actual data |
| Over-fetching | Getting more fields than needed (REST problem GraphQL solves) |
| Under-fetching | Needing multiple requests for related data (REST problem GraphQL solves) |
| N+1 Problem | Nested queries triggering too many DB calls — fix with DataLoader |
| Single Endpoint | Everything goes to `POST /graphql` |
| Introspection | Clients can query the schema itself to discover available operations |

---

## One-Line Formula

```
GraphQL = One Endpoint + Client-Defined Response + Schema Contract + Resolver Logic
        = No Over-fetching + No Under-fetching + Perfect for Mobile & Microservices
```
