# Hoardarr Implementation Tasks

**Total Duration**: ~4 weeks (1-2 hours per task)

---

## Week 1: Backend Foundation

### Task 1: Basic Flask Setup ⏱️ 1-2 hours

**Goal**: Get Flask running with health check.

**What to build**:
- `backend/app.py` - Flask app initialization
- `backend/config.py` - Load environment variables
- Health check endpoint

**Files to create**:
- `app.py`
- `config.py`

**Success criteria**:
```bash
python app.py
# Visit http://localhost:5000/health
# Should return: {"status": "healthy"}
```

**Ask Claude Code**:
```
"Guide me through creating app.py with Flask basics.
I'll write the code, you review each part."
```

---

### Task 2: Couchbase Connection ⏱️ 2-3 hours

**Goal**: Connect to Couchbase Capella.

**What to build**:
- `services/couchbase_service.py`
- Methods: connect, store, query, delete

**Reference**: `docs/database.md`

**Success criteria**:
```python
# test_couchbase.py
from services.couchbase_service import CouchbaseService

db = CouchbaseService()
db.connect()
doc = {"test": "data", "user_id": "test_user"}
db.store("reddit_content", doc)
print("Success!")
```

**Ask Claude Code**:
```
"I need to implement CouchbaseService.
Guide me through connection setup and CRUD operations."
```

---

### Task 3: Base Provider Abstraction ⏱️ 1 hour

**Goal**: Create provider base classes.

**What to build**:
- `providers/base_provider.py`
- `providers/api_provider.py` (extends BaseProvider)
- `providers/file_upload_provider.py` (extends BaseProvider)

**Reference**: `docs/providers.md`

**Success criteria**:
```python
# Try creating test provider
class TestProvider(APIProvider):
    pass

# Should fail - abstract methods not implemented
```

**Ask Claude Code**:
```
"Help me understand abstract base classes in Python.
Then guide me through creating BaseProvider."
```

---

### Task 4: Reddit OAuth Flow ⏱️ 3-4 hours

**Goal**: Implement Reddit OAuth end-to-end.

**What to build**:
- `routes/auth.py` (Reddit endpoints)
- Token encryption utility
- Store tokens in Couchbase

**Reference**: `docs/providers.md` (Reddit section)

**Success criteria**:
```bash
# 1. Visit http://localhost:5000/api/auth/reddit
# 2. Authorize on Reddit
# 3. Redirected back successfully
# 4. Check Couchbase: sync_metadata document exists
```

**Ask Claude Code**:
```
"I'm implementing Reddit OAuth.
Walk me through the flow step by step."
```

---

### Task 5: Reddit Provider ⏱️ 3-4 hours

**Goal**: Fetch Reddit saved items.

**What to build**:
- `providers/reddit_provider.py`
- Implement all abstract methods
- Handle pagination

**Reference**: `docs/providers.md` (Reddit implementation)

**Success criteria**:
```python
# test_reddit_provider.py
provider = RedditProvider()
items = provider.fetch_items(access_token)
print(f"Fetched {len(items)} items")

# Verify:
# - Items have correct structure
# - Pagination works
# - Both posts and comments handled
```

**Ask Claude Code**:
```
"I need to implement RedditProvider.fetch_items() with pagination.
Show me the structure, I'll write it."
```

---

## Week 2: Vectorization & Sync

### Task 6: Embedding Service ⏱️ 2 hours

**Goal**: Integrate OpenAI embeddings.

**What to build**:
- `services/embedding_service.py`
- Handle rate limits and errors
- Batch processing

**Reference**: `docs/vectorization.md`

**Success criteria**:
```python
# test_embeddings.py
embedder = EmbeddingService(api_key)
text = "Test text for embedding"
embedding = embedder.embed(text)

print(f"Dimensions: {len(embedding)}")  # Should be 1536
```

**Ask Claude Code**:
```
"Guide me through implementing OpenAI embedding service.
How should I handle errors and rate limits?"
```

---

### Task 7: Vectorizer Service ⏱️ 2 hours

**Goal**: Coordinate vectorization pipeline.

**What to build**:
- `services/vectorizer.py`
- Text preprocessing
- Coordinate provider + embedder

**Reference**: `docs/vectorization.md`

**Success criteria**:
```python
# Test full pipeline
vectorizer = Vectorizer(embedding_service)
item = reddit_provider.fetch_items()[0]
db_doc = vectorizer.vectorize_item(reddit_provider, item)

# Verify db_doc has:
# - All required fields
# - 1536-dim embedding
```

**Ask Claude Code**:
```
"Help me implement Vectorizer that ties together
provider text extraction and embedding service."
```

---

### Task 8: Sync Orchestration ⏱️ 4-5 hours

**Goal**: Implement delta sync + reconciliation.

**What to build**:
- `routes/sync.py`
- Delta sync logic
- Full reconciliation every 10th sync

**Reference**: `docs/sync-strategy.md`

**Success criteria**:
```bash
# Trigger sync
curl -X POST http://localhost:5000/api/sync/reddit \
  -d '{"user_id": "username"}'

# Verify:
# - Items in Couchbase with embeddings
# - sync_metadata updated
# - Trigger again - should be fast (delta sync)
# - On 10th sync - reconciliation runs
```

**Ask Claude Code**:
```
"I need to implement sync orchestration with delta sync.
Walk me through the algorithm step by step."
```

---

## Week 3: GitHub & RAG

### Task 9: GitHub OAuth & Provider ⏱️ 3-4 hours

**Goal**: Add GitHub as second provider.

**What to build**:
- GitHub OAuth in `routes/auth.py`
- `providers/github_provider.py`

**Reference**: `docs/providers.md` (GitHub section)

**Success criteria**:
```bash
# 1. OAuth flow works
# 2. Sync works
curl -X POST http://localhost:5000/api/sync/github \
  -d '{"user_id": "github_user"}'

# 3. Verify items in Couchbase
```

**Ask Claude Code**:
```
"I'm adding GitHub provider following the Reddit pattern.
Help me identify GitHub-specific differences."
```

---

### Task 10: Content Retrieval Routes ⏱️ 2-3 hours

**Goal**: API to list content with filters.

**What to build**:
- `routes/content.py`
- Endpoints for Reddit, GitHub
- Filtering (subreddit, topic)
- Pagination

**Reference**: `docs/api-endpoints.md`

**Success criteria**:
```bash
# Get Reddit content
curl "http://localhost:5000/api/content/reddit?user_id=alice&subreddit=programming"

# Get GitHub stars
curl "http://localhost:5000/api/content/github?user_id=bob&topic=python"

# Both return paginated, filtered results
```

**Ask Claude Code**:
```
"Guide me through implementing content retrieval endpoints
with filtering and pagination."
```

---

### Task 11: Vector Search Setup ⏱️ 1-2 hours

**Goal**: Create and test vector index.

**Manual steps**:
1. Create `content_vector_index` in Couchbase UI
2. Test vector search queries

**What to build**:
- Vector search method in `couchbase_service.py`

**Reference**: `docs/database.md` (Vector Search section)

**Success criteria**:
```python
# test_vector_search.py
query_embedding = embedder.embed("Python tutorials")
results = db.vector_search(
    "content_vector_index",
    query_embedding,
    top_k=5,
    user_id="test_user"
)

print(f"Found {len(results)} similar items")
# Should return relevant items
```

**Ask Claude Code**:
```
"Help me understand Couchbase vector search syntax.
How do I query with filters?"
```

---

### Task 12: RAG Service ⏱️ 3-4 hours

**Goal**: Implement chat with Claude.

**What to build**:
- `services/rag_service.py`
- `routes/chat.py`
- Context building
- Claude integration

**Reference**: `docs/vectorization.md` (RAG section)

**Success criteria**:
```bash
curl -X POST http://localhost:5000/api/chat \
  -d '{"user_id": "alice", "message": "What Python content did I save?"}'

# Should return:
# - Relevant response from Claude
# - Source citations (Reddit posts, GitHub repos)
```

**Ask Claude Code**:
```
"Guide me through implementing RAG with Claude.
How should I structure the prompt and context?"
```

---

## Week 4: TikTok & Frontend

### Task 13: TikTok Provider ⏱️ 4-5 hours

**Goal**: Handle TikTok file uploads.

**What to build**:
- `providers/tiktok_provider.py`
- `routes/upload.py`
- Metadata fetching

**Reference**: `docs/providers.md` (TikTok section)

**Success criteria**:
```bash
# Upload user_data.json
curl -F "file=@user_data.json" \
     -F "user_id=tiktok_user" \
     http://localhost:5000/api/upload/tiktok

# Verify:
# - Videos parsed
# - Metadata fetched
# - Stored in Couchbase with embeddings
```

**Ask Claude Code**:
```
"I'm implementing TikTok file upload provider.
How should I handle the metadata fetching?"
```

---

### Task 14: Vue App Setup ⏱️ 2 hours

**Goal**: Get Vue running with router.

**What to build**:
- `src/main.js`
- `src/App.vue`
- `src/router/index.js`

**Success criteria**:
```bash
npm run dev
# Opens http://localhost:3000
# Shows basic layout
```

**Ask Claude Code**:
```
"Guide me through setting up Vue 3 with Composition API
and vue-router."
```

---

### Task 15: Home View (OAuth) ⏱️ 3 hours

**Goal**: OAuth connection buttons + status.

**What to build**:
- `views/Home.vue`
- `services/api.js` (Axios client)
- OAuth flow handling

**Success criteria**:
- Click "Connect Reddit" → OAuth flow works
- Click "Connect GitHub" → OAuth flow works
- Show connection status

**Ask Claude Code**:
```
"Help me implement OAuth flow in Vue.
How do I handle the redirects?"
```

---

### Task 16: Reddit & GitHub Views ⏱️ 3-4 hours

**Goal**: List views with filtering.

**What to build**:
- `views/RedditView.vue`
- `views/GitHubView.vue`
- `components/ContentList.vue`
- Filtering dropdowns

**Success criteria**:
- Shows Reddit saved items
- Filter by subreddit works
- Shows GitHub stars
- Filter by topic works

**Ask Claude Code**:
```
"Guide me through building the content list views
with filtering and pagination."
```

---

### Task 17: TikTok & Chat Views ⏱️ 4 hours

**Goal**: TikTok upload + Chat interface.

**What to build**:
- `views/TikTokView.vue` (with file upload)
- `views/ChatView.vue`
- `components/ChatMessage.vue`

**Success criteria**:
- Can upload TikTok data file
- Shows upload progress
- Lists TikTok favorites
- Chat works with source citations

**Ask Claude Code**:
```
"Help me implement file upload in Vue and
create a chat interface with message history."
```

---

## After All Tasks

### Final Integration Testing
- [ ] Full OAuth flow (all providers)
- [ ] Sync all providers
- [ ] Filter views work
- [ ] Chat returns good responses
- [ ] TikTok upload works end-to-end

### Optional Enhancements
- Add loading spinners
- Better error messages
- Sync status indicators
- Export functionality
- Advanced filters (date range, score)

---

**Remember**: Each task should be completed fully before moving to the next.
Test thoroughly, commit working code, then proceed.

---

**Working on a task?** 
1. Read task description
2. Check relevant docs
3. Ask Claude Code for guidance
4. Write code yourself
5. Test thoroughly
6. Mark complete in `claude.md`
