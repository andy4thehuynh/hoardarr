# Hoardarr Architecture

## System Overview

```
┌─────────────────┐
│   Vue Frontend  │
│  (Port 3000)    │
└────────┬────────┘
         │ HTTP/REST
         ▼
┌─────────────────────────────────────┐
│        Flask Backend (Port 5000)    │
│  ┌──────────────────────────────┐  │
│  │   Routes Layer               │  │
│  │  - auth.py                   │  │
│  │  - sync.py                   │  │
│  │  - content.py                │  │
│  │  - chat.py                   │  │
│  │  - upload.py (TikTok)        │  │
│  └──────────┬───────────────────┘  │
│             ▼                       │
│  ┌──────────────────────────────┐  │
│  │   Services Layer             │  │
│  │  - embedding_service.py      │  │
│  │  - vectorizer.py             │  │
│  │  - couchbase_service.py      │  │
│  │  - rag_service.py            │  │
│  └──────────┬───────────────────┘  │
│             ▼                       │
│  ┌──────────────────────────────┐  │
│  │   Providers Layer            │  │
│  │  - base_provider.py          │  │
│  │  - file_upload_provider.py   │  │
│  │  - reddit_provider.py        │  │
│  │  - github_provider.py        │  │
│  │  - tiktok_provider.py        │  │
│  └──────────────────────────────┘  │
└──────────┬──────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│   External Services                  │
│  - Reddit API                        │
│  - GitHub API                        │
│  - TikTok (user uploads)             │
│  - OpenAI (embeddings)               │
│  - Anthropic (Claude chat)           │
└──────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│   Couchbase Capella                  │
│  - reddit_content collection         │
│  - github_stars collection           │
│  - tiktok_favorites collection       │
│  - sync_metadata collection          │
│  - content_vector_index (1536-dim)   │
└──────────────────────────────────────┘
```

---

## Core Patterns

### 1. Provider Abstraction

**Problem**: Need to support multiple content sources with different APIs
**Solution**: Abstract base classes for shared behavior

```
BaseProvider (abstract)
├── get_vectorization_text()
├── transform_to_db_model()
├── get_item_id()
│
├── APIProvider (abstract) - for OAuth-based sources
│   ├── fetch_items()
│   ├── RedditProvider
│   └── GitHubProvider
│
└── FileUploadProvider (abstract) - for file-based sources
    ├── parse_upload_file()
    ├── extract_items()
    └── TikTokProvider
```

**Benefits**:
- Add new providers by implementing 3-5 methods
- Consistent interface for sync orchestration
- Easy testing in isolation

---

### 2. Service Layer

**Separation of Concerns**:
- **Routes**: Handle HTTP, validate input, return JSON
- **Services**: Business logic, orchestration
- **Providers**: External API communication

**Example Flow**:
```
POST /api/sync/reddit
  ↓
sync.py route validates user_id
  ↓
Calls sync_orchestrator.sync()
  ↓
Orchestrator calls RedditProvider.fetch_items()
  ↓
Orchestrator calls Vectorizer.vectorize_item()
  ↓
Orchestrator calls CouchbaseService.store()
```

---

### 3. Vectorization Pipeline

**Goal**: Transform raw content → searchable embeddings

```
Raw Content
  ↓
Provider.get_vectorization_text()
  ├─ Reddit: extract title or body
  ├─ GitHub: combine name + description  
  └─ TikTok: combine caption + description + hashtags
  ↓
Vectorizer.preprocess()
  ├─ Trim whitespace
  ├─ Truncate to 32k chars (~8k tokens)
  └─ Remove excessive newlines
  ↓
EmbeddingService.embed()
  └─ OpenAI API: text-embedding-3-small
  ↓
1536-dimensional vector
  ↓
Provider.transform_to_db_model()
  └─ Combine metadata + embedding
  ↓
CouchbaseService.store()
```

---

### 4. RAG Architecture

**Retrieval-Augmented Generation**:

```
User Query: "What Python articles did I save?"
  ↓
EmbeddingService.embed(query)
  ↓
Vector: [0.234, -0.567, ...]
  ↓
CouchbaseService.vector_search()
  ├─ Search across all collections
  ├─ Filter by user_id
  ├─ Top K=5 most similar
  ↓
Retrieved Items:
  - Reddit post: "10 Python Tips"
  - GitHub repo: "awesome-python"
  - TikTok: "Python tutorial #coding"
  ↓
RAGService.build_context()
  └─ Format items into readable context
  ↓
Claude API with system prompt + context + query
  ↓
Response: "You saved several Python resources..."
  └─ Include source citations
```

---

## Tech Stack Rationale

### Flask over FastAPI
**Decision**: Flask
**Why**: 
- Sync operations are batch jobs, not real-time streaming
- Simpler mental model for learning project
- Easier debugging with synchronous code
- Can add async later if needed

**Trade-off**: Slower concurrent API calls (acceptable for personal use)

---

### Couchbase over PostgreSQL
**Decision**: Couchbase Capella
**Why**:
- Native vector search (no pgvector extension needed)
- JSON document model fits varied provider schemas
- Flexible schema for adding providers
- Managed cloud service (no ops burden)

**Trade-off**: Less familiar than PostgreSQL (acceptable for learning)

---

### Vue 3 over React
**Decision**: Vue 3 (Composition API)
**Why**:
- Simpler for quick prototypes
- Less boilerplate than React
- Single-file components
- Good for this scale of app

**Trade-off**: Smaller ecosystem (not needed for this project)

---

## Data Flow Examples

### Reddit Sync Flow
```
1. User clicks "Sync Reddit"
   POST /api/sync/reddit {"user_id": "alice"}

2. Backend loads sync_metadata
   last_sync_at: "2025-10-10T12:00:00Z"
   sync_count: 3

3. RedditProvider.fetch_items(since="2025-10-10T12:00:00Z")
   - Paginate through /user/alice/saved
   - Stop when created_utc <= last_sync_at
   - Return 15 new items

4. For each item:
   - Extract title/body
   - Call OpenAI to get embedding
   - Store in reddit_content collection

5. Update sync_metadata:
   last_sync_at: "2025-10-17T14:30:00Z"
   sync_count: 4

6. Return to frontend:
   {"status": "completed", "items_added": 15}
```

---

### TikTok Upload Flow
```
1. User uploads user_data.json
   POST /api/upload/tiktok
   multipart/form-data

2. Backend parses JSON
   Extract "Favorite Videos" array
   Found 120 video URLs

3. For each URL:
   - Check if already in tiktok_favorites
   - If new: Fetch metadata (caption, author, etc.)
   - Extract text for vectorization
   - Get embedding from OpenAI
   - Store in tiktok_favorites

4. Remove items not in upload
   (user unfavorited them)

5. Return status:
   {"processed": 120, "new": 15, "removed": 3}
```

---

### RAG Query Flow
```
1. User asks: "Show me articles about machine learning"
   POST /api/chat {"message": "...", "user_id": "alice"}

2. Embed query
   OpenAI API → [0.123, -0.456, ...]

3. Vector search
   Couchbase FTS query:
   - Search content_vector_index
   - Filter: user_id = "alice"
   - Top 5 similar items

4. Results:
   - Reddit post: "ML basics explained"
   - GitHub repo: "ml-course"
   - TikTok: "Neural networks 101"

5. Build context
   "Context from your saved content:
    1. Reddit (r/machinelearning): ML basics...
    2. GitHub (username/ml-course): ...
    3. TikTok (@ai_teacher): ..."

6. Send to Claude
   System: "You are a helpful assistant..."
   Context: [formatted above]
   User: "Show me articles about machine learning"

7. Claude responds with citations
   Return to frontend with source links
```

---

## Security Considerations

### OAuth Token Storage
- Tokens encrypted using Fernet (symmetric encryption)
- Encryption key in environment variable
- Never logged or exposed in API responses

### User Data Isolation
- All queries filtered by `user_id`
- No cross-user data access
- OAuth state tokens prevent CSRF

### API Key Management
- All keys in `.env` (gitignored)
- Separate keys for dev/prod
- Rate limiting to prevent abuse

---

## Scalability Notes

**Current Design**: Single user / small team
**Bottlenecks if scaling**:
1. Sync is synchronous (blocks request)
2. No job queue for background processing
3. Single Flask instance

**If scaling needed**:
1. Add Celery for background jobs
2. Use Redis for job queue
3. Deploy multiple Flask workers
4. Add caching layer (Redis)
5. Implement proper rate limiting

**For now**: Keep it simple, single user works great

---

## Error Handling Strategy

### Layer-by-layer:

**Routes**: 
- Validate input
- Return 400 for bad requests
- Return 500 for unexpected errors
- Log with context

**Services**:
- Try/catch around external calls
- Return meaningful error objects
- Don't expose internal details

**Providers**:
- Handle API rate limits
- Retry with exponential backoff
- Log API errors with request details

**Database**:
- Handle connection failures
- Validate before storing
- Transaction-like updates where possible

---

## Testing Strategy

### Unit Tests
- Each provider method independently
- Mock external APIs
- Test text extraction logic
- Verify data transformations

### Integration Tests
- OAuth flows end-to-end
- Sync with real APIs (dev accounts)
- Vector search accuracy
- RAG response quality

### Manual Testing
- Click through every UI flow
- Test error cases (bad tokens, etc.)
- Verify filters work correctly
- Check chat with various queries

---

## Monitoring & Debugging

### What to Log
- All API calls (provider, method, status)
- Sync operations (items processed, errors)
- Vector search queries (query, results count)
- Errors with full context

### What NOT to Log
- OAuth tokens or API keys
- User data content
- Full embeddings (too large)

### Debug Checklist
1. Check `.env` loaded correctly
2. Verify API tokens not expired
3. Check rate limits not hit
4. Inspect Couchbase data directly
5. Test embedding service isolated
6. Check vector index exists

---

**Next**: See `providers.md` for implementing new providers
