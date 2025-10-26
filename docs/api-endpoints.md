# Hoardarr API Endpoints

## Authentication

### Reddit OAuth

#### `GET /api/auth/reddit`
Initiate Reddit OAuth flow.

**Response**: 302 redirect to Reddit authorization

---

#### `GET /api/auth/reddit/callback`
Handle Reddit OAuth callback.

**Query Params**:
- `code`: Authorization code
- `state`: CSRF state token

**Response**:
```json
{
  "status": "success",
  "user_id": "reddit_username"
}
```

---

### GitHub OAuth

#### `GET /api/auth/github`
Initiate GitHub OAuth flow.

**Response**: 302 redirect to GitHub authorization

---

#### `GET /api/auth/github/callback`
Handle GitHub OAuth callback.

**Query Params**:
- `code`: Authorization code
- `state`: CSRF state token

**Response**:
```json
{
  "status": "success",
  "user_id": "github_username"
}
```

---

## Sync Operations

### Sync Reddit

#### `POST /api/sync/reddit`
Trigger Reddit sync (delta or full reconciliation).

**Request Body**:
```json
{
  "user_id": "reddit_username",
  "full_reconciliation": false
}
```

**Response**:
```json
{
  "status": "completed",
  "items_processed": 45,
  "items_added": 12,
  "items_removed": 3
}
```

---

### Sync GitHub

#### `POST /api/sync/github`
Trigger GitHub sync (delta or full reconciliation).

**Request Body**:
```json
{
  "user_id": "github_username",
  "full_reconciliation": false
}
```

**Response**:
```json
{
  "status": "completed",
  "items_processed": 87,
  "items_added": 8,
  "items_removed": 1
}
```

---

## Upload Operations (TikTok)

### Upload TikTok Data

#### `POST /api/upload/tiktok`
Upload TikTok data export file.

**Content-Type**: `multipart/form-data`

**Form Data**:
- `file`: user_data.json (or .zip)
- `user_id`: TikTok username

**Response**:
```json
{
  "status": "processing",
  "job_id": "upload_abc123",
  "videos_found": 245,
  "message": "Processing favorites"
}
```

---

### Check Upload Status

#### `GET /api/upload/status/:job_id`
Get status of upload processing job.

**Response**:
```json
{
  "status": "completed",
  "videos_processed": 245,
  "videos_successful": 240,
  "videos_failed": 5,
  "message": "Processing complete"
}
```

---

## Content Retrieval

### Get Reddit Content

#### `GET /api/content/reddit`
Get Reddit saved content with optional filtering.

**Query Params**:
- `user_id` (required): Reddit username
- `subreddit` (optional): Filter by subreddit
- `limit` (optional, default=50): Items per page
- `offset` (optional, default=0): Pagination offset

**Response**:
```json
{
  "total": 234,
  "items": [
    {
      "id": "abc123",
      "kind": "t3",
      "title": "Cool article",
      "subreddit": "programming",
      "url": "https://...",
      "created_utc": 1698765432.0,
      "score": 234
    }
  ]
}
```

---

### Get GitHub Stars

#### `GET /api/content/github`
Get GitHub starred repos with optional filtering.

**Query Params**:
- `user_id` (required): GitHub username
- `topic` (optional): Filter by topic
- `limit` (optional, default=50): Items per page
- `offset` (optional, default=0): Pagination offset

**Response**:
```json
{
  "total": 156,
  "items": [
    {
      "id": 1296269,
      "name": "Hello-World",
      "full_name": "octocat/Hello-World",
      "description": "My first repo",
      "topics": ["api"],
      "html_url": "https://github.com/...",
      "starred_at": "2025-01-15T10:30:00Z"
    }
  ]
}
```

---

### Get TikTok Favorites

#### `GET /api/content/tiktok`
Get TikTok favorited videos with optional filtering.

**Query Params**:
- `user_id` (required): TikTok username  
- `hashtag` (optional): Filter by hashtag
- `limit` (optional, default=50): Items per page
- `offset` (optional, default=0): Pagination offset

**Response**:
```json
{
  "total": 89,
  "items": [
    {
      "video_id": "1234567890",
      "caption": "Cool trick",
      "hashtags": ["fyp", "tutorial"],
      "author_username": "creator",
      "link": "https://www.tiktok.com/@creator/video/...",
      "favorited_at": "2024-10-15T14:23:45Z"
    }
  ]
}
```

---

## RAG Chat

### Chat with Content

#### `POST /api/chat`
RAG-powered chat with saved content.

**Request Body**:
```json
{
  "user_id": "username",
  "message": "What Python articles did I save?",
  "top_k": 5
}
```

**Response**:
```json
{
  "response": "Based on your saved content, you have several Python articles...",
  "sources": [
    {
      "type": "reddit",
      "title": "Python tips for beginners",
      "url": "https://reddit.com/r/programming/...",
      "collection": "reddit_content"
    },
    {
      "type": "github",
      "name": "awesome-python",
      "url": "https://github.com/vinta/awesome-python",
      "collection": "github_stars"
    }
  ]
}
```

---

## Error Responses

All endpoints return errors in this format:

```json
{
  "error": "Error message",
  "details": "Additional context",
  "status": 400
}
```

**Status Codes**:
- `400`: Bad request (invalid input)
- `401`: Unauthorized (missing/invalid auth)
- `404`: Not found
- `429`: Rate limit exceeded
- `500`: Server error
