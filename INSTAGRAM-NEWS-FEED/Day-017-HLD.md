# Day 017 — Instagram News Feed: High-Level Design (HLD)

---

## Features Covered

| Feature | Description |
|---|---|
| 1 | Create Post (text, image, video) |
| 2 | Optimised Media Uploads |
| 3 | Like a Post |
| 4 | Comment on a Post |

---

## Overall Architecture

```
                        Client (Mobile / Web)
                                │
                                ▼
                         Load Balancer
                                │
                         API Gateway
                    (Auth, Rate Limiting, Routing)
                                │
          ┌─────────────────────┼──────────────────────┐
          │                     │                      │
          ▼                     ▼                      ▼
    Post Service           Like Service          Comment Service
          │                     │                      │
          │                     │                      │
    ┌─────┴──────┐         Redis Cache           Comment DB
    │            │               │                     │
Metadata DB  Media Upload        └──────────┬──────────┘
  (SQL)       Service                       │
                 │                          ▼
           Object Storage         Kafka Event Bus
           (S3 / GCS)                      │
                 │          ┌──────────────┼──────────────┐
                CDN         ▼              ▼              ▼
                       Feed Service  Notification   Analytics /
                            │         Service       ML Engine
                            ▼
                      Followers' Feeds
```

---

## Feature 1 — Create Post

### What Happens

A user creates a post (text, image, or video). The post is stored as metadata in a database. Media files are stored in object storage and served via CDN. Followers see the post in their news feed.

### Architecture Flow

```
Client
  │  POST /api/v1/posts/media
  ▼
API Gateway (validate JWT, rate limit)
  │
  ▼
Post Service
  ├──→ Save metadata to SQL DB
  │     (postId, userId, caption, mediaUrl, timestamp, visibility)
  │
  ├──→ Media Upload Service
  │         │
  │         ▼
  │    Object Storage (S3)
  │         │
  │         ▼
  │    Media Processor (async via Kafka)
  │         ├── Compress image/video
  │         ├── Generate thumbnail
  │         ├── Convert to WebP / multiple resolutions
  │         └── Save processed URLs back to DB
  │
  └──→ Publish "PostCreated" event to Kafka
            │
            ├──→ Feed Service   (push to followers' feeds)
            ├──→ Notification   (notify tagged users)
            └──→ Analytics      (trending, recommendations)
```

### Database Schema

**Posts Table**
```
PostID      VARCHAR  PRIMARY KEY
UserID      VARCHAR  NOT NULL
Caption     TEXT
MediaURL    VARCHAR             ← CDN URL, not the file itself
MediaType   ENUM(image, video, carousel, text)
Visibility  ENUM(public, followers, private)
LikeCount   INT      DEFAULT 0  ← denormalised counter
CommentCount INT     DEFAULT 0  ← denormalised counter
CreatedAt   TIMESTAMP
```

**Users Table**
```
UserID      VARCHAR  PRIMARY KEY
Username    VARCHAR
ProfilePic  VARCHAR  (CDN URL)
FollowerCount INT
```

### Why These Design Choices

| Decision | Reason |
|---|---|
| SQL for metadata | Strong consistency, relational queries (JOIN user + post) |
| Object storage for media | Cheap, durable, scales to PB — never store files in DB |
| CDN in front of S3 | Low latency global delivery, reduces S3 costs |
| Denormalised counters | Avoid `COUNT(*)` on every feed load — just read the column |
| Kafka for post creation | Decouples feed update, notification, and analytics |

---

## Feature 2 — Optimised Media Uploads

### The Problem

Routing large file uploads through API servers wastes bandwidth and creates bottlenecks.

**Bad approach:**
```
Client → API Server → Object Storage
         ↑ bottleneck — handles all bytes
```

**Good approach — Signed URL:**
```
Client → API Server (request upload URL)
       ← Signed URL returned
Client → Object Storage directly (upload bytes)
Client → API Server (notify upload complete)
```

The API server never touches the file. It only generates a temporary signed URL.

### Image Optimisation Pipeline

```
Original upload (e.g. 5 MB PNG)
        │
        ▼
1. Compress     → 5 MB → ~700 KB
2. Resize       → 8000px → 1080px max
3. Convert      → PNG → WebP (60% smaller)
4. Multi-res    → thumbnail (200px), medium (720px), original (1080px)
        │
        ▼
Store all versions in S3 → index URLs in DB
```

### Video Optimisation Pipeline

```
Original upload (e.g. 2 GB MP4)
        │
        ▼
1. Chunked upload  → split into 5–10 MB chunks
   (each uploaded independently, parallelisable, resumable)
        │
        ▼
2. Background processing via Kafka:
   ├── Transcode to multiple resolutions: 240p, 360p, 720p, 1080p
   ├── Generate thumbnail at 3s mark
   └── Generate HLS manifest for adaptive streaming
        │
        ▼
Store all variants in S3 → serve via CDN
```

### Lazy Loading

Only images visible in the viewport are downloaded:

```
Feed loaded:
  Post 1  ← downloaded ✅
  Post 2  ← downloaded ✅
  Post 3  ← downloaded ✅
  ─────── (scroll boundary)
  Post 50 ← not downloaded yet ⏳
```

### Adaptive Streaming

Instead of serving one fixed video quality:

```
User on 4G   → 1080p
User on 3G   → 720p
User on 2G   → 360p
User on weak → 240p
```

The player auto-switches quality based on network speed using HLS (HTTP Live Streaming).

---

## Feature 3 — Like a Post

### Challenge

Likes are both read-heavy AND write-heavy:
- Millions of like/unlike actions per second
- Every feed request reads like counts for every post
- Must prevent duplicate likes from the same user

### Architecture Flow

```
User clicks Like
      │
      ▼
Like Service
      │
      ├──→ Check Redis: has user already liked this post?
      │         YES → return "already liked" (idempotent)
      │         NO  ↓
      │
      ├──→ Write to Redis:
      │         SET liked:{userId}:{postId} = 1
      │         INCR likecount:{postId}
      │
      ├──→ Return success to client (fast — Redis only)
      │
      └──→ Async via Kafka:
                ├── Persist like to SQL DB (eventually consistent)
                ├── Notify post owner
                └── Update analytics / recommendations
```

### Why Redis for Likes?

| Approach | Latency | Problem |
|---|---|---|
| Write directly to SQL | High | DB becomes bottleneck at millions/sec |
| Write to Redis, sync to SQL async | Low | Slight eventual consistency (acceptable) |

### Database Schema

**Likes Table** (persisted asynchronously)
```
UserID   VARCHAR
PostID   VARCHAR
LikedAt  TIMESTAMP
PRIMARY KEY (UserID, PostID)   ← composite key prevents duplicates
```

The composite primary key is the simplest way to prevent a user liking the same post twice — attempting to insert a duplicate row fails at the DB level.

### API

```
POST   /api/v1/posts/{postId}/like    ← like
DELETE /api/v1/posts/{postId}/like    ← unlike
```

---

## Feature 4 — Comment on a Post

### Challenges

- Write-heavy (millions of comments/day)
- Must support nested replies
- Must be paginated (can't load 10,000 comments at once)
- Must update comment count on the post

### Architecture Flow

```
User submits comment
      │
      ▼
Comment Service
      │
      ├──→ Save to Comment DB
      │
      ├──→ Increment CommentCount on Post (SQL update)
      │
      └──→ Publish "CommentCreated" to Kafka
                ├── Notification Service (notify post owner)
                └── Feed Service (update engagement signals)
```

### Database Schema

**Comments Table**
```
CommentID        VARCHAR  PRIMARY KEY
PostID           VARCHAR  NOT NULL
UserID           VARCHAR  NOT NULL
Text             TEXT
ParentCommentID  VARCHAR  NULL   ← NULL = top-level, non-NULL = reply
CreatedAt        TIMESTAMP

INDEX on (PostID, CreatedAt)     ← for fast paginated fetching
```

Nested replies are handled with a self-referencing `ParentCommentID`. A `NULL` parent means top-level comment. A non-null parent means it's a reply.

### Pagination

Never fetch all comments at once. Use cursor-based pagination:

```
GET /api/v1/posts/{postId}/comments?limit=20&cursor=<cursor>

First load:  returns comments 1–20, nextCursor = "c021"
Next load:   returns comments 21–40, nextCursor = "c041"
End:         nextCursor = null, hasMore = false
```

### API

```
POST   /api/v1/posts/{postId}/comments      ← add comment
GET    /api/v1/posts/{postId}/comments      ← get comments (paginated)
DELETE /api/v1/comments/{commentId}         ← delete comment
```

---

## Kafka Event Bus — Full Picture

Kafka decouples all services. Each service publishes events and other services consume independently:

```
Post Service   → topic: posts       → Feed Service, Notification, Analytics
Like Service   → topic: likes       → Notification, Analytics, Recommendations
Comment Service → topic: comments   → Notification, Feed Service, Analytics
Media Service  → topic: media       → Media Processor (compress, transcode, thumbnail)
```

If Notification Service goes down temporarily, it replays missed events from Kafka when it recovers. No data loss.

---

## Interview Decision Table

| Feature | Design Choice | Reason |
|---|---|---|
| Media storage | Object Storage (S3) | Cheap, durable, infinite scale |
| Metadata | SQL Database | Consistency, JOINs, transactions |
| Image delivery | CDN | Low latency, global caching |
| Upload flow | Signed URL | API servers never touch file bytes |
| Video upload | Chunked | Resume support, parallel chunks |
| Image size | Compress + WebP + multi-res | Bandwidth and load time |
| Video quality | Transcode + adaptive HLS | Works on all network speeds |
| Like count | Redis + async SQL sync | Handles millions/sec writes |
| Duplicate likes | Composite PK (UserID, PostID) | DB-level enforcement |
| Comment replies | ParentCommentID (self-join) | Simple, flexible nesting |
| Comment loading | Cursor-based pagination | Scales, no OFFSET scan |
| Service decoupling | Kafka event bus | Independent scaling, replay, resilience |
