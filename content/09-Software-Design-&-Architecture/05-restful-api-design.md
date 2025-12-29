# RESTful API Design

## Concept Overview

REST (Representational State Transfer) is an architectural style for designing distributed systems, particularly web services. It defines how clients and servers communicate through HTTP using a standardized set of principles. REST APIs have become the de facto standard for web service integration because they leverage existing HTTP infrastructure effectively and provide clear, predictable contracts.

Unlike application-specific protocols, REST deliberately constrains design decisions to achieve stateless, cacheable, and independently deployable systems. **The goal is to create APIs that are intuitive to use, maintainable over time, easy to version, and resilient to evolution.**

## Problems This Concept Solves

### 1. **Unclear API Contracts**
Without standards, each API defines its own conventions for success, errors, pagination, and filtering. Clients must learn a new pattern for each API. Integration becomes error-prone.

### 2. **Tight Coupling Between Client and Server**
APIs designed without clear separation of concerns entangle client code with implementation details. Changing the server breaks all clients. No independent evolution is possible.

### 3. **Inefficient Communication**
Custom protocols often over-fetch or under-fetch data. A client needs three pieces of information but must make five round-trips, or gets ten pieces of data it doesn't need.

### 4. **Inability to Cache**
Caching requires understanding which operations are safe, idempotent, and idempotent. Without standard semantics, caching becomes impossible or inconsistent.

### 5. **Scaling and Intermediary Support**
Proxies, load balancers, CDNs, and caches are built on HTTP semantics. APIs that violate HTTP semantics can't leverage these intermediaries effectively.

### 6. **Versioning Chaos**
Without clear versioning strategies, supporting multiple API versions becomes a maintenance nightmare. Clients are locked to specific versions; you can't evolve without breaking someone.

### 7. **Error Handling Inconsistency**
Each API invents its own error format. Is the error code a number or string? Does it have details? Is it wrapped in a response envelope? Clients must parse each differently.

## Why These Problems Exist

**The Core Issue:** In the absence of constraints, systems naturally tend toward the path of least immediate resistance. Engineers optimize for their current use case without considering:

- **Future compatibility:** "We'll never need to version this API"
- **Other clients:** "The web frontend is the only client"
- **Operational reality:** "Caching isn't important for us"
- **Long-term evolution:** "This design is final"

REST constrains these decisions intentionally:
- **Statelessness** ensures any server instance can handle any request (enables scaling)
- **Resource orientation** creates a uniform interface (enables caching, proxying)
- **Standard methods** make semantics explicit (enables intermediaries to understand intent)
- **Explicit status codes** document outcomes consistently (enables client logic)
- **Hypermedia** enables evolution without breaking clients (enables long-term stability)

Without these constraints, systems drift toward:
- RPC-style APIs where the URL is meaningless and all logic is in method calls
- Stateful protocols that require session management
- Custom, undocumented behavior that surprises clients

## Common Solutions / Approaches

### Core REST Principles

#### Resource Orientation

**Principle:** Everything is a resource identified by a URL.

**What it means:**
- `/orders` represents the collection of orders
- `/orders/123` represents order 123
- `/orders/123/items` represents items in order 123
- `/customers/456/orders` represents orders by customer 456

**Anti-pattern:** RPC style
```
/createOrder
/updateOrder
/deleteOrder
/getOrderItems
```

This obscures that you're operating on resources. It treats every operation as a unique method call.

**Why it matters:** URLs become predictable. A client can guess `/orders/456` exists. Browsers cache based on URL. Proxies can route intelligently.

**Real-world problem solved:** Without resource orientation, clients must memorize the URL structure for every API. With it, structure is discoverable and intuitive.

#### HTTP Methods as Semantic Operations

**GET:** Retrieves a resource. Safe (doesn't modify state) and idempotent (multiple calls have the same effect).

**POST:** Creates a new resource or triggers an action. Not safe, not idempotent (each call might create a new resource).

**PUT:** Replaces a resource entirely. Idempotent (replace with same data multiple times = same result).

**PATCH:** Partially updates a resource. May or may not be idempotent (depends on the patch operation).

**DELETE:** Removes a resource. Idempotent (deleting twice has the same effect as once).

**Important distinction:**

```
POST /orders → Creates new order (123, 124, 125 on retry)
PUT /orders/123 → Replaces order 123 (always same state)
PATCH /orders/123 → Updates order 123 fields
GET /orders/123 → Retrieves order 123 (no side effects)
DELETE /orders/123 → Removes order 123 (idempotent)
```

**Why it matters:** Intermediaries (proxies, caches, routers) understand semantics.

- Caches can cache GET but not POST
- Load balancers can retry idempotent operations on timeout
- Browsers can warn before submitting POST (state change) but not GET

**Real-world problem solved:** Custom HTTP methods or using POST for everything loses semantic information. Intermediaries don't know whether an operation is safe or idempotent, so they can't optimize.

#### Statelessness

**Principle:** The server should not store client context. Each request contains everything needed to understand and process it.

**What it means:**
- No session state on the server (or if there is, it's short-lived and cache-like)
- Authentication credentials are sent with each request (JWT in header, not cookie)
- Client doesn't depend on server remembering previous requests

**Bad approach:**
```
GET /authenticate?username=alice&password=secret
→ Server creates session, returns session ID

GET /orders
→ Server looks up session, retrieves client context, returns orders
```

Session ID is a form of server state. If the session expires or the server crashes, the client is confused.

**Good approach:**
```
GET /orders
Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
→ Server decodes JWT, identifies client, returns orders
```

Each request contains authentication. Server doesn't store state.

**Why it matters:**
- Any server instance can handle any request (scale horizontally)
- No session synchronization across server replicas
- Stateless servers are simple to reason about
- Clients control their context

**Real-world problem solved:** Stateful servers are bottlenecks. If requests must route to the specific server that holds the session, you can't load-balance freely.

#### Standard Status Codes

**2xx (Success):**
- **200 OK:** Request succeeded, response body contains result
- **201 Created:** Resource was created; response includes new resource or Location header
- **204 No Content:** Request succeeded, but no content to return
- **202 Accepted:** Request accepted for processing, but not yet complete

**3xx (Redirection):**
- **301 Moved Permanently:** Resource moved; clients should update their URLs
- **304 Not Modified:** Client has cached version; no need to send full response

**4xx (Client Error):**
- **400 Bad Request:** Request malformed (invalid JSON, missing fields)
- **401 Unauthorized:** Authentication required or failed
- **403 Forbidden:** Authenticated but not authorized for this resource
- **404 Not Found:** Resource doesn't exist
- **409 Conflict:** Request conflicts with current state (e.g., concurrent update)
- **422 Unprocessable Entity:** Request well-formed but business logic rejects it
- **429 Too Many Requests:** Rate limiting—client exceeded quota

**5xx (Server Error):**
- **500 Internal Server Error:** Unexpected server error
- **503 Service Unavailable:** Server temporarily down

**Why it matters:** Status codes are machine-readable. Clients can:
- Implement retry logic (5xx are retryable, 4xx are not)
- Cache appropriately (200 is cacheable, 404 is cacheable, 500 is not)
- Show appropriate UI (403 → "Access Denied", 404 → "Not Found")

**Real-world problem solved:** Custom status codes (always 200, with error details in body) prevent intermediaries from understanding response semantics.

#### Idempotency

**Principle:** Operations should be designed so that repeating them produces the same result as executing once.

**Idempotent operations:**
- `PUT /orders/123 {status: "shipped"}` → Multiple executions = same state
- `DELETE /orders/123` → Multiple deletes = resource is gone (same end state)
- `GET /orders/123` → Multiple reads = same data

**Non-idempotent operations:**
- `POST /orders` → Each execution creates a new order
- `PATCH /orders/123 {increment_count: 1}` → Each execution increments

**How to make operations idempotent:**
1. Use idempotency keys: client generates unique ID for operation, server caches result
2. Replace instead of modify: use PUT instead of PATCH when possible
3. Design state transitions carefully: if state allows only one transition, operation is idempotent

**Real-world example:**
```
Request: POST /orders {items: [...], amount: 100}
Response: 201 Created {id: 123}
Network timeout, client retries
Request: POST /orders {items: [...], amount: 100}
→ Server has idempotency key, returns same 201 {id: 123}
→ Order 123 is not duplicated
```

**Why it matters:** Networks fail. Timeouts occur. Without idempotency, retrying is dangerous (might duplicate the operation). With idempotency, retry is safe.

#### Pagination, Filtering, and Sorting

**Pagination:** Handle large datasets.

```
GET /orders?page=2&limit=20
→ Returns items 20-40

GET /orders?cursor=abc123&limit=20
→ Cursor-based pagination (more robust for updates)
```

**Why cursor-based > offset-based:** If items are added/removed between requests, offset-based pagination has gaps or duplicates. Cursors are stable.

**Filtering:** Narrow results.

```
GET /orders?status=shipped&customer=123
→ Returns orders for customer 123 with status "shipped"
```

**Sorting:**

```
GET /orders?sort=-date,+amount
→ Sort by date descending, then amount ascending
```

**Real problem solved:** Without standard pagination, large datasets cause:
- Timeouts (returning 1 million records)
- Memory issues (client and server)
- Slow responses
- Inability to build responsive UIs

#### Versioning

**The problem:** APIs evolve. Clients depend on specific versions. You need to support multiple versions simultaneously.

**Common approaches:**

**URL-based versioning:**
```
GET /v1/orders
GET /v2/orders
```

**Header-based versioning:**
```
GET /orders
Header: Accept: application/vnd.myapi.v2+json
```

**Parameter-based versioning:**
```
GET /orders?version=2
```

**Semantic versioning guidance:**
- **Major version change:** Breaking changes (field removed, contract changed)
- **Minor version change:** Additive changes (new optional field)
- **Patch version change:** Bug fixes

**Best practice:** Support multiple major versions simultaneously, retire after deprecation period.

**Real problem solved:** Without versioning, adding a single required field breaks all existing clients. Versioning lets clients and server evolve independently.

#### Error Responses

**Standard error format:**

```json
{
  "code": "INVALID_ORDER_STATE",
  "message": "Order cannot be shipped when status is pending",
  "details": {
    "current_status": "pending",
    "valid_transitions": ["processing", "cancelled"]
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "request_id": "req_abc123"
}
```

**Why each field matters:**
- **code:** Machine-readable error identifier (enables error handling logic)
- **message:** Human-readable explanation
- **details:** Additional context for specific errors
- **timestamp:** When error occurred (debugging distributed systems)
- **request_id:** Trace requests through logs

**Real problem solved:** Generic error messages prevent clients from distinguishing different failure modes. `"ERROR"` could mean network error, validation error, or server error—each requires different handling.

#### Content Negotiation

**Principle:** Support multiple response formats based on client preference.

```
GET /orders/123
Header: Accept: application/json
→ Returns JSON

GET /orders/123
Header: Accept: application/xml
→ Returns XML

GET /orders/123
Header: Accept: text/csv
→ Returns CSV
```

**Real-world benefit:** Different clients have different needs. A web client wants JSON. A legacy system needs XML. A reporting tool wants CSV. One endpoint can serve all.

#### HATEOAS (Hypermedia As The Engine Of Application State)

**Principle:** Responses include links to related resources and actions.

```json
{
  "id": 123,
  "status": "processing",
  "amount": 100,
  "_links": {
    "self": { "href": "/orders/123" },
    "cancel": { "href": "/orders/123", "method": "DELETE" },
    "ship": { "href": "/orders/123", "method": "PATCH", "body": {"status": "shipped"} },
    "customer": { "href": "/customers/456" },
    "items": { "href": "/orders/123/items" }
  }
}
```

**Why it matters:** Clients don't need to hardcode API structure. They follow links. This enables:
- API evolution without breaking clients (new links can be added)
- Discovery (clients see available actions)
- State management (server indicates which actions are valid)

**Maturity level:** HATEOAS is the highest level of REST maturity. Most "REST" APIs are level 2 (resource-oriented with HTTP methods). Full HATEOAS is less common because it's more complex to implement.

## Trade-offs and Limitations

### Trade-off 1: REST Verbosity vs. Efficiency

**Cost:** REST over HTTP can be chatty.

```
GET /orders/123          → JSON response with order summary
GET /orders/123/items    → JSON response with items
GET /orders/123/customer → JSON response with customer
```

**Result:** Three round-trips for data that could be fetched once.

**Solutions:**
1. **JSON response includes related data:** `/orders/123?include=items,customer`
2. **GraphQL:** Query exactly what you need in one request
3. **Accept over-fetching:** JSON is small; network is fast; simplicity wins

**Reality:** For most applications, REST's simplicity outweighs minor inefficiency.

### Trade-off 2: Cacheability vs. Freshness

**Cost:** Caching assumes resources are stable. If orders change frequently, caching is wrong.

```
GET /orders/123
Header: Cache-Control: max-age=3600
→ CDN serves cached version for 1 hour
→ User sees stale data
```

**Solutions:**
1. **Short cache TTLs:** Cache for minutes, not hours
2. **Invalidation:** When resource changes, invalidate cache
3. **Cache-busting:** Clients explicitly request fresh data
4. **Don't cache:** For mutable resources, set Cache-Control: no-cache

**Reality:** Caching is most valuable for read-heavy APIs. For write-heavy or real-time data, benefits are marginal.

### Trade-off 3: Statelessness vs. Session Efficiency

**Cost:** Without session state, every request requires authentication validation.

```
Stateful:  Auth once, reuse session
Stateless: Validate JWT/token every request
```

**Result:** Stateless authentication has overhead—cryptographic validation, database lookups.

**Solutions:**
1. **JWT with signature validation:** Fast, no database lookup
2. **Caching:** Cache authentication results
3. **Accept the cost:** In modern systems, validation is cheap relative to other operations

**Reality:** Statelessness enables horizontal scaling, which outweighs authentication overhead.

### Trade-off 4: Standard Methods vs. Semantic Precision

**Issue:** Some operations don't fit neatly into GET/POST/PUT/PATCH/DELETE.

**Example:** "Activate an order" doesn't fit standard methods.

**Solutions:**
1. **Use POST:** `POST /orders/123/activate`
2. **Use PATCH:** `PATCH /orders/123 {status: "active"}`
3. **Use custom header:** `Header: X-Action: activate`

**Best practice:** Favor PATCH over POST for actions. Keep custom methods rare.

## Common Pitfalls and Misuses

### Pitfall 1: Using POST for Everything

**Mistake:**
```
POST /getOrders {customer_id: 123}
POST /deleteOrder {id: 123}
POST /updateOrder {id: 123, status: "shipped"}
```

**Why it's wrong:**
- GET /orders?customer=123 is clearer
- DELETE /orders/123 is more semantically correct
- PUT /orders/123 or PATCH /orders/123 is better

**Cost:** Lose caching, idempotency guarantees, semantics.

### Pitfall 2: Inconsistent Error Responses

**Mistake:**
```
Error 1: {"error": "Invalid request"}
Error 2: {"message": "Order not found"}
Error 3: [{"field": "amount", "error": "required"}]
```

**Cost:** Client must parse different formats. Error handling becomes complex.

**Better approach:** Standardize all errors.

### Pitfall 3: Ignoring Idempotency

**Mistake:** Design POST endpoint that isn't idempotent.

```
POST /orders {items: [...]}
→ Multiple POSTs create multiple orders
```

**Problem:** Network timeouts cause duplicates.

**Better approach:**
```
POST /orders with idempotency key header
Server: Remember idempotency key, return cached response
Multiple requests with same key → Same order ID
```

### Pitfall 4: Exposing Database Structure

**Mistake:**
```
GET /orders/123
→ {id: 123, created_at, updated_at, deleted_at, internal_status, ...}
```

**Problems:**
- Clients depend on internal fields
- Can't refactor database without breaking API
- Over-exposure of implementation details

**Better approach:** Only expose business-relevant fields in API responses.

### Pitfall 5: No Rate Limiting

**Mistake:** Allowing unlimited requests.

**Problems:**
- DoS attacks
- Runaway clients (bugs causing request loops)
- Resource exhaustion

**Better approach:**
```
Header: X-RateLimit-Limit: 1000
Header: X-RateLimit-Remaining: 999
Header: X-RateLimit-Reset: 1234567890
→ Client understands quotas, can back off intelligently
```

### Pitfall 6: Ignoring Security

**Mistake:**
```
GET /orders (no authentication)
→ Returns all orders in the system
```

**Problems:**
- Data exposure
- Privacy violations
- Compliance issues

**Better approach:**
- Authenticate every request
- Authorize based on user context
- Return only resources user can access

## Key Takeaways

1. **REST is about constraints that enable scalability and maintainability.** Resource orientation, statelessness, and standard methods aren't arbitrary—they solve real problems.

2. **HTTP semantics matter.** Using correct status codes, methods, and headers enables intermediaries (caches, proxies, CDNs) to optimize automatically.

3. **Idempotency is essential in distributed systems.** Network failures happen. Retrying must be safe. Design operations to be idempotent.

4. **Versioning is inevitable.** Plan for it from the start. Support multiple versions simultaneously. Deprecate gracefully.

5. **Error responses must be standardized.** Different errors require different handling. Provide code, message, and details.

6. **Pagination prevents disasters.** Always paginate large datasets. Use cursor-based pagination for robustness.

7. **HATEOAS is optional but powerful.** Most APIs don't need it. But full hypermedia enables long-term evolution without breaking clients.

8. **Caching multiplies efficiency.** Properly cacheable APIs leverage CDNs and proxies. Improper caching causes subtle inconsistencies.

9. **REST is a guideline, not a law.** There are legitimate reasons to deviate. But understand what you're sacrificing.

10. **Test your API like you test code.** Document contracts. Test edge cases. Version carefully. Deprecate mindfully.

## Related Concepts

- GraphQL (alternative query language for APIs)
- gRPC (binary protocol for high-performance services)
- OpenAPI/Swagger (documentation and code generation)
- API Gateway Pattern (centralized API management)
- Rate Limiting and Quota Management
- Security (authentication, authorization, HTTPS)
