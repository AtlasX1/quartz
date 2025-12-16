# Object-Oriented TypeScript: L4 Engineering Guide

## Part 1: Classes & Access Modifiers

### 1.1 Class Fundamentals

TypeScript classes extend ES6 classes with type annotations and access modifiers. **Access modifiers** control visibility: `public` (default), `protected` (accessible in subclasses), `private` (not accessible outside class).

**Constructor parameters** can use access modifiers to automatically declare and initialize properties, reducing boilerplate.

```typescript
// Basic class
class Animal {
  public name: string; // public (default)
  protected age: number;
  private energy: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
    this.energy = 100;
  }

  public speak(): string {
    return `${this.name} makes a sound`;
  }

  protected getAge(): number {
    return this.age;
  }
}

// Parameter properties: access modifier on constructor parameter declares property
class Dog extends Animal {
  constructor(
    public name: string,
    protected age: number,
    private breed: string
  ) {
    super(name, age);
  }

  speak(): string {
    return `${this.name} barks`;
  }
}

// Getters and setters
class User {
  private _email: string = "";

  get email(): string {
    return this._email;
  }

  set email(value: string) {
    if (!value.includes("@")) throw new Error("Invalid email");
    this._email = value;
  }
}
```

### 1.2 Abstract Classes & Interfaces for OOP

**Abstract classes** define base classes with some abstract methods that subclasses must implement. **Interfaces** define contracts that classes implement.

**Distinction:** Abstract classes can have implementation and state; interfaces cannot (though newer TypeScript relaxes this slightly with interface properties).

```typescript
// Abstract class
abstract class Shape {
  abstract getArea(): number; // Must be implemented by subclasses

  describe(): string {
    return `Area: ${this.getArea()}`;
  }
}

class Circle extends Shape {
  constructor(private radius: number) {
    super();
  }

  getArea(): number {
    return Math.PI * this.radius ** 2;
  }
}

// Interface for contracts
interface Drawable {
  draw(): void;
}

interface Resizable {
  resize(scale: number): void;
}

class Rectangle implements Drawable, Resizable {
  draw() { /* ... */ }
  resize(scale: number) { /* ... */ }
}
```

---

## Part 2: Inheritance & Polymorphism

### 2.1 Class Inheritance & Super

**Inheritance** enables subclasses to extend parent classes, inheriting properties and methods. **`super` keyword** calls parent constructor and methods.

```typescript
// Inheritance hierarchy
class Vehicle {
  constructor(protected model: string) {}

  start(): string {
    return `${this.model} starting`;
  }
}

class Car extends Vehicle {
  constructor(model: string, private doors: number) {
    super(model); // Must call parent constructor
  }

  start(): string {
    return `${super.start()} with ${this.doors} doors`;
  }
}

// Method override with type safety
class Motorcycle extends Vehicle {
  start(): string {
    return `${this.model} revving`;
  }
}
```

### 2.2 Polymorphism & Method Binding

**Polymorphism** enables treating objects of different classes uniformly through a common base type. **Method binding** determines which implementation executes; TypeScript's type system ensures safety.

```typescript
// Polymorphic behavior
const vehicles: Vehicle[] = [
  new Car("Tesla", 4),
  new Motorcycle("Harley")
];

vehicles.forEach(v => console.log(v.start())); // Correct method called for each

// Type-safe polymorphism
function startVehicle(vehicle: Vehicle): string {
  return vehicle.start(); // Calls the appropriate subclass method
}

// Generics + inheritance for type-safe collections
class Repository<T extends { id: string | number }> {
  private items: T[] = [];

  save(item: T) {
    const index = this.items.findIndex(i => i.id === item.id);
    if (index >= 0) {
      this.items[index] = item;
    } else {
      this.items.push(item);
    }
  }

  find(id: T["id"]): T | undefined {
    return this.items.find(i => i.id === id);
  }
}
```

---

## Part 3: Mixins & Composition

### 3.1 Mixin Pattern in TypeScript

**Mixins** combine behaviors from multiple sources without inheritance hierarchies. They're implemented using higher-order functions or object composition.

```typescript
// Mixin function pattern
type Constructor<T = {}> = new (...args: any[]) => T;

function withTimestamp<T extends Constructor>(Base: T) {
  return class extends Base {
    timestamp = new Date();
  };
}

function withValidation<T extends Constructor>(Base: T) {
  return class extends Base {
    validate(): boolean {
      return true;
    }
  };
}

class User {
  constructor(public name: string) {}
}

// Apply mixins
const ValidatedUserWithTimestamp = withTimestamp(withValidation(User));
const user = new ValidatedUserWithTimestamp("John");
// user has: name, timestamp, validate()

// Composition over inheritance
interface Logger {
  log(message: string): void;
}

interface Validator {
  validate(data: unknown): boolean;
}

class UserService implements Logger, Validator {
  log(message: string) { console.log(message); }
  validate(data: unknown): boolean { return true; }

  createUser(name: string) {
    this.log(`Creating user: ${name}`);
    if (!this.validate(name)) throw new Error("Invalid");
    // ... create user
  }
}
```

---

## Part 4: Static Members & Singleton Pattern

### 4.1 Static Properties & Methods

**Static members** belong to the class itself, not instances. Useful for utility methods, factory functions, and shared state.

```typescript
class Math {
  static readonly PI = 3.14159;
  static readonly E = 2.71828;

  static add(a: number, b: number): number {
    return a + b;
  }

  static distance(x1: number, y1: number, x2: number, y2: number): number {
    return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
  }
}

Math.PI; // 3.14159
Math.add(5, 3); // 8

// Singleton pattern using static
class Database {
  private static instance: Database;

  private constructor() {}

  static getInstance(): Database {
    if (!Database.instance) {
      Database.instance = new Database();
    }
    return Database.instance;
  }
}

const db1 = Database.getInstance();
const db2 = Database.getInstance();
console.log(db1 === db2); // true (same instance)
```

---

## Interview Questions

**Q1: Explain access modifiers in TypeScript. Why are they important?**

Access modifiers (`public`, `protected`, `private`) control visibility. `public` is default; `protected` allows subclass access; `private` restricts to the class. They enforce encapsulation, prevent accidental misuse, and document API contracts.

**Q2: When should you use inheritance vs composition?**

Inheritance models "is-a" relationships but creates tight coupling. Composition models "has-a" relationships and is more flexible. Prefer composition for most cases; use inheritance for true subtype relationships where polymorphism is needed.

**Q3: What's the mixin pattern and when would you use it?**

Mixins combine behaviors from multiple sources without deep inheritance hierarchies. Useful when multiple unrelated classes need the same functionality. Implemented as higher-order functions that enhance a class.

**Q4: Explain the singleton pattern. Is it always appropriate?**

Singleton pattern restricts a class to one instance, useful for shared resources (database, logger). However, singletons complicate testing (can't isolate) and introduce global state. Prefer dependency injection for better testability.

---

## Key Takeaways

1. **Access modifiers enforce encapsulation** - `private`/`protected`/`public` control visibility
2. **Parameter properties reduce boilerplate** - Access modifier on parameter declares the property
3. **Abstract classes define subclass contracts** - Combine interface contracts with state/implementation
4. **Polymorphism enables treating subclasses uniformly** - Through base class references
5. **Mixins avoid deep inheritance hierarchies** - Combine behaviors from multiple sources
6. **Composition is more flexible than inheritance** - Prefer composition for code reuse
7. **Static members belong to the class, not instances** - Useful for utilities and singleton patterns