# Data Replication Diagrams

---

## 1. Primary-Replica Replication

Primary handles writes; replicas serve reads.

```mermaid
flowchart LR
    Primary[(Primary DB)]

    Primary --> Replica1[(Replica 1)]
    Primary --> Replica2[(Replica 2)]
    Primary --> Replica3[(Replica 3)]
```

---

## 2. Read Replica Architecture

Application routes writes to primary and reads to replicas.

```mermaid
flowchart TB
    App[Application]

    App -->|Writes| Primary[(Primary)]
    App -->|Reads| Replica1[(Replica 1)]
    App -->|Reads| Replica2[(Replica 2)]
    App -->|Reads| Replica3[(Replica 3)]
```

---

## 3. Failover

When primary fails, a replica is promoted automatically.

```mermaid
flowchart TB
    subgraph Before [Before Failure]
        P1[(Primary ✅)]
        R1[(Replica 1)]
        R2[(Replica 2)]
        P1 --> R1
        P1 --> R2
    end

    subgraph After [After Failure]
        P2[(Primary ❌)]
        NP[(Replica 1 → New Primary ✅)]
        R3[(Replica 2)]
        P2 -.->|Fails| NP
        NP --> R3
    end
```

---

## 4. Multi-Leader Replication

Both nodes accept writes and sync with each other.

```mermaid
flowchart LR
    PrimaryA[(Primary A)] <-->|Sync| PrimaryB[(Primary B)]

    PrimaryA --> ReplicaA[(Replica A)]
    PrimaryB --> ReplicaB[(Replica B)]
```
