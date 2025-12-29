# Microservices Patterns

## Concept Overview

Microservices patterns are proven solutions to recurring architectural problems in distributed systems. They address challenges specific to systems composed of many independent services: **how do services find each other, how do they coordinate complex operations, how do they manage failure, and how do they evolve without breaking contracts?**

These patterns are more specialized and domain-specific than Gang of Four patterns. They emerged from building large-scale distributed systems and are documented in resources like "Microservices Patterns" by Chris Richardson. They address problems that are irrelevant in monolithic systems but critical in microservices.

## Problems These Patterns Solve

### 1. **Service Location and Discovery**
Services are created, destroyed, scaled up and down dynamically. How does Service A know where to find Service B?

### 2. **Managing Cascading Failures**
One slow or failing service cascades failures to its callers. System degrades gracefully instead of collapsing completely.

### 3. **Coordinating Transactions Across Services**
In monoliths, ACID transactions are natural. In microservices, coordinating updates across multiple databases is complex. How do you maintain consistency?

### 4. **Temporal Decoupling**
Service B doesn't need to process a request immediately. Event-driven communication allows decoupling in time.

### 5. **Query Across Multiple Services**
Getting a complete picture of a customer requires aggregating data from multiple services. Querying across services is non-trivial.

### 6. **Identifying Service Boundaries**
Choosing where to draw boundaries between services is the fundamental microservices challenge.

### 7. **Gradual Monolit-to-Microservices Transition**
Rewriting a monolith from scratch is risky and slow. How do you migrate incrementally?

## Why These Problems Exist

**Root Cause:** Decomposing systems into independent services creates problems that don't exist in monoliths.

- **Monoliths are simple:** Everything runs in one process, uses one database, communicates through method calls. The architectural model is straightforward.

- **Microservices distribute complexity:** By making services independent, you distribute:
  - **Location:** Services can be anywhere in the network
  - **Time:** Services don't synchronously wait for each other
  - **Failure modes:** One service's failure doesn't directly crash others
  - **State management:** Each service has its own data

- **Distributed systems are fundamentally harder:** Two-phase commit, eventual consistency, partial failures, network partitions—these are new problems that didn't exist before.

Patterns address these by providing **named solutions with known trade-offs**. Rather than inventing solutions, engineers apply proven patterns.

## Common Solutions / Approaches

### API Gateway Pattern

**Problem:** Clients directly call multiple services. Clients must:
- Know all service locations
- Handle authentication/authorization for each
- Manage versions (Service A uses v1, Service B uses v2)
- Implement resilience (retries, timeouts)

**Solution:** Single entry point (API Gateway) that routes requests to services.

```
Client → API Gateway → Order Service
                    → Payment Service
                    → Inventory Service
```

**Responsibilities of API Gateway:**
1. **Routing:** Route requests to appropriate service
2. **Composition:** Aggregate responses from multiple services
3. **Authentication:** Validate JWT or API keys
4. **Rate limiting:** Enforce quotas per client
5. **Protocol translation:** Convert REST to gRPC, for example
6. **Request/response transformation:** Transform for client compatibility

**Benefits:**
- Clients see single API (simpler contracts)
- Services can evolve behind the gateway
- Cross-cutting concerns (auth, rate-limiting) are centralized
- Service locations are hidden

**Trade-off:** API Gateway becomes a bottleneck and single point of failure.

**Mitigation:**
- Multiple gateway instances (load-balanced)
- Lightweight implementation
- Fallback behavior if gateway fails

**Common implementations:** Kong, AWS API Gateway, Traefik, NGINX.

### Circuit Breaker Pattern

**Problem:** Service B is failing or slow. Service A keeps calling it, timing out, and retrying. These wasted calls consume resources that could be used elsewhere.

**Solution:** Circuit breaker tracks failures. State transitions:

```
CLOSED (normal)
    ↓ (failures exceed threshold)
OPEN (fast-fail)
    ↓ (after timeout, try recovery)
HALF_OPEN (test if recovered)
    ↓ (if request succeeds, return to CLOSED)
CLOSED (normal)
```

**States explained:**

1. **CLOSED:** Normal operation. Requests pass through. Failures are counted.
2. **OPEN:** Threshold exceeded. Requests fail immediately without calling the service. Gives service time to recover.
3. **HALF_OPEN:** After a timeout, allow one request to test if the service has recovered. If it succeeds, return to CLOSED. If it fails, return to OPEN.

**Example:**

```
PaymentService.charge(amount):
  if circuit.isOpen():
    throw ServiceUnavailable("Payment service is down, try later")
  
  try:
    result = call PaymentService
    circuit.recordSuccess()
    return result
  except:
    circuit.recordFailure()
    if circuit.failureCountExceeds(threshold):
      circuit.open()
    throw
```

**Benefits:**
- Prevents cascading failures (stops calling a failing service)
- Gives services time to recover (back-off reduces load)
- Enables fast-fail behavior (clients don't wait for timeouts)

**Implementation libraries:** Resilience4j (Java), Polly (C#), Hystrix (Java).

### Bulkhead Pattern

**Problem:** One slow service causes thread starvation. All threads are waiting on that service, leaving no capacity for other requests.

**Solution:** Isolate resources. Dedicated thread pools for different concerns.

```
Thread Pool A: Order Service calls (5 threads)
Thread Pool B: Payment Service calls (10 threads)
Thread Pool C: Inventory calls (8 threads)

If Payment Service is slow:
  All 10 threads in Pool B get blocked
  Pools A and C continue normally
```

**Benefits:**
- Isolates failure domains
- Prevents one dependency from starving others
- Enables better resource utilization

**Trade-off:** Overhead of managing multiple thread pools, complexity in tuning.

### Saga Pattern

**Problem:** Updating data across multiple services requires coordination. A saga is a distributed transaction.

**Scenario:**
```
1. Order Service creates order
2. Payment Service charges customer
3. Inventory Service reserves items
4. Shipping Service schedules delivery

If any step fails, previous steps must be compensated (undone)
```

**Two implementations:**

#### Choreography Saga

**How it works:** Services emit events, others react.

```
1. OrderService publishes "OrderCreated"
2. PaymentService listens, charges customer, publishes "PaymentProcessed"
3. InventoryService listens to "PaymentProcessed", reserves items, publishes "ItemsReserved"
4. ShippingService listens to "ItemsReserved", schedules delivery
5. If error at step 3: InventoryService publishes "ItemReservationFailed"
6. PaymentService listens, refunds customer
```

**Characteristics:**
- **Decoupled:** Services don't know about each other
- **Hard to understand:** Business logic is scattered across services
- **Hard to test:** Must simulate all event sequences
- **No central view:** Difficult to see overall saga state

#### Orchestration Saga

**How it works:** Central orchestrator coordinates all steps.

```
SagaOrchestrator:
  1. Call OrderService.create()
  2. Call PaymentService.charge()
     If fails: call PaymentService.refund()
  3. Call InventoryService.reserve()
     If fails: call PaymentService.refund(), call InventoryService.release()
  4. Call ShippingService.schedule()
     If fails: compensate all previous steps
```

**Characteristics:**
- **Centralized logic:** Saga flow is in one place
- **Easy to understand:** Clear sequence of operations
- **Easier to test:** Test orchestrator directly
- **Single point of failure:** Orchestrator is critical

**Trade-off:** Choreography vs. Orchestration is about centralization vs. decoupling.

**Best practice:** Orchestration for complex, critical sagas (order processing). Choreography for simpler, autonomous event reactions.

### Event Sourcing Pattern

**Problem:** Traditional databases store current state. If you need to understand "how did we get here?", you must audit tables or logs separately.

**Solution:** Store state as an immutable sequence of events.

```
Traditional:
  Order table: id=123, status="shipped", amount=100

Event Sourcing:
  events:
    OrderCreated(id=123, items=[...], amount=100)
    PaymentProcessed(id=123, amount=100)
    ItemsReserved(id=123, items=[...])
    OrderShipped(id=123)
```

**How it works:**

1. **Events are immutable:** Never update events, only append
2. **State is computed:** Load all events, apply in order to compute current state
3. **Audit trail is free:** All events are stored; complete history is available
4. **Replay is possible:** Apply events up to any point in time to see historical state

**Benefits:**
- **Complete audit trail:** Know everything that happened
- **Replay:** Reproduce bugs or test "what if" scenarios
- **Temporal queries:** "What was the order status at 3pm?"
- **Recovery:** Lost current state? Replay events to rebuild

**Trade-offs:**
- **Storage:** Storing all events uses more space than storing current state
- **Complexity:** Event evolution (what if event schema changes?)
- **Query performance:** Computing state from events is slower than querying current state

**Mitigation:**
- Snapshots: periodically save state, so you don't replay entire history
- Event stream databases: specialized databases for event data (EventStoreDB, Postgres JSONB)

### CQRS (Command Query Responsibility Segregation) Pattern

**Problem:** Read and write patterns are often different.

- **Writes:** Happen less frequently, require validation, must be consistent
- **Reads:** Happen frequently, must be fast, can tolerate eventual consistency

Trying to optimize for both in a single model compromises both.

**Solution:** Separate models for reads and writes.

```
Write Model (Command):
  Order aggregate, strict validation, ACID transactions
  
Read Model (Query):
  Denormalized Order view optimized for reads
  Includes customer name, item details, shipping info
```

**How it works:**

1. Command writes to write model (Order Service)
2. Command publishes event (OrderCreated)
3. Read model subscriber receives event
4. Read model updates denormalized view
5. Query reads from read model (eventually consistent)

**Benefits:**
- **Scalability:** Read model scales independently (replicate to multiple databases)
- **Performance:** Denormalized data structure optimized for queries
- **Simplicity:** Write model is focused on consistency, read model on speed

**Trade-off:**
- **Eventual consistency:** Queries see stale data briefly
- **Complexity:** Two models to maintain, synchronization logic
- **Operational overhead:** More databases/services to monitor

**Real-world example:** E-commerce search.

```
Write Model: Canonical product data (inventory, details)
Read Model: Elasticsearch index optimized for search, filtering, faceting
When product is updated:
  1. Update canonical data
  2. Publish ProductUpdated event
  3. Elasticsearch index updates
  4. Search queries use fresh index
```

### Strangler Fig Pattern

**Problem:** You need to migrate from monolith to microservices without massive rewrite.

**Solution:** Gradually replace monolith functionality with microservices.

```
Phase 1:
  API Gateway routes requests to monolith
  (everything still in monolith)

Phase 2:
  Extract OrderService as microservice
  API Gateway routes /orders/* to OrderService
  Other paths still route to monolith

Phase 3:
  Extract PaymentService, InventoryService
  Monolith is now smaller (gets strangled)

Phase N:
  Monolith is gone, all functionality in microservices
```

**Benefits:**
- **Low risk:** Revert easily by routing back to monolith
- **Independent progress:** Extract services as ready
- **Parallel development:** Monolith and services can evolve together
- **Validates boundaries:** See if proposed service boundaries work before full commitment

**How to extract:**

1. **Identify a capability:** "Orders" is a good candidate
2. **Create the microservice:** OrderService with its own database
3. **Duplicate data:** Copy relevant data from monolith to OrderService database
4. **Add facade in monolith:** Monolith delegates to OrderService through API
5. **Update API Gateway:** Route to OrderService instead of monolith
6. **Monitor and verify:** Ensure service works as expected
7. **Stop syncing data:** Monolith no longer needs order data
8. **Remove from monolith:** Eventually retire order logic from monolith

**Trade-off:** Maintaining two systems in parallel increases complexity temporarily, but risk is lower than rewrite.

### Database-Per-Service Pattern

**Pattern:** Each service owns its database.

**Characteristics:**
- **Services are independent:** Can choose database technology per service
- **Schema is private:** Other services can't directly query
- **Data ownership is clear:** One service owns each piece of data

**Consequences:**
- **Sharing data is complex:** Must go through APIs or events
- **Consistency is eventually consistent:** No distributed ACID transactions
- **Query complexity:** Joining data across services is application logic, not SQL

**Implementation:**

```
Order Service has order_db with orders, line_items tables
Payment Service has payment_db with payments, transactions tables

If PaymentService needs order details:
  1. Call OrderService API: GET /orders/123
  2. OrderService queries its database, returns JSON
  3. PaymentService uses the data
  
(Never query order_db directly from PaymentService)
```

**Why this matters:** If you share databases, services are coupled. You can't:
- Migrate database (change RDBMS to NoSQL)
- Scale database independently
- Refactor schema
- Use different database technology

### Service Mesh Pattern

**Problem:** Managing inter-service communication, resilience, observability at scale is complex. Each service must implement retries, timeouts, circuit breakers, logging, tracing.

**Solution:** Sidecar proxy for each service handles cross-cutting concerns.

```
Actual request: Service A → Sidecar Proxy A → Network → Sidecar Proxy B → Service B
                                                         (Handles resilience, logging, etc.)
```

**Sidecar responsibilities:**
- Retries and timeouts
- Circuit breaking
- Request/response transformation
- Metrics and logging
- mTLS encryption between services
- Load balancing
- Rate limiting

**Benefits:**
- **Business logic is pure:** Services don't include resilience boilerplate
- **Consistent behavior:** All services have same resilience policies
- **Operational control:** Change behavior without touching service code

**Trade-off:**
- **Added latency:** Requests go through sidecar proxy
- **Operational complexity:** Manage sidecar proxies (usually with Kubernetes)
- **Resource overhead:** Extra process per service

**Popular service mesh implementations:** Istio, Linkerd, Consul.

## Trade-offs and Limitations

### Trade-off 1: Pattern Complexity vs. Simplicity

**Cost:** Each pattern adds architectural complexity. Sagas have compensation logic. CQRS has two models. Circuit breakers require failure detection and state management.

**When to apply:** Use patterns when the problem they solve is significant. Circuit breaker is essential in microservices. CQRS is optional—only if read/write patterns differ significantly.

**Principle:** Don't apply patterns speculatively. Apply them to solve concrete problems.

### Trade-off 2: Consistency vs. Availability

**Problem:** Sagas and event-driven patterns lead to eventual consistency. During the window of inconsistency, the system shows outdated information.

**Mitigation:**
- Make windows short (milliseconds to seconds)
- Design for inconsistency (show "processing" status instead of wrong status)
- Use idempotency to handle retries safely

### Trade-off 3: Debugging Difficulty

**Cost:** Patterns like choreography sagas and event-driven systems are harder to debug. Business logic is distributed.

**Mitigation:**
- Excellent logging and tracing
- Simulation/testing of saga flows
- Clear event contracts and documentation

### Trade-off 4: Operational Burden

**Cost:** Each pattern requires operational support. Circuit breakers need metrics. Sagas need compensation handlers. Service mesh needs orchestration platform.

**Mitigation:** Invest in infrastructure and tooling. Use managed services (AWS/GCP/Azure provide managed service meshes).

## Common Pitfalls and Misuses

### Pitfall 1: Over-applying Patterns

**Mistake:** Using Saga pattern for every update. Creating CQRS for every service even if read/write patterns are identical.

**Better approach:** Apply patterns to specific problems:
- Use Saga when you need to coordinate across services
- Use CQRS when read and write patterns differ
- Use Event Sourcing when audit trail is important

### Pitfall 2: Choreography Sagas Without Clear Contracts

**Mistake:** Services emit events without documenting schema. Subscribers guess what events mean.

**Result:** Events change, subscribers break. Saga flows are implicit and hard to test.

**Better approach:**
- Define event schemas clearly (Avro, JSON Schema, Protocol Buffers)
- Document saga flows (even if choreography-based)
- Version events to enable evolution

### Pitfall 3: CQRS Without Eventual Consistency Design

**Mistake:** Implementing CQRS without considering how to handle stale reads.

**Problem:** User updates order, immediately views order status, sees old status.

**Better approach:**
- Accept eventual consistency in UI (show "updating" status)
- Use event IDs to detect staleness
- Re-query if data seems wrong
- Design for inconsistency rather than pretending it doesn't exist

### Pitfall 4: Event Sourcing Without Snapshots

**Mistake:** Storing all events but not periodically snapshotting. For long-lived aggregates with thousands of events, replaying takes seconds.

**Better approach:**
- Snapshot every N events
- Replay only events after last snapshot
- Consider event stream database (better performance than replaying)

### Pitfall 5: Ignoring Idempotency in Sagas

**Mistake:** Saga compensation steps aren't idempotent. If compensation is retried, it might partially undo twice, causing inconsistency.

**Example:**
```
Refund(amount=100):
  balance -= 100      // First call: balance 900 - 100 = 800
  balance -= 100      // Retry: balance 800 - 100 = 700 (WRONG!)
```

**Better approach:**
```
Refund(amount=100, refund_id=abc123):
  if refund_id already processed:
    return cached result
  balance -= 100
  record refund_id as processed
  return result
```

### Pitfall 6: Service Mesh Without Understanding Trade-offs

**Mistake:** Deploying service mesh (Istio) for small system with 5 services.

**Cost:** Added latency, operational complexity, resource overhead don't justify benefits for small scale.

**Better approach:** Service mesh is most valuable at scale (50+ services) where manual resilience implementation across services is error-prone.

## Key Takeaways

1. **API Gateway is the first pattern to implement.** Single entry point simplifies client code and centralizes cross-cutting concerns.

2. **Circuit Breaker is essential.** In distributed systems, failures cascade. Circuit breaker prevents cascades.

3. **Sagas coordinate transactions.** Choose orchestration for complex, critical flows. Choreography for simple, decoupled reactions.

4. **Event Sourcing provides the complete story.** Useful when audit trail is important (financial systems, compliance). Overkill for simple CRUD.

5. **CQRS enables independent optimization.** Use when read and write patterns differ significantly. Not needed for balanced workloads.

6. **Strangler Fig minimizes risk.** Migrate gradually. Measure, verify, and extract services one at a time.

7. **Database-per-Service is non-negotiable.** Services that share databases are not truly independent.

8. **Idempotency is critical.** Networks fail. Every operation that can be retried must be safe to retry.

9. **Service Mesh is for scale.** Don't add it until manual resilience becomes unbearable.

10. **Patterns solve real problems.** Apply them when the pain point is acute, not speculatively.

## Related Concepts

- Bounded Contexts (DDD) — inform service boundaries
- Distributed Transactions and CAP Theorem
- Event-Driven Architecture
- Microservices Observability (logging, tracing, metrics)
- API Versioning and Backward Compatibility
- Testing Microservices (contract tests, integration tests)
