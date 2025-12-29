# Microservices & Communication: Distributed Systems Architecture and Inter-Service Protocols

## Conceptual Overview

Microservices architecture decomposes monolithic applications into independently deployable, loosely coupled services. Each service owns a domain, has its own database, and communicates with others through well-defined interfaces. This provides scalability benefits (scale services independently), organizational benefits (teams own services), and operational challenges (distributed complexity, eventual consistency, network failures). Understanding different communication protocols (REST, gRPC, GraphQL, message queues), service discovery, resilience patterns, and observability in distributed systems is essential for building systems that scale beyond the capabilities of monoliths.

---

## Internal Mechanics: Service Communication Protocols

### REST: HTTP-Based Communication

**Principles**: Resources identified by URLs; standard HTTP methods (GET, POST, PUT, DELETE).

```javascript
// Client
GET /api/v1/users/123
POST /api/v1/users { "name": "Alice" }
PUT /api/v1/users/123 { "name": "Bob" }
DELETE /api/v1/users/123

// Server
app.get('/api/v1/users/:id', (req, res) => {
  res.json(getUser(req.params.id));
});
```

**Characteristics**:
- **Text-based (JSON)**: Human-readable; easy to debug; larger payloads.
- **Loose coupling**: Client and server evolve independently.
- **Stateless**: Each request contains all information needed.

**Trade-offs**:
- Pros: Simple; HTTP widely supported; easy to understand; cacheable via HTTP caching.
- Cons: Chatty (N requests to fetch N resources—N+1 problem); large payloads; no strong typing.

### gRPC: High-Performance RPC

**Protocol**: Binary (Protocol Buffers); HTTP/2 multiplexing.

```protobuf
// user.proto
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (User) {}
  rpc ListUsers(Empty) returns (stream User) {}
  rpc CreateUser(CreateUserRequest) returns (User) {}
}

message GetUserRequest {
  int32 id = 1;
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

**Generated code**:
```javascript
// Node.js generated client
const { UserServiceClient } = require('./user_pb');
const grpc = require('@grpc/grpc-js');

const client = new UserServiceClient('localhost:50051', grpc.credentials.createInsecure());

client.getUser({ id: 123 }, (error, response) => {
  if (error) console.error(error);
  else console.log(response);
});
```

**Characteristics**:
- **Binary protocol**: Compact (5-10x smaller than JSON); faster parsing.
- **HTTP/2 multiplexing**: True concurrency; no head-of-line blocking.
- **Code generation**: Type-safe; client and server stubs generated from .proto files.
- **Streaming**: Bidirectional streaming; server can stream responses.

**Trade-offs**:
- Pros: Ultra-fast; efficient; strong typing; true streaming.
- Cons: Steep learning curve; Protocol Buffers syntax; less human-readable; harder debugging.

### GraphQL: Query Language and Optimization

**Concept**: Clients specify exactly what data they need; server resolves graph of data.

```graphql
query GetUserWithPosts {
  user(id: 123) {
    name
    email
    posts(first: 5) {
      id
      title
      content
    }
  }
}
```

**Server**: Resolves nested fields through resolver functions.

```javascript
const resolvers = {
  Query: {
    user: async (_, { id }) => db.getUser(id),
  },
  User: {
    posts: async (parent) => db.getPosts(parent.id),  // parent is user
  }
};
```

**Characteristics**:
- **Client-driven**: Clients request exactly what's needed; reduces over-fetching.
- **Strongly typed schema**: Self-documenting API.
- **Single endpoint**: `/graphql` handles all queries.

**Trade-offs**:
- Pros: Eliminates over-fetching; strongly typed; introspection enables tooling.
- Cons: Complexity (resolvers, caching, batching); N+1 query problem (mitigated by dataloader); not suitable for file uploads.

### Message Queues: Asynchronous Communication

**Pattern**: Producer sends message; broker stores; consumer retrieves.

```
Producer: "Process order #123"
  ↓
Message Broker (RabbitMQ, Kafka, NATS)
  ↓
Consumer(s): Receive and process

Decoupling: Producer and consumer operate independently; no synchronous coupling.
```

**Use case**: Asynchronous, fire-and-forget operations.

```javascript
// Producer
const queue = new Queue('order-processing', redisConnection);
await queue.add({ orderId: 123, userId: 456 });

// Consumer (separate process)
queue.process(async (job) => {
  const order = await db.getOrder(job.data.orderId);
  await paymentService.charge(order);
  await inventoryService.reserve(order);
});
```

---

## Service Discovery and Communication Patterns

### Service Discovery: Static vs. Dynamic

**Static (hardcoded)**:
```javascript
const userService = 'http://user-service:3000';
const orderService = 'http://order-service:3001';
```

**Limitation**: If services move (redeploy, failure), clients have stale URLs.

**Dynamic (service registry)**:
```
Service starts → Registers with registry: "user-service at 10.0.0.1:3000"
Client needs service → Queries registry: "Where is user-service?"
Registry: "10.0.0.1:3000"
Client: Connects to 10.0.0.1:3000

Service crashes → Deregisters or registry detects via heartbeat
Registry: "user-service unavailable"
Client: Tries next instance or fails
```

**Tools**: Consul, etcd, Kubernetes service discovery.

### API Gateway: Single Entry Point

```
Clients
  ↓
API Gateway (Load balancer, auth, rate limiting, routing)
  ↓
Microservices (internal network)
```

**Responsibilities**:
- Route requests to correct service
- Authentication/authorization
- Rate limiting
- Request/response transformation
- Logging and monitoring

**Tools**: Kong, AWS API Gateway, Istio (service mesh).

---

## Resilience Patterns in Distributed Systems

### Circuit Breaker: Fail Fast Under Load

Monitors failure rate; prevents cascading failures.

```javascript
const breaker = new CircuitBreaker(
  async () => fetch('http://slow-service:3000'),
  {
    timeout: 5000,                      // 5 second timeout
    errorThresholdPercentage: 50,       // Open if 50% fail
    resetTimeout: 30000                 // Try recovering after 30s
  }
);

// States: CLOSED (normal) → OPEN (failing) → HALF_OPEN (testing) → CLOSED
try {
  const result = await breaker.fire();
} catch (error) {
  // Circuit is open; fail fast without waiting
  res.status(503).send('Service unavailable');
}
```

### Retry with Exponential Backoff

Retry failed requests with increasing delays:

```javascript
async function retryWithBackoff(fn, maxAttempts = 3) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      
      const delay = Math.pow(2, attempt - 1) * 1000;  // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

const response = await retryWithBackoff(() => 
  fetch('http://external-api.com/data')
);
```

### Timeout: Prevent Indefinite Waits

```javascript
const timeout = (promise, ms) => 
  Promise.race([
    promise,
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), ms)
    )
  ]);

const result = await timeout(
  fetch('http://slow-service:3000'),
  5000  // 5 second timeout
);
```

### Bulkhead: Resource Isolation

Limit resource usage per operation to prevent one failure from affecting others:

```javascript
const semaphore = new Semaphore(10);  // Max 10 concurrent requests

async function callExternalService() {
  await semaphore.acquire();
  try {
    return await fetch('http://external-api:3000');
  } finally {
    semaphore.release();
  }
}
```

---

## Distributed Tracing and Observability

### Correlation IDs: Tracking Requests Across Services

```javascript
const correlationId = req.headers['x-correlation-id'] || uuid.v4();

// Log with correlation ID
logger.info({ correlationId, action: 'order-created' });

// Pass to downstream services
const response = await fetch('http://payment-service:3000', {
  headers: { 'x-correlation-id': correlationId }
});
```

**Benefit**: Trace request flow across multiple services in logs.

### OpenTelemetry: Standard Tracing

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

const sdk = new NodeSDK({
  traceExporter: new JaegerExporter({
    serviceName: 'order-service',
    endpoint: 'http://jaeger:14268/api/traces'
  })
});

sdk.start();

// Automatic instrumentation of HTTP, database, etc.
// Manual instrumentation if needed
const tracer = require('@opentelemetry/api').trace.getTracer('my-app');
const span = tracer.startSpan('processOrder');
// ... do work
span.end();
```

---

## Problems & Challenges

### 1. Distributed Transaction Problem

**The problem**: Transactions that span multiple services are difficult to implement. Distributed consensus (2-phase commit) is complex and slow.

**Example**: Order service needs to reserve inventory and charge payment. What if payment succeeds but inventory reservation fails?

### 2. Data Consistency and Eventual Consistency

**The problem**: Services have independent databases; data isn't immediately consistent across services. 

**Scenario**: User updates profile in user-service; order-service still sees old profile temporarily.

### 3. Network Failures and Timeouts

**The problem**: Calls between services can fail silently (network partition, timeout). Client doesn't know if request was processed.

### 4. Service Discovery and Dynamic Routing

**The problem**: Services scale up/down; instances crash and restart. Static URLs don't work.

### 5. Deployment Complexity

**The problem**: Deploying 20 microservices with dependencies (service A depends on service B) is complex. Versioning mismatches break integration.

### 6. Debugging Production Issues

**The problem**: Request spans multiple services; tracing execution flow is difficult without correlation IDs and centralized logging.

### 7. API Versioning

**The problem**: Services need to maintain multiple API versions for backward compatibility.

---

## Solutions & Architectural Approaches

### 1. Saga Pattern for Distributed Transactions

**Choreography** (event-driven): Services emit events; others react.

```
OrderService: emit 'order-created'
  ↓ [event]
PaymentService: emit 'payment-processed'
  ↓ [event]
InventoryService: emit 'inventory-reserved'
```

**Orchestration**: Central coordinator manages steps.

```
Coordinator:
  1. Call OrderService.create()
  2. Call PaymentService.charge()
  3. If payment fails, call OrderService.cancel()
  4. Call InventoryService.reserve()
  5. If fails, rollback payment and order
```

### 2. Implement Idempotency

Ensure operations can be safely retried.

```javascript
// Idempotent: Multiple calls with same idempotency key return same result
app.post('/orders', async (req, res) => {
  const idempotencyKey = req.body.idempotencyKey || uuid.v4();
  
  // Check if already processed
  const existing = await db.getOrder({ idempotencyKey });
  if (existing) return res.json(existing);
  
  // Process
  const order = await db.createOrder({ ...req.body, idempotencyKey });
  res.json(order);
});
```

### 3. Use API Versioning

```javascript
app.get('/api/v1/users/:id', (req, res) => {
  // v1 API returns subset of fields
  res.json({ id: user.id, name: user.name });
});

app.get('/api/v2/users/:id', (req, res) => {
  // v2 API includes additional fields
  res.json({ id: user.id, name: user.name, email: user.email, role: user.role });
});
```

### 4. Implement Health Checks

Services expose readiness and liveness probes:

```javascript
// Liveness: Is service running?
app.get('/health/live', (req, res) => {
  res.json({ status: 'alive' });
});

// Readiness: Can service handle requests?
app.get('/health/ready', async (req, res) => {
  try {
    await Promise.all([
      db.ping(),
      cache.ping(),
      messageBroker.ping()
    ]);
    res.json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});
```

### 5. Use Message Queues for Async Communication

```javascript
// Producer: Fire-and-forget
app.post('/orders', async (req, res) => {
  const order = await db.createOrder(req.body);
  await queue.add({ orderId: order.id }, { delay: 0, attempts: 3 });
  res.json({ orderId: order.id });
});

// Consumer: Process asynchronously
queue.process(async (job) => {
  await paymentService.charge(job.data.orderId);
  await emailService.sendConfirmation(job.data.orderId);
});
```

---

## Trade-offs & Limitations

### Monolith vs. Microservices

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Complexity** | Low | High |
| **Scaling** | All-or-nothing | Fine-grained |
| **Deployment** | Single binary | Multiple services |
| **Debugging** | Easy (local calls) | Hard (distributed) |
| **Database** | Single | Multiple |
| **Team coordination** | Tight coupling | Loose coupling |

### REST vs. gRPC vs. GraphQL

| Aspect | REST | gRPC | GraphQL |
|--------|------|------|---------|
| **Ease of use** | Simple | Complex | Moderate |
| **Performance** | Moderate | Fast | Moderate |
| **Type safety** | Weak | Strong | Strong |
| **Debugging** | Easy | Harder | Moderate |
| **Ecosystem** | Mature | Growing | Growing |

---

## Common Pitfalls

### 1. Tightly Coupled Services

```javascript
// Anti-pattern: Services depend on internal details of others
// OrderService calls UserService directly with expectation of exact response format
const user = await fetch('http://user-service/internal/get-user');
// If UserService changes internal format, OrderService breaks

// Better: Services provide public APIs
const user = await fetch('http://user-service/api/v1/users/:id');
// Contract is stable; internal changes don't break clients
```

### 2. No Timeout on External Calls

```javascript
// Anti-pattern: Can hang indefinitely
const response = await fetch('http://slow-service:3000');

// Better: Set timeout
const response = await fetch('http://slow-service:3000', { 
  timeout: 5000 
});
```

### 3. Lost Correlation IDs

```javascript
// Anti-pattern: Doesn't propagate correlation ID
const response = await fetch('http://downstream-service:3000', {
  headers: { /* no correlation ID */ }
});

// Better: Forward correlation ID
const response = await fetch('http://downstream-service:3000', {
  headers: { 'x-correlation-id': req.headers['x-correlation-id'] }
});
```

### 4. Ignoring Network Failures

```javascript
// Anti-pattern: No retry or circuit breaker
const result = await fetch('http://external-api:3000');

// Better: Resilience patterns
const breaker = new CircuitBreaker(async () => 
  fetch('http://external-api:3000')
);
const result = await breaker.fire();
```

### 5. No API Versioning

```javascript
// Anti-pattern: Upgrade breaks all clients
app.get('/users/:id', (req, res) => {
  // Added new required field; old clients break
  res.json({ id: user.id, version: user.version });
});

// Better: Version APIs
app.get('/api/v1/users/:id', oldHandler);
app.get('/api/v2/users/:id', newHandler);
```

---

## How This Affects System Architecture

Microservices choices influence architecture:

- **Service boundaries**: Domain-driven design determines service split.
- **Communication overhead**: gRPC vs. REST affects performance and complexity.
- **Consistency model**: Saga pattern trades consistency for availability.
- **Observability**: Correlation IDs and centralized tracing are essential.
- **Team structure**: Microservices enable independent team ownership but require clear contracts.

---

## Key Takeaways

1. **REST is simple and widely understood; gRPC is faster and more efficient; GraphQL optimizes for client needs—choose based on use case.**
2. **Circuit breakers, retries, and timeouts are essential for preventing cascade failures in distributed systems.**
3. **Service discovery and health checks enable dynamic service routing and automatic failover.**
4. **Correlation IDs and distributed tracing are critical for debugging production issues across services.**
5. **Idempotency and saga patterns enable reliable operations in distributed transactions.**
6. **API versioning enables services to evolve independently without breaking clients.**
7. **Message queues decouple services and enable asynchronous, resilient communication.**
