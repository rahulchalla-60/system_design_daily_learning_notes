# Load Balancer Diagrams

---

## 1. Basic Load Balancer Architecture

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

## 2. Round Robin Load Balancing

Requests cycle through servers in order.

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

## 3. Sticky Session (Session Affinity)

Each user is always routed to the same server.

```mermaid
flowchart LR
    UserA --> LB
    UserB --> LB

    LB -->|Always routes to| S1[Server 1]
    LB -->|Always routes to| S2[Server 2]

    UserA -.->|Pinned| S1
    UserB -.->|Pinned| S2
```
