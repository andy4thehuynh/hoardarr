# Hoardarr Development Guide

## Working with Claude Code

### Good Interaction Patterns

**✅ Ask for structure first**
```
"I need to implement RedditProvider.fetch_items(). 
Can you explain the pagination logic, then I'll write it?"
```

**✅ Request code review**
```
"Here's my implementation of delta sync. 
Does this follow the patterns in docs/sync-strategy.md?"
```

**✅ Debug together**
```
"My vector search returns no results. Here's my query code.
What could be wrong?"
```

**✅ Clarify concepts**
```
"Why do we use sync_count % 10 for reconciliation?
What problem does this solve?"
```

---

### Avoid These Patterns

**❌ Don't ask for complete code**
```
"Write the entire RedditProvider class for me"
```

**❌ Don't skip learning**
```
"Just give me the solution without explaining"
```

**❌ Don't ask vague questions**
```
"My code doesn't work, fix it"
```

Better: Show code, describe error, explain what you've tried

---

## Task Workflow

### Before Starting a Task

1. **Read task description**: `docs/implementation-tasks.md`
2. **Review relevant docs**:
   - Providers? → `docs/providers.md`
   - Database? → `docs/database.md`
   - Sync? → `docs/sync-strategy.md`
3. **Ask Claude Code**: "Guide me through [task]"

---

### During the Task

1. **Write code yourself**
2. **Test incrementally** (don't write everything at once)
3. **Ask for review**: Show Claude Code your code
4. **Iterate based on feedback**
5. **Add logging** for debugging

---

### After Completing Task

1. **Test thoroughly** (see test criteria in task)
2. **Write quick test script** if needed
3. **Update `claude.md`**: Mark task complete
4. **Commit to git**: Clear commit message
5. **Move to next task**

---

## Testing Strategies

### Unit Testing (Optional but Recommended)

```python
# test_reddit_provider.py
import unittest
from providers.reddit_provider import RedditProvider

class TestRedditProvider(unittest.TestCase):
    def setUp(self):
        self.provider = RedditProvider()
    
    def test_get_vectorization_text_post(self):
        item = {
            'kind': 't3',
            'data': {
                'title': 'Test Post',
                'selftext': 'Post body'
            }
        }
        
        text = self.provider.get_vectorization_text(item)
        self.assertIn('Test Post', text)
        self.assertIn('Post body', text)
    
    def test_get_vectorization_text_comment(self):
        item = {
            'kind': 't1',
            'data': {
                'body': 'Comment text'
            }
        }
        
        text = self.provider.get_vectorization_text(item)
        self.assertEqual(text, 'Comment text')
```

---

### Manual Testing

**After OAuth implementation**:
```bash
# Test Reddit OAuth
1. Visit http://localhost:5000/api/auth/reddit
2. Authorize app on Reddit
3. Should redirect back with success
4. Check Couchbase for sync_metadata document
```

**After sync implementation**:
```bash
# Test Reddit sync
curl -X POST http://localhost:5000/api/sync/reddit \
  -H "Content-Type: application/json" \
  -d '{"user_id": "your_reddit_username"}'

# Check response
# Check Couchbase for reddit_content documents
# Verify embeddings are 1536 dimensions
```

**After RAG implementation**:
```bash
# Test chat
curl -X POST http://localhost:5000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"user_id": "username", "message": "What Python content did I save?"}'

# Check response quality
# Verify sources are included
```

---

## Debugging Checklist

### "OAuth isn't working"
- [ ] Check `.env` has correct client ID/secret
- [ ] Verify redirect URI matches OAuth app settings
- [ ] Check state token is being set and validated
- [ ] Look at browser network tab for errors

### "Sync returns no items"
- [ ] Check OAuth token not expired
- [ ] Verify `last_sync_at` timestamp is correct
- [ ] Test API call with curl/Postman directly
- [ ] Check rate limits not exceeded
- [ ] Look for errors in Flask logs

### "Vector search returns nothing"
- [ ] Verify vector index exists in Couchbase
- [ ] Check embeddings are actually stored
- [ ] Verify embedding dimensions (should be 1536)
- [ ] Test query without filters first
- [ ] Check user_id filter is correct

### "RAG responses are poor"
- [ ] Verify vector search returns relevant items
- [ ] Check context being sent to Claude
- [ ] Test Claude prompt in Claude.ai first
- [ ] Verify enough content in database
- [ ] Check if embeddings are similar to query

---

## Common Gotchas

### Reddit
**Issue**: 429 rate limit errors
**Fix**: Add User-Agent header
```python
headers = {'User-Agent': 'Hoardarr/1.0'}
```

**Issue**: Pagination doesn't work
**Fix**: `after` is from response, not created_utc
```python
after = data['data']['after']  # Use this
# NOT: after = item['created_utc']
```

---

### GitHub
**Issue**: No `starred_at` in response
**Fix**: Use correct Accept header
```python
headers = {'Accept': 'application/vnd.github.star+json'}
```

**Issue**: Rate limit hit quickly
**Fix**: GitHub limits are per hour, not minute. Check headers:
```python
remaining = response.headers.get('X-RateLimit-Remaining')
reset_time = response.headers.get('X-RateLimit-Reset')
```

---

### TikTok
**Issue**: Can't get video metadata
**Fix**: Video may be deleted or private
```python
try:
    metadata = fetch_metadata(url)
except VideoNotFound:
    # Skip this video
    continue
```

**Issue**: Upload processing is slow
**Fix**: Expected - fetching 100+ videos takes time
Consider showing progress bar

---

### Couchbase
**Issue**: Vector search fails
**Fix**: Index might not be created yet
1. Go to Couchbase UI
2. Search → Indexes
3. Verify `content_vector_index` exists

**Issue**: Connection timeout
**Fix**: Check connection string and credentials
```python
# Should be: couchbases://cluster-name.cloud.couchbase.com
# NOT: couchbase://... (missing 's')
```

---

### OpenAI
**Issue**: Embeddings fail
**Fix**: Check API key is valid
```bash
export OPENAI_API_KEY=sk-...
```

**Issue**: Token limit exceeded
**Fix**: Truncate text before embedding
```python
text = text[:32000]  # ~8k tokens
```

---

## Logging Best Practices

### What to Log
```python
import logging

# Setup
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Good logs:
logger.info(f"Starting Reddit sync for user: {user_id}")
logger.info(f"Fetched {len(items)} items from Reddit")
logger.error(f"Failed to fetch from Reddit: {error}", exc_info=True)
```

### What NOT to Log
```python
# DON'T log sensitive data:
logger.info(f"Access token: {access_token}")  # ❌
logger.info(f"User content: {item['body']}")  # ❌ (PII)

# DO log safely:
logger.info(f"Access token exists: {bool(access_token)}")  # ✅
logger.info(f"Processing item ID: {item['id']}")  # ✅
```

---

## Git Workflow

### Commit Messages
```bash
# Good commits:
git commit -m "feat: implement Reddit OAuth flow"
git commit -m "fix: handle pagination edge case in GitHub provider"
git commit -m "test: add vector search integration test"

# Not so good:
git commit -m "stuff"
git commit -m "fixed bug"
git commit -m "updates"
```

### Branch Strategy (If Using Branches)
```bash
# Create feature branch
git checkout -b feature/reddit-provider

# Work on feature...
git add providers/reddit_provider.py
git commit -m "feat: implement RedditProvider base methods"

# Merge when done
git checkout main
git merge feature/reddit-provider
```

---

## Environment Setup Reminders

### Backend
```bash
cd backend
source venv/bin/activate  # or venv\Scripts\activate on Windows

# Check .env loaded
python -c "from dotenv import load_dotenv; load_dotenv(); import os; print(os.getenv('FLASK_SECRET_KEY'))"

# Run server
python app.py
```

### Frontend
```bash
cd frontend
npm run dev

# Opens http://localhost:3000
```

---

## Daily Checklist

**Starting Work**:
- [ ] Activate backend venv
- [ ] Check which task you're on
- [ ] Read relevant docs
- [ ] Start Flask and frontend servers

**During Work**:
- [ ] Write code incrementally
- [ ] Test after each small change
- [ ] Ask Claude Code for reviews
- [ ] Add logging for debugging

**Ending Work**:
- [ ] Test current implementation
- [ ] Commit working code
- [ ] Update task status in `claude.md`
- [ ] Note any blockers for tomorrow

---

## Resources

### Documentation
- OpenAI API: https://platform.openai.com/docs
- Anthropic API: https://docs.anthropic.com
- Couchbase SDK: https://docs.couchbase.com/python-sdk
- Reddit API: https://www.reddit.com/dev/api
- GitHub API: https://docs.github.com/en/rest

### Testing Tools
- Postman/Insomnia: Test API endpoints
- Couchbase Web UI: View data directly
- Chrome DevTools: Debug frontend
- Python debugger: `import pdb; pdb.set_trace()`
