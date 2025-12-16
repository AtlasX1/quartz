# OOP & SOLID Principles: Theoretical Foundations and Architectural Patterns

## Table of Contents
1. Object-Oriented Programming Paradigm
2. Core OOP Concepts and Mathematical Foundations
3. SOLID Principles: Theory and Application
4. Design Patterns and Their Classification
5. Architectural Patterns and Enterprise Design
6. Dependency Injection and Inversion of Control
7. GRASP Principles and Responsibility Assignment
8. Production Systems and Trade-offs

---

## 1. Object-Oriented Programming Paradigm

### 1.1 Historical Context and Paradigm Shift

Object-oriented programming emerged in the 1960s-1970s as a response to the limitations of procedural programming at scale. While procedural programming organizes code around **functions** (behavior), OOP organizes code around **objects** (data + behavior). This fundamental shift addresses several key challenges:

**Procedural vs. Object-Oriented Decomposition:**
- Procedural: Functions operate on shared global state → tight coupling, difficult maintenance
- OOP: Objects encapsulate state and behavior → loose coupling, encapsulation, modularity

The theoretical foundation of OOP rests on three pillars:
1. **Encapsulation**: Bundling data with methods, controlling access
2. **Inheritance**: Establishing type hierarchies and code reuse
3. **Polymorphism**: Runtime behavior variation through method overriding

### 1.2 The Abstraction Problem

OOP solves the **abstraction problem** in software design: how to represent complex systems using simpler mental models. This maps to category theory concepts in mathematics:

- Objects are instances of **categories** with defined interfaces
- Inheritance creates **hierarchical category relationships**
- Method dispatch is a form of **natural transformation** between object types

The abstraction principle states: *"The abstraction selected determines what aspects of the system are visible to the developer and what aspects are hidden."*

For example, a `PaymentProcessor` abstraction hides:
- Currency conversion details
- Payment gateway specifics
- Retry logic and failure handling
- Audit trail generation

While exposing:
- `processPayment(amount: Money, card: CreditCard): Result<PaymentId>`
- `refund(paymentId: PaymentId): Result<RefundId>`

### 1.3 Type Systems and Subtype Polymorphism

OOP is fundamentally about **subtype polymorphism**, formalized by Liskov's Substitution Principle (LSP):

*If S is a subtype of T, then objects of type S may be substituted for objects of type T without altering the desirable properties of the program.*

This can be expressed mathematically. For types $S$ and $T$:
$$S <: T \iff \forall \text{ operations } f \text{ where } T \text{ is valid, } S \text{ produces equivalent results}$$

**Example - Behavioral Subtyping:**
```
Type Circle extends Shape
Type Square extends Shape

// Both must satisfy Shape's contract
Shape.getArea(): Number
Shape.getPerimeter(): Number
```

The violation case: `Rectangle extends Square` violates LSP because:
- Square expects: `width == height` invariant
- Rectangle violates this invariant
- LSP predicts maintenance issues (which indeed arise in practice)

---

## 2. Core OOP Concepts and Mathematical Foundations

### 2.1 Encapsulation: Information Hiding

Encapsulation is the **bundling of data with methods** and the hiding of internal details. This addresses the fundamental problem of managing coupling:

**Coupling Metric:**
$$\text{Coupling} = \frac{\text{External dependencies exposed}}{(\text{Total methods} + \text{Total fields})}$$

Encapsulation reduces this ratio by:
1. Making fields private (not exposed)
2. Exposing behavior through limited, intentional interfaces
3. Controlling how internal state is accessed and modified

**Invariant Maintenance:**
Encapsulation enables maintaining object invariants. For a `BankAccount`:
```
Invariant: balance >= 0
Invariant: transactions.length <= maxTransactionHistory
```

Private methods like `_validateBalance()` and `_rotateTransactions()` maintain these invariants without external interference.

**Access Levels and Information Boundaries:**
- `private`: No external access (intra-class only)
- `protected`: Accessible to subclasses (intra-hierarchy)
- `package/internal`: Accessible within module/package
- `public`: Full external access

Each level represents a trade-off between flexibility and encapsulation strength.

### 2.2 Inheritance: Type Hierarchies and Code Reuse

Inheritance establishes **is-a relationships** between types, enabling:

1. **Code Reuse**: Common behavior defined once in a parent class
2. **Polymorphic Behavior**: Different implementations via method overriding
3. **Type Hierarchies**: Establishing substitutable types

**Inheritance Trade-offs:**

| Aspect | Benefit | Cost |
|--------|---------|------|
| Code reuse | DRY principle, reduced duplication | Tight parent-child coupling |
| Polymorphism | Behavior variation at runtime | Method resolution overhead |
| Hierarchy clarity | Models natural IS-A relationships | Rigid structure, fragile base class problem |
| Interface contracts | Guarantees subclass behavior | LSP violations if not careful |

**The Fragile Base Class Problem:**
When a base class is modified, derived classes may break unexpectedly:
```
Base class: removeElement(index) → removes at index
Derived class: List extends ArrayList
  Invariant: maintains sorted order

// If Base class changes removeElement() implementation,
// Derived class's sorted order invariant may be violated
```

**Single vs. Multiple Inheritance:**
Most modern OOP languages (Java, C#, Python 3 MRO) restrict to single inheritance or use interfaces (multiple inheritance of interface, not implementation):

- **Single Inheritance**: Simpler, avoids diamond problem, but less expressive
- **Multiple Inheritance**: More expressive, but creates ambiguity (C++ diamond problem)
- **Multiple Interface Implementation**: Compromise solution (Java, C#, TypeScript)

### 2.3 Polymorphism: Runtime Behavior Variation

Polymorphism enables **calling the same method name on different objects** and getting different behaviors. This manifests in three forms:

#### 2.3.1 Subtype Polymorphism (Runtime Dispatch)

```
interface Shape {
  area(): number
}

class Circle implements Shape {
  constructor(radius: number) { this.r = radius }
  area() { return Math.PI * this.r * this.r }
}

class Rectangle implements Shape {
  constructor(w: number, h: number) { this.w = w; this.h = h }
  area() { return this.w * this.h }
}

// Polymorphic behavior
const shapes: Shape[] = [new Circle(5), new Rectangle(4, 6)]
shapes.forEach(s => console.log(s.area()))  // Different implementation per type
```

**Virtual Method Tables (VMT):**
Modern OOP implementations use VMT to resolve method calls:
- Each object carries a pointer to its class's VMT
- VMT maps method names to actual implementations
- Lookup happens at runtime (O(1) with hash table or array indexing)

#### 2.3.2 Ad-hoc Polymorphism (Function Overloading)

Same method name, different signatures. Resolved at **compile time**:
```
class Calculator {
  add(a: int, b: int): int
  add(a: double, b: double): double
  add(a: string, b: string): string
}
```

#### 2.3.3 Parametric Polymorphism (Generics)

Type parameters allow abstract code that works with multiple types:
```
class Container<T> {
  private items: T[] = []
  
  add(item: T): void { this.items.push(item) }
  get(index: number): T { return this.items[index] }
}
```

This is fundamentally different from subtype polymorphism—the same code path handles all types, compared to different implementations per type.

### 2.4 Composition vs. Inheritance

**Composition (Has-A)**: Objects contain other objects, delegating behavior:

```
class Engine { start(): void }
class Car {
  private engine: Engine = new Engine()
  start(): void { this.engine.start() }
}
```

**Inheritance (Is-A)**: Objects extend other objects, inheriting behavior:

```
class Vehicle { start(): void }
class Car extends Vehicle { }
```

**Composition Advantages:**
1. Flexible behavior changes at runtime
2. Avoids fragile base class problem
3. Multiple behavior combinations without multiple inheritance
4. Better encapsulation (internal objects not exposed)

**Inheritance Advantages:**
1. Cleaner is-a relationships
2. Natural polymorphism for type hierarchies
3. Less verbose for modeling hierarchies

**Modern Recommendation**: *"Favor composition over inheritance"* (Gang of Four)
- Use inheritance for **type relationships only**
- Use composition for **behavior reuse and combination**

---

## 3. SOLID Principles: Theory and Application

### 3.1 Single Responsibility Principle (SRP)

**Definition**: A class should have one, and only one, reason to change.

**Mathematical Formulation:**
$$R = \frac{\text{Number of reasons to change}}{1} \leq 1$$

A class has multiple responsibilities if it has multiple reasons to change. For example:

```
// VIOLATES SRP: Two responsibilities
class User {
  saveToDatabase(): void { }  // Persistence concern
  sendWelcomeEmail(): void { }  // Communication concern
  validateEmail(): void { }  // Data validation concern
}

// COMPLIES WITH SRP
class User {
  validateEmail(): void { }  // Only data integrity
}

class UserRepository {
  save(user: User): void { }  // Only persistence
}

class WelcomeEmailSender {
  send(user: User): void { }  // Only communication
}
```

**Cohesion Metric:**
$$\text{Cohesion} = \frac{\text{Methods that use shared fields}}{\text{Total methods}}$$

Higher cohesion (methods sharing state) indicates SRP compliance.

**Benefits:**
- Reduced coupling: Changes to persistence don't affect email logic
- Increased testability: Each concern tested independently
- Improved reusability: Persistence layer usable by multiple contexts
- Easier maintenance: Clear responsibility boundaries

**Violations in Practice:**
1. **God Classes**: Large classes with many responsibilities
2. **Feature Envy**: Methods accessing too much external state (indicates misplaced responsibility)
3. **Inappropriate Intimacy**: Classes knowing too much about each other

### 3.2 Open/Closed Principle (OCP)

**Definition**: Software entities should be open for extension but closed for modification.

**Formal Expression:**
A class is OCP-compliant if new behavior can be added without modifying existing code:
$$\text{OCP-Compliant} \iff \forall \text{ new features } f, \text{ existing code remains unchanged}$$

**Achieving OCP Through Abstraction:**

```
// VIOLATES OCP: Must modify PaymentProcessor for each new method
class PaymentProcessor {
  process(method: string) {
    if (method === 'credit-card') { /* credit card logic */ }
    if (method === 'paypal') { /* paypal logic */ }
    if (method === 'crypto') { /* crypto logic */ }  // NEW: Must modify!
  }
}

// COMPLIES WITH OCP: New methods without modification
interface PaymentMethod {
  process(amount: Money): Result<PaymentId>
}

class PaymentProcessor {
  private methods: Map<string, PaymentMethod> = new Map()
  
  register(name: string, method: PaymentMethod): void {
    this.methods.set(name, method)
  }
  
  process(methodName: string, amount: Money): Result<PaymentId> {
    return this.methods.get(methodName)?.process(amount) ?? fail()
  }
}

// NEW: Add without modifying PaymentProcessor
class CryptoPayment implements PaymentMethod {
  process(amount: Money): Result<PaymentId> { /* crypto logic */ }
}

processor.register('crypto', new CryptoPayment())
```

**Abstraction Level Trade-off:**
- **Too Concrete**: Tightly coupled, requires modification for each variation
- **Too Abstract**: Over-engineered, unnecessary indirection
- **Optimal**: Abstract just enough to accommodate anticipated extensions

**Implementation Strategies:**
1. **Template Method Pattern**: Abstract algorithm skeleton, subclasses fill details
2. **Strategy Pattern**: Encapsulate algorithm variations
3. **Decorator Pattern**: Add behavior dynamically
4. **Dependency Injection**: Inject implementations rather than hardcoding

### 3.3 Liskov Substitution Principle (LSP)

**Definition**: Subtypes must be substitutable for their base types without breaking the program.

**Formal Statement** (Barbara Liskov, 1987):
If $S$ is a subtype of $T$, then objects of type $S$ may be substituted for objects of type $T$ without altering any of the desirable properties of that program.

**Contract Preservation:**
```
interface Shape {
  area(): number
  perimeter(): number
}

// COMPLIES WITH LSP
class Circle implements Shape {
  area() { return Math.PI * this.r * this.r }
  perimeter() { return 2 * Math.PI * this.r }
}

// VIOLATES LSP
class Square implements Shape {
  setWidth(w: number) { this.width = w; this.height = w }
  setHeight(h: number) { this.width = h; this.height = h }
  
  area() { return this.width * this.height }
  perimeter() { return 4 * this.width }
  
  // PROBLEM: Caller expects independent width/height
  // Code using Shape: shape.setWidth(5); shape.setHeight(3)
  // Expectation: width=5, height=3
  // Reality (Square): width=3, height=3 (violated invariant!)
}
```

**Behavioral Contract Violations:**

| Violation Type | Example | Impact |
|---|---|---|
| Precondition Strengthening | Subclass requires more restrictive input | Caller's valid input rejected |
| Postcondition Weakening | Subclass provides weaker guarantees | Caller's assumptions broken |
| Invariant Violation | Subclass breaks class invariants | Unexpected state corruption |
| Exception Addition | Subclass throws unexpected exceptions | Uncaught exception crashes |

**LSP in Collections:**
```
// VIOLATES LSP
class CountdownList<T> extends ArrayList<T> {
  @Override add(element: T): boolean {
    // Add to beginning instead of end
    this.items.unshift(element)
    return true
  }
}

// Caller expects ArrayList behavior (FIFO)
let list: ArrayList<int> = new CountdownList()
list.add(1); list.add(2); list.add(3)
console.log(list[0])  // Expected: 1, Got: 3 (LSP violation!)
```

**Benefits:**
- Program correctness guaranteed through type system
- Refactoring safety: Can replace implementations without testing all callers
- Enables polymorphic collections without runtime surprises

### 3.4 Interface Segregation Principle (ISP)

**Definition**: Clients should not be forced to depend on interfaces they do not use.

**Formal Expression:**
$$\text{ISP-Compliant} \iff \forall \text{ clients } c, \text{ unused methods count} = 0$$

**Dependency Bloat Anti-pattern:**

```
// VIOLATES ISP: Worker does much; Robots forced to depend on everything
interface Worker {
  work(): void
  eat(): void
  sleep(): void
}

class Robot implements Worker {
  work(): void { /* robot works */ }
  eat(): void { throw new Error('Robots do not eat!') }
  sleep(): void { throw new Error('Robots do not sleep!') }
}

// COMPLIES WITH ISP: Segregated interfaces
interface Workable {
  work(): void
}

interface Eatable {
  eat(): void
}

class Robot implements Workable {
  work(): void { /* robot works */ }
}

class Human implements Workable, Eatable {
  work(): void { /* human works */ }
  eat(): void { /* human eats */ }
}
```

**ISP Benefits:**
1. **Reduced Coupling**: Clients depend only on needed methods
2. **Easier Mocking**: Test doubles implement only necessary behavior
3. **Better Documentation**: Interface clearly shows expected contract
4. **Flexibility**: Different clients use different interface subsets

**Anti-pattern Recognition:**
```
// Sign of ISP violation:
// 1. Large interfaces (>6-8 methods)
// 2. NotImplementedException in implementations
// 3. Methods that don't match the class's core responsibility
```

### 3.5 Dependency Inversion Principle (DIP)

**Definition**: 
1. High-level modules should not depend on low-level modules. Both should depend on abstractions.
2. Abstractions should not depend on details. Details should depend on abstractions.

**Visual Representation:**
```
// VIOLATES DIP: High-level depends on low-level
BusinessLogic → DatabaseImpl → PhysicalDisk

// COMPLIES WITH DIP: Both depend on abstraction
BusinessLogic → IRepository ← DatabaseImpl
                            ← FileSystemImpl
```

**Dependency Flow:**
```
// ANTI-PATTERN: Direct coupling
class OrderService {
  private mysql: MySQLDatabase = new MySQLDatabase()
  
  getOrder(id: string): Order {
    return this.mysql.query('SELECT * FROM orders WHERE id = ?', id)
  }
}

// PATTERN: Abstraction through DI
interface IRepository {
  getOrder(id: string): Promise<Order>
}

class OrderService {
  constructor(private repo: IRepository) { }
  
  async getOrder(id: string): Promise<Order> {
    return this.repo.getOrder(id)
  }
}
```

**Benefits:**
1. **Testability**: Inject mock repositories in tests
2. **Flexibility**: Swap implementations without changing business logic
3. **Maintainability**: Changes to persistence layer don't cascade
4. **Reusability**: Business logic independent of storage mechanism

**Practical Implementation:**
```
// Production
const repo = new MySQLRepository()
const service = new OrderService(repo)

// Testing
const mockRepo = new MockRepository()
const service = new OrderService(mockRepo)

// Future: Change to document database
const repo = new MongoDBRepository()
const service = new OrderService(repo)  // Same code!
```

---

## 4. Design Patterns and Their Classification

### 4.1 Gang of Four (GoF) Pattern Taxonomy

Design patterns are **reusable solutions to common problems** in object-oriented design. Cataloged by Gang of Four (1994), they fall into three categories:

#### 4.1.1 Creational Patterns

Control **object instantiation mechanisms**:

**Singleton Pattern:**
```
class Configuration {
  private static instance: Configuration
  private constructor() { }
  
  static getInstance(): Configuration {
    if (!this.instance) {
      this.instance = new Configuration()
    }
    return this.instance
  }
}
```
- Use: Single instance coordination (loggers, config)
- Cost: Global state, testing complexity, thread-safety issues

**Factory Pattern:**
```
interface AnimalFactory {
  create(): Animal
}

class DogFactory implements AnimalFactory {
  create(): Animal { return new Dog() }
}
```
- Use: Decouple creation logic from usage
- Benefit: Single point for object creation

**Builder Pattern:**
```
class QueryBuilder {
  private query: Query = new Query()
  
  select(fields: string[]): QueryBuilder { this.query.fields = fields; return this }
  where(condition: string): QueryBuilder { this.query.condition = condition; return this }
  build(): Query { return this.query }
}

const q = new QueryBuilder()
  .select(['id', 'name'])
  .where('age > 18')
  .build()
```
- Use: Complex object construction
- Benefit: Readable, fluent API

**Object Pool Pattern:**
```
class ConnectionPool {
  private available: Connection[] = []
  private inUse: Set<Connection> = new Set()
  
  acquire(): Connection {
    const conn = this.available.length > 0 ? this.available.pop() : new Connection()
    this.inUse.add(conn)
    return conn
  }
  
  release(conn: Connection): void {
    this.inUse.delete(conn)
    this.available.push(conn)
  }
}
```
- Use: Expensive object reuse (DB connections, threads)
- Benefit: Reduced allocation overhead

#### 4.1.2 Structural Patterns

Deal with **object composition and relationships**:

**Decorator Pattern:**
```
interface Component {
  operation(): string
}

class ConcreteComponent implements Component {
  operation(): string { return "Basic operation" }
}

abstract class Decorator implements Component {
  constructor(protected component: Component) { }
  operation(): string { return this.component.operation() }
}

class AuthDecorator extends Decorator {
  operation(): string {
    return `[Auth check] ${this.component.operation()}`
  }
}

const basic = new ConcreteComponent()
const decorated = new AuthDecorator(basic)
```
- Use: Add responsibilities dynamically
- Alternative to: Inheritance (more flexible)

**Adapter Pattern:**
```
interface TargetInterface {
  request(): string
}

class Adapter implements TargetInterface {
  constructor(private adaptee: IncompatibleInterface) { }
  
  request(): string {
    return this.adaptee.specificMethod()
  }
}
```
- Use: Make incompatible interfaces compatible
- Real-world: USB-C adapter, voltage converter

**Facade Pattern:**
```
class LibraryFacade {
  private catalog: Catalog = new Catalog()
  private checkout: Checkout = new Checkout()
  private payment: Payment = new Payment()
  
  borrowBook(bookId: string, userId: string): Result {
    // Coordinate complex subsystem interactions
    const book = this.catalog.findBook(bookId)
    this.checkout.reserve(book, userId)
    const fee = this.payment.calculateFee(book, userId)
    return this.payment.charge(userId, fee)
  }
}
```
- Use: Simplify complex subsystems
- Benefit: Hide internal complexity

**Proxy Pattern:**
```
interface Subject {
  request(): void
}

class RealSubject implements Subject {
  request(): void { console.log("Expensive operation") }
}

class Proxy implements Subject {
  private realSubject: RealSubject
  
  request(): void {
    if (this.isAccessAllowed()) {
      this.realSubject.request()
    }
  }
  
  private isAccessAllowed(): boolean { /* auth logic */ }
}
```
- Use: Control access, lazy loading, caching
- Example: Database connection proxy

#### 4.1.3 Behavioral Patterns

Control **object collaboration and responsibility**:

**Strategy Pattern:**
```
interface SortingStrategy {
  sort(array: number[]): number[]
}

class QuickSort implements SortingStrategy {
  sort(array: number[]): number[] { /* quick sort */ }
}

class MergeSort implements SortingStrategy {
  sort(array: number[]): number[] { /* merge sort */ }
}

class Sorter {
  constructor(private strategy: SortingStrategy) { }
  
  sort(array: number[]): number[] {
    return this.strategy.sort(array)
  }
}
```
- Use: Multiple algorithm implementations
- Benefit: Runtime algorithm selection

**Observer Pattern:**
```
interface Observer {
  update(data: any): void
}

class Subject {
  private observers: Observer[] = []
  
  attach(observer: Observer): void {
    this.observers.push(observer)
  }
  
  notify(data: any): void {
    this.observers.forEach(obs => obs.update(data))
  }
}
```
- Use: Event systems, reactive updates
- Real-world: UI event listeners, pub/sub systems

**Command Pattern:**
```
interface Command {
  execute(): void
  undo(): void
}

class PrintCommand implements Command {
  constructor(private document: Document) { }
  
  execute(): void { this.document.print() }
  undo(): void { /* remove from print queue */ }
}

class Invoker {
  private history: Command[] = []
  
  execute(command: Command): void {
    command.execute()
    this.history.push(command)
  }
  
  undo(): void {
    this.history.pop()?.undo()
  }
}
```
- Use: Undo/redo, transaction queuing, macros
- Benefit: Encapsulate requests as objects

**State Pattern:**
```
interface State {
  handle(context: Context): void
}

class OnState implements State {
  handle(context: Context): void {
    console.log("Already on")
    context.setState(new OnState())
  }
}

class Context {
  private state: State = new OffState()
  
  setState(state: State): void { this.state = state }
  
  request(): void { this.state.handle(this) }
}
```
- Use: Objects with state-dependent behavior
- Example: Vending machine, traffic light

---

## 5. Architectural Patterns and Enterprise Design

### 5.1 Layered Architecture

The most common enterprise pattern. Organizes system into horizontal layers:

```
┌─────────────────────────────────┐
│   Presentation Layer            │ (UI, API endpoints)
├─────────────────────────────────┤
│   Business Logic Layer          │ (Use cases, rules)
├─────────────────────────────────┤
│   Persistence Layer             │ (Database access)
├─────────────────────────────────┤
│   Infrastructure Layer          │ (Utilities, config)
└─────────────────────────────────┘
```

**Characteristics:**
- Each layer has specific responsibilities
- Layers communicate vertically (presentation → business → persistence)
- Typically no layer skipping (presentation shouldn't access persistence directly)

**Advantages:**
- Simple to understand and implement
- Clear separation of concerns
- Easy to test each layer independently

**Disadvantages:**
- Can lead to "spaghetti" if boundaries blur
- Performance overhead from layer traversal
- May encourage database-driven design (anemic models)

### 5.2 Hexagonal Architecture (Ports and Adapters)

Alternative to layered, emphasizing domain isolation:

```
        ┌────────────────────────────┐
        │   Domain Logic (Core)      │
        │   Business entities,       │
        │   use cases, rules         │
        └────────────────────────────┘
           ▲                    ▲
           │ Adapter            │ Adapter
           │                    │
    ┌──────┴──┐          ┌──────┴──┐
    │ Web API  │          │Database │
    │ Adapter  │          │Adapter  │
    └──────────┘          └──────────┘
```

**Ports**: Interfaces defining boundaries
**Adapters**: Implementations specific to external systems

**Advantages:**
- Domain logic completely independent of infrastructure
- Highly testable (mock adapters easily)
- Technology agnostic core
- Easy to swap implementations

**Disadvantages:**
- More complex than layered
- Requires disciplined abstraction
- May be overkill for simple systems

### 5.3 Event-Driven Architecture

Systems communicate through **events** rather than direct calls:

```
┌──────────────┐         Event         ┌──────────────┐
│ Event Source │────────────────────────→ Event Handler│
└──────────────┘      (UserCreated)     └──────────────┘
                                        
                ┌─────────────────────────────────────┐
                │    Event Bus / Message Broker       │
                │  (RabbitMQ, Kafka, EventBridge)    │
                └─────────────────────────────────────┘
```

**Characteristics:**
- Components communicate asynchronously through events
- Temporal decoupling: Producers don't wait for consumers
- Event sourcing: System state derived from event history

**Advantages:**
- Scalability: Easy to add new handlers
- Loose coupling: Components don't know about each other
- Audit trail: Events provide complete history

**Disadvantages:**
- Eventual consistency: Harder to guarantee strong consistency
- Debugging complexity: Flow harder to trace
- At-least-once delivery semantics: Handle duplicate events

---

## 6. Dependency Injection and Inversion of Control

### 6.1 Inversion of Control (IoC) Container

An IoC container manages **object lifecycle and dependency resolution**:

```typescript
// Manual DI
const db = new PostgresConnection()
const repo = new UserRepository(db)
const service = new UserService(repo)
const controller = new UserController(service)

// IoC Container (automatic)
const container = new DIContainer()
container.register(UserService, { 
  dependencies: [UserRepository] 
})
container.register(UserRepository, { 
  dependencies: [Database] 
})

const service = container.resolve(UserService)
// Container automatically resolves dependencies
```

**Benefits:**
1. **Reduced boilerplate**: Container handles wiring
2. **Centralized configuration**: All dependencies in one place
3. **Lifecycle management**: Singleton vs. transient vs. scoped instances
4. **Easy testing**: Swap implementations globally

**Common IoC Features:**
- **Singleton**: One instance per container lifetime
- **Transient**: New instance each time
- **Scoped**: One instance per request/scope
- **Factory functions**: Custom instantiation logic
- **Auto-wiring**: Automatic dependency detection

### 6.2 Dependency Injection Patterns

#### 6.2.1 Constructor Injection (Recommended)

Dependencies provided through constructor:
```typescript
class UserService {
  constructor(private repo: IUserRepository) { }
}
```

**Advantages:**
- Immutable dependencies
- Clear dependencies (visible in constructor)
- Impossible to use uninitialized
- Excellent for testing

**Disadvantages:**
- Large constructors for many dependencies (code smell)
- Circular dependency detection (compile-time)

#### 6.2.2 Setter Injection

Dependencies set via properties:
```typescript
class UserService {
  private repo: IUserRepository
  setRepository(repo: IUserRepository) {
    this.repo = repo
  }
}
```

**Advantages:**
- Optional dependencies
- No large constructors
- Can change dependencies at runtime

**Disadvantages:**
- Can use before injection (null reference errors)
- Dependencies not immediately clear
- Harder to test

#### 6.2.3 Interface Injection

Special interface for dependency registration:
```typescript
interface InjectionTarget {
  inject(container: DIContainer): void
}

class UserService implements InjectionTarget {
  private repo: IUserRepository
  inject(container: DIContainer) {
    this.repo = container.resolve(IUserRepository)
  }
}
```

**Disadvantages:**
- Verbose and intrusive
- Rarely used in practice
- Less type-safe than constructor injection

**Recommendation**: Constructor injection for production, property injection for optional cross-cutting concerns.

---

## 7. GRASP Principles and Responsibility Assignment

GRASP (General Responsibility Assignment Software Patterns) provides heuristics for assigning responsibilities to classes:

### 7.1 Creator Pattern

**Who should create object X?**

**Heuristic**: B should create A if:
- B aggregates A
- B contains A
- B has the data needed to initialize A
- B logs/tracks A

```typescript
// GOOD: OrderFactory knows how to create Order
class OrderFactory {
  createOrder(customerId: string, items: Item[]): Order {
    return new Order(customerId, items, DateTime.now())
  }
}

// BAD: Random service creating orders
class EmailService {
  sendOrderConfirmation(order: Order) {
    // Shouldn't create orders here
    const newOrder = new Order(...)
  }
}
```

### 7.2 Information Expert Pattern

**Who should be responsible for a given task?**

**Heuristic**: Assign to the class with most information needed to perform the task.

```typescript
// GOOD: Order knows about items and totals
class Order {
  getTotal(): Money {
    return this.items.reduce((sum, item) => sum + item.price, Money.zero())
  }
}

// BAD: Calculator doesn't have order context
class OrderCalculator {
  calculateTotal(items: Item[]): Money {
    // Weak design - shouldn't know all item details
    return items.reduce((sum, item) => sum + item.price, Money.zero())
  }
}
```

### 7.3 Low Coupling Pattern

**How to reduce coupling?**

**Heuristic**: Keep dependencies minimal and abstract.

```typescript
// HIGH COUPLING: Direct dependencies on concrete classes
class ReportGenerator {
  private mysql: MySQLDatabase = new MySQLDatabase()
  private fileWriter: LocalFileWriter = new LocalFileWriter()
  private emailer: GmailEmailer = new GmailEmailer()
}

// LOW COUPLING: Dependencies on abstractions
class ReportGenerator {
  constructor(
    private db: IDatabase,
    private output: IReportOutput,
    private notifier: INotifier
  ) { }
}
```

### 7.4 High Cohesion Pattern

**How to keep classes focused?**

**Heuristic**: Elements should be strongly related; responsibilities should be related.

```typescript
// LOW COHESION: Mixed concerns
class User {
  authenticate(): void { /* auth logic */ }
  sendEmail(): void { /* email logic */ }
  logActivity(): void { /* logging */ }
  generateReport(): void { /* reporting */ }
}

// HIGH COHESION: Focused responsibility
class User {
  authenticate(): void { /* auth logic */ }
  getEmail(): string { }
  getId(): string { }
  // All related to user identity and authentication
}
```

**Cohesion Formula:**
$$C = \frac{\sum \text{(methods sharing fields)}}{\text{total method pairs}}$$

### 7.5 Polymorphism Pattern

**How to handle type-based variations?**

**Heuristic**: Use polymorphism rather than switch statements.

```typescript
// ANTI-PATTERN: Switch-based variation
class PaymentProcessor {
  process(type: string, amount: Money) {
    switch(type) {
      case 'credit': return this.creditCard(amount)
      case 'paypal': return this.paypal(amount)
      case 'crypto': return this.crypto(amount)  // Must modify!
    }
  }
}

// PATTERN: Polymorphic variation
interface PaymentMethod {
  process(amount: Money): Result
}

class PaymentProcessor {
  constructor(private method: PaymentMethod) { }
  
  process(amount: Money): Result {
    return this.method.process(amount)
  }
}
```

---

## 8. Production Systems and Trade-offs

### 8.1 Scalability Considerations

**Vertical Layered Architecture Problems:**
- Monolithic deployment: One change requires full deployment
- Cascading changes: Layer modifications affect entire system
- Testing complexity: Integration testing required for small changes
- Database coupling: Layer tightly tied to persistence model

**Enterprise Solutions:**
- **Service-oriented architecture (SOA)**: Services with clear boundaries
- **Microservices**: Fine-grained service decomposition
- **Serverless**: Function-as-a-service, event-driven
- **CQRS**: Separate command and query models

### 8.2 Common Production Anti-patterns

**Anemic Domain Model:**
```typescript
// ANTI-PATTERN: Objects are just data containers
class Order {
  customerId: string
  items: OrderItem[]
  status: string
}

// PATTERN: Objects encapsulate behavior
class Order {
  private customerId: string
  private items: OrderItem[]
  private status: OrderStatus
  
  canCancel(): boolean { return this.status === OrderStatus.PENDING }
  cancel(): void { if (this.canCancel()) this.status = OrderStatus.CANCELLED }
}
```

**Leaky Abstractions:**
```typescript
// ANTI-PATTERN: Database leaking through abstraction
interface Repository {
  find(id: string): Promise<Entity>  // Returns database concept
  executeSQL(sql: string): any       // Direct SQL exposure
}

// PATTERN: Clean abstraction
interface Repository {
  find(id: string): Promise<Entity>
  findByStatus(status: Status): Promise<Entity[]>
}
```

**Premature Abstraction:**
```typescript
// ANTI-PATTERN: Over-engineered for single use case
interface EmailProvider {
  send(email: Email): Promise<void>
}
class SMTPEmailProvider implements EmailProvider { }
class MockEmailProvider implements EmailProvider { }

// When only SMTP is used and never changes. Better:
class EmailService {
  send(email: Email): Promise<void> { /* SMTP */ }
}
```

### 8.3 Performance Implications

**OOP Overhead:**
| Operation | Cost | Mitigation |
|-----------|------|-----------|
| Virtual method dispatch | 1-5ns per call | JIT inlining (modern VMs) |
| Object allocation | 10-100ns | Object pooling |
| Abstraction layers | Latency per layer | Inline hot paths |
| Polymorphic collections | Cache misses | Type specialization |

**Measured in practice:**
- Direct method call: ~1ns
- Virtual method call: ~2-5ns  
- Reflection-based access: ~100ns
- Interface dispatch in polymorphic collections: ~5-20ns

Most overhead is negligible compared to I/O operations (network: ~100,000ns, disk: ~1,000,000ns).

### 8.4 Testing Strategies

**Unit Testing:**
```typescript
describe('UserService', () => {
  it('creates user with repository', () => {
    const mockRepo = new MockRepository()
    const service = new UserService(mockRepo)
    
    service.create({ name: 'John' })
    
    expect(mockRepo.created).toContain({ name: 'John' })
  })
})
```

**Integration Testing:**
```typescript
describe('UserService with real database', () => {
  it('persists user to database', async () => {
    const db = new TestDatabase()
    const repo = new PostgresRepository(db)
    const service = new UserService(repo)
    
    await service.create({ name: 'John' })
    
    const user = await db.query('SELECT * FROM users WHERE name = ?', 'John')
    expect(user).toBeDefined()
  })
})
```

---

## Key Takeaways

1. **OOP Fundamentals**: Encapsulation, inheritance, and polymorphism form the foundation for extensible, maintainable systems.

2. **SOLID Principles** provide actionable heuristics:
   - SRP: One reason to change per class
   - OCP: Extend without modification via abstraction
   - LSP: Behavioral contracts matter for type safety
   - ISP: Clients shouldn't depend on unused methods
   - DIP: Depend on abstractions, not concretions

3. **Design Patterns** are proven solutions to recurring problems, categorized as:
   - **Creational**: Control instantiation (Factory, Builder, Singleton)
   - **Structural**: Object composition (Decorator, Adapter, Facade)
   - **Behavioral**: Object collaboration (Strategy, Observer, Command)

4. **Architectural Patterns** shape large systems:
   - Layered: Simple, common, but prone to blurring
   - Hexagonal: Domain-focused, highly testable
   - Event-driven: Loosely coupled, complex debugging

5. **Dependency Injection** through constructor parameters is the standard approach for testability and flexibility.

6. **GRASP Principles** guide responsibility assignment:
   - Creator, Information Expert, Low Coupling, High Cohesion, Polymorphism

7. **Production Trade-offs**: Abstraction provides flexibility but adds complexity. Premature abstraction is worse than tight coupling initially; refactor as requirements stabilize.

8. **Testing is paramount**: Strong typing and dependency injection enable comprehensive unit testing, which catches LSP violations early.

9. **Modern Reality**: Most systems combine patterns:
   - Layered architecture + hexagonal core
   - Service-oriented + event-driven
   - Microservices + CQRS

10. **The Meta-principle**: *"Favor composition over inheritance, depend on abstractions, and assign responsibilities to high-cohesion, low-coupling classes."*
