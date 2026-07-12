# Day 019 — Designing YouTube (Video Upload & Streaming)

> Scope: Video Upload · Video Processing · Video Streaming · Data Modelling

---

## Part 1 — Requirements

---

### Functional Requirements

#### Video Upload
- Upload videos in common formats: MP4, MOV, AVI, MKV
- System stores the raw video safely in durable object storage
- System generates multiple resolutions (240p → 1080p)
- System generates a thumbnail automatically
- System extracts and stores metadata (title, duration, owner, etc.)
- Video becomes available for streaming after processing completes

#### Video Streaming
- User opens a video → system authenticates and returns a manifest URL
- Player fetches manifest (`.m3u8`) listing available quality levels
- Player fetches small video segments sequentially
- Player automatically switches quality based on network speed (ABR)
- Supports seeking (jump to any timestamp)
- Supports buffering and resume playback

---

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| **Availability** | 99.99% uptime |
| **Scalability** | Millions of uploads and concurrent streams |
| **Durability** | Videos must never be lost |
| **Low Latency** | Video playback starts within 2–3 seconds |
| **Fault Tolerance** | Encoding failure must not affect other uploads |
| **Concurrency** | Millions of simultaneous streams supported |
| **Metadata Consistency** | Strong — metadata must always be correct |
| **Media Consistency** | Eventual — encoded copies appear after upload completes |

---

### Assumptions (Scale Baseline)

| Metric | Value |
|---|---|
| MAU | 500 million |
| DAU | 100 million |
| Uploads per day | 20 million |
| Average video size | 500 MB |
| Average viewers per day | 100 million |
| Average watch time | 30 minutes |

---

## Part 2 — Capacity Estimation

---

### Traffic Pattern

YouTube is **strongly read-heavy**:

```
Uploads (writes):   20 million/day    ← small fraction
Stream views (reads): billions/day    ← dominant workload

Read : Write ratio ≈ 10,000 : 1
```

This drives the entire architecture — caching and CDN are the most important investments.

---

### Upload Throughput

```
20M uploads/day ÷ 86,400 sec = ~231 uploads/sec (average)
Peak (5× average)             = ~1,150 uploads/sec
```

---

### Storage Estimation

| Item | Calculation | Result |
|---|---|---|
| Raw uploads/day | 20M × 500 MB | **10 PB/day** |
| Encoded copies (3× avg) | 10 PB × 3 | **~30 PB/day** |
| Metadata/day | 20M × 1 KB | **~20 GB/day** |
| 1-year total storage | 40 PB × 365 | **~14.6 EB/year** |

> Encoded copies are stored at multiple resolutions (240p/360p/480p/720p/1080p). Each adds storage but enables adaptive streaming.

---

### Streaming Bandwidth

```
Concurrent viewers (peak)  = ~10 million
Average bitrate (720p)     = ~2.5 Mbps
Total outbound bandwidth   = 10M × 2.5 Mbps = 25 Tbps
```

No single data centre can serve 25 Tbps. This is why CDN is non-negotiable — popular content must be cached at hundreds of edge locations worldwide.

---

### Cache / Memory

What goes in memory:
- Video metadata (title, duration, status, thumbnail URL)
- Frequently requested manifests (`.m3u8` files)
- Popular video segment identifiers
- Session and auth data

What does NOT go in memory:
- Raw video files
- Encoded video segments (too large — these live on CDN/object storage)

---

## Part 3 — API Design

---

### API Summary

| Operation | Method | Endpoint |
|---|---|---|
| Initiate upload | POST | `/api/v1/videos/upload` |
| Get upload status | GET | `/api/v1/videos/{id}/status` |
| Get video metadata | GET | `/api/v1/videos/{id}` |
| Get stream manifest | GET | `/api/v1/videos/{id}/manifest` |
| Fetch manifest file | GET | `/manifest/{videoId}/master.m3u8` |
| Fetch video segment | GET | `/segments/{videoId}/720p/seg001.ts` |
| Delete video | DELETE | `/api/v1/videos/{id}` |
| Search videos | GET | `/api/v1/search?q=<query>&limit=20&cursor=<cursor>` |

---

### 1. Initiate Upload

```
POST /api/v1/videos/upload
Authorization: Bearer <token>
```

**Request body:**
```json
{
  "title": "My Travel Vlog",
  "description": "Trip to Manali",
  "category": "Travel",
  "privacy": "public",
  "fileSize": 524288000,
  "fileType": "video/mp4"
}
```

**Response `201 Created`:**
```json
{
  "videoId": "v_abc123",
  "uploadUrl": "https://storage.googleapis.com/bucket/v_abc123?X-Goog-Signature=...",
  "uploadUrlExpiresAt": "2026-07-12T11:30:00Z",
  "status": "awaiting_upload"
}
```

The client uploads the file directly to `uploadUrl` — the API server never handles the bytes.

---

### 2. Get Upload / Processing Status

```
GET /api/v1/videos/{id}/status
```

**Response `200 OK`:**
```json
{
  "videoId": "v_abc123",
  "uploadStatus": "complete",
  "processingStatus": "encoding",
  "availableResolutions": [],
  "estimatedCompletionSeconds": 120
}
```

**Processing status values:** `awaiting_upload` → `uploaded` → `encoding` → `ready` → `failed`

---

### 3. Get Video Metadata

```
GET /api/v1/videos/{id}
```

**Response `200 OK`:**
```json
{
  "videoId": "v_abc123",
  "title": "My Travel Vlog",
  "description": "Trip to Manali",
  "ownerID": "u_xyz",
  "duration": 425,
  "thumbnailUrl": "https://cdn.youtube.com/thumbs/v_abc123.jpg",
  "viewCount": 15420,
  "likeCount": 830,
  "availableResolutions": ["240p", "360p", "480p", "720p", "1080p"],
  "manifestUrl": "https://cdn.youtube.com/manifest/v_abc123/master.m3u8",
  "createdAt": "2026-07-10T08:00:00Z"
}
```

---

### 4. Fetch HLS Manifest

```
GET /manifest/{videoId}/master.m3u8
```

**Response (HLS master playlist):**
```
#EXTM3U
#EXT-X-VERSION:3

#EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=426x240
240p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480
480p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
720p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/playlist.m3u8
```

---

### 5. Fetch Video Segment

```
GET /segments/{videoId}/720p/seg001.ts
```

Returns a 2–10 second binary video segment. Served entirely from CDN.

---

## Part 4 — High-Level Design

---

### Upload Flow

```
Client
  │  POST /api/v1/videos/upload (metadata only)
  ▼
API Gateway (auth, rate limit)
  │
  ▼
Upload Service
  ├── Save metadata to DB (status: awaiting_upload)
  └── Generate pre-signed URL from Object Storage
                │
                ▼  (returns URL to client)
Client uploads file directly → Object Storage (S3/GCS)
                │
                ▼  (storage triggers event)
Kafka: "VideoUploaded" event
                │
          ┌─────┴──────────────────┐
          ▼                        ▼
   Encoding Workers         Thumbnail Worker
   (FFmpeg cluster)         (extract frame)
          │
   ┌──────┼──────────────────────┐
   ▼      ▼      ▼      ▼       ▼
 240p   360p   480p   720p   1080p
          │
          ▼
   Object Storage (encoded assets)
          │
          ▼
   Kafka: "EncodingComplete" event
          │
          ▼
   Update DB: status=ready, add resolution URLs
          │
          ▼
   CDN pre-warms popular content
```

---

### Streaming Flow

```
Client
  │  GET /api/v1/videos/{id}
  ▼
API Gateway
  │
  ▼
Video Service
  │
  ├── Check Redis cache for metadata
  │      Hit → return immediately
  │      Miss → query Metadata DB → cache result
  │
  └── Return manifestUrl to client

Client
  │  GET /manifest/{id}/master.m3u8 (via CDN)
  ▼
CDN Edge Server
  │
  ├── Cache Hit  → return manifest
  └── Cache Miss → fetch from Object Storage → cache → return

Client player parses manifest, starts requesting segments:
  GET /segments/{id}/720p/seg001.ts
  GET /segments/{id}/720p/seg002.ts
  ...  (all served from CDN)
```

---

## Part 5 — Upload Deep Dive

---

### Why Pre-Signed URLs?

```
Without pre-signed URL:
  Client → (500 MB file) → API Server → Object Storage
  Problem: API server handles all bytes → bandwidth bottleneck

With pre-signed URL:
  Client → POST /upload (metadata) → API Server → returns signed URL
  Client → (500 MB file) → Object Storage directly
  API Server never sees the file
```

**What a signed URL contains:**
```
https://storage.googleapis.com/bucket/v_abc123.mp4
  ?X-Goog-Algorithm=GOOG4-RSA-SHA256
  &X-Goog-Expires=600              ← 10-minute window
  &X-Goog-SignedHeaders=host
  &X-Goog-Signature=abc123...      ← tamper-proof
```

---

### Chunked Upload for Large Files

For videos > 100 MB, use chunked (resumable) uploads:

```
500 MB video → split into 5 MB chunks
  chunk_001.bin → upload
  chunk_002.bin → upload
  chunk_003.bin → network drops → retry chunk_003 only
  chunk_004.bin → upload
  chunk_005.bin → upload
  → server assembles chunks → complete file
```

Benefits:
- Resume after network failure — don't restart from zero
- Parallel chunk uploads (faster on good connections)
- Better reliability on mobile networks

---

### Event-Driven Encoding

Encoding is CPU-intensive and can take minutes. It must never block the upload response:

```
Upload completes at 10:00:00
→ Return "upload successful" to client immediately
→ Kafka event fires asynchronously
→ Encoding starts at 10:00:01
→ 720p ready at 10:02:30
→ 1080p ready at 10:04:00
→ All resolutions ready at 10:05:00
→ DB updated: status = ready
```

If encoding fails, the Kafka consumer retries. Other uploads are completely unaffected.

---

### HLS Encoding — How Segmentation Works

Each resolution is split into small `.ts` segments plus a playlist:

```
720p/
  playlist.m3u8      ← lists all segments
  seg001.ts          ← seconds 0–4
  seg002.ts          ← seconds 4–8
  seg003.ts          ← seconds 8–12
  ...
```

**Why segments?**
- Player can start after downloading only 1 segment (~4 seconds)
- Seeking jumps to the right segment without downloading the whole video
- Quality switching happens at segment boundaries — seamless
- CDN caches individual segments — popular segments get massive cache hit rates

---

## Part 6 — Streaming Deep Dive

---

### How the Player Works

```
1. Player downloads master.m3u8 (tiny file, <1 KB)
        ↓
2. Player selects initial quality (usually 360p or auto)
        ↓
3. Player downloads quality playlist: 360p/playlist.m3u8
        ↓
4. Player requests segments in order:
     seg001.ts → buffer
     seg002.ts → buffer
     seg003.ts → ...
        ↓
5. Every ~10 seconds, player measures download speed
   Fast  → switch to 720p at next segment boundary
   Slow  → switch to 240p at next segment boundary
```

Because all resolutions are cut at identical timestamps, the switch is seamless — the viewer never notices.

---

### CDN Caching Strategy

```
First viewer (cold cache):
  Player → CDN Edge → Cache Miss → Origin (S3) → Cache segment → Serve

All subsequent viewers (warm cache):
  Player → CDN Edge → Cache Hit → Serve immediately
```

Popular segments (top 1% of videos) serve 99% of traffic entirely from CDN edges. Origin storage is rarely hit.

**Cache TTL guidance:**
- Manifest files (`.m3u8`): short TTL (60–300 sec) — content may update
- Video segments (`.ts`): long TTL (days–weeks) — immutable once encoded
- Thumbnails: medium TTL (hours)

---

### Adaptive Bitrate (ABR) — Quality Ladder

```
Network speed       Resolution    Bitrate    Use case
─────────────────────────────────────────────────────
< 500 kbps          240p          400 kbps   Slow 2G/3G
500 kbps – 1 Mbps   360p          800 kbps   Moderate mobile
1 – 2 Mbps          480p          1.5 Mbps   Good mobile
2 – 5 Mbps          720p          2.8 Mbps   WiFi / 4G
> 5 Mbps            1080p         5.0 Mbps   Fast WiFi / 5G
```

---

## Part 7 — Data Modelling

---

### Videos Table

```sql
CREATE TABLE Videos (
  videoID          VARCHAR(36)  PRIMARY KEY,
  ownerID          VARCHAR(36)  NOT NULL,
  title            VARCHAR(255) NOT NULL,
  description      TEXT,
  duration         INT,                           -- seconds
  uploadStatus     ENUM('awaiting','uploaded','encoding','ready','failed'),
  processingStatus ENUM('pending','processing','complete','failed'),
  thumbnailURL     VARCHAR(500),                  -- CDN URL
  viewCount        BIGINT       DEFAULT 0,
  likeCount        INT          DEFAULT 0,
  privacy          ENUM('public','unlisted','private') DEFAULT 'public',
  createdAt        TIMESTAMP    DEFAULT NOW(),
  updatedAt        TIMESTAMP    DEFAULT NOW(),

  INDEX idx_owner   (ownerID),
  INDEX idx_status  (processingStatus),
  INDEX idx_created (createdAt DESC)
);
```

### VideoAssets Table (one row per encoded resolution)

```sql
CREATE TABLE VideoAssets (
  assetID      VARCHAR(36)  PRIMARY KEY,
  videoID      VARCHAR(36)  NOT NULL REFERENCES Videos(videoID),
  resolution   ENUM('240p','360p','480p','720p','1080p','4k'),
  codec        VARCHAR(20)  DEFAULT 'h264',
  bitrate      INT,                               -- kbps
  manifestURL  VARCHAR(500),                      -- CDN URL to playlist.m3u8
  storagePath  VARCHAR(500),                      -- S3/GCS path
  fileSize     BIGINT,                            -- bytes
  createdAt    TIMESTAMP    DEFAULT NOW(),

  INDEX idx_video (videoID)
);
```

### Thumbnails Table

```sql
CREATE TABLE Thumbnails (
  thumbnailID  VARCHAR(36)  PRIMARY KEY,
  videoID      VARCHAR(36)  NOT NULL REFERENCES Videos(videoID),
  imageURL     VARCHAR(500),                      -- CDN URL
  width        INT,
  height       INT,
  isDefault    BOOLEAN      DEFAULT FALSE,
  createdAt    TIMESTAMP    DEFAULT NOW(),

  INDEX idx_video (videoID)
);
```

### Entity Relationships

```
Videos ──< VideoAssets   (one video → many encoded resolutions)
Videos ──< Thumbnails    (one video → multiple thumbnail options)
Users  ──< Videos        (one user → many videos)
```

---

## Part 8 — Common Bottlenecks & Solutions

| Bottleneck | Cause | Solution |
|---|---|---|
| Encoding workers saturated | Upload spike (live events, viral content) | Auto-scale worker pool, priority queues |
| Origin storage overwhelmed | CDN cache hit rate drops | Pre-warm CDN for popular videos, increase CDN TTL |
| Metadata DB hotspot | Extremely popular video's row locked constantly | Use Redis to serve view/like counts, async DB sync |
| Large upload fails | Unstable mobile network | Chunked resumable uploads — retry only failed chunks |
| Cold CDN cache | New upload or infrequently watched video | Accept cold start latency, no practical solution at low scale |
| Manifest staleness | Video still processing when user opens it | Poll `/status` endpoint, show "processing" state in UI |

---

## Part 9 — Trade-off Decisions

| Decision | Choice | Reason |
|---|---|---|
| Video storage | Object Storage (S3/GCS) | Cheap, durable, scales to exabytes |
| Metadata storage | SQL (PostgreSQL) | Strong consistency, relational queries |
| Like/view counters | Redis + async SQL sync | SQL can't handle millions of increments/sec |
| Upload mechanism | Pre-signed URL (client → storage direct) | API server never handles large file bytes |
| Large video uploads | Chunked resumable | Reliable on mobile networks, resumable |
| Encoding trigger | Kafka event (async) | Decouples upload from encoding, retries, fault isolation |
| Streaming format | HLS (.m3u8 + .ts segments) | Universal support, enables ABR, CDN-friendly |
| Quality switching | Adaptive Bitrate (ABR) | Seamless quality changes based on network |
| Content delivery | CDN (CloudFront/Akamai) | 25 Tbps+ cannot be served from a single origin |
| Metadata caching | Redis | Thousands of requests/sec for popular video metadata |

---

## Part 10 — Interview Discussion Points

Be ready to explain each of these clearly:

1. **Why not store videos in a relational database?** — Binary blobs at petabyte scale destroy query performance, backup times, and replication lag
2. **Why are uploads asynchronous and event-driven?** — Encoding takes minutes; blocking the upload response would be terrible UX
3. **Why pre-signed URLs?** — API servers handling gigabytes of file data would become the bottleneck immediately
4. **How does HLS enable adaptive bitrate streaming?** — Segments at identical timestamps across all resolutions allow seamless quality switching
5. **Why is CDN non-negotiable for YouTube-scale?** — 25 Tbps+ outbound bandwidth cannot come from a single origin; edge caching is the only solution
6. **How are metadata and media separated?** — SQL holds structured data (1 KB/video), S3 holds binary files (500 MB+)
7. **How does the system scale independently?** — Upload service, encoding workers, streaming service, and CDN all scale horizontally and independently
8. **Strong vs eventual consistency trade-off** — Metadata (title, owner) is strongly consistent via SQL; encoded copies are eventually consistent — a video may show "processing" briefly after upload

---

## End-to-End Flow Summary

```
Upload:
  Client → POST /upload (metadata) → API → pre-signed URL
  Client → PUT file → Object Storage (direct)
  Storage → Kafka: "VideoUploaded"
  Kafka → Encoding Workers → FFmpeg (5 resolutions)
  Workers → Object Storage (encoded assets)
  Workers → Kafka: "EncodingComplete"
  Kafka → Update DB: status=ready

Stream:
  Client → GET /videos/{id} → API → Redis cache → DB
  API → return manifestUrl
  Client → GET manifest.m3u8 → CDN
  CDN cache hit → serve segments
  CDN cache miss → fetch from S3 → cache → serve
  Player → ABR: measures bandwidth → switches quality at segment boundaries
```
