# System Design Diagrams

## 1. Load Balancer Architecture

```mermaid
flowchart TD

    U1[User 1]
    U2[User 2]
    U3[User 3]

    LB[Load Balancer]

    S1[Server 1]
    S2[Server 2]
    S3[Server 3]

    U1 --> LB
    U2 --> LB
    U3 --> LB

    LB --> S1
    LB --> S2
    LB --> S3
```

---

# 2. Round Robin Load Balancing

```mermaid
sequenceDiagram

    participant LB as Load Balancer
    participant S1 as Server 1
    participant S2 as Server 2
    participant S3 as Server 3

    LB->>S1: Request 1
    LB->>S2: Request 2
    LB->>S3: Request 3
    LB->>S1: Request 4
    LB->>S2: Request 5
    LB->>S3: Request 6
```

---

# 3. Sticky Session Load Balancing

```mermaid
flowchart LR

    UserA --> LB
    UserB --> LB

    LB -->|Always| S1[Server 1]
    LB -->|Always| S2[Server 2]

    UserA -.-> S1
    UserA -.-> S1

    UserB -.-> S2
    UserB -.-> S2
```

---

# 4. Vertical Scaling

```mermaid
flowchart TB

    Server1[Server<br/>4 CPU<br/>8GB RAM]

    Server2[Server<br/>16 CPU<br/>64GB RAM]

    Server1 -->|Upgrade Hardware| Server2
```

---

# 5. Horizontal Scaling

```mermaid
flowchart LR

    LB[Load Balancer]

    S1[Server 1]
    S2[Server 2]
    S3[Server 3]
    S4[Server 4]

    LB --> S1
    LB --> S2
    LB --> S3
    LB --> S4
```

---

# 6. CDN Architecture

```mermaid
flowchart LR

    User

    CDN1[Edge Server India]
    CDN2[Edge Server USA]
    CDN3[Edge Server Europe]

    Origin[Origin Server]

    User --> CDN1

    CDN1 -->|Cache Hit| User

    CDN1 -->|Cache Miss| Origin
```

---

# 7. CDN Cache Flow

```mermaid
flowchart TD

    User --> Request

    Request --> Cache{Content Cached?}

    Cache -->|Yes| Return[Return from CDN]

    Cache -->|No| Origin[Fetch from Origin]

    Origin --> Store[Store in Cache]

    Store --> Return
```

---

# 8. Monolithic Architecture

```mermaid
flowchart TB

    Client

    Client --> Monolith

    subgraph Monolith_Application
        Auth
        Product
        Orders
        Payments
        Notification
    end
```

---

# 9. Microservices Architecture

```mermaid
flowchart TB

    Client --> APIGateway

    APIGateway --> AuthService
    APIGateway --> ProductService
    APIGateway --> OrderService
    APIGateway --> PaymentService
    APIGateway --> NotificationService

    AuthService --> DB1[(DB)]
    ProductService --> DB2[(DB)]
    OrderService --> DB3[(DB)]
    PaymentService --> DB4[(DB)]
```

---

# 10. Data Sharding

```mermaid
flowchart TB

    App

    App --> Router

    Router --> Shard1[(Shard 1)]
    Router --> Shard2[(Shard 2)]
    Router --> Shard3[(Shard 3)]

    Shard1 --> U1[Users 1-1M]
    Shard2 --> U2[Users 1M-2M]
    Shard3 --> U3[Users 2M-3M]
```

---

# 11. Hash-Based Sharding

```mermaid
flowchart LR

    UserID

    UserID --> HashFunction

    HashFunction --> Shard1
    HashFunction --> Shard2
    HashFunction --> Shard3
```

---

# 12. Data Replication

```mermaid
flowchart LR

    Primary[(Primary DB)]

    Replica1[(Replica 1)]
    Replica2[(Replica 2)]
    Replica3[(Replica 3)]

    Primary --> Replica1
    Primary --> Replica2
    Primary --> Replica3
```

---

# 13. Read Replica Architecture

```mermaid
flowchart TB

    App

    App -->|Writes| Primary[(Primary)]

    App -->|Reads| Replica1[(Replica1)]
    App -->|Reads| Replica2[(Replica2)]
    App -->|Reads| Replica3[(Replica3)]
```

---

# 14. Cache Architecture

```mermaid
flowchart LR

    User --> App

    App --> Cache[(Redis)]

    Cache -->|Miss| DB[(Database)]

    DB --> Cache

    Cache --> App
```

---

# 15. Cache Aside Pattern

```mermaid
flowchart TD

    Request --> Cache{Cache Hit?}

    Cache -->|Yes| Return

    Cache -->|No| DB

    DB --> Store[Store in Cache]

    Store --> Return
```

---

# 16. Complete High-Level System Design

```mermaid
flowchart LR

    User

    User --> CDN

    CDN --> LB

    LB --> App1
    LB --> App2
    LB --> App3

    App1 --> Cache
    App2 --> Cache
    App3 --> Cache

    Cache --> DBPrimary

    DBPrimary --> Replica1
    DBPrimary --> Replica2

    DBPrimary --> Shard1
    DBPrimary --> Shard2
```

---

# 17. System Design Interview Revision Diagram

```mermaid
flowchart TB

    User --> CDN

    CDN --> LoadBalancer

    LoadBalancer --> AppServers

    AppServers --> Cache

    Cache --> Database

    Database --> Replication

    Database --> Sharding

    AppServers --> Microservices
```

---

# Recommended Notes Repository Structure


