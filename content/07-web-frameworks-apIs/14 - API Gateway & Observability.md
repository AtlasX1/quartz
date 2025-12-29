# API Gateway, Monitoring & Observability: Infrastructure for Production APIs

## Introduction & Purpose

An API is only useful if it's **observable**—if you can understand what's happening, detect problems, and debug issues in production. As systems grow from single servers to distributed microservices, observability becomes mission-critical.

This document explores the infrastructure that makes production APIs reliable: API gateways for request routing and policy enforcement, logging/tracing for understanding behavior, and metrics for detecting anomalies before they become outages.

## Core Concepts & Internal Architecture

### API Gateway Architecture

An **API gateway** is a reverse proxy that sits between clients and backend services:

```
Client → API Gateway → Backend Services
              ↓
    (Routing, Auth, Rate Limiting,
     Caching, Logging, Tracing)
```

**Responsibilities:**

1. **Request routing** — Route requests to appropriate backend
2. **Authentication/Authorization** — Verify identity and permissions
3. **Rate limiting** — Prevent abuse
4. **Request/response transformation** — Modify requests/responses
5. **Caching** — Cache responses to reduce backend load
6. **Load balancing** — Distribute across backend instances
7. **Logging/Tracing** — Record all requests for debugging

**Gateway Tools:**

- **NGINX** (lightweight, fast, C-based)
- **Kong** (feature-rich, Lua-based plugins)
- **AWS API Gateway** (managed, cloud-native)
- **Envoy** (modern, service mesh integration)
- **Traefik** (Docker/Kubernetes-native)

### Rate Limiting

**Rate limiting** prevents abuse by restricting requests per time period:

```
User makes 10 requests/second
Rate limit: 5 requests/second
    ↓
6th+ requests rejected with 429 Too Many Requests
```

**Algorithms:**

**Token Bucket:**
```
Bucket: 100 tokens (capacity)
Refill rate: 10 tokens/second

Request arrives:
    If bucket has ≥1 token: Remove token, allow
    Else: Reject

Over time:
    Allows burst (100 requests if client waits)
    Then sustainable rate (10/second)
```

**Sliding Window:**
```
Window: Last 60 seconds
Limit: 100 requests per minute

For each request:
    Count requests in last 60 seconds
    If count < 100: Allow
    Else: Reject
```

**Implementation (Redis):**

```lua
-- Token bucket in Redis
local bucket = redis.call('GET', 'rate:' .. userId)
local tokens = tonumber(bucket or 100)
local refill = math.floor((time.now - lastRefill) * refillRate)
local newTokens = math.min(100, tokens + refill)

if newTokens >= 1 then
    redis.call('SET', 'rate:' .. userId, newTokens - 1)
    return 'ALLOW'
else
    return 'REJECT'
end
```

**Architectural consequence:** Rate limits must be **distributed** (Redis, not in-memory) to work across multiple servers.

### Authentication at Gateway Level

The gateway can enforce authentication before requests reach backends:

```javascript
// Gateway middleware
app.use((req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1]
    
    if (!token) {
        return res.status(401).json({ error: 'Unauthorized' })
    }
    
    try {
        const decoded = jwt.verify(token, secret)
        req.user = decoded
        next()
    } catch (err) {
        res.status(401).json({ error: 'Invalid token' })
    }
})
```

**Advantage:** Backend services don't need to authenticate; they trust the gateway.

**Architecture:**

```
Untrusted Request
    ↓
Gateway validates JWT
    ↓
If invalid, reject here (fast)
    ↓
If valid, inject user context
    ↓
Backend service receives pre-authenticated request
```

### Structured Logging

**Logging** records application behavior:

**Naive logging:**
```javascript
console.log('User created: Alice')
```

**Structured logging:**
```javascript
logger.info('user_created', {
    userId: 123,
    email: 'alice@example.com',
    timestamp: '2024-01-15T10:30:45Z',
    duration_ms: 145,
    source: 'POST /api/users'
})
```

Output (JSON):
```json
{
    "level": "info",
    "event": "user_created",
    "userId": 123,
    "email": "alice@example.com",
    "timestamp": "2024-01-15T10:30:45Z",
    "duration_ms": 145,
    "source": "POST /api/users"
}
```

**Libraries:**
- **Winston** (Node.js, flexible)
- **Pino** (Node.js, fast)
- **Bunyan** (Node.js, JSON-first)

**Architectural benefit:** Structured logs are **machine-readable**. Can query/analyze programmatically:

```
Find all errors in last hour:
$ curl logs-service.example.com/search?level=error&time=1h

Find slowest requests:
$ curl logs-service.example.com/search?sort=-duration_ms&limit=10
```

### Distributed Tracing

**Tracing** follows requests through distributed systems:

**Problem without tracing:**

```
Client makes request
    ↓
API Gateway processes
    ↓
Service A makes call to Service B
    ↓
Service B makes call to database
    ↓
Response takes 2 seconds total

Where did 2 seconds go? API Gateway: 100ms? Service A: 500ms? Service B: 1400ms?
```

**Solution: Distributed trace**

```
Trace ID: abc123 (unique ID for this request)

Gateway receives request
    └─ Span: gateway_processing (100ms)
        └─ Calls Service A
            └─ Span: service_a_processing (500ms)
                └─ Calls Service B
                    └─ Span: service_b_processing (1000ms)
                        └─ Database query (400ms)

Total: 2000ms = 100 + 500 + 1000
Database: 400ms (bottleneck)
```

**Implementation (OpenTelemetry):**

```typescript
import { trace } from '@opentelemetry/api'

const tracer = trace.getTracer('my-service')

app.get('/users/:id', async (req, res) => {
    const span = tracer.startSpan('get_user')
    span.setAttributes({ userId: req.params.id })
    
    try {
        const span2 = tracer.startSpan('database_query', { parent: span })
        const user = await db.findById(req.params.id)
        span2.end()
        
        res.json(user)
    } finally {
        span.end()
    }
})
```

**Trace visualization (Jaeger, DataDog):**

```
Timeline visualization:
Gateway [========]
  ↓ calls Service A [==============]
           ↓ calls Service B [======]
                   ↓ Database query [===]

Can see: A is waiting for B, B is waiting for database
Solution: Add caching before database
```

### Metrics and Alerting

**Metrics** quantify API behavior over time:

**Red metrics:**
```
Rate: Requests per second
Errors: Errors per second
Duration: Response time (latency)
```

**Key metrics:**

```
http_requests_total (counter)
    Labels: method, path, status
    Example: http_requests_total{method="GET", path="/users", status="200"} 1523

http_request_duration_seconds (histogram)
    Labels: method, path
    Example: 
        .001 buckets: 50 (< 1ms)
        .005 buckets: 120 (< 5ms)
        .05 buckets: 800 (< 50ms)
        → Tells: 50 requests < 1ms, 120 total < 5ms, etc.

http_requests_in_flight (gauge)
    Current number of ongoing requests
    Example: 45
```

**Tools:**
- **Prometheus** (metrics database + scraping)
- **Grafana** (visualization)
- **StatsD** (metrics collection)
- **CloudWatch** (AWS-managed)

**Alerting:**

```yaml
# Prometheus alert rule
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
for: 5m
annotations:
  summary: "High error rate detected"
  description: "{{ $value }}% of requests are errors"

# If >5% errors for >5 minutes, trigger alert
```

### Health Checks

**Health checks** verify service is functional:

```javascript
app.get('/health', (req, res) => {
    // Check dependencies
    const dbHealthy = await checkDatabase()
    const cacheHealthy = await checkRedis()
    
    if (dbHealthy && cacheHealthy) {
        res.status(200).json({ status: 'healthy' })
    } else {
        res.status(503).json({ status: 'unhealthy' })
    }
})
```

**Kubernetes liveness/readiness:**

```yaml
livenessProbe:
    httpGet:
        path: /health
        port: 3000
    initialDelaySeconds: 10
    periodSeconds: 10

readinessProbe:
    httpGet:
        path: /health
        port: 3000
    initialDelaySeconds: 5
    periodSeconds: 5
```

Kubernetes:
- **Liveness:** If unhealthy, restart pod
- **Readiness:** If unhealthy, remove from load balancer

## Common Problems & Failure Scenarios

### Problem 1: Cascading Failures

**Scenario:**

```
Service A → Service B → Service C

Service C becomes slow (database overloaded)
    ↓
Service B's requests to C timeout
    ↓
Service B runs out of connection pool (connections hang waiting)
    ↓
Service A can't reach Service B
    ↓
Entire system appears down

Root cause: One slow service brings down others
```

**Solution: Circuit breaker**

```javascript
const CircuitBreaker = require('opossum')

const breaker = new CircuitBreaker(async () => {
    return await serviceB.call()
}, {
    timeout: 3000,  // 3 second timeout
    errorThresholdPercentage: 50,  // Open if 50%+ errors
    resetTimeout: 30000  // Try again after 30s
})

breaker.on('open', () => console.log('Circuit open; failing fast'))

try {
    const result = await breaker.fire()
} catch (err) {
    // Handle failure gracefully
    return cachedResponse
}
```

### Problem 2: Observability Blindness

**Scenario:**

```
System goes down
No logs to show what happened
No traces to show request flow
No metrics to show what changed

"Why did it fail?" — No idea
```

**Root cause:** No observability infrastructure.

**Solution:** Instrument everything

```
Every request:
    → Structured log (INFO level)
    → Distributed trace (OpenTelemetry)
    → Metrics (Prometheus)

Every error:
    → Error log (ERROR level)
    → Trace shows context
    → Alert configured
```

### Problem 3: Alert Fatigue

**Scenario:**

```
Alerts fire constantly
Engineers ignore them (hundreds per day)
Real problem happens
Buried in noise of false alerts
Outage goes unnoticed
```

**Root cause:** Alerts not tuned; too many false positives.

**Solution: Alert well**

```
❌ Bad alerts:
   "CPU > 70%" (normal for healthy systems)
   "Any error" (occasional errors are normal)

✅ Good alerts:
   "Error rate > 5% for 5+ minutes"
   "Response time p99 > 1000ms for 5+ minutes"
   "Database connection pool exhausted"
```

## Design Decisions & Trade-offs

### Trade-off 1: Centralized Gateway vs. Service Meshes

**Centralized Gateway:**
- Pros: Single point of control, easy to understand, simpler
- Cons: Single point of failure, bottleneck

**Service Mesh (Istio, Linkerd):**
- Pros: Distributed, no single point of failure, automatic tracing
- Cons: Complex, many moving parts, operational overhead

**Trade-off:** Gateways for entry point; service mesh for internal communication.

### Trade-off 2: Sampling vs. Recording All

**Sample traces (1% of requests):**
- Pros: Low overhead, manageable storage
- Cons: Might miss rare issues

**Record all requests:**
- Pros: Complete picture, don't miss anything
- Cons: High storage/processing cost

**Trade-off:** Sample in production; record all in development.

## When to Implement

### Essential for Production

- ✅ Structured logging (at least INFO level)
- ✅ Basic metrics (requests/errors/duration)
- ✅ Health checks (liveness/readiness)
- ✅ Rate limiting (prevent abuse)
- ✅ Authentication at gateway
- ✅ Circuit breakers (prevent cascades)

### Important for Scale

- ✅ Distributed tracing (OpenTelemetry)
- ✅ Comprehensive alerting
- ✅ Request correlation IDs
- ✅ Centralized log aggregation

### Nice to Have

- ⚠️ Advanced analytics
- ⚠️ Custom metrics
- ⚠️ Distributed caching at gateway level

---

## Conclusion

**Observability is not a feature; it's a prerequisite for production systems.** You cannot operate systems you cannot observe.

Key principles:

1. **Instrument everything** — Logs, traces, metrics
2. **Structure logs** — Machine-readable, queryable
3. **Trace requests** — Follow through distributed systems
4. **Metrics matter** — Understand behavior over time
5. **Alert well** — Few, meaningful alerts; not noise
6. **Health checks** — Kubernetes/orchestrators depend on them
7. **API gateway** — Single point for cross-cutting concerns (auth, rate limiting, logging)

The goal is to detect and debug issues before they become customer-facing outages.
