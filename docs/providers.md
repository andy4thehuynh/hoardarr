# Hoardarr Providers Guide

## Provider Types

### BaseProvider (Abstract)
All providers inherit from this. Defines the contract.

```python
from abc import ABC, abstractmethod

class BaseProvider(ABC):
    @abstractmethod
    def get_vectorization_text(self, item):
        """Extract text to embed from item"""
        pass
    
    @abstractmethod
    def transform_to_db_model(self, item, embedding):
        """Transform API item to Couchbase document"""
        pass
    
    @abstractmethod
    def get_item_id(self, item):
        """Extract unique identifier"""
        pass
```

---

### APIProvider (for OAuth sources)
Extends BaseProvider for API-based sources.

```python
class APIProvider(BaseProvider):
    @abstractmethod
    def fetch_items(self, access_token, since=None):
        """Fetch items from API with pagination"""
        pass
```

**Used by**: Reddit, GitHub

---

### FileUploadProvider (for file-based sources)
Extends BaseProvider for file uploads.

```python
class FileUploadProvider(BaseProvider):
    @abstractmethod
    def parse_upload_file(self, file_data):
        """Parse uploaded file (JSON, CSV, etc.)"""
        pass
    
    @abstractmethod
    def extract_items(self, parsed_data):
        """Extract items from parsed file"""
        pass
```

**Used by**: TikTok

---

## Reddit Provider

### API Details
- **Base URL**: `https://oauth.reddit.com`
- **Endpoint**: `GET /user/{username}/saved`
- **Auth**: OAuth 2.0, scope `history`
- **Pagination**: `after` parameter (opaque string)
- **Rate Limit**: 60 requests/minute

### Required Headers
```python
headers = {
    'Authorization': f'Bearer {access_token}',
    'User-Agent': 'Hoardarr/1.0'  # REQUIRED!
}
```

**Critical**: Reddit API fails without User-Agent.

---

### Response Structure
```json
{
  "data": {
    "children": [
      {
        "kind": "t3",  // Post
        "data": {
          "id": "abc123",
          "name": "t3_abc123",
          "title": "Cool post title",
          "selftext": "Post body text",
          "subreddit": "programming",
          "permalink": "/r/programming/comments/abc123/...",
          "created_utc": 1698765432.0,
          "url": "https://example.com/article",
          "score": 234
        }
      },
      {
        "kind": "t1",  // Comment
        "data": {
          "id": "xyz789",
          "name": "t1_xyz789",
          "body": "Comment text here",
          "subreddit": "AskReddit",
          "permalink": "/r/AskReddit/comments/.../xyz789",
          "created_utc": 1698765000.0
        }
      }
    ],
    "after": "t3_abc123"  // Pagination token
  }
}
```

---

### Implementation Pattern

```python
class RedditProvider(APIProvider):
    def fetch_items(self, access_token, since=None):
        """Fetch saved items with pagination"""
        items = []
        after = None
        
        while True:
            # Build request
            params = {'limit': 100}
            if after:
                params['after'] = after
            
            # Make request
            response = requests.get(
                f'{BASE_URL}/user/me/saved',
                headers=self._get_headers(access_token),
                params=params
            )
            
            data = response.json()
            children = data['data']['children']
            
            # Process items
            for child in children:
                item_data = child['data']
                
                # Check if we've reached old items
                if since and item_data['created_utc'] <= since:
                    return items
                
                items.append(child)
            
            # Check pagination
            after = data['data']['after']
            if not after:
                break
        
        return items
    
    def get_vectorization_text(self, item):
        """Extract text based on kind"""
        kind = item['kind']
        data = item['data']
        
        if kind == 't3':  # Post
            if data.get('selftext'):
                return f"{data['title']}. {data['selftext']}"
            return data['title']
        
        elif kind == 't1':  # Comment
            return data['body']
        
        return ""
    
    def transform_to_db_model(self, item, embedding):
        """Transform to Couchbase document"""
        kind = item['kind']
        data = item['data']
        
        return {
            '_type': 'reddit_saved',
            'id': data['id'],
            'name': data['name'],
            'kind': kind,
            'title': data.get('title', ''),
            'selftext': data.get('selftext', ''),
            'body': data.get('body', ''),
            'subreddit': data['subreddit'],
            'permalink': data['permalink'],
            'created_utc': data['created_utc'],
            'url': data.get('url', ''),
            'score': data.get('score', 0),
            'embedding': embedding,
            'synced_at': datetime.utcnow().isoformat()
        }
    
    def get_item_id(self, item):
        """Get unique ID"""
        return item['data']['id']
```

---

### Delta Sync Logic
```python
# Load last sync time
last_sync_at = get_sync_metadata(user_id, 'reddit')['last_sync_at']

# Fetch only new items
items = reddit_provider.fetch_items(
    access_token,
    since=last_sync_at  # Stop at this timestamp
)

# Update metadata
update_sync_metadata(user_id, 'reddit', {
    'last_sync_at': datetime.utcnow().isoformat(),
    'sync_count': sync_count + 1
})
```

---

## GitHub Provider

### API Details
- **Base URL**: `https://api.github.com`
- **Endpoint**: `GET /user/starred`
- **Auth**: OAuth 2.0, scope `read:user`
- **Pagination**: `page` parameter (starts at 1)
- **Rate Limit**: 5,000 requests/hour (authenticated)

### Required Headers
```python
headers = {
    'Authorization': f'Bearer {access_token}',
    'Accept': 'application/vnd.github.star+json',  # Get starred_at!
    'X-GitHub-Api-Version': '2022-11-28'
}
```

**Critical**: Must use `application/vnd.github.star+json` to get `starred_at` timestamp.

---

### Response Structure
```json
{
  "starred_at": "2025-01-15T10:30:00Z",
  "repo": {
    "id": 1296269,
    "full_name": "octocat/Hello-World",
    "name": "Hello-World",
    "description": "My first repository",
    "html_url": "https://github.com/octocat/Hello-World",
    "topics": ["api", "octocat", "github"],
    "language": "JavaScript",
    "stargazers_count": 80
  }
}
```

---

### Implementation Pattern

```python
class GitHubProvider(APIProvider):
    def fetch_items(self, access_token, since=None):
        """Fetch starred repos with pagination"""
        items = []
        page = 1
        
        while True:
            params = {
                'per_page': 100,
                'page': page,
                'sort': 'created',
                'direction': 'desc'  # Newest first
            }
            
            response = requests.get(
                f'{BASE_URL}/user/starred',
                headers=self._get_headers(access_token),
                params=params
            )
            
            starred_repos = response.json()
            
            if not starred_repos:
                break
            
            for starred_item in starred_repos:
                # Check if we've reached old stars
                starred_at = starred_item['starred_at']
                if since and starred_at <= since:
                    return items
                
                items.append(starred_item)
            
            page += 1
        
        return items
    
    def get_vectorization_text(self, item):
        """Combine name and description"""
        repo = item['repo']
        name = repo['name']
        description = repo.get('description') or ''
        
        return f"{name}: {description}"
    
    def transform_to_db_model(self, item, embedding):
        """Transform to Couchbase document"""
        repo = item['repo']
        
        return {
            '_type': 'github_star',
            'id': repo['id'],
            'full_name': repo['full_name'],
            'name': repo['name'],
            'description': repo.get('description', ''),
            'topics': repo.get('topics', []),
            'html_url': repo['html_url'],
            'language': repo.get('language', ''),
            'stargazers_count': repo['stargazers_count'],
            'starred_at': item['starred_at'],
            'embedding': embedding,
            'synced_at': datetime.utcnow().isoformat()
        }
    
    def get_item_id(self, item):
        """Get unique ID"""
        return str(item['repo']['id'])
```

---

## TikTok Provider

### Data Source
**No API available** - uses manual data export.

User exports data from TikTok:
1. Settings → Privacy → Download your data
2. Select JSON format
3. Wait 2-7 days for file
4. Upload `user_data.json` to app

---

### Export Structure
```json
{
  "Activity": {
    "Favorite Videos": {
      "FavoriteVideoList": [
        {
          "Date": "2024-10-15 14:23:45",
          "Link": "https://www.tiktok.com/@username/video/1234567890"
        }
      ]
    }
  }
}
```

**Problem**: Only contains URLs, not video metadata!

**Solution**: Fetch metadata from each URL after upload.

---

### Implementation Pattern

```python
class TikTokProvider(FileUploadProvider):
    def parse_upload_file(self, file_data):
        """Parse JSON export"""
        try:
            data = json.loads(file_data)
            return data
        except json.JSONDecodeError:
            raise ValueError("Invalid JSON file")
    
    def extract_items(self, parsed_data):
        """Extract favorite videos"""
        try:
            favorites = parsed_data['Activity']['Favorite Videos']['FavoriteVideoList']
            return favorites
        except KeyError:
            raise ValueError("No favorites found in export")
    
    def fetch_metadata(self, video_url):
        """Fetch video metadata from URL"""
        # Option 1: TikTok embed API (limited data)
        # Option 2: Unofficial API like TikAPI (paid)
        # Option 3: Web scraping with Playwright
        
        # Example with embed API:
        video_id = self._extract_video_id(video_url)
        embed_url = f'https://www.tiktok.com/oembed?url={video_url}'
        
        response = requests.get(embed_url)
        data = response.json()
        
        return {
            'video_id': video_id,
            'caption': data.get('title', ''),
            'author_username': data.get('author_name', ''),
            'link': video_url
        }
    
    def get_vectorization_text(self, item):
        """Combine caption, description, hashtags"""
        caption = item.get('caption', '')
        description = item.get('video_description', '')
        hashtags = ' '.join(item.get('hashtags', []))
        
        return f"{caption}. {description}. Tags: {hashtags}"
    
    def transform_to_db_model(self, item, embedding):
        """Transform to Couchbase document"""
        return {
            '_type': 'tiktok_favorite',
            'video_id': item['video_id'],
            'link': item['link'],
            'favorited_at': item['favorited_at'],
            'caption': item.get('caption', ''),
            'video_description': item.get('video_description', ''),
            'author_username': item.get('author_username', ''),
            'hashtags': item.get('hashtags', []),
            'like_count': item.get('like_count', 0),
            'view_count': item.get('view_count', 0),
            'embedding': embedding,
            'uploaded_at': datetime.utcnow().isoformat()
        }
    
    def get_item_id(self, item):
        """Get video ID from URL"""
        return item['video_id']
```

---

### Upload Flow
```python
# In routes/upload.py
@app.route('/api/upload/tiktok', methods=['POST'])
def upload_tiktok():
    file = request.files['file']
    user_id = request.form['user_id']
    
    # Parse file
    file_data = file.read()
    parsed_data = tiktok_provider.parse_upload_file(file_data)
    
    # Extract video URLs
    items = tiktok_provider.extract_items(parsed_data)
    
    # Process each video (async recommended)
    for item in items:
        # Fetch metadata
        metadata = tiktok_provider.fetch_metadata(item['Link'])
        
        # Add favorited date from export
        metadata['favorited_at'] = item['Date']
        
        # Vectorize
        text = tiktok_provider.get_vectorization_text(metadata)
        embedding = embedding_service.embed(text)
        
        # Store
        doc = tiktok_provider.transform_to_db_model(metadata, embedding)
        couchbase_service.store('tiktok_favorites', doc)
    
    return {'status': 'completed', 'processed': len(items)}
```

---

## Adding a New Provider

### Example: Twitter Bookmarks

**Step 1**: Determine type
- Has API? → Extend `APIProvider`
- File export only? → Extend `FileUploadProvider`

Twitter has API, so:

```python
class TwitterProvider(APIProvider):
    pass
```

---

**Step 2**: Implement required methods

```python
def fetch_items(self, access_token, since=None):
    """Fetch bookmarked tweets"""
    # Call Twitter API /2/users/:id/bookmarks
    # Paginate with pagination_token
    # Return list of tweets
    pass

def get_vectorization_text(self, item):
    """Extract tweet text"""
    return item['text']

def transform_to_db_model(self, item, embedding):
    """Transform to document"""
    return {
        '_type': 'twitter_bookmark',
        'id': item['id'],
        'text': item['text'],
        'author': item['author_id'],
        'created_at': item['created_at'],
        'embedding': embedding,
        'synced_at': datetime.utcnow().isoformat()
    }

def get_item_id(self, item):
    return item['id']
```

---

**Step 3**: Add OAuth flow
In `routes/auth.py`, add endpoints similar to Reddit/GitHub.

---

**Step 4**: Add sync route
In `routes/sync.py`, add sync endpoint.

---

**Step 5**: Create Couchbase collection
Create `twitter_bookmarks` collection in Couchbase UI.

---

**Step 6**: Add frontend view
Create `TwitterView.vue` similar to RedditView.

---

**Done!** New provider integrated following the same pattern.

---

## Common Gotchas

### Reddit
- **Forget User-Agent**: 429 errors
- **Wrong kind check**: Mixing posts (t3) and comments (t1)
- **Pagination token**: `after` is opaque, not a timestamp

### GitHub
- **Missing Accept header**: No `starred_at` field
- **Rate limiting**: 5k/hour, track in headers
- **Topics can be empty**: Always check `topics` exists

### TikTok
- **URLs only in export**: Must fetch metadata separately
- **Video may be deleted**: Handle 404s gracefully
- **Metadata services**: Unofficial APIs may have limits

---

**Next**: See `sync-strategy.md` for sync implementation details
