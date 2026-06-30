# Day 014 — Instagram News Feed: Requirements

---

## Functional Requirements

*What the system must do.*

### Content & Feed

| # | Requirement |
|---|---|
| 1 | User can create posts (images and videos) |
| 2 | Display a personalised news feed for each user |
| 3 | Show posts from newest to oldest, or ranked by relevance |
| 4 | Infinite scrolling to load more posts |
| 5 | Refresh feed to get new posts |
| 6 | Show advertisements and recommended posts |

### Engagement

| # | Requirement |
|---|---|
| 7 | Like and unlike posts |
| 8 | Comment on posts |
| 9 | Share posts |
| 10 | Save / bookmark posts |

### Notifications

| # | Requirement |
|---|---|
| 11 | Receive notifications for likes on own posts |
| 12 | Receive notifications for comments on own posts |

---

## Non-Functional Requirements

*How well the system must work.*

### Performance

| Requirement | Target |
|---|---|
| **Low Latency** | Feed loads in ≤ 200ms |
| **High Throughput** | Millions of feed requests, likes, and comments per second |
| **Caching** | Cache popular feeds and posts to reduce DB load |
| **CDN Support** | Serve images and videos from geographically nearest servers |

### Reliability & Resilience

| Requirement | What It Means |
|---|---|
| **High Availability** | Feed is always accessible — no unplanned downtime |
| **Fault Tolerance** | System keeps working even if some servers fail |
| **Reliability** | No loss of posts, likes, or comments |
| **Durability** | Uploaded content is never lost, even after crashes |

### Scale & Architecture

| Requirement | What It Means |
|---|---|
| **Scalability** | Handle millions of users and posts — grows with demand |
| **Load Balancing** | Requests distributed evenly across multiple servers |
| **Eventual Consistency** | Likes and comment counts may take a few seconds to appear everywhere |

### Security & Experience

| Requirement | What It Means |
|---|---|
| **Security** | Protect user data, authenticate all requests |
| **Usability** | Simple, responsive, smooth UI experience |
| **Monitoring** | Track server health, latency, errors, and throughput |

---

## Requirements Tree

```
Instagram News Feed
│
├── Functional (What it does)
│   ├── Content & Feed
│   │   ├── Create posts (image/video)
│   │   ├── Personalised feed
│   │   ├── Newest / ranked order
│   │   ├── Infinite scrolling
│   │   ├── Feed refresh
│   │   └── Ads & recommendations
│   │
│   ├── Engagement
│   │   ├── Like / Unlike
│   │   ├── Comment
│   │   ├── Share
│   │   └── Save / Bookmark
│   │
│   └── Notifications
│       ├── Like notifications
│       └── Comment notifications
│
└── Non-Functional (How well it works)
    ├── Performance
    │   ├── Latency ≤ 200ms
    │   ├── High throughput
    │   ├── Caching
    │   └── CDN
    │
    ├── Reliability & Resilience
    │   ├── High availability
    │   ├── Fault tolerance
    │   ├── Reliability
    │   └── Durability
    │
    ├── Scale & Architecture
    │   ├── Scalability
    │   ├── Load balancing
    │   └── Eventual consistency
    │
    └── Security & Experience
        ├── Security
        ├── Usability
        └── Monitoring
```

---

## Interview Quick Reference

| Category | Key Points |
|---|---|
| **Functional** | Create posts · Personalised feed · Like/Comment/Share/Save · Infinite scroll · Notifications · Ads |
| **Performance** | ≤ 200ms latency · High throughput · CDN · Caching |
| **Reliability** | Always available · Fault tolerant · No data loss · Durable storage |
| **Scale** | Millions of users · Load balanced · Horizontal scaling |
| **Consistency** | Eventual consistency — likes/counts may lag slightly |
| **Security** | Auth on all endpoints · Encrypted data |
| **Ops** | Monitoring · Alerting · Health checks |
