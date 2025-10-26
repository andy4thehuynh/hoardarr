# Hoardarr Sync Strategy

## Delta Sync (Default)

**Goal**: Only fetch new items since last sync.

**Algorithm**:
```
1. Load sync_metadata for user+provider
2. Get last_sync_at timestamp
3. Fetch items from provider API with since=last_sync_at
4. For each item:
   - If created_at > last_sync_at: process and store
   - If created_at <= last_sync_at: stop (reached old content)
5. Update last_sync_at to now
6. Increment sync_count
```

**Benefits**:
- Fast (only fetch new data)
- Lower API usage
- Reduced processing time

**Limitation**:
- Doesn't detect deleted items (user unsaved/unstarred)

---

## Full Reconciliation (Every 10th Sync)

**Goal**: Detect and remove items user deleted.

**Algorithm**:
```
1. Check if sync_count % 10 == 0
2. If yes, trigger full reconciliation:
   a. Fetch ALL item IDs from provider (lightweight query)
   b. Fetch ALL item IDs from Couchbase for this user
   c. Identify items in DB but not in API → user deleted them
   d. Delete those items from Couchbase
   e. Reset sync_count to 0
   f. Update last_full_reconciliation timestamp
```

**Benefits**:
- Keeps DB in sync with provider
- Removes stale data

**Cost**:
- Slower (fetches all IDs)
- More API calls

---

## Implementation Example (Reddit)

### Delta Sync
```python
def sync_reddit_delta(user_id, access_token):
    # Load metadata
    metadata = get_sync_metadata(user_id, 'reddit')
    last_sync_at = metadata.get('last_sync_at', 0)
    sync_count = metadata.get('sync_count', 0)
    
    # Fetch new items
    provider = RedditProvider()
    items = provider.fetch_items(access_token, since=last_sync_at)
    
    # Process each item
    processed = 0
    for item in items:
        # Vectorize
        text = provider.get_vectorization_text(item)
        embedding = embedding_service.embed(text)
        
        # Store
        doc = provider.transform_to_db_model(item, embedding)
        doc['user_id'] = user_id
        couchbase_service.store('reddit_content', doc)
        
        processed += 1
    
    # Update metadata
    update_sync_metadata(user_id, 'reddit', {
        'last_sync_at': datetime.utcnow().isoformat(),
        'sync_count': sync_count + 1
    })
    
    return {'items_added': processed}
```

---

### Full Reconciliation
```python
def sync_reddit_full_reconciliation(user_id, access_token):
    provider = RedditProvider()
    
    # Get all IDs from Reddit
    reddit_ids = set()
    items = provider.fetch_items(access_token)  # All items
    for item in items:
        reddit_ids.add(provider.get_item_id(item))
    
    # Get all IDs from Couchbase
    query = """
    SELECT id FROM hoardarr._default.reddit_content
    WHERE user_id = $1
    """
    result = cluster.query(query, user_id)
    db_ids = {row['id'] for row in result}
    
    # Find items to delete
    to_delete = db_ids - reddit_ids
    
    # Delete from Couchbase
    for item_id in to_delete:
        delete_query = """
        DELETE FROM hoardarr._default.reddit_content
        WHERE user_id = $1 AND id = $2
        """
        cluster.query(delete_query, user_id, item_id)
    
    # Update metadata
    update_sync_metadata(user_id, 'reddit', {
        'last_full_reconciliation': datetime.utcnow().isoformat(),
        'sync_count': 0
    })
    
    return {'items_removed': len(to_delete)}
```

---

## TikTok Upload Strategy

**Different pattern**: No API, so no automatic sync.

**Algorithm**:
```
1. User uploads user_data.json
2. Parse file to get list of video URLs
3. For each URL:
   a. Extract video_id from URL
   b. Check if video_id exists in tiktok_favorites
   c. If exists: skip (already have it)
   d. If new: fetch metadata and store
4. Get all video_ids from upload
5. Get all video_ids from Couchbase for this user
6. Delete items in DB but not in upload (user unfavorited)
```

**This is like "full reconciliation" every upload.**

---

### TikTok Upload Implementation
```python
def process_tiktok_upload(user_id, file_data):
    provider = TikTokProvider()
    
    # Parse file
    parsed = provider.parse_upload_file(file_data)
    items = provider.extract_items(parsed)
    
    # Track video IDs in upload
    upload_video_ids = set()
    
    # Process each video
    new_count = 0
    for item in items:
        video_id = provider.extract_video_id(item['Link'])
        upload_video_ids.add(video_id)
        
        # Check if exists
        query = """
        SELECT video_id FROM hoardarr._default.tiktok_favorites
        WHERE user_id = $1 AND video_id = $2
        """
        result = cluster.query(query, user_id, video_id)
        
        if not list(result):
            # New video - fetch metadata
            metadata = provider.fetch_metadata(item['Link'])
            metadata['favorited_at'] = item['Date']
            
            # Vectorize and store
            text = provider.get_vectorization_text(metadata)
            embedding = embedding_service.embed(text)
            doc = provider.transform_to_db_model(metadata, embedding)
            doc['user_id'] = user_id
            
            couchbase_service.store('tiktok_favorites', doc)
            new_count += 1
    
    # Remove videos not in upload
    query = """
    SELECT video_id FROM hoardarr._default.tiktok_favorites
    WHERE user_id = $1
    """
    result = cluster.query(query, user_id)
    db_video_ids = {row['video_id'] for row in result}
    
    to_delete = db_video_ids - upload_video_ids
    
    for video_id in to_delete:
        delete_query = """
        DELETE FROM hoardarr._default.tiktok_favorites
        WHERE user_id = $1 AND video_id = $2
        """
        cluster.query(delete_query, user_id, video_id)
    
    return {
        'processed': len(items),
        'new': new_count,
        'removed': len(to_delete)
    }
```

---

## Sync Metadata Structure

```python
{
    'provider': 'reddit',  # or 'github'
    'user_id': 'username',
    'last_sync_at': '2025-10-17T12:00:00Z',
    'last_full_reconciliation': '2025-10-10T12:00:00Z',
    'sync_count': 5,
    'access_token': 'encrypted...',
    'refresh_token': 'encrypted...'
}
```

**Stored as**: Document key = `{provider}:{user_id}`

---

## Sync Orchestration

### Main Sync Function
```python
def sync_provider(user_id, provider_name):
    # Load metadata
    metadata = get_sync_metadata(user_id, provider_name)
    sync_count = metadata.get('sync_count', 0)
    
    # Determine sync type
    if sync_count % 10 == 0:
        result = full_reconciliation(user_id, provider_name, metadata)
    else:
        result = delta_sync(user_id, provider_name, metadata)
    
    return result
```

---

## Rate Limiting

**Reddit**: 60 requests/minute
**GitHub**: 5000 requests/hour

**Strategy**:
- Track request count in memory
- Sleep if approaching limit
- Log rate limit headers

```python
def make_reddit_request(url, headers):
    global reddit_request_count
    
    # Check rate limit
    if reddit_request_count >= 55:  # Buffer of 5
        time.sleep(60)  # Wait 1 minute
        reddit_request_count = 0
    
    response = requests.get(url, headers=headers)
    reddit_request_count += 1
    
    return response
```

---

## Error Handling During Sync

**Network Errors**: Retry with exponential backoff
**API Errors**: Log and skip item
**Rate Limits**: Wait and retry
**Invalid Tokens**: Notify user to re-auth

```python
def sync_with_retry(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt  # Exponential backoff
            time.sleep(wait_time)
```

