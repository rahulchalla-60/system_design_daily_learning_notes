# CDN Diagrams

---

## 1. CDN Architecture

Content is served from the edge server closest to the user.

```mermaid
flowchart LR
    User
    CDN1[Edge Server — India]
    CDN2[Edge Server — USA]
    CDN3[Edge Server — Europe]
    Origin[Origin Server]

    User --> CDN1
    CDN1 -->|Cache Hit| User
    CDN1 -->|Cache Miss| Origin
```

---

## 2. CDN Cache Flow

How a request moves through a CDN.

```mermaid
flowchart TD
    User --> Request
    Request --> Cache{Content Cached?}

    Cache -->|Yes| Return[Return from CDN Edge]
    Cache -->|No| Origin[Fetch from Origin Server]

    Origin --> Store[Store in Edge Cache]
    Store --> Return
```

---

## 3. Global CDN Distribution

```mermaid
flowchart LR
    Origin[Origin Server USA]

    Origin --> EdgeUSA[Edge — USA]
    Origin --> EdgeIndia[Edge — India]
    Origin --> EdgeEU[Edge — Europe]

    EdgeUSA --> UsersUSA[US Users]
    EdgeIndia --> UsersIndia[India Users]
    EdgeEU --> UsersEU[EU Users]
```
