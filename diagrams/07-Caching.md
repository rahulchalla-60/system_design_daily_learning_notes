# Caching Diagrams

---

## 1. Basic Cache Architecture

```mermaid
flowchart LR
    User --> App[Application]
    App --> Cache[(Redis Cache)]
    Cache -->|Miss| DB[(Database)]
    DB --> Cache
    Cache --> App
```

---

## 2. Cache Aside Pattern (Lazy Loading)

Application checks cache first; fetches from DB only on a miss.

```mermaid
flowchart TD
    Request --> Cache{Cache Hit?}

    Cache -->|Yes| Return[Return Cached Data]
    Cache -->|No| DB[(Database)]

    DB --> Store[Store in Cache]
    Store --> Return
```

---

## 3. Write Through Pattern

Every write goes to cache and database simultaneously.

```mermaid
flowchart LR
    App[Application]
    App --> Cache[(Cache)]
    App --> DB[(Database)]
    Cache -.->|Always in sync| DB
```

---

## 4. Write Back Pattern

Write to cache first; database updated asynchronously.

```mermaid
flowchart LR
    App[Application]
    App -->|Immediate| Cache[(Cache)]
    Cache -->|Async later| DB[(Database)]
```

---

## 5. Cache Stampede Problem

Many users miss at the same time and overwhelm the DB.

```mermaid
flowchart TD
    TTLExpires[Cache TTL Expires]
    TTLExpires --> R1[Request 1]
    TTLExpires --> R2[Request 2]
    TTLExpires --> R3[Request 3]
    TTLExpires --> RN[... 100K Requests]

    R1 --> DB[(Database)]
    R2 --> DB
    R3 --> DB
    RN --> DB

    DB --> Overload[DB Overloaded ❌]
```

---

## 6. Distributed Cache Cluster

Cache partitioned across multiple nodes.

```mermaid
flowchart LR
    App1[App Server 1]
    App2[App Server 2]
    App3[App Server 3]

    App1 --> Cluster
    App2 --> Cluster
    App3 --> Cluster

    subgraph Cluster [Redis Cluster]
        Node1[(Node 1)]
        Node2[(Node 2)]
        Node3[(Node 3)]
    end
```
