# Microservices Architecture

## Concept Overview

Microservices Architecture is an approach to building distributed systems by decomposing them into small, independently deployable services. Each service:
- Owns a specific business capability
- Runs in its own process
- Communicates through well-defined APIs
- Can be deployed, scaled, and evolved independently

Microservices is fundamentally about **managing complexity through decomposition**. As systems grow, a single codebase becomes unwieldy. Microservices acknowledge this reality and structure the system accordingly.

This is not primarily about technical innovation—it's about **organizational structure aligned with technical boundaries**. Conway's Law states that system architecture mirrors the organization's communication structure. Microservices intentionally designs both to evolve together.

## Problems These Concepts Solve

### 1. **Monolithic Bottlenecks**
Large codebases with many teams create merge conflicts, deployment coordination, and fear of change. A small fix requires understanding sprawling code. Deployments take hours because the entire system must be validated.

### 2. **Inflexible Scaling**
A monolith scales as a unit. If 10% of traffic targets one feature, you scale the entire application, wasting resources. Microservices enable scaling specific services under load.

### 3. **Technology Heterogeneity**
Monoliths lock to one technology stack. If the right tool for one problem is a different language, database, or framework, you can't use it without major refactoring.

### 4. **Organizational Friction**
Large teams working on monoliths create overhead: merge conflicts, deployment delays, coordination meetings. Microservices enable team autonomy—each team owns a service end-to-end.

### 5. **Isolation of Failures**
In monoliths, one bad deployment affects everything. A memory leak in one component crashes the entire system. Microservices isolate failures—one service fails, others continue.

### 6. **Technology Debt Accumulation**
Monoliths accumulate patterns and decisions that become increasingly costly to change. Extracting a microservice allows starting fresh for that capability.

## Why These Problems Exist

**The Root Cause:** Monoliths are appropriate at inception but resist change as systems grow.

- **Initial simplicity wins:** Deploying one artifact is easier than coordinating multiple services. This lasts until the codebase reaches ~100K lines or teams exceed 20 people.

- **Organizational scaling:** Companies grow. As teams expand, the communication overhead of shared codebases increases exponentially. Microservices align technical and organizational structures to reduce friction.

- **Change velocity variance:** Some features need rapid iteration; others are stable. In monoliths, all features share deployment cycles. Microservices decouple these.

- **Technology innovation:** New tools emerge. In monoliths, adopting them requires migrating the entire system. Microservices allow adopting new tech incrementally.

- **Scaling asymmetry:** Traffic doesn't hit all features equally. Search is busy, checkout is occasional. Monoliths can't capture this asymmetry. Microservices enable independent scaling per feature.

## Common Solutions / Approaches

### Service Decomposition

#### Domain-Driven Decomposition

**Approach:** Each service maps to a bounded context (DDD) or business capability.

**Example e-commerce system:**

- **Order Service:** Manages order lifecycle (creation, processing, fulfillment)
- **Payment Service:** Handles payment processing, refunds, reconciliation
- **Inventory Service:** Manages product stock, reservations, availability
- **Shipping Service:** Coordinates fulfillment, tracking, returns
- **Customer Service:** Manages customer profiles, preferences, loyalty
- **Notification Service:** Sends emails, SMS, push notifications
- **Search Service:** Full-text product search, filtering, recommendations

**Why this works:**
- Clear ownership (one team per service)
- Business logic is cohesive (all order logic together)
- Services have natural boundaries (orders don't manage inventory directly)
- Teams can evolve services independently

**Why it's hard:**
- Requires deep domain knowledge
- Boundaries must be stable (refactoring service boundaries is expensive)
- Some cross-service logic is complex (orders need inventory, payments)

**Decision framework:**
- **Can this capability be owned by one team independently?** If yes, it's a service.
- **Do other services need real-time access to this capability?** If yes, they're tightly coupled.
- **Can this scale independently from others?** If yes, separate service.
- **Is this capability likely to evolve differently from others?** If yes, separate service.

#### Functional Decomposition (Anti-pattern)

**Mistake approach:** Services organized by technical function.

```
User Service (manages users)
Product Service (manages products)
Order Service (manages orders)
Auth Service (handles authentication)
Payment Service (manages payments)
```

This **looks** reasonable but fails because:
- Services are **vertically sliced** (each service handles one layer)
- Changes to user-related business logic span multiple services
- No team owns a feature end-to-end
- Services become tightly coupled at the data layer

**Better approach:** Services are **horizontally sliced** (each service owns a complete capability, from UI to database).

### Service Communication

#### Synchronous Communication (Request/Response)

**Pattern:** Service A calls Service B and waits for a response.

```
Order Service calls Inventory Service: "Is item X available?"
Inventory Service responds: "Yes, 5 in stock"
Order Service proceeds with order
```

**Characteristics:**
- **Simple to implement:** Direct method calls across process boundaries
- **Immediate feedback:** Caller knows if request succeeded
- **Tight coupling:** If Inventory Service is slow/down, Order Service is blocked
- **Timeout risk:** Long operations block callers, creating cascading timeouts

**When to use:**
- Queries that don't require durability
- Operations on the critical path (place order)
- Low-latency requirements

**Best practices:**
- Set explicit timeouts
- Implement retries with exponential backoff
- Use circuit breakers to fail fast
- Cache responses when safe

#### Asynchronous Communication (Message-Driven)

**Pattern:** Service A publishes an event; Service B subscribes and processes it asynchronously.

```
Order Service publishes: "OrderPlaced" event
Payment Service subscribes, charges customer
Inventory Service subscribes, reserves items
Notification Service subscribes, sends email
```

**Characteristics:**
- **Loose coupling:** Services don't know about each other
- **Scalable:** Messages queue if consumers are slow
- **Eventual consistency:** Operation completes over time, not immediately
- **Harder to debug:** No direct response; errors are implicit

**When to use:**
- Secondary concerns (notifications, analytics, caching)
- Operations where eventual consistency is acceptable
- Fire-and-forget operations
- Decoupling services that change at different rates

**Technologies:**
- **Message Queues:** RabbitMQ, AWS SQS, Azure Service Bus
- **Event Streaming:** Kafka, AWS Kinesis, Pulsar
- **Pub/Sub:** AWS SNS, Google Cloud Pub/Sub, Redis

**Key difference:**
- **Queues:** Each message consumed by one consumer (work distribution)
- **Pub/Sub:** Each message consumed by all subscribers (event broadcast)
- **Event Streaming:** Ordered, durable, replayable event log (audit trail)

**Real example:** Payment processing

```
Synchronous (bad):
Order Service calls Payment Service → wait for response → proceed
If Payment Service is down, orders can't be placed
If it's slow, order placement is slow

Asynchronous (better):
Order Service publishes "PaymentRequested" event
Payment Service consumes, charges customer
Order Service publishes "PaymentProcessed" event
Other services react as needed
If Payment Service is slow, orders still complete
```

#### Hybrid Approach

**Reality:** Most systems use both patterns.

```
Synchronous:
- User data validation (must be immediate)
- Inventory availability check (must be accurate)
- Payment authorization (must be confirmed)

Asynchronous:
- Confirmation emails (can be sent later)
- Analytics updates (eventual consistency acceptable)
- Cache invalidation (best-effort)
- Audit logging (durability matters, latency doesn't)
```

### Service Discovery

**Problem:** Services need to find each other. Service A needs to call Service B, but Service B might be at any IP address, might be running on multiple servers.

#### Client-Side Discovery

**How it works:**
1. Service B registers its location with a registry (Consul, Eureka)
2. Service A queries the registry to find Service B's location
3. Service A calls Service B directly

**Characteristics:**
- **Clients are aware:** Client code includes discovery logic
- **Flexible routing:** Clients can make routing decisions (prefer cached instances)
- **Tight coupling to registry:** Client must know how to query registry

#### Server-Side Discovery (API Gateway Pattern)

**How it works:**
1. Services register with a registry
2. API Gateway queries the registry and maintains service locations
3. Clients call API Gateway; it routes to appropriate service

**Characteristics:**
- **Clients are unaware:** Client just calls Gateway
- **Gateway is central point:** Becomes bottleneck and single point of failure
- **Decouples clients from discovery:** Clients don't need discovery logic

**In practice:** Most systems use server-side discovery with an API Gateway.

**Tools:**
- **Kubernetes Service Discovery:** Built-in, automatic registration
- **Consul:** Explicit registration, health checks, DNS interface
- **Eureka:** Spring Cloud's service registry
- **API Gateway:** AWS API Gateway, Kong, Traefik

### Data Management in Microservices

#### Database-Per-Service Pattern

**Rule:** Each service owns its database. Other services cannot access it directly.

**Why this matters:**
- **Services are truly independent:** No shared schema dependency
- **Can choose optimal database:** Each service uses the right tool
- **Prevents tight coupling:** Can't share tables across services

**Implementation:**
```
Order Service → PostgreSQL
Payment Service → MySQL
Search Service → Elasticsearch
Analytics Service → Data Warehouse
```

**Problem it creates:** Sharing data between services becomes complex.

#### Data Consistency Challenges

**Scenario:** An order must reserve inventory.

```
Order Service: Create order
Inventory Service: Decrement inventory
```

**Issue:** If Inventory Service fails after Order Service commits, you have an order with no inventory.

**Solutions:**

1. **Synchronous with Compensation (Saga Pattern):**
   - Order Service calls Inventory Service to reserve
   - If payment fails, call Inventory Service to release reservation
   - Essentially manual transaction management

2. **Event Sourcing + CQRS:**
   - Store order as event: "OrderPlaced"
   - Inventory Service subscribes, decrements stock
   - If error, publish "OrderCancelled", Inventory rescinds

3. **Eventual Consistency:**
   - Accept that order and inventory are temporarily inconsistent
   - Reconcile periodically or through compensation logic

4. **Distributed Transactions (not recommended):**
   - Two-phase commit (2PC) across services
   - Synchronous, blocking, high latency
   - Violates service autonomy

**Best practice:** Design for eventual consistency. Compensate for temporary inconsistency.

### Resilience Patterns

#### Circuit Breaker

**Problem:** Service A calls Service B repeatedly, but Service B is failing. A keeps timing out, wasting resources, and retrying.

**Solution:** Circuit Breaker tracks failures. When failure rate exceeds threshold:
- **Open:** Stop calling Service B, fail immediately
- **Half-Open:** Periodically attempt a call to see if Service B recovers
- **Closed:** Resume normal operation

**Benefit:** Prevents cascading failures. Gives Service B time to recover without wasting resources on doomed calls.

**Implementation:** Libraries like Resilience4j, Polly, or Hystrix.

#### Retries and Exponential Backoff

**Problem:** Temporary failures (network hiccup, service overload) shouldn't cause permanent failure.

**Solution:** Retry failed requests with exponential backoff.

```
Attempt 1: immediate
Attempt 2: wait 1 second
Attempt 3: wait 2 seconds
Attempt 4: wait 4 seconds
```

**Caveat:** Only retry idempotent operations. Retrying "charge customer" twice charges twice.

#### Bulkhead Pattern

**Problem:** One slow dependency causes cascading failure across the system.

**Solution:** Isolate resources. If Payment Service is slow, use dedicated thread pools for payment calls—don't starve other operations.

```
Thread pool 1: Payment calls (5 threads)
Thread pool 2: Inventory calls (10 threads)
Thread pool 3: Other calls (20 threads)

If Payment Service is slow, payment calls block their own threads
Other operations continue unaffected
```

#### Timeout and Fail-Fast

**Problem:** Waiting indefinitely for a slow service causes resource exhaustion.

**Solution:** Set explicit timeouts. Fail fast rather than waiting.

```
GET /orders?timeout=5s
If response doesn't arrive in 5 seconds, fail immediately
```

#### Graceful Degradation

**Problem:** Dependent service fails. Primary service must still function.

**Solution:** Provide degraded service.

```
Search Service is down
→ Return cached results or limited results
→ Don't crash the entire application
```

### Observability

#### Distributed Tracing

**Problem:** Request spans multiple services. One service is slow. Which one?

```
Client → Order Service → Payment Service → Bank API
                      → Inventory Service → Warehouse API
```

Without tracing, you don't know where the latency is.

**Solution:** Assign unique trace ID to request, propagate through all calls.

```
Trace ID: abc123
Order Service: received request, processing
Inventory Service: called by Order Service (trace ID: abc123), processing
Payment Service: called by Order Service (trace ID: abc123), slow!
```

**Tools:** OpenTelemetry, Jaeger, Zipkin.

#### Centralized Logging

**Problem:** Logs are scattered across multiple services on multiple servers.

```
Error in Order Service
Look in /var/log on order-service-1, 2, 3
Not there... check payment-service servers
...
```

**Solution:** Centralized log aggregation.

```
All services log to Elasticsearch
Kibana visualizes logs from all services
Search: "ORDER_PAYMENT_FAILED" finds all relevant logs across services
```

**Tools:** ELK Stack, Splunk, Datadog.

#### Metrics and Monitoring

**Problem:** System is slow. Which service is the bottleneck?

**Solution:** Collect metrics from all services.

```
Order Service: 100 requests/sec, 50ms avg latency
Payment Service: 80 requests/sec, 500ms avg latency → bottleneck!
Inventory Service: 150 requests/sec, 10ms avg latency
```

**Tools:** Prometheus, Grafana, Datadog.

### Deployment and Orchestration

#### Containerization

**Benefit:** Each service runs in a container (Docker). Container includes the service and all dependencies. Guaranteed to work on any machine.

#### Orchestration

**Problem:** Deploying, scaling, and monitoring dozens of containers is complex.

**Solution:** Container orchestration platform (Kubernetes).

```
Define service: "Run 3 instances of Order Service"
Kubernetes: Starts 3 containers, manages networking, health checks, restart on failure
Scale to 10: Kubernetes spins up 7 more containers
Service fails: Kubernetes restarts it
```

**Tools:** Kubernetes (de facto standard), Docker Swarm, AWS ECS.

## Trade-offs and Limitations

### Trade-off 1: Operational Complexity vs. Development Simplicity

**Cost of microservices:**
- Multiple deployments to coordinate
- Service discovery and routing
- Monitoring dozens of services
- Debugging across services
- Data consistency management

**Benefit:** Faster development once operational infrastructure is in place.

**Decision:** Microservices are worth it at scale (50+ engineers, complex domain) or when you have strong DevOps culture.

### Trade-off 2: Network Latency vs. Independence

**Cost:** Microservices communicate over the network. Latency is orders of magnitude worse than in-process calls.

```
In-process call:    <1ms
Network call:       5-100ms
Cascading calls:    100-500ms
```

**Solution:**
- Cache results
- Use asynchronous patterns
- Design APIs to minimize round-trips
- Optimize network infrastructure

**Reality:** Network is no longer the bottleneck in most systems. Accept the latency for the independence gains.

### Trade-off 3: Consistency vs. Scalability

**Cost:** Strong ACID consistency is not possible across services. Must design for eventual consistency.

**Benefit:** System is more available and resilient to failures.

**Reality:** Most business processes can tolerate eventual consistency if handled properly (within seconds).

### Trade-off 4: Testing Complexity

**Cost:** Integration tests require running multiple services. Test infrastructure becomes non-trivial.

**Solutions:**
- Unit test services independently
- Use test containers (TestContainers) to spin up dependencies
- Use contract tests to validate service interactions without full integration
- Use end-to-end tests sparingly (they're slow)

## Common Pitfalls and Misuses

### Pitfall 1: Too Many Services Too Soon

**Mistake:** Every small piece of logic becomes a microservice. 100 services for a business that's 20 services large.

**Result:** Coordination overhead exceeds the benefit. Deployments become slow. Integration testing is painful.

**Better approach:** Right-size services. A service should have:
- Clear business capability
- Enough logic to justify operational overhead
- Boundaries that are unlikely to change

### Pitfall 2: Technical Microservices

**Mistake:** Services organized by technical function (`UserService`, `AuthService`, `LoggingService`).

**Result:** Services are not independently deployable. Changes cascade. You haven't actually decomposed the problem.

**Better approach:** Services = business capabilities. Auth is infrastructure, shared library, or part of API Gateway.

### Pitfall 3: Insufficient Observability

**Mistake:** Deploying microservices without logging, tracing, or metrics.

**Result:** Debugging is impossible. You deploy something, it breaks, and you have no way to find out why.

**Cost:** Observability infrastructure (logging, tracing, metrics) is essential. Budget for it.

### Pitfall 4: Ignoring Data Consistency

**Mistake:** Designing services without addressing how they share data.

**Result:** Orphaned orders (order created, payment failed, order is in inconsistent state).

**Better approach:** Explicitly design consistency model. Use sagas or event-driven patterns.

### Pitfall 5: Synchronous Everything

**Mistake:** Every service call is synchronous. Service A waits for B, which waits for C, which waits for D...

**Result:** Cascading failures. If any service is slow, the entire chain is slow. P99 latency suffers.

**Better approach:** Use asynchronous patterns where appropriate. Only synchronous for critical path.

### Pitfall 6: No Service Isolation

**Mistake:** Services share the same database, libraries, or code.

**Result:** Services are not independent. Changing one breaks others. You've just reorganized the monolith, not decomposed it.

**Better approach:** Each service owns its data and code. Integration through APIs.

## Key Takeaways

1. **Microservices are an answer to organizational and operational complexity, not technical complexity.** Use them when team size, domain complexity, or scaling requirements demand decomposition.

2. **Service boundaries are the critical decision.** Get them wrong and you've created a distributed monolith. Domain-driven decomposition usually works best.

3. **Database-per-service is non-negotiable.** Services that share databases are not truly independent.

4. **Asynchronous patterns are your friend.** Synchronous service chains are fragile. Design for loose coupling through events.

5. **Resilience patterns are essential.** Circuit breakers, retries, timeouts, and bulkheads prevent cascading failures.

6. **Observability is not optional.** Distributed systems are impossible to debug without excellent logging, tracing, and metrics.

7. **Eventual consistency requires careful design.** ACID transactions are not possible across services. Use sagas, events, or idempotency to manage consistency.

8. **Operational infrastructure is a prerequisite.** Container orchestration, service discovery, monitoring, and deployment automation are table-stakes.

9. **Conway's Law works for you or against you.** If team structure matches service boundaries, development flows. If not, you have constant friction.

10. **Start with a monolith, extract services when needed.** Premature decomposition is expensive. Extract when pain points emerge.

## Related Concepts

- Bounded Contexts (DDD)
- Event-Driven Architecture
- API Gateway Pattern
- Saga Pattern for distributed transactions
- CQRS (Command Query Responsibility Segregation)
- Service Mesh (Istio, Linkerd) for managing service communication
- Container Orchestration (Kubernetes)
