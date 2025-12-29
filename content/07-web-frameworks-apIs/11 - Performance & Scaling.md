# Performance & Scaling APIs: Architectural Patterns for High-Throughput Systems

## Introduction & Purpose

APIs are often built for current demands, then retrofitted when they fail at scale. Understanding performance and scaling requires understanding **where bottlenecks occur, how to measure them, and when to optimize**.

This document explores the architectural patterns and infrastructure decisions that enable APIs to handle thousands of requests per second: caching strategies, connection pooling, load balancing, and async I/O optimization.

## Core Concepts & Internal Architecture

### Benchmarking and Measurement

Before optimizing, you must **measure**. Without data, optimization is guessing.

#### Load Testing Tools

**autocannon** (Node.js-focused):
```bash
autocannon -c 100 -d 30 -l http://localhost:3000
# -c: 100 concurrent connections
# -d: 30 seconds duration
# Outputs: requests/sec, latency percentiles, errors
```

**k6** (scenario-based):
```javascript
import http from 'k6/http'
import { check } from 'k6'

export let options = {
    vus: 100,  // 100 virtual users
    duration: '30s'
}

export default function() {
    let res = http.get('http://localhost:3000/users')
    check(res, {
        'status is 200': (r) => r.status === 200,
        'latency < 500ms': (r) => r.timings.duration < 500
    })
}
```

**wrk** (simple, fast):
```bash
wrk -t4 -c100 -d30s http://localhost:3000
# -t4: 4 threads
# -c100: 100 connections
# -d30s: 30 seconds
```

#### Key Metrics

**Throughput:** Requests per second (RPS)
```
1000 RPS = 1,000 successful requests/second
```

**Latency:** Response time (how long user waits)
```
p50 (median): 50% of requests faster than this
p95: 95% of requests faster than this
p99: 99% of requests faster than this
p99.9: 99.9% of requests faster than this
```

**Amdahl's Law:** Performance limited by bottleneck
```
If 10% of requests are database-bound (slow)
And 90% are CPU-bound (fast)
Optimizing CPU doesn't help overall performance much
```

### Caching Strategies

Caching stores **frequently accessed data** to avoid expensive recomputation:

#### HTTP Caching (Client/Browser/CDN)

**Response headers control caching:**

```
GET /api/users/123
Response:
Cache-Control: public, max-age=3600
ETag: "abc123"
```

Browser/CDN caches response for 1 hour. Next request for same URL served from cache without hitting server.

**Cache validation:**
```
Cached response has ETag: "abc123"
    ↓
Browser checks if content changed: If-None-Match: "abc123"
    ↓
Server responds: 304 Not Modified (no body sent)
    ↓
Browser uses cached response
```

**Architectural benefit:** Reduces server load for frequently accessed resources.

**Limitation:** Only works for GET requests; client must check often for updates.

#### Redis In-Memory Cache

```javascript
const redis = require('redis')
const client = redis.createClient()

app.get('/users/:id', async (req, res) => {
    const cacheKey = `user:${req.params.id}`
    
    // Check cache first
    const cached = await client.get(cacheKey)
    if (cached) {
        return res.json(JSON.parse(cached))
    }
    
    // Cache miss; fetch from database
    const user = await database.findById(req.params.id)
    
    // Store in cache for 5 minutes
    await client.setex(cacheKey, 300, JSON.stringify(user))
    
    res.json(user)
})
```

**Caching layers:**

```
Request hits:
1. Browser cache (if GET & cacheable) → Instant
2. CDN cache (if response cached) → ~10ms
3. Redis in-memory cache → ~1-5ms
4. Database (slowest) → ~10-100ms
```

#### Cache Invalidation Problem

**The hard part:** Keeping cache consistent with data.

**Strategies:**

1. **Time-based (TTL):** Cache expires after time period
   ```javascript
   await redis.setex(key, 300, value)  // 5 minute expiry
   ```
   Pros: Simple
   Cons: Stale data possible (up to 5 minutes old)

2. **Event-based:** Invalidate on update
   ```javascript
   app.post('/users/:id', async (req, res) => {
       const user = await database.update(req.params.id, req.body)
       await redis.del(`user:${req.params.id}`)  // Clear cache
       res.json(user)
   })
   ```
   Pros: Always fresh
   Cons: Must remember to invalidate everywhere

3. **Dependency-based:** Track what data caches depend on
   ```javascript
   // user:1 cache depends on all posts by user 1
   // When post created, invalidate user:1
   ```
   Pros: Automatically consistent
   Cons: Complex to implement

### Connection Pooling

Databases are expensive to connect to. **Pooling reuses connections**:

**Without pooling:**
```
Request 1: Create connection, execute query, close
Request 2: Create connection, execute query, close
Request 3: Create connection, execute query, close
...
(Overhead: 10ms connection time × 3 = 30ms overhead)
```

**With pooling:**
```
Pool: [Connection 1, Connection 2, Connection 3, ...]
Request 1: Use Connection 1, execute query, return to pool
Request 2: Use Connection 2, execute query, return to pool
Request 3: Reuse Connection 1, execute query, return to pool
(Overhead: minimal; connections already open)
```

**Configuration:**

```javascript
const pool = new Pool({
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    max: 20,  // Maximum 20 connections
    idleTimeoutMillis: 30000,  // Close idle connections after 30s
    connectionTimeoutMillis: 2000  // Fail if can't get connection in 2s
})

app.get('/users/:id', async (req, res) => {
    const connection = await pool.connect()  // Get from pool
    try {
        const result = await connection.query('SELECT * FROM users WHERE id = $1', [req.params.id])
        res.json(result.rows[0])
    } finally {
        connection.release()  // Return to pool
    }
})
```

**Pool sizing:**

Too small:
```
Requests queue up waiting for connections
Latency increases
```

Too large:
```
Database runs out of memory (connections are expensive)
Server memory usage high
Connections idle, wasting resources
```

**Ideal:** Pool size ≈ (Core Count × 2) + Effective Spindle Count. For web apps, typically 10-20 connections.

### Load Balancing

**Load balancing** distributes requests across multiple servers:

```
Client
    ↓
Load Balancer (reverse proxy)
    ├─ → Server 1
    ├─ → Server 2
    ├─ → Server 3
    └─ → Server 4
```

**Algorithms:**

1. **Round-robin:** Distribute sequentially
   ```
   Request 1 → Server 1
   Request 2 → Server 2
   Request 3 → Server 3
   Request 4 → Server 1
   ```
   Pros: Simple, fair
   Cons: Doesn't account for server load

2. **Least connections:** Route to server with fewest active connections
   ```
   Server 1: 5 connections
   Server 2: 2 connections ← Route here
   Server 3: 3 connections
   ```
   Pros: Balances load
   Cons: Slightly more overhead

3. **IP hash:** Route based on client IP
   ```
   Client A (IP 1.2.3.4) always goes to Server 1
   Client B (IP 1.2.3.5) always goes to Server 2
   ```
   Pros: Session affinity ("sticky sessions")
   Cons: Uneven load if few clients

**Tools:**
- **NGINX** (reverse proxy, load balancer)
- **HAProxy** (highly tuned load balancer)
- **AWS Application Load Balancer (ALB)**
- **Envoy** (modern service mesh proxy)

### Clustering

**Clustering** runs multiple Node.js processes on one machine:

```javascript
const cluster = require('cluster')
const os = require('os')
const app = require('./app')

if (cluster.isMaster) {
    // Master process
    const numWorkers = os.cpus().length
    
    for (let i = 0; i < numWorkers; i++) {
        cluster.fork()  // Spawn worker process
    }
    
    // Load balance across workers
    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`)
        cluster.fork()  // Respawn if worker dies
    })
} else {
    // Worker process
    app.listen(3000)  // All workers listen on same port
}
```

**Architectural consequence:**

```
Master Process
├─ Worker 1 (PID 1001) → Listen on 3000
├─ Worker 2 (PID 1002) → Listen on 3000
├─ Worker 3 (PID 1003) → Listen on 3000
└─ Worker 4 (PID 1004) → Listen on 3000

OS load balances kernel connections across workers
```

**Benefits:**
- Use all CPU cores on single machine
- Automatic worker restart if one dies

### Compression

**Compression** reduces response payload:

```javascript
app.use(require('compression')())

app.get('/users', async (req, res) => {
    const users = await database.getAll()
    // Response compressed with gzip or brotli before sending
    res.json(users)
})
```

**Size reduction:**
```
Original JSON: 100KB
Gzip compressed: 15-20KB (80-85% reduction)
Brotli compressed: 12-15KB (85-88% reduction)
```

**Trade-off:**
- Pros: 80%+ bandwidth reduction; faster over networks
- Cons: CPU overhead for compression (milliseconds)

On slow networks or for large payloads, compression is almost always worth it. For fast local networks, overhead may not justify benefit.

### Async I/O and Streams

Node.js is **event-driven**; proper async patterns prevent blocking:

**Blocking (bad):**
```javascript
// Reads entire file into memory
const data = fs.readFileSync('large-file.json')
const processed = JSON.parse(data)
// If file is 1GB, blocks server for seconds
```

**Non-blocking (good):**
```javascript
// Reads file in chunks
fs.createReadStream('large-file.json')
    .pipe(JSONStream.parse('*'))  // Parse as it streams
    .on('data', (record) => {
        // Process each record as it arrives
        processRecord(record)
    })
```

**Architectural benefit:** Server remains responsive; handles other requests while streaming data.

**Backpressure:** Important for streams

```javascript
const writeStream = fs.createWriteStream('output.json')
const readStream = fs.createReadStream('input.json')

readStream.pipe(writeStream)
// .pipe() automatically handles backpressure
// If writeStream is slow, readStream pauses automatically
```

## Common Problems & Failure Scenarios

### Problem 1: Cache Stampede

**Scenario:**

```
Cache expires at 12:00:00
    ↓
1000 concurrent requests hit at 12:00:01
    ↓
All miss cache, all query database simultaneously
    ↓
Database receives 1000x normal traffic
    ↓
Database slow/crashes
```

**Root cause:** All requests cache expires simultaneously.

**Solution: Cache warming**
```javascript
setInterval(async () => {
    const freshData = await database.query()
    await cache.set(key, freshData)
}, 250000)  // Refresh before expiry
```

### Problem 2: Connection Pool Exhaustion

**Scenario:**

```
Pool size: 10 connections
Concurrent requests: 15
    ↓
5 requests wait for connections
    ↓
Database slow; connections take 500ms to process
    ↓
Queueing backlog grows
    ↓
Timeout errors
```

**Root cause:** Requests slower than expected; queue builds.

**Mitigation:** Monitor pool usage; scale servers if consistently near capacity.

### Problem 3: Load Imbalance

**Scenario:**

```
Server 1: 50 requests/sec
Server 2: 5 requests/sec
Server 3: 5 requests/sec
(Total: 60 RPS; should be ~20 each)
```

**Root cause:** Sticky sessions + uneven client distribution.

**Solution:** Load balancer with proper algorithm.

## When to Optimize

**Don't premature optimize.** Measure first:

```
1. Benchmark current performance
2. Profile to identify bottleneck
3. Optimize bottleneck only
4. Measure improvement
5. Repeat
```

### Scale Considerations

| RPS | Architecture | Notes |
|-----|---|---|
| <100 | Single server | No optimization needed |
| 100-1k | Single server + caching | Redis cache probably sufficient |
| 1k-5k | 2-3 servers + load balancing | Add connection pooling |
| 5k-10k | 5+ servers + Redis + optimization | Optimize hot paths |
| 10k+ | Cluster + CDN + extensive caching | Specialized infrastructure |

---

## Conclusion

API performance at scale requires **multiple layers of optimization**:

1. **Measurement** — Benchmark and profile before optimizing
2. **Caching** — Reduce expensive operations (database, external APIs)
3. **Connection pooling** — Reuse database connections
4. **Load balancing** — Distribute across servers
5. **Compression** — Reduce bandwidth
6. **Async/streams** — Prevent blocking
7. **Infrastructure** — CDN, clustering, read replicas

Scaling is fundamentally about **identifying the bottleneck and removing it**. For most APIs, the bottleneck is the database, not the application server. Cache well, pool connections, distribute load—these solve 90% of performance problems.
