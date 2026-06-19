# Data Sharding Diagrams

---

## 1. Basic Sharding Architecture

A shard router distributes data across multiple database shards.

```mermaid
flowchart TB
    App[Application]
    App --> Router[Shard Router]

    Router --> Shard1[(Shard 1)]
    Router --> Shard2[(Shard 2)]
    Router --> Shard3[(Shard 3)]

    Shard1 --> U1[Users 1 – 1M]
    Shard2 --> U2[Users 1M – 2M]
    Shard3 --> U3[Users 2M – 3M]
```

---

## 2. Hash-Based Sharding

The hash function determines which shard stores a given record.

```mermaid
flowchart LR
    UserID --> HashFunction[Hash Function]

    HashFunction -->|Hash mod 3 = 0| Shard1[(Shard 1)]
    HashFunction -->|Hash mod 3 = 1| Shard2[(Shard 2)]
    HashFunction -->|Hash mod 3 = 2| Shard3[(Shard 3)]
```

---

## 3. Sharding + Replication (Production Pattern)

Each shard has its own replicas for read scalability and fault tolerance.

```mermaid
flowchart TB
    Router[Shard Router]

    Router --> P1[(Shard 1 Primary)]
    Router --> P2[(Shard 2 Primary)]
    Router --> P3[(Shard 3 Primary)]

    P1 --> R1a[(Shard 1 Replica)]
    P2 --> R2a[(Shard 2 Replica)]
    P3 --> R3a[(Shard 3 Replica)]
```
