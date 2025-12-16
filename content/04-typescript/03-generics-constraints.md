# Generics & Type Constraints: L4 Engineering Guide

## Part 1: Generic Fundamentals

### 1.1 Generic Functions & Types

**Generics** allow writing code that works with multiple types while maintaining type safety. A generic parameter (convention: single uppercase letter like `T`, `U`, `K`) is a placeholder for any type.

**Generic resolution:** The compiler infers or explicitly specifies the generic type when the function is called. Type checking ensures operations on generic types are valid for all possible substitutions.

```typescript
// Generic function
function getFirst<T>(array: T[]): T {
  return array[0];
}

getFirst([1, 2, 3]); // T = number, returns 1
getFirst(["a", "b"]); // T = string, returns "a"

// Multiple generic parameters
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

pair(1, "hello"); // [number, string]

// Generic interface
interface Container<T> {
  value: T;
  getValue(): T;
  setValue(value: T): void;
}

const numContainer: Container<number> = {
  value: 42,
  getValue() { return this.value; },
  setValue(v) { this.value = v; }
};
```

### 1.2 Generic Constraints & Defaults

**Constraints** limit which types can substitute a generic parameter. Without constraints, only operations valid for `unknown` are allowed (essentially nothing).

**Default type parameters** specify a fallback type if none is provided.

```typescript
// Constraint: T must have length property
function getLength<T extends { length: number }>(x: T): number {
  return x.length;
}

getLength("hello"); // string has length
getLength([1, 2, 3]); // array has length
// getLength(123); // Error: number doesn't have length

// Constraint: U must be keyof T
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "John", age: 30 };
getProperty(user, "name"); // OK: "name" is keyof user
// getProperty(user, "invalid"); // Error: "invalid" not in keyof user

// Default type parameter
type Box<T = string> = { value: T };
const stringBox: Box = { value: "hello" }; // T defaults to string
const numBox: Box<number> = { value: 42 };
```

### 1.3 Generic Scope & Nested Generics

Generic parameters can be nested: generic types containing other generic types. Scope rules determine which generic parameters are in scope.

```typescript
// Nested generics
function compose<A, B, C>(
  f: (a: A) => B,
  g: (b: B) => C
): (a: A) => C {
  return (a) => g(f(a));
}

const toString = (n: number) => String(n);
const toUpperCase = (s: string) => s.toUpperCase();
const composed = compose(toString, toUpperCase);
// composed(42) returns "42"

// Generic class with multiple parameters
class Graph<T, E> {
  nodes: T[] = [];
  edges: Map<T, E[]> = new Map();

  addNode(node: T) {
    this.nodes.push(node);
  }

  addEdge(from: T, to: T, edge: E) {
    if (!this.edges.has(from)) {
      this.edges.set(from, []);
    }
    this.edges.get(from)!.push(edge);
  }
}
```

---

## Part 2: Advanced Generic Patterns

### 2.1 Generic Type Narrowing

**Type narrowing** within generics uses constraints and type guards to refine generic types. This enables type-safe operations even with constrained generics.

```typescript
// Generic narrowing with type guard
function process<T extends string | number>(value: T): string {
  if (typeof value === "string") {
    return value.toUpperCase(); // T narrowed to string
  } else {
    return value.toFixed(2); // T narrowed to number
  }
}

// Constraint + conditional type narrowing
function deepClone<T extends object>(obj: T): T {
  if (Array.isArray(obj)) {
    return obj.map(item => deepClone(item)) as T; // T narrowed to array
  } else if (typeof obj === "object" && obj !== null) {
    const cloned = {} as T;
    for (const key in obj) {
      if (obj.hasOwnProperty(key)) {
        cloned[key] = deepClone(obj[key]);
      }
    }
    return cloned;
  }
  return obj;
}
```

### 2.2 Generic Builders & Fluent APIs

**Builder pattern with generics** creates type-safe fluent interfaces, ensuring proper method chaining while maintaining type accuracy through method return types.

```typescript
// Generic builder with chaining
class QueryBuilder<T> {
  private conditions: string[] = [];

  where<K extends keyof T>(key: K, operator: string, value: T[K]): QueryBuilder<T> {
    this.conditions.push(`${String(key)} ${operator} ${value}`);
    return this;
  }

  build(): string {
    return `SELECT * WHERE ${this.conditions.join(" AND ")}`;
  }
}

interface User {
  name: string;
  age: number;
  email: string;
}

const query = new QueryBuilder<User>()
  .where("name", "=", "John")
  .where("age", ">", 18)
  .build();

// Type-safe: only User properties allowed, and types must match
// .where("invalid", "=", "value"); // Error
```

---

## Part 3: Generic Constraints & Relationships

### 3.1 Higher-Order Generics

**Higher-order generics** work with generic types themselves, enabling powerful abstractions. Examples: utilities that transform generic containers.

```typescript
// Higher-order generic: map over container type
type Map<F, T> = F extends Array<infer U> ? T[] : T;

// Higher-order function: transform generic container
function transform<T, U, Container extends readonly unknown[]>(
  container: Container,
  transform: (x: unknown) => U
): Map<Container, U>[] {
  return container.map(transform) as Map<Container, U>[];
}

// Distribution generic type
type Flatten<T> = T extends Array<infer U> ? U : T;
type FlattendUnion = Flatten<string[] | number>; // string | number
```

### 3.2 Generic Best Practices

**Don't over-generalize:** Add constraints to capture minimal requirements. **Use meaningful names:** `T` is conventional but `TEntity`, `TKey` improve clarity.

**Avoid deep nesting:** Complex nested generics are hard to reason about and debug. **Prefer union types:** Sometimes `T | U` is simpler than overly generic abstractions.

```typescript
// Good: clear constraint and purpose
function findById<T extends { id: number | string }>(
  items: T[],
  id: T["id"]
): T | undefined {
  return items.find(item => item.id === id);
}

// Bad: too generic, unclear intent
function find<T, K, V>(items: T[], key: K, value: V): T | undefined {
  // Unclear what T should be or how K/V relate
  return undefined;
}

// Good: specific and clear
interface Entity {
  id: number | string;
}

function findEntity<T extends Entity>(items: T[], id: T["id"]): T | undefined {
  return items.find(item => item.id === id);
}
```

---

## Interview Questions

**Q1: Explain generic constraints. Why are they necessary?**

Generic constraints (`T extends U`) limit which types can substitute the generic parameter. Without constraints, only operations valid for `unknown` are allowed (practically none). Constraints enable type-safe operations on the generic type.

**Q2: What's a practical use case for default type parameters?**

Default type parameters provide fallback types when not explicitly specified. Example: `type Container<T = string>` defaults to string if no type is provided. Useful for APIs where a common case should be the default.

**Q3: Explain the difference between generic constraints at function and type level.**

Function-level generics constrain individual function calls; type-level generics constrain type aliases/interfaces. Function constraints enable per-call type safety; type constraints ensure class instances are type-safe throughout their lifetime.

**Q4: When would you use a generic constraint vs a union type?**

Generic constraints preserve type identity across operations; unions don't. Use generics when you need to return the same type that was passed in. Use unions for simpler scenarios where type identity isn't important.

---

## Key Takeaways

1. **Generics enable type-safe reusable code** - Type parameters preserve type information
2. **Constraints ensure operations are valid** - `T extends U` restricts possible types
3. **Default type parameters provide convenience** - Fallback type if not specified
4. **Nested generics enable sophisticated patterns** - Compose complex type relationships
5. **Type narrowing works in generics** - Use type guards and conditionals to refine types
6. **Builder pattern + generics = fluent type-safe APIs** - Method chaining with accurate return types
7. **Meaningful names improve code clarity** - Use descriptive names beyond single letters