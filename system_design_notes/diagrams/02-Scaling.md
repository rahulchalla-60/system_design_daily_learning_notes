# Scaling Diagrams

---

## 1. Vertical Scaling (Scale Up)

Upgrade the existing server with more CPU and RAM.

```mermaid
flowchart TB
    Server1[Server
    4 CPU / 8GB RAM]

    Server2[Server
    16 CPU / 64GB RAM]

    Server1 -->|Upgrade Hardware| Server2
```

---

## 2. Horizontal Scaling (Scale Out)

Add more servers behind a load balancer.

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

## Key Difference

| | Vertical | Horizontal |
|---|---|---|
| Method | Bigger server | More servers |
| Limit | Hardware ceiling | Almost unlimited |
| Fault Tolerance | Low | High |
