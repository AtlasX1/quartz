# Domain-Driven Design (DDD)

## Concept Overview

Domain-Driven Design is an architectural approach that prioritizes modeling the business domain as the central concern of software design. Rather than organizing code around technical layers (controllers, services, repositories), DDD organizes it around **business concepts and their relationships**.

The core insight: **The most critical complexity in enterprise systems is business complexity, not technical complexity.** DDD addresses this by creating a shared vocabulary between engineers and domain experts, then structuring code to reflect that vocabulary.

At its essence, DDD is about **making domain logic discoverable and the system's boundaries explicit**. It transforms the codebase into a conversation between technical and business stakeholders rather than a technical artifact disconnected from business reality.

## Problems These Principles Solve

### 1. **Disconnect Between Code and Business Reality**
Business stakeholders and engineers speak different languages. A business person talks about "orders," "fulfillment," "disputes," while engineers discuss "objects," "tables," "services." This gap creates misunderstandings, requirements that don't translate to code, and features that don't solve actual business problems.

### 2. **Scattered Business Logic**
Core business rules are scattered across controllers, services, repositories, and utilities. Understanding what an "order" really is requires reading code in ten different files. Changing a business rule means hunting through the codebase.

### 3. **Anemic Data Models**
Objects become mere data containers without behavior. An `Order` is just a collection of fields; the logic that defines what an order is (validation rules, state transitions, calculations) lives elsewhere. This violates encapsulation and makes the domain model useless for understanding the business.

### 4. **Scaling Complexity**
As systems grow, adding features becomes slower. Developers must understand the entire system to make changes safely. Large teams step on each other because there's no clear responsibility boundaries. The codebase becomes a monolithic knowledge silo.

### 5. **Database-Driven Design**
Architecture is inverted: the database schema (foreign keys, normalization) drives the code structure. Business concepts are distorted to fit database constraints. Adding new aggregates or changing data ownership requires architectural changes.

### 6. **Unclear Integration Points**
When systems integrate, it's unclear which parts should integrate. Are you sharing database tables (coupling)? Raw domain models (coupling)? There's no principled way to decide. Systems become tightly bound.

## Why These Problems Exist

**The Root Cause:** Without a systematic way to model domains, engineers fall back on technical architecture as the organizing principle. They create layers (MVC, three-tier, layered architecture) that are comfortable and familiar but don't express business structure.

- **Business domains are complex:** Understanding what an "order" means requires understanding inventory, fulfillment, returns, disputes, pricing, taxes, and a hundred other things. This complexity has nowhere to live in a technical architecture.

- **Business rules change frequently:** Requirements like "apply discount if customer is bulk buyer" start simple but evolve into complex conditional logic. Without a clear home for these rules, they scatter across the codebase.

- **Integration is chaotic:** When two systems integrate, you don't know which concepts are truly shared and which are just coincidentally named the same. An "Order" in billing means something different from an "Order" in fulfillment.

- **Knowledge is oral:** Business context lives in conversations, emails, and wiki pages rather than the code. New team members can't learn the domain from the codebase alone.

- **Refactoring is risky:** Without clear domain boundaries, changing business logic risks affecting unexpected parts of the system.

DDD addresses these by making the domain the centerpiece of architecture and encoding business understanding directly into code.

## Common Solutions / Approaches

### Ubiquitous Language

**What it is:** A shared vocabulary used consistently by both domain experts and engineers. Terms from this language appear in code, conversations, and documentation.

**Why it matters:** Language is how humans think. If engineers and business people use different terms for the same concept, they'll have different mental models. A "customer" might be a person who buys something or an organization that has a contract.

**Establishing it:**
- Hold domain workshops with experts and engineers together
- Create a glossary of key terms with precise definitions
- Challenge ambiguous terms. If people explain it differently, it's not truly ubiquitous
- Encode it in code. Class names, method names, variable names should all use ubiquitous language

**Real example:** An e-commerce platform might establish:
- A "Product" is something you can purchase
- An "Item" is a product instance in inventory
- An "OrderLineItem" is a product in a customer's order
- A "SKU" is the unique identifier for a product variant

Using these precisely throughout code and documentation prevents confusion.

**Architectural benefit:** When code uses the same language as the business, developers can discuss logic with business people without translation. "Why does the code do X?" "Because the business rule says order adjustments can't exceed 10% of subtotal."

### Entities and Value Objects

**Entities:** Objects with identity. Two orders with the same data are different if they have different order IDs. Entities have a lifecycle (created, updated, shipped, archived). Their identity is constant even if their properties change.

**Real problem solved:** Without distinguishing entities from values, code treats all objects the same. An `Address` object might be an entity with an ID (because someone's address is tracked through time) or a value object (because addresses are compared by content, not identity). This confusion leads to design mistakes.

**Value Objects:** Objects compared by their contents, not identity. Two `Money(100, "USD")` objects are identical regardless of which object they are. Value objects are immutable. They have no identity beyond their values.

**Architectural implications:**
- **Entities need IDs** that persist through their lifecycle
- **Entities should have encapsulated behavior** reflecting business rules about what can be done to them
- **Value objects should be immutable** to prevent unexpected side effects
- **Value objects should have sensible equality** based on their values, not object reference

**Real example:** In an order system:
- `Order` is an entity (has unique ID, lifecycle, mutable state)
- `OrderLineItem` is an entity (has ID, can be modified or removed)
- `Money` is a value object (immutable, compared by amount and currency)
- `Address` might be a value object (compared by contents, not identity) or an entity (if you track address changes through time—depends on business requirements)

### Aggregates and Aggregate Roots

**What it is:** A cluster of related entities and value objects that change together. An aggregate has a single entry point (the aggregate root).

**Why it matters:** In order-heavy systems, customers, orders, line items, and shipments are all related. But they don't all change together. Customers are independent. Orders depend on customers. Line items depend on orders. Shipments depend on orders.

An aggregate bundles the parts that genuinely must change together:
- An `Order` (aggregate root) contains `OrderLineItems` and `OrderNotes`
- These are tightly bound: you can't modify a line item without the order's knowledge
- But `Customer`, `Inventory`, and `Shipment` are separate aggregates
- `Order` maintains an ID reference to `Customer`, but doesn't own or encapsulate it

**Rules for aggregates:**
- Each aggregate has one root entity (the aggregate root)
- External references must point to the aggregate root, not internal entities
- Changes to the aggregate are atomic (all succeed or all fail)
- The aggregate enforces its own invariants (business rules about valid states)
- Only the aggregate root can be queried directly from repositories

**Architectural benefit:** Aggregates are the unit of transaction, consistency, and change. They make clear which parts of the system must be consistent and which can be eventually consistent.

**Real problem solved:** Without aggregates, you might create a massive `Customer` aggregate that includes orders, payments, reviews, preferences, and support tickets. Changing preferences would lock the entire customer object, creating contention. Aggregates force you to ask: "What must change together?" and "What can be independent?"

**Example aggregate design:**
```
Customer (aggregate root)
  - customerId
  - name, email, address
  - methods: updateProfile(), changeAddress()

Order (separate aggregate root)
  - orderId
  - customerId (reference, not ownership)
  - OrderLineItems (contained)
    - lineItemId
    - productId (reference)
    - quantity, price
  - methods: addLineItem(), removeLineItem(), place()

Shipment (separate aggregate root)
  - shipmentId
  - orderId (reference)
  - items, address
```

Each aggregate is independently loaded, modified, and persisted.

### Repositories

**What it is:** An abstraction over data storage that presents collections of aggregates as if they were in-memory collections.

**Why it exists:** Data access is a cross-cutting concern that shouldn't leak into business logic. A `Repository<Order>` hides whether orders are stored in PostgreSQL, MongoDB, Redis, or a service API.

**Distinction from Data Access Objects (DAOs):** Repositories work with aggregates (business concepts), while DAOs work with individual tables (technical concepts). A repository might combine data from multiple tables to reconstruct an aggregate.

**Architectural principle:** Every aggregate root should have a corresponding repository. You don't create repositories for value objects (they're accessed through the aggregate root). You don't create repositories for non-root entities (they're accessed through the aggregate).

**Repository responsibilities:**
- Load aggregates by ID
- Save aggregates (insert or update)
- Query for aggregates by business criteria
- Delete aggregates
- Abstract storage technology completely

**Why this matters:** The domain model remains pure—it contains only business logic. Data access concerns are isolated in repositories. You can test domain logic without touching databases.

### Domain Services

**What it is:** Business logic that naturally belongs with entities and value objects but involves multiple aggregates.

**Real problem:** An order must be placed, but placing an order involves:
- Validating the order structure (belongs in `Order` aggregate)
- Charging the customer (requires `Order` and `Customer` aggregates)
- Reserving inventory (requires `Order` and `Inventory` aggregates)

Where does this orchestration logic live? It doesn't fit in any one aggregate.

**Solution:** Create a `PlaceOrderService` domain service that:
- Takes order, customer, and inventory repositories
- Validates the order
- Charges the customer
- Reserves inventory
- Raises an event that the order was placed

**Important distinction:** Domain services are about business logic orchestration, not technical concerns like logging or database access. Those belong in application services or cross-cutting infrastructure.

**Architectural benefit:** Domain services keep aggregates focused. Orders know about order rules; customers know about customer rules; the `PlaceOrderService` orchestrates their interaction.

### Bounded Contexts

**What it is:** An explicit boundary within which a domain model is valid. Different parts of a large system have different domain models.

**Real problem:** "Customer" means different things in different contexts:
- In **Billing context**: a customer has billing address, credit card, payment history, tax ID
- In **Support context**: a customer has name, email, support tickets, satisfaction score
- In **Analytics context**: a customer is a set of behavioral events

If you try to create one universal `Customer` class that includes everything, it becomes bloated, confusing, and coupled to unrelated concerns.

**Solution:** Define explicit bounded contexts:
- `BillingContext` has its own `Customer` model
- `SupportContext` has its own `Customer` model
- `AnalyticsContext` doesn't model customers directly—it models events
- Each context has its own ubiquitous language
- Models are translated at integration points

**Architectural benefit:** Contexts can evolve independently. Teams own separate contexts. Complexity is divide-and-conquered.

### Context Mapping and Integration

**The challenge:** Different bounded contexts must integrate. How do they interact?

**Common patterns:**

1. **Shared Kernel:** Billing and Payments contexts share a common `Money` value object and `Currency` enum. They're defined in a shared library.
   - **Trade-off:** Creates coupling between contexts. Changes to the kernel affect both.

2. **Anticorruption Layer:** One context provides an unstable API. The other context creates an adapter that translates its own domain model to the external model.
   - **Use when:** You're integrating with legacy systems or external APIs you don't control.

3. **Published Language:** One context publishes events or a stable API contract. Others consume it.
   - **Use when:** One context is upstream (auth, payment) and others depend on it.

4. **Open Host Service / Published Interface:** A context explicitly designs an API for other contexts to consume.
   - **Difference from published language:** More formal, versioned, documented.

5. **Separate Ways:** Two contexts don't integrate. Each solves its domain independently.
   - **Use when:** Contexts have no meaningful interaction or integration cost exceeds benefit.

**Real example:** An e-commerce platform:
- **Billing context** publishes "Payment Processed" events
- **Order context** consumes these events and updates order status
- **Inventory context** publishes "Item Reserved" events
- **Shipping context** consumes inventory events and creates shipments
- Each context has its own data model; they communicate through events, not direct coupling

## Trade-offs and Limitations

### Trade-off 1: Domain Modeling Requires Domain Knowledge

**The cost:** To design a good domain model, you must understand the domain deeply. This requires ongoing collaboration with domain experts, which is time-consuming and may not always be available.

**When to invest:** For systems where the domain is the primary source of complexity and change (most business systems). Not necessary for simple CRUD applications or infrastructure tools.

**Mitigation:** Build domain expertise incrementally. Start with basic contexts and refine as understanding grows.

### Trade-off 2: Multiple Models Increase Development Complexity

**The cost:** Different bounded contexts maintain different models for the same concept. This adds complexity—translation logic, potential inconsistency, more cognitive load.

**Reality:** In large systems, this is unavoidable. Different contexts genuinely need different models. The question is whether to acknowledge it (DDD's approach) or pretend one universal model works (creates confusion).

### Trade-off 3: Aggregate Consistency vs. Scalability

**The issue:** DDD aggregates are consistent within themselves but may be eventually consistent with other aggregates. This creates temporary inconsistencies. For example, an order reserves inventory, but there's a window where the inventory isn't actually reserved if the system crashes.

**Solutions:**
- Use transactions within aggregates (strong consistency)
- Publish events from aggregates and use sagas to coordinate across aggregates (eventual consistency)
- Decide consistency requirements per interaction

**Key principle:** Not everything needs strong consistency. Most business processes can tolerate eventual consistency if handled properly.

### Trade-off 4: ORM Impedance Mismatch

**The problem:** Object-oriented domain models and relational databases think differently. Aggregates with complex object graphs don't map cleanly to normalized tables. You end up with complex ORM configuration or abandoning encapsulation.

**Solutions:**
- Use document databases that map more naturally to aggregates (MongoDB, DynamoDB)
- Accept some impedance mismatch and manage it explicitly
- Use event sourcing to store aggregates as event streams rather than snapshots
- Create intentional mapping layers between domain model and persistence model

## Common Pitfalls and Misuses

### Pitfall 1: Anemic Domain Models in DDD Clothing

**Mistake:** Creating classes that look like DDD entities but are still just data containers. Logic remains in services. The domain model doesn't encode business rules.

**Symptom:** Services with names like `OrderValidationService`, `OrderCalculationService`, `OrderProcessingService`. All the logic is in services; entities are passive.

**Better approach:** Entities should encapsulate behavior. An `Order` should have a `place()` method that validates state, applies rules, and raises events. The service orchestrates, but doesn't contain logic.

### Pitfall 2: Creating Aggregates That Are Too Large

**Mistake:** Including everything related to a customer in the `Customer` aggregate: orders, addresses, preferences, payment methods, support tickets, preferences.

**Result:** Bottlenecks. Loading a customer means loading everything. Modifying preferences locks the entire customer. Scaling becomes difficult.

**Better approach:** Ask "What must be consistent?" Customer preferences might be a separate aggregate that references customer ID.

### Pitfall 3: Creating Aggregates That Are Too Small

**Mistake:** Every entity is its own aggregate. `Order` is an aggregate with ID, but `OrderLineItem` is also an aggregate with ID.

**Result:** Aggregates should have clear boundaries. Queries become complex. Transaction management is confusing.

**Better approach:** If line items are always accessed through orders, never independently, they're not aggregates—they're entities within the `Order` aggregate.

### Pitfall 4: Applying DDD to Everything

**Mistake:** Designing every part of a system with DDD rigor. Building ubiquitous language for the admin interface, report generation, or batch processing.

**Reality:** DDD is expensive. It pays off in domains with complex business logic. Use it there. For CRUD-heavy, reporting-heavy, or simple utility parts of the system, lighter approaches work fine.

### Pitfall 5: Ignoring Domain Experts

**Mistake:** Engineers design the domain model based on their understanding of "good architecture" without consulting domain experts.

**Result:** The model doesn't match business thinking. Changes are needed, but engineers resist because "it would require refactoring." The system becomes disconnected from reality.

**Better approach:** Domain experts and engineers should design together. Architects should facilitate this collaboration.

### Pitfall 6: Over-Formalizing Everything

**Mistake:** Creating elaborate event storming sessions, context maps, and documentation for every interaction. DDD becomes ceremony instead of clarity.

**Better approach:** Use DDD tools (event storming, context mapping) when they provide clarity. Skip the ceremony when everything is already clear.

## Key Takeaways

1. **DDD centers on the business domain as the primary complexity.** Code structure should reflect business structure, not technical layers.

2. **Ubiquitous language is essential.** If business people and engineers speak different languages, you'll build the wrong thing.

3. **Entities and value objects are different.** Entities have identity and lifecycle; value objects are immutable and compared by content.

4. **Aggregates are units of consistency and change.** Design them around business transactionality, not database normalization.

5. **Repositories abstract data access.** Domain logic never touches database concerns.

6. **Domain services orchestrate across aggregates.** But don't make them god objects.

7. **Bounded contexts acknowledge that one model can't fit all.** Different parts of large systems have different models.

8. **Context mapping is where integration happens.** Design explicitly how contexts interact to avoid accidental coupling.

9. **DDD is most valuable for domains with complex business logic.** For simple CRUD applications, it's overhead. Know when to apply it.

10. **Aggregate design is difficult.** There's no algorithm. It requires understanding the business deeply and iterating with domain experts.

## Related Concepts

- Event Sourcing (natural way to represent aggregate changes)
- CQRS (separates commands that change state from queries that read it)
- Sagas (coordinate transactions across aggregates)
- Event-Driven Architecture (contexts communicate through events)
- Microservices (each service often maps to bounded contexts)
