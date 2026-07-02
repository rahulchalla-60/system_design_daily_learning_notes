# Day 016 — Instagram News Feed: API Design

---

## Design Goals

- Define how mobile/web clients communicate with the backend
- Follow REST conventions — correct HTTP methods and status codes
- Use cursor-based pagination for infinite scrolling at scale
- Keep media (images/videos) out of the API — store in object storage, return URLs only
- Version all endpoints (`/api/v1/`) so future changes don't break existing clients

---

## API Summary Table

| # | Operation | Method | Endpoint |
|---|---|---|---|
| 1 | Create text post | POST | `/api/v1/posts/text` |
| 2 | Create image/video post | POST | `/api/v1/posts/media` |
| 3 | Delete a post | DELETE | `/api/v1/posts/{postId}` |
| 4 | Get news feed | GET | `/api/v1/feed?limit=20&cursor=<cursor>` |
| 5 | Follow a user | POST | `/api/v1/users/{userId}/follow` |
| 6 | Unfollow a user | DELETE | `/api/v1/users/{userId}/follow` |
| 7 | Like a post | POST | `/api/v1/posts/{postId}/like` |
| 8 | Unlike a post | DELETE | `/api/v1/posts/{postId}/like` |
| 9 | Add a comment | POST | `/api/v1/posts/{postId}/comments` |
| 10 | Get comments | GET | `/api/v1/posts/{postId}/comments?limit=20&cursor=<cursor>` |
| 11 | Save a post | POST | `/api/v1/posts/{postId}/save` |
| 12 | Unsave a post | DELETE | `/api/v1/posts/{postId}/save` |
| 13 | Get notifications | GET | `/api/v1/notifications?limit=20&cursor=<cursor>` |

> All endpoints require authentication via `Authorization: Bearer <token>` header.

---

## 1. Create a Text Post

```
POST /api/v1/posts/text
```

**Request body:**
```json
{
  "caption": "Enjoying my vacation!"
}
```

**Response `201 Created`:**
```json
{
  "postId": "p123",
  "message": "Post created successfully"
}
```

---

## 2. Create an Image / Video Post

```
POST /api/v1/posts/media
Content-Type: multipart/form-data
```

**Request body (form fields):**
```
file:     beach.jpg  (or trip.mp4)
caption:  Sunset at Goa 🌅
location: Goa
```

**Response `201 Created`:**
```json
{
  "postId": "p124",
  "mediaUrl": "https://cdn.instagram.com/posts/p124.jpg",
  "message": "Media uploaded successfully"
}
```

> The file is stored in Object Storage (S3 / GCS). The API returns only the CDN URL — the client never receives the raw file bytes through this endpoint.

**Upload flow:**
```
Client
  │  POST /api/v1/posts/media (multipart)
  ▼
API Server
  │  store file
  ▼
Object Storage (S3)
  │  generates URL
  ▼
CDN caches file
  │  URL returned to client
  ▼
Client displays post via CDN URL
```

---

## 3. Delete a Post

```
DELETE /api/v1/posts/{postId}
```

**Response `200 OK`:**
```json
{
  "message": "Post deleted successfully"
}
```

---

## 4. Get News Feed

```
GET /api/v1/feed?limit=20&cursor=abc123
```

**Query parameters:**

| Parameter | Type | Description |
|---|---|---|
| `limit` | integer | Number of posts to return (default: 20, max: 50) |
| `cursor` | string | Pagination cursor from previous response |

**Response `200 OK`:**
```json
{
  "posts": [
    {
      "postId": "p101",
      "userId": "u10",
      "username": "rahul_dev",
      "caption": "Morning Coffee ☕",
      "mediaUrl": "https://cdn.instagram.com/posts/p101.jpg",
      "likes": 520,
      "comments": 30,
      "saved": false,
      "timestamp": "2026-07-02T10:30:00Z"
    }
  ],
  "nextCursor": "xyz789",
  "hasMore": true
}
```

> `nextCursor` is `null` and `hasMore` is `false` when there are no more posts to load.

### Cursor vs Page-Based Pagination

| | Cursor-Based | Page-Based (`?page=2`) |
|---|---|---|
| Works when new posts arrive | ✅ Yes | ❌ Posts shift, duplicates appear |
| Performance at scale | ✅ Fast (index seek) | ⚠️ Slow (OFFSET scans all rows) |
| Used for infinite scrolling | ✅ Perfect | ❌ Poor fit |

---

## 5. Follow a User

```
POST /api/v1/users/{userId}/follow
```

**Response `200 OK`:**
```json
{
  "message": "Followed successfully"
}
```

---

## 6. Unfollow a User

```
DELETE /api/v1/users/{userId}/follow
```

**Response `200 OK`:**
```json
{
  "message": "Unfollowed successfully"
}
```

---

## 7. Like a Post

```
POST /api/v1/posts/{postId}/like
```

**Response `200 OK`:**
```json
{
  "postId": "p101",
  "likes": 521
}
```

---

## 8. Unlike a Post

```
DELETE /api/v1/posts/{postId}/like
```

**Response `200 OK`:**
```json
{
  "postId": "p101",
  "likes": 520
}
```

---

## 9. Add a Comment

```
POST /api/v1/posts/{postId}/comments
```

**Request body:**
```json
{
  "text": "Beautiful shot! 🔥"
}
```

**Response `201 Created`:**
```json
{
  "commentId": "c456",
  "postId": "p101",
  "userId": "u25",
  "text": "Beautiful shot! 🔥",
  "timestamp": "2026-07-02T11:00:00Z"
}
```

---

## 10. Get Comments on a Post

```
GET /api/v1/posts/{postId}/comments?limit=20&cursor=<cursor>
```

**Response `200 OK`:**
```json
{
  "comments": [
    {
      "commentId": "c456",
      "userId": "u25",
      "username": "priya_dev",
      "text": "Beautiful shot! 🔥",
      "timestamp": "2026-07-02T11:00:00Z"
    }
  ],
  "nextCursor": "c457",
  "hasMore": true
}
```

---

## 11. Save a Post

```
POST /api/v1/posts/{postId}/save
```

**Response `200 OK`:**
```json
{
  "message": "Post saved successfully"
}
```

---

## 12. Unsave a Post

```
DELETE /api/v1/posts/{postId}/save
```

**Response `200 OK`:**
```json
{
  "message": "Post unsaved successfully"
}
```

---

## 13. Get Notifications

```
GET /api/v1/notifications?limit=20&cursor=<cursor>
```

**Response `200 OK`:**
```json
{
  "notifications": [
    {
      "notificationId": "n789",
      "type": "like",
      "fromUserId": "u30",
      "fromUsername": "amit_codes",
      "postId": "p101",
      "message": "amit_codes liked your post",
      "read": false,
      "timestamp": "2026-07-02T11:05:00Z"
    },
    {
      "notificationId": "n790",
      "type": "comment",
      "fromUserId": "u25",
      "fromUsername": "priya_dev",
      "postId": "p101",
      "message": "priya_dev commented on your post",
      "read": false,
      "timestamp": "2026-07-02T11:00:00Z"
    }
  ],
  "nextCursor": "n780",
  "hasMore": false
}
```

---

## HTTP Status Codes Used

| Code | Meaning | When |
|---|---|---|
| `200 OK` | Success | GET, DELETE, action endpoints |
| `201 Created` | Resource created | POST (new post, comment) |
| `400 Bad Request` | Invalid input | Missing required fields |
| `401 Unauthorized` | Not authenticated | Missing or invalid token |
| `403 Forbidden` | Not allowed | Deleting someone else's post |
| `404 Not Found` | Resource missing | Post/user doesn't exist |
| `429 Too Many Requests` | Rate limit hit | Spam protection |

---

## API Design Principles Applied

| Principle | How It's Applied |
|---|---|
| Versioning | `/api/v1/` prefix on all endpoints |
| Correct HTTP methods | POST=create, GET=read, DELETE=remove |
| Cursor pagination | All list endpoints use cursor, not page numbers |
| Media via CDN | API returns URL only — files never in API response body |
| Authentication | All endpoints require Bearer token |
| Idempotent DELETEs | Unfollowing/unliking twice returns same result, no error |
