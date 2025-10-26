# Hoardarr - Personal Content RAG System

## What This Is
A web app that aggregates saved content from Reddit, GitHub, and TikTok, vectorizes it, and provides semantic search + RAG chat. Built for learning - user writes all code with AI guidance.

---

## Quick Reference

**Current Status**: [Update as you progress]
- [ ] Backend foundation complete
- [ ] Reddit provider working
- [ ] GitHub provider working
- [ ] TikTok provider working
- [ ] RAG chat functional
- [ ] Frontend views complete

**Working on**: [Current task from docs/implementation-tasks.md]

**Tech Stack**: Flask, Vue 3, Couchbase Capella, OpenAI embeddings, Claude 3.5 Haiku

---

## Core Concepts

### The Provider Pattern
All content sources inherit from `BaseProvider` (API-based) or `FileUploadProvider` (file-based). This makes adding new sources trivial.
- **Reddit & GitHub**: OAuth + API polling
- **TikTok**: Manual file upload + metadata fetching

See: `docs/providers.md`

### Sync Strategy
- **Delta sync** (default): Only fetch new items
- **Full reconciliation** (every 10th sync): Detect deleted items
- **TikTok**: Full replacement on each upload

See: `docs/sync-strategy.md`

### Data Flow
1. Content fetched → 2. Text extracted → 3. Embedded (1536-dim) → 4. Stored in Couchbase → 5. Vector search retrieves → 6. Claude generates response

See: `docs/architecture.md`

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Flask over FastAPI | Sync is batch-based, not real-time. Simpler for learning. |
| Provider abstraction | Easy to add Twitter, YouTube, etc. later |
| Delta + reconciliation | Efficient syncing without missing deleted items |
| File upload for TikTok | No API for favorites; acceptable for personal tool |
| Demo-quality UI | Prove functionality first, polish later |

See: `docs/architecture.md` for full rationale

---

## Documentation Map

📁 **docs/**
- `architecture.md` - System design, patterns, component relationships
- `providers.md` - How to implement providers (Reddit, GitHub, TikTok examples)
- `database.md` - Couchbase schema, collections, vector index
- `api-endpoints.md` - All routes with request/response formats
- `sync-strategy.md` - Delta sync, reconciliation, TikTok uploads
- `vectorization.md` - Embedding service, RAG flow, text extraction
- `development-guide.md` - How to work with Claude Code, testing, debugging
- `implementation-tasks.md` - 17-step build plan (4 weeks)

📄 **Project root files:**
- `.env.example` - All required environment variables
- `README.md` - User-facing project description

---

## When Working on a Task

1. **Check current task**: `docs/implementation-tasks.md`
2. **Understand the component**: Read relevant doc (e.g., `providers.md` for provider tasks)
3. **Ask Claude Code**: "Guide me through [task]. I'll write code, you review."
4. **Test thoroughly**: Every task has test criteria
5. **Update status**: Mark task complete in this file

---

## Essential Fields Quick Reference

**Reddit** (vectorize `title` or `body`):
- id, name, kind, title, body, subreddit, permalink, created_utc

**GitHub** (vectorize `name: description`):
- id, full_name, name, description, topics, html_url, starred_at

**TikTok** (vectorize `caption + description + hashtags`):
- video_id, link, caption, video_description, hashtags, author_username, favorited_at

Full schemas: `docs/database.md`

---

## Common Commands

```bash
# Backend
cd backend
source venv/bin/activate
python app.py

# Frontend  
cd frontend
npm run dev

# Test Couchbase connection
python test_couchbase.py

# Manual sync trigger
curl -X POST http://localhost:5000/api/sync/reddit -d '{"user_id":"username"}'
```

---

## Important Notes

- **OAuth tokens**: Encrypted before storing in Couchbase
- **Rate limits**: Reddit 60/min, GitHub 5000/hr, TikTok metadata service varies
- **Vector dimensions**: Always 1536 (text-embedding-3-small)
- **Sync count**: Stored in `sync_metadata`, triggers reconciliation at 10
- **TikTok metadata**: Fetched after upload using unofficial API/scraping

---

## Getting Help from Claude Code

**Good questions:**
- "Guide me through implementing RedditProvider.fetch_items()"
- "Review this code I wrote for delta sync"
- "Why isn't my vector search returning results?"
- "How should I structure the TikTok upload endpoint?"

**Avoid:**
- "Write the entire provider for me"
- "Build the backend"

See: `docs/development-guide.md` for detailed interaction patterns

---

## Critical Gotchas

⚠️ **Reddit API**: Requires User-Agent header or requests fail
⚠️ **GitHub API**: Need `Accept: application/vnd.github.star+json` for `starred_at`
⚠️ **TikTok Export**: Only contains video URLs, not metadata (must fetch separately)
⚠️ **Couchbase Vector Index**: Must be created manually in UI before querying
⚠️ **OpenAI Embeddings**: Truncate text to ~8000 tokens (32k chars)

Full list: `docs/development-guide.md`

---

## Project Philosophy

- **Learning first**: Write code yourself, get guidance
- **Small steps**: Complete one task before moving on
- **Test everything**: Each task has clear success criteria
- **Extensible design**: Easy to add new providers
- **Functional over fancy**: Prove it works, polish later

---

**Next Steps**: Start with Task 1 in `docs/implementation-tasks.md`
