# Performance & Scaling: System-Level Optimization and Throughput Maximization

## Conceptual Overview

Scaling Node.js applications involves two dimensions: **vertical scaling** (optimizing a single instance) and **horizontal scaling** (distributing load across multiple instances). Vertical scaling addresses CPU, memory, and I/O bottlenecks through profiling, optimization, and caching. Horizontal scaling requires stateless architecture, load balancing, and distributed consensus mechanisms. Understanding where bottlenecks arise, how to measure performance, and when to apply each optimization is essential for maintaining sub-100ms latencies at 10,000+ RPS.

The challenge is avoiding premature optimization while remaining aware of scalability constraints early enough to prevent costly rewrites late in development.

---

## Internal Mechanics: Throughput and Latency

### Throughput vs. Latency

**Throughput**: Requests processed per second (RPS, queries per second).

**Latency**: Time from request arrival to response (milliseconds).

These are coupled but distinct:

```
Service: 100 RPS capacity, 100ms per request
At 50 RPS:
  - Queuing: 0ms (requests served immediately)
  - Latency: 100ms (request processing time)
  - Total: 100ms

At 100 RPS (at capacity):
  - Queuing: 0ms (no backlog)
  - Latency: 100ms
  - Total: 100ms

At 150 RPS (overloaded):
  - Queuing: 50ms (requests wait in queue)
  - Latency: 100ms (request processing time)
  - Total: 150ms (perceived latency)
```

**Little's Law**: `L = λ × W` where L = queue length, λ = arrival rate, W = average wait time.

At overload, queue grows linearly; perceived latency increases. This is why overload prevention (load shedding, circuit breakers) is critical.

### Amdahl's Law: Parallelism Limits

Amdahl's Law describes speedup from parallelization:

```
Speedup = 1 / (S + P/N)
```

Where S = serial fraction (can't parallelize), P = parallel fraction, N = processors.

**Example**:
- 80% of code can parallelize (P = 0.8)
- 20% is serial (S = 0.2)
- With 4 processors: Speedup = 1 / (0.2 + 0.8/4) = 2.5x

**Implication**: Even with infinite processors, speedup is capped at 1/S. For 20% serial code, maximum speedup is 5x.

This explains why scaling Node.js servers with clustering yields diminishing returns.

---

## Clustering and Load Distribution

### Cluster Mode: Multi-Process on Single Machine

The `cluster` module creates multiple worker processes listening on the same port:

```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  // Master: Create workers (one per CPU core)
  for (let i = 0; i < os.cpus().length; i++) {
    cluster.fork();
  }
  
  // Master restarts crashed workers
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    cluster.fork();  // Respawn
  });
} else {
  // Worker: Run the actual application
  app.listen(3000);
}
```

**How it works**: The master process creates worker processes. Each worker binds to port 3000. The OS kernel (or Node.js) distributes incoming connections across workers. This is simpler than external load balancing but limited to single machine.

**Limitations**:
- Restart loses all in-memory state (sessions, caches)
- Adding machines requires external load balancer
- Resource sharing between workers is implicit (shared file descriptors)

### Load Balancing Algorithms

**Round-robin**: Distribute requests evenly across workers.

```
Request 1 → Worker 1
Request 2 → Worker 2
Request 3 → Worker 3
Request 4 → Worker 1 (cycles back)
```

**Least connections**: Route requests to worker with fewest active connections.

**IP hash**: Route based on client IP; ensures session affinity.

**Weighted round-robin**: Distribute based on worker capacity (some workers handle more load).

---

## Caching Strategies

### In-Process Memory Cache

```javascript
const lru = require('lru-cache');

const cache = new lru({
  max: 1000,           // Max 1000 items
  maxSize: 50 * 1024 * 1024,  // Max 50MB
  ttl: 1000 * 60 * 5   // 5 minutes
});

function getUserWithCache(id) {
  if (cache.has(id)) return cache.get(id);
  
  const user = db.getUser(id);
  cache.set(id, user);
  return user;
}
```

**Pros**: Ultra-fast (memory access).

**Cons**: Not shared across processes; size limited by available RAM; invalidation is manual.

### Redis Cache

```javascript
const redis = require('redis');
const client = redis.createClient();

async function getUserWithCache(id) {
  const cached = await client.get(`user:${id}`);
  if (cached) return JSON.parse(cached);
  
  const user = await db.getUser(id);
  await client.set(`user:${id}`, JSON.stringify(user), 'EX', 3600);  // Expire after 1 hour
  return user;
}
```

**Pros**: Shared across processes; scalable; automatic expiration.

**Cons**: Network latency (microseconds, but adds up); requires Redis deployment; complex invalidation.

### Query Result Caching

```javascript
// Cache expensive database queries
const queryCache = new Map();

async function getExpensiveData() {
  const key = 'expensive_query';
  if (queryCache.has(key)) {
    return queryCache.get(key);
  }
  
  const result = await db.query(`SELECT ... (complex join)`);
  queryCache.set(key, result);
  
  // Invalidate after 5 minutes
  setTimeout(() => queryCache.delete(key), 5 * 60 * 1000);
  
  return result;
}
```

---

## Queue Systems for Asynchronous Processing

### Use Cases

Queues decouple request handling from time-consuming operations:

```
Request: Upload file
    ↓
API: Returns 202 Accepted, enqueues job
    ↓
Worker process: Reads job from queue, processes asynchronously
    ↓
Callback/Webhook: Notifies client when done
```

### Bull/BullMQ: Job Queue for Node.js

```javascript
const Queue = require('bullmq').Queue;

const uploadQueue = new Queue('file-upload', {
  connection: { host: '127.0.0.1', port: 6379 }  // Redis backend
});

// Producer: Enqueue job
app.post('/upload', async (req, res) => {
  const job = await uploadQueue.add(
    { file: req.file, userId: req.user.id },
    { delay: 0, attempts: 3, backoff: { type: 'exponential', delay: 2000 } }
  );
  res.json({ jobId: job.id });
});

// Consumer: Process jobs
uploadQueue.process(async (job) => {
  // Process file asynchronously
  return processFile(job.data.file);
});
```

**Benefits**:
- Non-blocking (request returns immediately)
- Automatic retries (configurable backoff)
- Visibility (monitor queue depth, job status)
- Reliability (persisted in Redis; survives restart)

### Message Brokers: RabbitMQ, Kafka

**RabbitMQ**: Pub/Sub with acknowledgments; reliable delivery.

**Kafka**: Event log; high throughput; retention-based; distributed.

**Trade-off**: Queues (Bull) are simple; brokers (RabbitMQ/Kafka) provide reliability and scaling at cost of complexity.

---

## Event-Driven Architecture

### Pub/Sub Pattern

Decouple services through events:

```javascript
const EventEmitter = require('events');

class UserService extends EventEmitter {
  async createUser(userData) {
    const user = await db.create(userData);
    this.emit('user-created', user);  // Emit event
    return user;
  }
}

const userService = new UserService();

// Subscribers
userService.on('user-created', async (user) => {
  await emailService.sendWelcomeEmail(user.email);
});

userService.on('user-created', async (user) => {
  await analyticsService.trackSignup(user);
});
```

**Benefits**: Services don't know about each other (loose coupling).

**Drawbacks**: Harder to trace execution flow; error handling is implicit.

### Event Sourcing: Event Log as Source of Truth

Instead of storing current state, store all events:

```javascript
// Traditional: Store current balance
{ accountId: 1, balance: 1000 }

// Event sourcing: Store events
[
  { type: 'account-created', accountId: 1, balance: 1000 },
  { type: 'deposit', accountId: 1, amount: 500 },
  { type: 'withdrawal', accountId: 1, amount: 200 }
]

// Derive current state by replaying events
let state = { balance: 0 };
for (const event of events) {
  if (event.type === 'deposit') state.balance += event.amount;
  if (event.type === 'withdrawal') state.balance -= event.amount;
}
// state.balance = 1300
```

**Benefits**: Complete audit trail; can reconstruct any point in time.

**Trade-offs**: Complex query logic; event versioning challenges; eventual consistency.

---

## Problems & Challenges

### 1. CPU Bottlenecks in Single-Threaded Model

**The problem**: CPU-bound work (crypto, compression, JSON parsing) blocks the event loop.

### 2. Memory Bloat and Garbage Collection Pauses

**The problem**: Excessive allocations trigger full GC pauses (50-100ms), causing request latency spikes.

### 3. Connection Pool Saturation

**The problem**: Database or external service connections exhaust pool; requests queue indefinitely.

### 4. Cascading Failures under Load

**The problem**: Overload in one service cascades to dependent services. Example: Payment service is slow; checkout service waits; timeout; user tries again, amplifying load.

### 5. Stateful Application Preventing Horizontal Scaling

**The problem**: In-memory sessions or caches aren't shared across instances. Session affinity is required, reducing scalability.

### 6. Queue Backlog Growth

**The problem**: Producers outpace consumers; job queue grows unbounded.

---

## Solutions & Architectural Approaches

### 1. Use Clustering for CPU-Bound Work

```javascript
// Master process coordinates; workers do computation
if (cluster.isMaster) {
  for (let i = 0; i < os.cpus().length; i++) {
    cluster.fork();
  }
} else {
  app.listen(3000);
}
```

### 2. Implement Circuit Breakers for Resilience

```javascript
const breaker = new CircuitBreaker(async () => {
  return fetch('https://slow-service.com/api');
}, { timeout: 5000, errorThresholdPercentage: 50 });

try {
  const result = await breaker.fire();
} catch (error) {
  // Circuit is open; fail fast
}
```

### 3. Monitor and Alert on Performance Metrics

```javascript
// Track key metrics
const prometheus = require('prom-client');

const httpDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  buckets: [0.1, 0.5, 1, 2, 5]
});

app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpDuration.observe(duration);
  });
  next();
});
```

### 4. Use Read Replicas for Database Scaling

Direct read-only queries to read replicas; write to primary.

```javascript
// Write to primary
const primary = createPool(primaryDb);

// Reads to replica
const replica = createPool(replicaDb);

app.get('/users/:id', async (req, res) => {
  const user = await replica.query('SELECT * FROM users WHERE id = $1', [req.params.id]);
  res.json(user);
});

app.post('/users', async (req, res) => {
  const user = await primary.query('INSERT INTO users ...');
  res.json(user);
});
```

### 5. Implement Request Timeout and Load Shedding

```javascript
app.use((req, res, next) => {
  req.setTimeout(30000);  // 30-second timeout
  next();
});

// Load shedding: Reject requests if queue is too long
const MAX_QUEUE = 100;
let currentQueue = 0;

app.use((req, res, next) => {
  if (currentQueue > MAX_QUEUE) {
    return res.status(503).json({ error: 'Service overloaded' });
  }
  currentQueue++;
  res.on('finish', () => currentQueue--);
  next();
});
```

### 6. Use Redis for Distributed Caching and Sessions

```javascript
const redis = require('redis');
const client = redis.createClient();

// Store sessions in Redis (shared across instances)
const RedisStore = require('connect-redis').default;
app.use(session({
  store: new RedisStore({ client }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false
}));
```

### 7. Profile and Optimize Hot Paths

```bash
# Generate CPU profile
node --prof app.js

# Isolate hot function
node --prof-process v8.log | grep "myExpensiveFunction"

# Use flame graphs
npx clinic.js doctor -- node app.js
```

---

## Trade-offs & Limitations

### Vertical vs. Horizontal Scaling

| Aspect | Vertical | Horizontal |
|--------|----------|-----------|
| **Complexity** | Simple | Complex (load balancer, session management) |
| **Cost** | Expensive hardware | Cheaper commodity hardware |
| **Limit** | Single machine capacity | Theoretically unlimited |
| **Latency** | No coordination overhead | Coordination overhead |

### Caching Strategies

| Strategy | Speed | Complexity | Invalidation |
|----------|-------|-----------|--------------|
| **In-memory** | Fastest | Simple | Manual |
| **Redis** | Fast | Moderate | TTL or manual |
| **CDN** | Ultra-fast (edge) | Moderate | Complex (purge) |
| **Database query cache** | Fast | Moderate | Complex |

---

## Common Pitfalls

### 1. Over-Caching Without Invalidation

```javascript
// Anti-pattern: Cache grows indefinitely
const cache = {};
function getUser(id) {
  if (!cache[id]) {
    cache[id] = db.getUser(id);
  }
  return cache[id];
}
// If user is updated, cache is stale

// Better: TTL or invalidation
const cache = new LRU({ max: 1000, ttl: 5 * 60 * 1000 });
```

### 2. Synchronous Operations in Handlers

```javascript
// Anti-pattern: Blocks event loop
app.get('/expensive', (req, res) => {
  const result = expensiveSyncComputation();
  res.json(result);
});

// Better: Offload to worker or make async
app.get('/expensive', async (req, res) => {
  const result = await expensiveAsyncComputation();
  res.json(result);
});
```

### 3. Not Implementing Timeouts

```javascript
// Anti-pattern: Can hang indefinitely
fetch('https://external-api.com/slow-endpoint').then(handleResponse);

// Better: Timeout
fetch('https://external-api.com/slow-endpoint', { timeout: 5000 });
```

### 4. Stateful Architecture Preventing Scaling

```javascript
// Anti-pattern: Session stored in memory
const sessions = {};
app.post('/login', (req, res) => {
  sessions[req.sessionID] = { userId: user.id };
});

// Multiple instances can't share sessions; requires sticky sessions

// Better: Redis-backed sessions
// Sessions automatically shared across instances
```

### 5. Queue Consumer Slower Than Producer

```javascript
// Anti-pattern: Enqueuing 1000/s but processing 100/s
producer.enqueue(job);  // 1000/s
consumer.process(job);  // 100/s
// Queue grows unbounded; eventually crashes

// Better: Monitor queue depth; scale consumers or rate-limit producer
```

---

## How This Affects System Architecture

Performance decisions influence architecture:

- **Scalability**: Stateless design enables horizontal scaling.
- **Resilience**: Circuit breakers and timeouts prevent cascade failures.
- **Consistency**: Caching and event-driven systems introduce eventual consistency trade-offs.
- **Observability**: Monitoring and profiling are essential for identifying bottlenecks.
- **Complexity**: Event-driven architecture scales better but is harder to reason about.

---

## Key Takeaways

1. **Throughput and latency are coupled via queuing theory; overload increases perceived latency exponentially.**
2. **Clustering provides vertical scaling to a single machine's CPU cores; horizontal scaling requires external load balancing.**
3. **Caching is high-impact but invalidation is complex; TTL-based expiration is simpler than manual invalidation.**
4. **Event-driven and queue-based architectures decouple components, improving scalability and resilience.**
5. **Circuit breakers and timeouts prevent cascade failures under load.**
6. **Profiling and monitoring are essential; optimize based on data, not assumptions.**
