# Monolith vs Microservices Diagrams

---

## 1. Monolithic Architecture

All modules live inside one deployable unit.

```mermaid
flowchart TB
    Client --> Monolith

    subgraph Monolith [Monolith Application]
        Auth
        Product
        Orders
        Payments
        Notification
    end
```

**Characteristics:**
- Simple to develop initially
- Single deployment unit
- Harder to scale individual parts
- One failure can affect everything

---

## 2. Microservices Architecture

Each service is independent, owns its own database, and communicates via API.

```mermaid
flowchart TB
    Client --> APIGateway[API Gateway]

    APIGateway --> AuthService[Auth Service]
    APIGateway --> ProductService[Product Service]
    APIGateway --> OrderService[Order Service]
    APIGateway --> PaymentService[Payment Service]
    APIGateway --> NotificationService[Notification Service]

    AuthService --> DB1[(Auth DB)]
    ProductService --> DB2[(Product DB)]
    OrderService --> DB3[(Order DB)]
    PaymentService --> DB4[(Payment DB)]
```

**Characteristics:**
- Each service deploys independently
- Scale only what needs scaling
- More operational complexity
- Fault isolation per service
