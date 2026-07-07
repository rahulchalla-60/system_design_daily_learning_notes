# Day 018 — Instagram News Feed: Deep Dive

> Topics: Database Selection · Data Modelling · Pre-Signed URLs · Media Processing

---

## Why a Deep Dive?

Instagram is fundamentally a **media storage and distribution system** — not just a social network. The hard problems are:

- Storing billions of posts across different data types
- Delivering media with low latency to users worldwide
- Handling massive concurrent reads and writes
- Keeping the system consistent without becoming a bottleneck

These four topics come up in almost every Instagram-style system design interview.

---

## 1. Database Selection

### Why Not One SQL Database for Everything?

Different data has completely different access patterns:

| Data | Size | Access Pattern | Best Storage |
|---|---|---|---|
| User profiles | Small | Read-heavy, relational | SQL |
| Post metadata | Small | Relational, transactional | SQL |
| Comments | Medium | Ordered, relational | SQL |
| Likes | High volume | Extremely write-heavy | SQL + Redis cache |
| Follower graph | Large | Graph traversal, fan-out | NoSQL / Graph DB |
| Feed cache | High volume | Read-heavy, TTL-based | Redis / NoSQL |
| Images | Large (2–5 MB each) | Binary, write-once, read-many | Object Storage |
| Videos | Very large (50 MB–2 GB) | Streaming, binary | Object Storage |
| Analytics | Massive | Aggregate queries (OLAP) | Data Warehouse |

Forcing everything into one SQL DB causes:
- Huge tables slowing down queries
- Binary files bloating backup and replication
- Write bottlenecks from high-volume tables (likes, events)

---

### Storage Architecture

```
                    Instagram
                        │
     ┌──────────────────┼──────────────────┐
     │                  │                  │
     ▼                  ▼                  ▼
Relational DB       NoSQL / Cache     Object Storage
(PostgreSQL/MySQL)   (Redis/Cassandra)  (S3/GCS/Blob)
     │                  │                  │
Users                Feed cache         Images
Posts metadata       Follower graph      Videos
Comments             Like counters       Thumbnails
Likes (persisted)    Session data        Processed variants
```

---

### SQL — What Goes Here and Why

SQL is the right choice when data is **relational, transactional, or requires consistency**:

```sql
-- A post belongs to a user
-- A like belongs to a user + post
-- A comment belongs to a post + user
-- These relationships are natural SQL territory
SELECT p.caption, u.username, COUNT(l.postId) as likes
FROM Posts p
JOIN Users u ON p.userId = u.userId
JOIN Likes l ON l.postId = p.postId
WHERE p.postId = 'p123';
```

**Why NOT store images in SQL:**
```
1 billion images × 5 MB average = 5 Petabytes in SQL
→ Queries become catastrophically slow
→ Backups take days
→ Replication lag grows unmanageable

Solution: SQL stores only the URL
  mediaURL = "https://cdn.instagram.com/media/abc123.jpg"
  The file lives in Object Storage
```

---

### Object Storage — What Goes Here and Why

Designed specifically for large binary objects:

| Property | Value |
|---|---|
| Durability | 99.999999999% (eleven nines) |
| Scalability | Virtually unlimited |
| Cost | ~$0.023/GB/month (S3 standard) |
| Access pattern | Write-once, read-many via URL |

**Examples:** Amazon S3, Google Cloud Storage, Azure Blob Storage

---

### NoSQL / Cache — What Goes Here and Why

Some data is too large or too fast for SQL:

**Follower graph:**
```
Celebrity with 500M followers:
  SQL: SELECT * FROM followers WHERE userId = 'celebrity'
  → Full table scan, expensive JOIN
  
NoSQL: { userId: "celebrity", followers: [id1, id2, ...] }
  → Fast document read
```

**Like counters in Redis:**
```
Redis: INCR likecount:p123    ← microsecond operation
SQL:  UPDATE posts SET likeCount = likeCount + 1 WHERE postId = 'p123'
  → Acceptable for low volume, bottleneck at millions/sec
```

---

## 2. Data Modelling

Good schema design matters more than database choice.

### Users Table

```sql
CREATE TABLE Users (
  userID      VARCHAR(36)  PRIMARY KEY,
  username    VARCHAR(50)  UNIQUE NOT NULL,
  email       VARCHAR(100) UNIQUE NOT NULL,
  bio         TEXT,
  profilePic  VARCHAR(500),          -- CDN URL
  createdAt   TIMESTAMP    DEFAULT NOW()
);
```

### Posts Table

```sql
CREATE TABLE Posts (
  postID        VARCHAR(36)  PRIMARY KEY,
  userID        VARCHAR(36)  NOT NULL REFERENCES Users(userID),
  caption       TEXT,
  mediaURL      VARCHAR(500),         -- CDN URL, not the file
  mediaType     ENUM('image','video','carousel','text'),
  visibility    ENUM('public','followers','private') DEFAULT 'public',
  likeCount     INT          DEFAULT 0,  -- denormalised counter
  commentCount  INT          DEFAULT 0,  -- denormalised counter
  createdAt     TIMESTAMP    DEFAULT NOW(),

  INDEX idx_user    (userID),
  INDEX idx_created (createdAt DESC)
);
```

> `likeCount` and `commentCount` are stored directly on the post row. This avoids running `COUNT(*)` on millions of rows every time a feed loads. Reading a count becomes O(1).

### Likes Table

```sql
CREATE TABLE Likes (
  userID    VARCHAR(36) NOT NULL,
  postID    VARCHAR(36) NOT NULL,
  createdAt TIMESTAMP   DEFAULT NOW(),

  PRIMARY KEY (userID, postID),    -- composite PK prevents duplicate likes
  INDEX idx_post (postID)
);
```

The composite primary key is the simplest duplicate prevention — a second `INSERT` for the same `(userID, postID)` pair fails at the DB level.

### Comments Table

```sql
CREATE TABLE Comments (
  commentID       VARCHAR(36) PRIMARY KEY,
  postID          VARCHAR(36) NOT NULL REFERENCES Posts(postID),
  userID          VARCHAR(36) NOT NULL REFERENCES Users(userID),
  text            TEXT        NOT NULL,
  parentCommentID VARCHAR(36) NULL,    -- NULL = top-level, non-NULL = reply
  createdAt       TIMESTAMP   DEFAULT NOW(),

  INDEX idx_post_time (postID, createdAt DESC)  -- fast paginated comment loading
);
```

Self-referencing `parentCommentID` handles nested replies without a separate table.

### Followers Table

```sql
CREATE TABLE Followers (
  followerID  VARCHAR(36) NOT NULL,   -- the user who follows
  followingID VARCHAR(36) NOT NULL,   -- the user being followed
  createdAt   TIMESTAMP   DEFAULT NOW(),

  PRIMARY KEY (followerID, followingID),
  INDEX idx_following (followingID)   -- "who follows celebrity X?"
);
```

### Hashtags

```sql
CREATE TABLE Hashtags (
  hashtagID VARCHAR(36)  PRIMARY KEY,
  name      VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE PostHashtags (   -- many-to-many junction
  postID    VARCHAR(36),
  hashtagID VARCHAR(36),
  PRIMARY KEY (postID, hashtagID)
);
```

### Entity Relationship Summary

```
Users ──< Posts ──< Likes ──> Users
           │
           ├──< Comments ──> Users
           │      └── (self-join for replies via parentCommentID)
           │
           └──< PostHashtags >── Hashtags

Users ──< Followers >── Users   (self-referencing)
```

### Index Strategy

| Query | Index Needed |
|---|---|
| Feed: posts by a user | `INDEX(userID)` on Posts |
| Feed: latest posts first | `INDEX(createdAt DESC)` on Posts |
| Likes count for a post | `INDEX(postID)` on Likes |
| Comments for a post | Composite `INDEX(postID, createdAt DESC)` |
| Who follows a user | `INDEX(followingID)` on Followers |

---

## 3. Pre-Signed URLs

One of the most commonly asked questions in media system design interviews.

### The Problem with Routing Uploads Through the API

```
Without Pre-Signed URLs:
  Client → (100 MB video) → API Server → Object Storage

Problems:
  - API server bandwidth saturated
  - Every byte passes through your servers (cost + latency)
  - Hard to scale for millions of concurrent uploads
```

### The Solution — Pre-Signed URL Flow

```
Step 1:  Client → POST /upload/initiate → API Server
Step 2:  API Server → generate signed URL → Object Storage (S3)
Step 3:  S3 returns signed URL to API
Step 4:  API returns signed URL to Client
Step 5:  Client → PUT {bytes} → S3 directly (bypasses API)
Step 6:  S3 confirms upload complete
Step 7:  Client → POST /posts { mediaURL: "https://cdn.../file.jpg" }
Step 8:  API stores metadata, publishes PostCreated event
```

```
Client                API Server              Object Storage
  │                        │                        │
  │── POST /upload/initiate─►                        │
  │                        │── generate signed URL ─►│
  │                        │◄─ signed URL ───────────│
  │◄─ signed URL ──────────│                        │
  │                        │                        │
  │── PUT file bytes ───────────────────────────────►│
  │◄─ 200 OK ───────────────────────────────────────│
  │                        │                        │
  │── POST /posts ─────────►│                        │
  │◄─ { postId: "p123" } ──│                        │
```

### What Makes It "Signed"?

A signed URL contains an embedded cryptographic signature that proves it was issued by your server:

```
https://s3.amazonaws.com/bucket/abc123.mp4
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Expires=600          ← expires in 10 minutes
  &X-Amz-SignedHeaders=host
  &X-Amz-Signature=abc...     ← tamper-proof signature
```

Properties:
- Only valid for the specific file name — can't overwrite other objects
- Expires after a short window (typically 5–15 minutes)
- Read-only or write-only — no other permissions granted
- Prevents malicious uploads to your storage bucket

### Benefits Summary

| Benefit | Impact |
|---|---|
| API never handles file bytes | Eliminates bandwidth bottleneck |
| Uploads go directly to S3 | Lower latency for users |
| Chunked + parallel uploads | Resumable on mobile networks |
| Signed = scoped permissions | Security — can't abuse the URL |
| Offloads compute | No file I/O on API servers |

---

## 4. Media Processing

Uploading the raw file is only the beginning. Instagram transforms every piece of media before it's served to users.

### Why Async Processing?

Video compression can take minutes. If processing happened synchronously:
```
User uploads → wait for compression → wait for transcoding → success ← unacceptable
```

Instead:
```
User uploads → S3 stores raw file → return success immediately
                     ↓ (background)
              Kafka event triggers Media Processor
                     ↓
              Processing happens asynchronously
                     ↓
              Processed URLs saved to DB
                     ↓
              CDN caches processed content
```

### Image Processing Pipeline

```
Original upload: photo.jpg (5 MB, 4000×3000 PNG)
        │
        ▼
1. Compression:     PNG → WebP → AVIF   (~70–80% size reduction)
2. Resize:          4000px → 1080px max  (mobile doesn't need 8K)
3. Multi-resolution:
     thumbnail  = 200px  (feed grid, stories)
     medium     = 720px  (mobile feed view)
     large      = 1080px (desktop, expanded view)
4. Metadata strip:  Remove EXIF GPS data (privacy)
        │
        ▼
Store all versions in S3:
  /posts/p123/thumb.webp
  /posts/p123/medium.webp
  /posts/p123/large.webp
        │
        ▼
Update Post.mediaURL with CDN URLs
```

### Video Processing Pipeline

```
Original upload: video.mp4 (2 GB, 4K)
        │
        ▼
1. Chunked upload:   Split into 64 MB chunks
                     Upload chunks in parallel
                     Resume on failure — only re-upload failed chunk
        │
        ▼
2. Store raw video in S3
        │
        ▼
3. Kafka event: "VideoUploaded"
        │
        ▼
4. Media Worker picks up event:
   ├── Transcode to: 240p, 360p, 480p, 720p, 1080p
   ├── Generate HLS manifest (for adaptive streaming)
   ├── Extract thumbnail at 3-second mark
   └── Strip GPS metadata
        │
        ▼
5. Store all variants in S3
6. Update Post.mediaURL with CDN URLs for each resolution
```

### Adaptive Streaming (HLS)

Instead of serving one fixed resolution, the player dynamically switches based on network speed:

```
Network speed   → Resolution served
─────────────────────────────────────
< 500 kbps      → 240p
500–1500 kbps   → 360p
1.5–5 Mbps      → 720p
> 5 Mbps        → 1080p
```

The HLS manifest file lists all available resolutions. The player measures bandwidth every few seconds and switches to the appropriate one — seamless, no manual selection needed.

### CDN Caching Layer

```
User requests: https://cdn.instagram.com/posts/p123/medium.webp
                    │
             CDN Edge (nearest PoP)
                    │
          ┌─────────┴─────────┐
       Cache Hit            Cache Miss
          │                    │
     Return file         Fetch from S3
                         Cache at edge
                         Return file
```

Popular content is served entirely from CDN edge locations — the origin S3 bucket may rarely be hit for hot content.

---

## End-to-End Media Lifecycle

```
User Creates Post
        │
        ▼
Request Upload Session (POST /upload/initiate)
        │
        ▼
API Generates Pre-Signed URL (10 min expiry)
        │
        ▼
Client Uploads File Directly to S3 (chunked if video)
        │
        ▼
Client Notifies API: POST /posts { mediaURL }
        │
        ▼
Post Service Saves Metadata to SQL DB
        │
        ▼
Kafka Event: "PostCreated" + "MediaUploaded"
        │
   ┌────┼──────────────────────────┐
   ▼    ▼                          ▼
Feed  Media Processing Workers   Notification
Svc   ├── Compress + convert      Service
      ├── Resize to multi-res
      ├── Transcode video
      └── Generate thumbnail
              │
              ▼
      Store processed files in S3
              │
              ▼
      Update DB with CDN URLs
              │
              ▼
      CDN caches on first request
              │
              ▼
      Followers see optimised content
```

---

## Interview Decision Table

| Topic | Decision | Reason |
|---|---|---|
| User/post metadata | SQL (PostgreSQL) | ACID, relational queries, constraints |
| Images/videos | Object Storage (S3) | Cheap, durable, scales to PB |
| Feed cache / hot data | Redis / NoSQL | Handles massive reads, TTL-based expiry |
| Follower graph at scale | NoSQL / Graph DB | Expensive to JOIN 500M rows in SQL |
| Like counters | Redis + async SQL sync | Handles millions of writes/sec |
| Denormalised counts | `likeCount` on Posts row | O(1) read vs O(n) COUNT(*) |
| Duplicate likes | Composite PK (userID, postID) | DB-level enforcement |
| Indexes | userID, postID, createdAt DESC | Covers 95% of common queries |
| Media upload | Pre-Signed URL (client → S3 direct) | API never handles file bytes |
| Image optimisation | Compress + WebP + multi-resolution | ~80% bandwidth reduction |
| Video optimisation | Chunked upload + HLS transcoding | Resumable, adaptive playback |
| Media processing | Async workers via Kafka | Keeps upload API fast |
| Content delivery | CDN (CloudFront / Akamai) | Low latency, reduces S3 load |
| EXIF / GPS stripping | In media processor | User privacy |
