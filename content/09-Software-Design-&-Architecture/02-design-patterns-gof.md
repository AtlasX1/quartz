# Design Patterns (Gang of Four)

## Concept Overview

Gang of Four (GoF) Design Patterns represent a cataloged collection of 23 reusable solutions to common problems in object-oriented design. Rather than inventing solutions from scratch, these patterns provide tested architectural solutions that have proven effective across diverse systems. They are not algorithms (which define a sequence of steps) but rather **structural templates for organizing code and managing dependencies**.

Design patterns solve the practical question: **How do I structure objects and their interactions to make my system flexible, testable, and maintainable?**

Patterns are organized into three categories:
- **Creational:** How objects are created
- **Structural:** How objects are composed and organized
- **Behavioral:** How objects interact and communicate

## Problems These Patterns Solve

### 1. **Uncontrolled Object Creation**
Creating objects directly scattered throughout the codebase makes it difficult to:
- Control instantiation logic (e.g., reusing objects, applying initialization rules)
- Mock objects in tests
- Switch between different implementations
- Enforce preconditions or invariants during creation

### 2. **Rigid Object Hierarchies**
Fixed inheritance hierarchies resist change. Adding a new variation requires creating new subclasses or modifying existing ones, violating OCP.

### 3. **Weak Abstraction Boundaries**
Without clear patterns for structure, components become entangled. Dependencies flow in unpredictable directions, making changes dangerous.

### 4. **Duplicated Structural Logic**
Common problems like managing state transitions, handling command execution, or controlling access are re-implemented differently throughout the codebase.

### 5. **Communication Complexity**
As systems scale, managing interactions between objects becomes chaotic. Without patterns for synchronization and messaging, coupling increases exponentially.

## Why These Problems Exist

**The Core Issue:** Object-oriented programming offers flexibility through composition and inheritance, but this flexibility comes with decision paralysis. Engineers must choose how to structure interactions without clear guidelines.

- **Inheritance is fragile:** Deep hierarchies are hard to reason about and modify. Sibling classes often share code but can't share implementations without creating unnecessary parent-child relationships.
- **Encapsulation creates boundaries, but crossing them is unclear:** How should objects collaborate? Through direct method calls, events, delegates, or something else?
- **Variation points are unpredictable:** You don't know in advance whether you'll need to swap implementations, so you don't know whether to introduce abstraction.
- **Copy-paste solutions accumulate:** Different teams solve the same problem differently, creating inconsistency and hidden complexity.

Patterns address these issues by providing **named, proven solutions** that communicate intent and structure to future maintainers.

## Common Solutions / Approaches

### **CREATIONAL PATTERNS**

#### Factory Method

**What it solves:** Creating objects without specifying their concrete classes.

**Real problem:** A `Document` application needs to create `WordDocument`, `PDFDocument`, and `ImageDocument` instances. Hard-coding `new WordDocument()` throughout the codebase makes it difficult to:
- Switch implementations
- Create documents with different backends (disk-based vs. in-memory)
- Mock documents in tests

**How it works:** Define an abstract `DocumentFactory` interface with a `createDocument()` method. Each document type implements its own factory. The application code depends on the factory interface, not concrete classes.

**Architectural benefit:** Decouples document creation from the application logic. Adding a new document type requires a new factory implementation, not changes to existing code (OCP).

**Trade-off:** Introduces indirection. Simple applications with few object types don't benefit—the overhead exceeds the flexibility gain.

#### Abstract Factory

**What it solves:** Creating families of related objects that must work together.

**Real problem:** A UI framework needs to support multiple themes (Dark theme, Light theme, Accessibility theme). Each theme has coordinated buttons, windows, dialogs, and menus. Creating them separately risks inconsistency.

**How it works:** Create an abstract factory that knows how to create all components for a theme. Concrete factories implement the theme-specific creation logic. The application requests components through the abstract factory.

**Architectural benefit:** Ensures theme consistency and makes swapping themes trivial. New themes are isolated from application logic.

**Trade-off:** Adds layers of indirection. Only worth it when you have multiple families of related objects that must vary together.

#### Builder

**What it solves:** Creating complex objects with many optional parameters and initialization logic.

**Real problem:** Creating an HTTP request with optional headers, timeouts, authentication, proxies, and custom body handlers. A constructor with dozens of parameters is unmaintainable. Positional parameters create confusion.

**How it works:** Create a builder class that accepts configuration one piece at a time, validating constraints at each step. The final `build()` method constructs the immutable object.

**Architectural benefit:** Makes complex object creation readable and validates at construction time. Enables incremental configuration and makes optional parameters explicit.

**Practical application:** Configuration objects, query builders, test fixtures.

**Limitation:** Adds code. Not needed for simple objects with few parameters.

#### Singleton

**What it solves:** Ensuring only one instance of a class exists and providing global access to it.

**Why it's dangerous:** Singleton is one of the most misused patterns. The problem: global state is a coupling mechanism. Any code can modify the singleton, making it difficult to predict behavior. Testing becomes complex because state persists across tests.

**Legitimate use cases:**
- Thread-safe lazy initialization of expensive resources (database connections, logger)
- Coordination points where global state is semantically correct (transaction managers, configuration managers)
- Managing system-wide resources (thread pools, caches)

**Legitimate alternatives:**
- Dependency injection (prefer this in modern systems)
- Static factories with controlled access
- Module-level singletons in functional languages

**Architectural warning:** If you find yourself using Singleton frequently, it's often a sign that you should be using dependency injection instead. Singletons violate DIP because they create hidden dependencies on global state.

### **STRUCTURAL PATTERNS**

#### Adapter

**What it solves:** Making incompatible interfaces work together.

**Real problem:** Your system expects a `PaymentProcessor` interface with `processPayment(amount)`. You integrate a third-party library that provides `MakePayment(transactionRequest)` with a different signature. You can't modify the third-party code.

**How it works:** Create an adapter that implements `PaymentProcessor` and translates calls to the third-party interface internally.

**Architectural benefit:** Isolates your system from external API changes. The third-party library can evolve without affecting your code. Makes dependencies on external systems replaceable.

**Real-world use case:** Wrapping legacy APIs, integrating incompatible libraries, creating stable interfaces over volatile dependencies.

**Key insight:** Adapters are defensive boundaries. They acknowledge that you depend on external systems you don't control, so you shield your core logic with an abstraction layer.

#### Decorator

**What it solves:** Adding behavior to objects dynamically without modifying their code or creating new subclasses.

**Real problem:** A `DataStreamWriter` writes data to disk. You need to add compression, encryption, and buffering. Creating subclasses for each combination (`CompressedEncryptedBufferedDataStreamWriter`) is combinatorial explosion.

**How it works:** Create decorator classes that implement the same interface and wrap the original object. Each decorator adds a single piece of behavior before delegating to the wrapped object.

**Architectural benefit:** Compose behaviors dynamically. Add functionality without modifying original classes (OCP). Each concern is isolated in its own class (SRP).

**Comparison to inheritance:** Inheritance is static and creates taxonomies. Decorators are dynamic and compositional. Decorators win when you have orthogonal concerns that combine unpredictably.

**Real-world use case:** I/O streams, UI component styling, caching layers, logging/monitoring wrappers.

#### Composite

**What it solves:** Treating individual objects and compositions of objects uniformly.

**Real problem:** A file system has files and directories. Directories can contain files and other directories recursively. You need to calculate total size, apply permissions, or search recursively. Hard-coding file-specific and directory-specific logic creates duplication.

**How it works:** Define a common interface for both files and directories. Both implement operations like `getSize()`, `delete()`, `getChildren()`. Directories recursively apply operations to their children.

**Architectural benefit:** Clients treat files and directories identically. New operations can be added without modifying file or directory classes. Tree structures are handled elegantly.

**Key insight:** Composite works well for hierarchical structures where operations naturally apply recursively. It doesn't work as well for heterogeneous structures where files and directories behave fundamentally differently.

**Real-world use case:** UI component hierarchies, organizational hierarchies, file system operations, menu structures.

#### Proxy

**What it solves:** Controlling access to another object or deferring expensive operations.

**Real problem:** Loading an image takes time. Rather than loading all images immediately, you want to load them only when they're actually displayed. Or you want to log every access to a sensitive object.

**How it works:** A proxy implements the same interface as the real object and intercepts calls. It can:
- Defer creation/loading (virtual proxy)
- Control access (protection proxy)
- Log operations (logging proxy)
- Implement remote access (remote proxy)

**Architectural benefit:** Decouples clients from implementation details. Enables cross-cutting concerns (logging, access control) without modifying the real object.

**Comparison to Decorator:** Both wrap objects, but proxies control access while decorators add functionality. A proxy might load the real object on first access; a decorator always delegates to the real object.

**Real-world use case:** ORM lazy loading, remote method invocation, access control layers, monitoring/metrics collection.

### **BEHAVIORAL PATTERNS**

#### Observer

**What it solves:** Creating loose coupling between objects that need to be notified of state changes.

**Real problem:** A `UserService` changes user data. Multiple subsystems need to react: 
- Send audit logs
- Clear related caches
- Trigger workflow steps
- Update analytics

Hard-coding notifications in `UserService` couples it to all these concerns.

**How it works:** `UserService` maintains a list of `Observer` instances. When data changes, it iterates through observers and calls `onStateChanged()`. Observers register themselves with the service.

**Architectural benefit:** `UserService` doesn't know about observers—it just notifies them. New observers can be added without modifying `UserService`. This is the foundation of reactive and event-driven architectures.

**Real-world use case:** UI frameworks (Model-View frameworks), reactive streams, pub-sub systems, event buses.

**Limitation:** In distributed systems, observer becomes impractical. Message queues and event buses provide better scaling.

#### Strategy

**What it solves:** Selecting algorithms at runtime without embedding conditional logic throughout the codebase.

**Real problem:** Sorting algorithms. Your system sorts data by different criteria (alphabetical, numeric, custom). Embedding if-else statements everywhere creates sprawl and violates OCP.

**How it works:** Define a `SortStrategy` interface. Create implementations for each sort algorithm. Pass the appropriate strategy to the code that sorts.

**Architectural benefit:** Algorithms are isolated. New sorting strategies don't affect existing code. The sorting code doesn't know or care about algorithm details.

**Pattern maturity:** Strategy is one of the most mature, proven patterns. It appears everywhere that algorithms must be interchangeable.

**Real-world use case:** Payment methods, compression algorithms, caching strategies, authentication mechanisms.

#### Command

**What it solves:** Encapsulating requests as objects, enabling queuing, logging, undo/redo, and delayed execution.

**Real problem:** A UI application needs to:
- Execute actions (edit, delete, copy)
- Undo them
- Queue them for later execution
- Replay them from logs

Spaghetti code results if each feature implements these concerns independently.

**How it works:** Wrap each action in a `Command` object that knows how to execute and undo itself. A `CommandQueue` manages execution, history, and replay.

**Architectural benefit:** Actions become first-class objects. You can queue them, log them, undo them, or distribute them across the network without understanding their implementation.

**Real-world use case:** Undo/redo systems, transaction logging, distributed task queues, scheduled job execution, API request modeling.

**Key insight:** Command is the bridge between imperative (do this now) and declarative (here's what to do) programming.

#### State

**What it solves:** Implementing objects whose behavior changes based on internal state, avoiding giant state machines with conditional logic.

**Real problem:** An `Order` can be pending, processing, shipped, or delivered. Each state has different valid operations:
- Pending → can be cancelled or processed
- Processing → can be shipped
- Shipped → can be delivered
- Delivered → no more operations

Hard-coding this with if-else statements creates brittle, error-prone code.

**How it works:** Create a state interface. Each state class implements behavior for that state. The order delegates operations to its current state object.

**Architectural benefit:** State transitions are explicit and local to each state. Adding a new state doesn't require modifying existing states. Each state is testable in isolation.

**Comparison to Strategy:** Both use interfaces, but Strategy is for behavior that clients choose, while State is for behavior that an object's internal condition determines.

**Real-world use case:** Workflow systems, TCP connection states, user account lifecycle, shopping cart checkout flow.

#### Mediator

**What it solves:** Reducing coupling between objects that communicate with each other. Instead of direct references, objects communicate through a mediator.

**Real problem:** A dialog with multiple controls (radio buttons, checkboxes, text fields). When one control changes, others must update:
- Select "express shipping" → disable "scheduled" option
- Check "same as billing" → copy address fields
- Change quantity → update price

Each control ends up with references to many others, creating a tangled dependency graph.

**How it works:** Create a `DialogMediator` that knows about all controls. When a control changes, it notifies the mediator, which updates other controls accordingly.

**Architectural benefit:** Controls are decoupled. They only know about the mediator, not other controls. Complex interactions are centralized in one place.

**Trade-off:** The mediator can become a "god object" knowing too much. Use it only when interactions genuinely belong together.

**Real-world use case:** Dialog/form coordination, UI component synchronization, workflow orchestration, air traffic control (yes, really—the pattern name comes from this analogy).

## Trade-offs and Limitations

### Trade-off 1: Abstraction Overhead vs. Flexibility

**The cost:** Patterns add indirection. A simple factory method introduces an extra class and interface. A strategy pattern multiplies classes.

**When to pay it:** In systems where variations are known or expected to occur. Legacy systems with high change rates benefit significantly.

**When to avoid it:** In simple scripts or small, focused tools. A command-line utility with one implementation of an algorithm doesn't need Strategy.

### Trade-off 2: Naming and Discoverability

**The challenge:** Patterns are useful because they're named and well-known. But they can also be overused, with engineers applying pattern names to justify overly complex designs.

**Principle:** Use pattern names when they genuinely fit the problem and communicate intent to future readers. Avoid using patterns for the sake of seeming sophisticated.

### Trade-off 3: Combinatorial Explosion

**The risk:** Combining multiple patterns can create complex hierarchies. A Decorator over a Strategy over an Adapter can become confusing.

**Reality check:** If you find yourself nesting multiple patterns, step back. Often, simpler solutions exist. Patterns should clarify, not obfuscate.

### Trade-off 4: Language Dependency

**Important note:** Some patterns are built into modern languages. In Python, decorators are language features, not design patterns. In functional languages, many GoF patterns are less relevant because they're addressing OOP-specific problems.

## Common Pitfalls and Misuses

### Pitfall 1: Pattern-Driven Design

**Mistake:** Starting with a pattern and fitting the problem to it. "We need a Factory!" "Let's use Composite!" without understanding the actual problem.

**Better approach:** Understand the problem first. Then, if a pattern fits, use it. Patterns emerge from problems, not the other way around.

### Pitfall 2: Premature Pattern Application

**Mistake:** Creating factories and strategies for variations that haven't materialized. You implement multiple implementations "just in case" but never actually use them.

**Reality principle:** Introduce patterns when:
1. You have multiple implementations or know they're coming
2. Variation points are explicitly in the roadmap
3. The pattern simplifies currently complex code

### Pitfall 3: God Objects and Mediators

**Mistake:** A mediator that knows about dozens of objects and coordinates all their interactions. It becomes a central bottleneck and violates SRP.

**Prevention:** Mediators should coordinate logically related interactions. If it's managing global state or coordinating unrelated domains, reconsider the design.

### Pitfall 4: Singleton Everywhere

**Mistake:** Using Singleton for "global access" to loggers, configuration, caches. This creates hidden dependencies and makes testing difficult.

**Modern alternative:** Dependency injection. Pass dependencies explicitly. Yes, it requires more plumbing, but it's worth it for testability and clarity.

### Pitfall 5: Adapter as a Dumping Ground

**Mistake:** Creating an adapter class that not only adapts the interface but also adds business logic, transformation, and validation.

**Clarity principle:** An adapter should translate interfaces, nothing more. Business logic belongs elsewhere.

### Pitfall 6: Command for Everything

**Mistake:** Using Command for every action, even simple one-off operations. This adds boilerplate without benefit.

**Right use:** Command shines when you need undo/redo, queuing, logging, or delayed execution. For simple imperative operations, just call methods.

## Key Takeaways

1. **Patterns are communication tools.** Their primary value is naming proven solutions and conveying intent to other engineers.

2. **Patterns address OOP-specific problems.** In functional programming or with modern language features, many GoF patterns are less relevant.

3. **Creational patterns control object creation.** They decouple code from concrete classes, making implementations interchangeable.

4. **Structural patterns manage composition.** They solve the problem of organizing classes into larger structures without tight coupling.

5. **Behavioral patterns manage interaction.** They define how objects communicate and how responsibilities are distributed.

6. **Patterns are not universal solutions.** Every pattern has trade-offs. Use them judiciously when they genuinely simplify code.

7. **Over-patterning is a real risk.** Excessive abstraction creates complexity without corresponding benefit. The goal is clear, maintainable code—patterns are a means to that end.

8. **Context matters.** A pattern valuable in a large, evolving system might be overkill in a simple utility. Always ask: "Does this pattern solve a real problem we have?"

## Related Concepts

- SOLID Principles (patterns implement these)
- Domain-Driven Design
- Refactoring (patterns emerge from refactoring)
- Anti-patterns (patterns misapplied)
- Architectural patterns (patterns at system scale)
