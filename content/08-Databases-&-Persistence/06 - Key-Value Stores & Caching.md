# Key-Value Stores & Caching: In-Memory Architecture & Strategies

## Purpose & Problem Space

Key-value stores and caching layers address a fundamental performance problem: databases are slow. Even the most optimized relational or document database has measurable latency (milliseconds); repeated accesses to the same data waste resources. Caching reduces latency and database load by keeping frequently accessed data in fast memory.

**Core problems addressed:**
- Database queries are slow (disk I/O, network latency)
- Repeated queries for the same data waste resources
- Write-through to disk is bottleneck (requests block on fsync)
- Cache invalidation is complex (how to keep cache fresh)
- High-throughput applications need sub-millisecond response times
- Session data and temporary information need fast access

Key-value stores (Redis, Memcached) provide in-memory storage with simple interfaces (GET/SET), enabling sub-millisecond access. They sacrifice rich querying for raw speed—the fundamental trade-off that makes caching effective.

---

## Core Concepts & Internal Architecture

### In-Memory Storage Model

**Memory vs. Disk Trade-offs:**

Data in memory:
- **Fast:** Sub-millisecond access (CPU cache + RAM)
- **Volatile:** Lost on server restart (data not persisted)
- **Expensive:** RAM costs more per byte than disk
- **Limited:** Server RAM ≤ 1TB practical; disk can be petabytes

Data on disk:
- **Durable:** Survives restarts
- **Slow:** 5-10ms access (mechanical disk), 1ms (SSD)
- **Cheap:** Much higher capacity
- **Complex:** Requires indexing, transaction management

**Cache Semantics:**
The cache is always a copy; the authoritative version lives elsewhere (database). This is crucial: cache is expendable. If cache is lost, data still exists in database (temporary inconsistency until repopulation).

### Redis: In-Memory Data Structure Store

Redis is more than a simple cache; it's an in-memory data structure server supporting multiple data types and operations.

**Data Structures:**

**Strings:**
Simple key-value pairs:
```
SET key value
GET key
APPEND key additional_text
```

Underlying implementation: allocated memory block, length tracking, automatic growth.

**Lists:**
Ordered collections accessible from both ends:
```
LPUSH list value  // Add to head
RPUSH list value  // Add to tail
LPOP list        // Remove from head
LRANGE list 0 -1 // All elements
```

Implementation: doubly-linked list with front/back pointers. O(1) operations on ends, O(n) on arbitrary indices.

**Sets:**
Unordered unique values:
```
SADD set member
SISMEMBER set member
SUNION set1 set2  // Union
SINTER set1 set2  // Intersection
```

Implementation: hash table (similar to dictionary). O(1) operations, efficient for membership testing and set operations.

**Sorted Sets:**
Sets with scores (weights), ordered by score:
```
ZADD leaderboard 100 player1
ZADD leaderboard 150 player2
ZRANGE leaderboard 0 -1 WITHSCORES  // All members by score
```

Implementation: skip list (probabilistic balanced tree). O(log n) insertion/deletion/lookup, supports range queries efficiently.

**Hashes:**
Maps of string keys to values:
```
HSET user:5 name "Alice" email "alice@example.com"
HGET user:5 name
HGETALL user:5
```

Implementation: hash table of strings. O(1) operations, efficient for storing multiple fields of an object.

**Bitmaps and HyperLogLog:**
Specialized structures for counting and membership tests with minimal memory (probabilistic, not exact).

### Memcached: Simple, Fast, Distributed

Memcached is intentionally minimal:
- Simple string key-value store (all values are strings)
- SET/GET operations only
- No persistence
- Built-in distributed caching (consistent hashing)

Contrasted with Redis:
- **Redis:** Complex, feature-rich, single-server (clustering added later)
- **Memcached:** Simple, fast, designed for distributed use

**Why Memcached Chose Simplicity:**
- Speed: less code = fewer CPU cycles per operation
- Clarity: no complex data types to debug
- Scalability: simple API works well distributed (each operation independent)

### Memory Management and Eviction

**Memory Allocation:**
When memory fills up, cache can't store new data. Solutions:

1. **Reject New Data:** Return error (unacceptable for most applications)
2. **Evict Old Data:** Remove existing entries to make room

**Eviction Policies (LRU, LFU):**

**LRU (Least Recently Used):**
Track last access time for every key; evict the key not accessed for longest time.

Implementation: doubly-linked list ordered by access time. On access, move key to head (most recent). On eviction, remove tail (least recent). O(1) with clever bookkeeping.

Rationale: Recently accessed data likely to be accessed again (temporal locality). Old data likely not needed.

Effectiveness: Excellent for working sets smaller than cache (cache hits). Poor for full sequential scans (thrashing—every access evicts previously accessed data).

**LFU (Least Frequently Used):**
Track access frequency; evict least frequently accessed key.

Implementation: Counter per key, min-heap or priority queue of counters.

Improvement over LRU: Distinguishes between occasional heavy users (few accesses, high frequency) and frequent light users. Prevents recently accessed but infrequently used data from dominating cache.

Example: Video streaming where popular videos accessed frequently; niche videos accessed rarely. LFU keeps popular videos, LRU might evict them if a niche video is accessed.

**TTL (Time-To-Live):**
Keys automatically expire after set duration:

```
SETEX key 3600 value  // Expires after 3600 seconds
```

Implementation: absolute expiration time per key, background process checks and deletes expired keys, or lazy deletion (check on access).

Advantages:
- Automatic cleanup (don't manually invalidate)
- Bounded cache size (no unbounded growth)
- Temporal data naturally expires (sessions, tokens)

### Persistence Options

Redis offers two persistence modes:

**RDB (Redis Database Snapshots):**
Periodically write entire dataset to disk:
```
BGSAVE  // Fork process, save to disk in background
```

Pros:
- Compact representation (compressed)
- Fast loading (single file read)
- Fast point-in-time recovery

Cons:
- Not point-accurate (lag between saves, latest data lost)
- Blocking during save (if foreground, blocking; background fork doubles memory)
- Large datasets slow to save

**AOF (Append-Only File):**
Log every write operation:
```
SET key value  // Logged to disk
LPUSH list item  // Logged to disk
```

On restart, replay log to restore state.

Pros:
- Point-accurate (every operation recorded)
- Human-readable (can inspect/debug operations)
- Can rewrite history (compact log)

Cons:
- Slower than RDB (write log every operation)
- Larger file (operation text vs. binary data)
- Slower loading (replay operations)

**Hybrid Approach:**
Use RDB for snapshots (fast recovery) + AOF for durability (point-accurate). On restart, load RDB snapshot then replay AOF changes since last snapshot.

---

## Consistency, Performance & Reliability Challenges

### Cache Invalidation

**The Hard Problem:**
"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Invalidation Challenges:**

**Stale Cache Problem:**
```
1. Read data from database, store in cache
2. Another process updates data in database
3. Cache still has old value
4. Application uses stale data
```

**Cache Invalidation Strategies:**

1. **Time-Based (TTL):**
```
SET user:5 value EX 3600  // Expires after 1 hour
```

Simple, but arbitrary TTL:
- Too short: cache misses increase (refresh often)
- Too long: stale data served (old information)

2. **Event-Based Invalidation:**
When data changes, explicitly invalidate cache:
```
UPDATE users SET name = 'Bob' WHERE id = 5
DELETE cache_key user:5
```

Pros: Immediate invalidation
Cons: Requires application to coordinate updates, easy to forget cases, complex in distributed systems

3. **Write-Through Cache:**
Update cache and database together:
```
SET cache_key value
UPDATE database SET ... WHERE ...
```

Pros: Cache always consistent with database
Cons: Double writes, slower

4. **Write-Behind (Write-Back) Cache:**
Write to cache immediately (fast), asynchronously write to database:
```
1. Set cache_key value (return immediately)
2. Queue write to database (background)
3. Database eventually updated
```

Pros: Very fast (don't wait for database)
Cons: Data loss if cache crashes before database write, inconsistency window

5. **Lazy Loading:**
On cache miss, load from database and cache:
```
def get_user(user_id):
    cached = redis.get(f"user:{user_id}")
    if cached:
        return cached
    value = database.find_user(user_id)
    redis.setex(f"user:{user_id}", 3600, value)
    return value
```

Simple, but "cache stampede" if key expires simultaneously for many requests (all miss, all hit database).

### Concurrency and Race Conditions

**Lost Update Problem in Cache:**
```
Request A: cache.get(counter) → 5
Request B: cache.get(counter) → 5
Request A: cache.set(counter, 6)
Request B: cache.set(counter, 6)
Result: counter = 6 (expected 7)
```

Redis transactions (MULTI/EXEC) provide ACID guarantees for multiple operations, but require careful implementation.

**Distributed Consistency:**
In multi-server cache setups (Memcached clusters), request A might hit different server than request B:
```
Server 1: user:5 = version 1
Server 2: user:5 = version 0
```

Consistency hash helps but doesn't prevent divergence.

### Cache Stampede (Thundering Herd)

**Scenario:**
Popular cache key expires simultaneously. Thousands of concurrent requests miss cache, all query database.

```
Time T: key expires
Time T+0.001s: 10,000 requests hit cache miss
All 10,000 queries hit database simultaneously
Database overload, slow responses
```

**Solutions:**

1. **Lock-Based Refresh:**
First cache miss acquires lock, regenerates cache, others wait
```
def get_expensive_data():
    cached = cache.get(key)
    if not cached:
        lock = cache.lock(key)
        if lock.acquire(timeout=1):
            cached = cache.get(key)  # Check again
            if not cached:
                value = expensive_computation()
                cache.set(key, value)
            lock.release()
        else:
            # Wait for other process
            wait_for_cache_set(key)
            cached = cache.get(key)
    return cached
```

2. **Probabilistic Early Refresh:**
Refresh cache before it expires, based on probability
```
// With 5% probability, regenerate even if cached
if cached and random() > 0.95:
    async_refresh(key)
return cached
```

3. **Increasing TTL:**
Key doesn't expire; manually invalidate or update

### Memory Exhaustion and OOM Issues

**Problem:** Cache grows unbounded until Out-of-Memory error causes crashes.

**Solutions:**
- Set max memory and eviction policy (MAXMEMORY + eviction in Redis)
- Monitor memory usage, alert when approaching limit
- Reduce TTL to age out data faster
- Implement multiple cache layers (L1: fast, small; L2: slower, large)

### Persistence Latency

**RDB Blocking:**
Background save process (BGSAVE) fork duplicates parent memory. If dataset is 10GB, fork requires 10GB free memory. Forks can pause Redis (milliseconds to seconds) during large operations.

**AOF Rewrite Latency:**
Rewriting log to compact file can cause latency spikes.

**Durability Cost:**
Full durability (fsync every write) turns Redis into a slow disk database. Most systems accept durability risk (cache can be lost) for performance.

---

## Design Decisions & Trade-offs

### Caching Strategy Selection

**Cache-Aside (Lazy Loading):**
Application responsible for populating cache:
```
def get_user(user_id):
    value = cache.get(user_id)
    if not value:
        value = db.get_user(user_id)
        cache.set(user_id, value, ttl=3600)
    return value
```

Pros: Simplicity, cache only stores used data
Cons: Cache miss cost (must query database), thundering herd risk, staleness

**Write-Through:**
Application writes to cache and database:
```
def update_user(user_id, data):
    cache.set(user_id, data)
    db.update_user(user_id, data)
```

Pros: Cache always consistent with database
Cons: Double writes, slows down writes

**Write-Behind:**
Application writes to cache only, database updated asynchronously:
```
def update_user(user_id, data):
    cache.set(user_id, data)
    queue.enqueue(lambda: db.update_user(user_id, data))
```

Pros: Very fast writes
Cons: Data loss risk, eventual consistency, complexity

**Refresh-Ahead:**
Proactively refresh cache before expiration:
```
If cache_access_time is old and access_frequency high:
    async_refresh(key)
```

Pros: Reduces cache misses for popular data
Cons: Wasted refreshes, complexity

**Best Practice Hybrid:**
Combine lazy loading (default) with TTL (automatic expiration) and event-based invalidation (critical updates). Use write-behind for non-critical data accepting eventual consistency.

### Multi-Level Caching

Combine multiple cache layers, each with different characteristics:

**L1: Application Cache (In-Process Memory)**
- Store frequently used objects in application memory
- Examples: parsed configs, connection pools
- Pros: Extreme speed (no network hop), no serialization
- Cons: Not shared across instances, memory overhead in each process

**L2: Distributed Cache (Redis/Memcached)**
- Shared across all application instances
- Pros: Shared across servers, reasonable speed (network latency)
- Cons: Network latency, serialization overhead, shared infrastructure

**L3: Database Query Cache**
- ORM caching (prepared statements, query results)
- Pros: Cache at right level (query results)
- Cons: Limited to single instance

**L4: Database**
- Authoritative source
- Slow but durable

**Cache Line Architecture:**
```
L1 (in-process) ←→ L2 (Redis) ←→ L3 (DB query) ←→ L4 (Database)
 <1us               5ms             50ms             5-10ms
```

Each miss goes down the hierarchy. Typical flow:
1. Check L1 (hit → return immediately)
2. Miss L1 → check L2 (hit → populate L1, return)
3. Miss L2 → query DB (hit → populate L2, L1, return)

### TTL Strategy

**Aggressive TTL (Short, e.g., 5 minutes):**
- More frequent misses
- Fresher data
- Less memory pressure
- When: Data changes frequently, staleness unacceptable

**Permissive TTL (Long, e.g., 24 hours):**
- Few misses (cache stays fresh)
- Stale data risk
- Memory pressure (data stays cached long)
- When: Data rarely changes, staleness acceptable

**Adaptive TTL:**
Adjust based on access patterns:
- Popular items: long TTL (stay cached, minimize misses)
- Unpopular items: short TTL (free up memory)

**No TTL (Manual Invalidation):**
- Cache never expires
- Application explicitly invalidates
- Risk: stale data forever if invalidation missed

### Redis vs. Memcached

| Aspect | Redis | Memcached |
|--------|-------|-----------|
| Data Types | Multiple (strings, lists, sets, etc.) | Strings only |
| Persistence | RDB, AOF | No persistence |
| Transactions | MULTI/EXEC | No |
| Pub/Sub | Yes | No |
| Lua Scripting | Yes | No |
| Distributed | Add-on (Redis Cluster) | Native (consistent hashing) |
| Speed | Slightly slower | Slightly faster |
| Memory Efficiency | Good | Excellent |

**Redis:** When you need data structures, persistence, or transactions. Single-server or clustered.

**Memcached:** When you need maximum simplicity and speed. Inherently distributed. Excellent for simple string caching.

---

## Alternative Models & Comparisons

### Cache Replacement Policies

| Policy | Best For | Weakness |
|--------|----------|----------|
| LRU | General working sets | Poor for sequential scans |
| LFU | Skewed access (some popular) | Frequency counting overhead |
| FIFO | Steady workloads | Ignores access patterns |
| W-TinyLFU | Modern, combines LRU+LFU | More complex |
| ARC | Adaptive | Moderate complexity |

LRU is most common; LFU improves in specific scenarios.

### Distributed Caching Strategies

**Consistent Hashing:**
Map keys to servers using hash function. When servers added/removed, only rehash affected keys.

**Replication:**
Cache the same data on multiple servers. Survives single server failure but wastes memory.

**Sharding:**
Partition cache by key. Each key on one server. Survives server failure with loss of that shard's data (acceptable for cache).

**Hybrid:**
Replicate important data, shard less critical data.

---

## When to Use and When to Avoid

### Ideal for Caching:

- **Read-Heavy Workloads:** High read-to-write ratios (80/20 or more)
- **Repeated Queries:** Same data accessed multiple times (user profile, config)
- **Acceptable Stale Data:** Eventual consistency acceptable (minutes old fine)
- **High Response Time Requirements:** Sub-millisecond latency needed
- **Database Overload:** Too many requests hitting database

### Avoid or Use Carefully:

- **Write-Heavy Workloads:** Cache updates slow down writes
- **Consistency Critical:** Financial transactions, accurate counters (complex with cache)
- **Small Dataset:** Cache overhead (network, serialization) not worth it
- **Volatile Data:** Constant changes make cache quickly stale
- **Single-Server System:** Cache overhead without shared benefit

### Use Key-Value Stores for:

- **Sessions:** User login state, temporary data
- **Leaderboards:** Frequent updates, range queries (sorted sets)
- **Rate Limiting:** Counter per IP/user
- **Pub/Sub Messaging:** Event distribution
- **Job Queues:** Redis lists as simple task queue
- **Real-Time Analytics:** Counters, aggregates

### Avoid Key-Value for:

- **Complex Queries:** SQL with JOINs better
- **Full-Text Search:** Elasticsearch better
- **Time-Series:** Specialized databases (InfluxDB)
- **Relationships:** Graph databases better

---

## Summary

Key-value stores and caching layers solve fundamental performance problems through in-memory storage, accepting volatility for speed. Redis provides rich data structures and persistence options; Memcached provides extreme simplicity and speed. Cache invalidation is hard—choosing appropriate strategies (time-based, event-based, lazy loading) is crucial. Multi-level caching architecture optimizes for different speeds. Understanding eviction policies, TTL strategies, and consistency challenges enables effective cache design that improves application performance while maintaining data correctness.
