# Hoardarr Vectorization & RAG

## Embedding Service

**Model**: OpenAI text-embedding-3-small
**Dimensions**: 1536
**Cost**: $0.02 per 1M tokens

### Implementation
```python
import openai

class EmbeddingService:
    def __init__(self, api_key):
        self.client = openai.OpenAI(api_key=api_key)
    
    def embed(self, text):
        """Get embedding for text"""
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=text,
            encoding_format="float"
        )
        return response.data[0].embedding
    
    def embed_batch(self, texts):
        """Batch embed multiple texts"""
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=texts,
            encoding_format="float"
        )
        return [item.embedding for item in response.data]
```

---

## Text Preprocessing

**Goals**:
- Remove excessive whitespace
- Truncate to model limits
- Clean markup/formatting

```python
class TextPreprocessor:
    def preprocess(self, text):
        # Remove excessive whitespace
        text = ' '.join(text.split())
        
        # Truncate to ~8000 tokens (rough: 32k chars)
        if len(text) > 32000:
            text = text[:32000]
        
        # Remove markdown formatting (optional)
        text = self._clean_markdown(text)
        
        return text
    
    def _clean_markdown(self, text):
        # Remove **bold**, *italic*, etc.
        import re
        text = re.sub(r'\*\*(.+?)\*\*', r'\1', text)
        text = re.sub(r'\*(.+?)\*', r'\1', text)
        return text
```

---

## Provider-Specific Text Extraction

### Reddit
```python
def get_vectorization_text(self, item):
    kind = item['kind']
    data = item['data']
    
    if kind == 't3':  # Post
        if data.get('selftext'):
            return f"{data['title']}. {data['selftext']}"
        return data['title']
    
    elif kind == 't1':  # Comment
        return data['body']
```

### GitHub
```python
def get_vectorization_text(self, item):
    repo = item['repo']
    name = repo['name']
    description = repo.get('description') or ''
    return f"{name}: {description}"
```

### TikTok
```python
def get_vectorization_text(self, item):
    caption = item.get('caption', '')
    description = item.get('video_description', '')
    hashtags = ' '.join(item.get('hashtags', []))
    return f"{caption}. {description}. Tags: {hashtags}"
```

---

## Vectorization Pipeline

```python
class Vectorizer:
    def __init__(self, embedding_service, preprocessor):
        self.embedding_service = embedding_service
        self.preprocessor = preprocessor
    
    def vectorize_item(self, provider, item):
        # 1. Extract text
        text = provider.get_vectorization_text(item)
        
        # 2. Preprocess
        text = self.preprocessor.preprocess(text)
        
        # 3. Get embedding
        embedding = self.embedding_service.embed(text)
        
        # 4. Transform to DB model
        db_item = provider.transform_to_db_model(item, embedding)
        
        return db_item
```

---

## RAG Service

### Vector Search
```python
class RAGService:
    def __init__(self, couchbase_service, embedding_service, claude_client):
        self.db = couchbase_service
        self.embedder = embedding_service
        self.claude = claude_client
    
    def search(self, user_id, query, top_k=5):
        # 1. Embed query
        query_vector = self.embedder.embed(query)
        
        # 2. Vector search
        results = self.db.vector_search(
            index="content_vector_index",
            vector=query_vector,
            top_k=top_k,
            filter=f'user_id:"{user_id}"'
        )
        
        return results
```

---

### Context Building
```python
def build_context(self, results):
    """Format search results into context"""
    context_parts = []
    
    for idx, item in enumerate(results, 1):
        if item['_type'] == 'reddit_saved':
            text = f"Reddit Post (r/{item['subreddit']}): {item['title']}"
            if item.get('selftext'):
                text += f"\n{item['selftext'][:500]}"
            elif item.get('body'):
                text += f"\n{item['body'][:500]}"
        
        elif item['_type'] == 'github_star':
            text = f"GitHub Repository: {item['full_name']}\n{item['description']}"
        
        elif item['_type'] == 'tiktok_favorite':
            text = f"TikTok Video (@{item['author_username']}): {item['caption']}"
        
        context_parts.append(f"[{idx}] {text}")
    
    return "\n\n".join(context_parts)
```

---

### Chat with Claude
```python
def chat(self, user_id, message, top_k=5):
    # 1. Search for relevant content
    results = self.search(user_id, message, top_k)
    
    # 2. Build context
    context = self.build_context(results)
    
    # 3. Create prompt
    system_prompt = """You are a helpful assistant that answers questions 
    based on the user's saved content from Reddit, GitHub, and TikTok. 
    Use the provided context to answer questions. If the context doesn't 
    contain relevant information, say so."""
    
    user_prompt = f"""Context from saved content:
{context}

Question: {message}

Please answer based on the context above."""
    
    # 4. Call Claude
    response = self.claude.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=1024,
        system=system_prompt,
        messages=[{
            "role": "user",
            "content": user_prompt
        }]
    )
    
    # 5. Extract sources
    sources = self._extract_sources(results)
    
    return {
        "response": response.content[0].text,
        "sources": sources
    }
```

---

### Source Extraction
```python
def _extract_sources(self, results):
    """Format sources for citation"""
    sources = []
    
    for item in results:
        if item['_type'] == 'reddit_saved':
            sources.append({
                "type": "reddit",
                "title": item['title'],
                "url": f"https://reddit.com{item['permalink']}",
                "subreddit": item['subreddit']
            })
        
        elif item['_type'] == 'github_star':
            sources.append({
                "type": "github",
                "name": item['full_name'],
                "url": item['html_url'],
                "description": item['description'][:100]
            })
        
        elif item['_type'] == 'tiktok_favorite':
            sources.append({
                "type": "tiktok",
                "caption": item['caption'],
                "url": item['link'],
                "author": item['author_username']
            })
    
    return sources
```

---

## Optimization Tips

### Batch Embedding
```python
# Instead of embedding one at a time:
for item in items:
    embedding = embedder.embed(text)

# Batch them:
texts = [provider.get_vectorization_text(item) for item in items]
embeddings = embedder.embed_batch(texts)
for item, embedding in zip(items, embeddings):
    # store with embedding
```

**Benefit**: Faster, fewer API calls

---

### Caching
```python
import hashlib

class CachedEmbeddingService:
    def __init__(self, embedding_service):
        self.service = embedding_service
        self.cache = {}
    
    def embed(self, text):
        # Hash text as cache key
        key = hashlib.md5(text.encode()).hexdigest()
        
        if key in self.cache:
            return self.cache[key]
        
        embedding = self.service.embed(text)
        self.cache[key] = embedding
        return embedding
```

**Benefit**: Avoid re-embedding duplicate content

---

## Testing Vector Quality

### Manual Test
```python
# Test similar items cluster together
items = [
    "Python programming tutorial",
    "Learn Python basics",
    "JavaScript framework comparison"
]

embeddings = [embedder.embed(text) for text in items]

# Calculate cosine similarity
from sklearn.metrics.pairwise import cosine_similarity
similarities = cosine_similarity(embeddings)

# Python items should be more similar to each other
# than to JavaScript item
```

---

### Search Quality Test
```python
# Test queries return relevant results
test_cases = [
    {
        "query": "machine learning",
        "expected_contains": ["neural", "ml", "ai", "model"]
    },
    {
        "query": "web development",
        "expected_contains": ["html", "css", "javascript", "react"]
    }
]

for test in test_cases:
    results = rag_service.search(user_id, test["query"])
    
    # Check if results contain expected terms
    all_text = " ".join([r.get('title', '') + r.get('description', '') 
                         for r in results])
    
    matches = sum(1 for term in test["expected_contains"] 
                  if term.lower() in all_text.lower())
    
    print(f"Query '{test['query']}': {matches}/{len(test['expected_contains'])} terms matched")
```
