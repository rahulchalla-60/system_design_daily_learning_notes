# Complete System Design Diagrams

---

## 1. High-Level System Design

Full production architecture combining all layers.

```mermaid
flowchart LR
    User --> CDN[CDN Edge]
    CDN --> LB[Load Balancer]

    LB --> App1[App Server 1]
    LB --> App2[App Server 2]
    LB --> App3[App Server 3]

    App1 --> Cache[(Redis Cache)]
    App2 --> Cache
    App3 --> Cache

    Cache --> DBPrimary[(Primary DB)]

    DBPrimary --> Replica1[(Replica 1)]
    DBPrimary --> Replica2[(Replica 2)]

    DBPrimary --> Shard1[(Shard 1)]
    DBPrimary --> Shard2[(Shard 2)]
```

---

## 2. System Design Interview Revision

Key components and how they connect.

```mermaid
flowchart TB
    User --> CDN[CDN]
    CDN --> LB[Load Balancer]
    LB --> AppServers[Application Servers]

    AppServers --> Cache[(Cache — Redis)]
    AppServers --> Microservices[Microservices]

    Cache --> DB[(Database)]
    DB --> Replication[Replication — Scale Reads]
    DB --> Sharding[Sharding — Scale Writes]
```

---

## 3. Complete Request Flow

What happens when a user makes a request end to end.

```mermaid
flowchart TD
    User -->|1 - Request| CDN{CDN Cache?}

    CDN -->|Hit| ReturnStatic[Return Static Asset]
    CDN -->|Miss| LB[Load Balancer]

    LB -->|2 - Route| App[App Server]

    App -->|3 - Check| RedisCache{Redis Cache?}

    RedisCache -->|Hit| ReturnData[Return Data to User]
    RedisCache -->|Miss| DB[(Database)]

    DB -->|4 - Fetch| App
    App -->|5 - Store| RedisCache
    App -->|6 - Respond| ReturnData
```

---

## 4. Internet-Scale Formula (Visual)

```mermaid
flowchart LR
    CDN --> LB[Load Balancer]
    LB --> AS[App Servers]
    AS --> Cache[(Redis Cache)]
    Cache --> DBP[(Primary DB)]
    DBP --> Rep[(Replicas)]
    DBP --> Shards[(Shards)]

    style CDN fill:#4A90D9,color:#fff
    style Cache fill:#E34C26,color:#fff
    style DBP fill:#336791,color:#fff
```
