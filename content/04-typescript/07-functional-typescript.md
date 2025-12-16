# Functional TypeScript: L4 Engineering Guide

## Part 1: Functional Patterns in TypeScript

### 1.1 Pure Functions & Immutability

**Pure functions** are functions that return the same output for the same inputs without side effects. TypeScript's static typing enables enforcing purity at the type level.

**Immutability** prevents unintended mutations. TypeScript provides `readonly` keyword for properties and `Readonly<T>` utility type.

```typescript
// Pure function
function add(a: number, b: number): number {
  return a + b; // Deterministic, no side effects
}

// Immutable data structures
interface User {
  readonly id: number;
  readonly name: string;
}

const user: User = { id: 1, name: "John" };
// user.name = "Jane"; // Error: cannot assign to readonly property

// Readonly array
const numbers: readonly number[] = [1, 2, 3];
// numbers.push(4); // Error: readonly array

// Readonly utility type
type ReadonlyUser = Readonly<User>;

// Immutable update patterns
const updatedUser: User = { ...user, name: "Jane" }; // New object
const updatedNumbers: readonly number[] = [...numbers, 4]; // New array
```

### 1.2 Higher-Order Functions & Composition

**Higher-order functions** work with functions as values. **Composition** combines functions into pipelines.

TypeScript's type system enables precise composition signatures.

```typescript
// Function composition
type Compose = <A, B, C>(
  f: (b: B) => C,
  g: (a: A) => B
) => (a: A) => C;

const compose: Compose = (f, g) => (a) => f(g(a));

// Function pipeline
type Pipe = <A, B, C>(
  f: (a: A) => B,
  g: (b: B) => C
) => (a: A) => C;

const pipe: Pipe = (f, g) => (a) => g(f(a));

// Practical composition
const toUpperCase = (s: string): string => s.toUpperCase();
const reverse = (s: string): string => s.split("").reverse().join("");

const composed = compose(reverse, toUpperCase);
composed("hello"); // "OLLEH"

const piped = pipe(toUpperCase, reverse);
piped("hello"); // "OLLEH"

// Currying: f(a, b) → f(a)(b)
function curry<A, B, C>(f: (a: A, b: B) => C): (a: A) => (b: B) => C {
  return (a) => (b) => f(a, b);
}

const curriedAdd = curry((a: number, b: number) => a + b);
curriedAdd(5)(3); // 8
```

### 1.3 Functional Data Transformations

**Declarative data transformations** use `map`, `filter`, `reduce` for clear, composable pipelines.

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

const products: Product[] = [
  { id: 1, name: "Laptop", price: 1000, category: "Electronics" },
  { id: 2, name: "Book", price: 20, category: "Books" },
  { id: 3, name: "Mouse", price: 30, category: "Electronics" }
];

// Declarative transformation
const expensiveElectronics = products
  .filter(p => p.category === "Electronics")
  .filter(p => p.price > 50)
  .map(p => ({ name: p.name, price: p.price }));

// Reduce for aggregation
const totalPrice = products.reduce((sum, p) => sum + p.price, 0);

const priceByCategory = products.reduce((acc, p) => {
  if (!acc[p.category]) {
    acc[p.category] = 0;
  }
  acc[p.category] += p.price;
  return acc;
}, {} as Record<string, number>);
```

---

## Part 2: Advanced Functional Patterns

### 2.1 Functional Composition with Types

TypeScript enables type-safe function composition that preserves type information through the pipeline.

```typescript
// Type-safe pipeline
interface Pipeline<T, R> {
  (input: T): R;
}

function createPipeline<T, R>(
  ...fns: Array<(x: any) => any>
): Pipeline<T, R> {
  return (input: T) => fns.reduce((acc, fn) => fn(acc), input) as R;
}

const processString = createPipeline<string, number>(
  (s: string) => s.toUpperCase(),
  (s: string) => s.length
);

processString("hello"); // 5

// Typefly builder pattern
type Builder<T, R> = {
  add: <S>(fn: (x: T) => S) => Builder<S, R>;
  build: () => (x: T) => R;
};

class FunctionalBuilder<T, R> implements Builder<T, R> {
  constructor(private transformers: Array<(x: any) => any> = []) {}

  add<S>(fn: (x: T) => S): Builder<S, R> {
    return new FunctionalBuilder([...this.transformers, fn]);
  }

  build(): (x: T) => R {
    return (x) => this.transformers.reduce((acc, fn) => fn(acc), x) as R;
  }
}

const builder = new FunctionalBuilder<string, number>()
  .add(s => s.toUpperCase())
  .add(s => s.length);

const result = builder.build()("hello"); // 5
```

### 2.2 Partial Application & Memoization

**Partial application** fixes some function arguments. **Memoization** caches function results for performance.

```typescript
// Partial application
function partial<T extends (...args: any[]) => any>(
  fn: T,
  ...fixedArgs: any[]
): (...args: any[]) => ReturnType<T> {
  return (...args: any[]) => fn(...fixedArgs, ...args);
}

const add = (a: number, b: number): number => a + b;
const add5 = partial(add, 5);
add5(3); // 8

// Memoization with types
function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, any>();

  return ((...args: any[]) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

const expensiveFn = memoize((n: number): number => {
  console.log(`Computing ${n}`);
  return n * 2;
});

expensiveFn(5); // Computes
expensiveFn(5); // Returns cached value
```

---

## Part 3: Functional Type Systems

### 3.1 Algebraic Data Types

**Algebraic data types** combine types using sum (unions) and product (tuples/records). Enables exhaustive pattern matching.

```typescript
// Sum type: one of several options
type Result<T, E> = { type: "ok"; value: T } | { type: "error"; error: E };

function processResult<T, E>(result: Result<T, E>): string {
  return result.type === "ok" ? String(result.value) : String(result.error);
}

// Product type: combination of types
type User = { name: string; age: number };

// Discriminated union for exhaustiveness
type Action =
  | { type: "INCREMENT"; payload: number }
  | { type: "DECREMENT"; payload: number }
  | { type: "RESET" };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT":
      return state + action.payload;
    case "DECREMENT":
      return state - action.payload;
    case "RESET":
      return 0;
  }
}
```

### 3.2 Monadic Patterns

**Monads** provide composable error handling and sequencing of operations. Common monads: `Option` (null-safety), `Either` (error handling), `Task` (async).

```typescript
// Option monad: handles null/undefined
class Option<T> {
  constructor(private value: T | null) {}

  static of<T>(value: T): Option<T> {
    return new Option(value);
  }

  map<U>(fn: (x: T) => U): Option<U> {
    return this.value === null ? new Option(null) : new Option(fn(this.value));
  }

  flatMap<U>(fn: (x: T) => Option<U>): Option<U> {
    return this.value === null ? new Option(null) : fn(this.value);
  }

  getOrElse(defaultValue: T): T {
    return this.value === null ? defaultValue : this.value;
  }
}

const maybeUser = Option.of<string>("John");
const greeting = maybeUser
  .map(name => `Hello, ${name}`)
  .map(greeting => greeting.toUpperCase())
  .getOrElse("Hello, Guest");

// Either monad: error handling
type Either<E, T> = { type: "left"; error: E } | { type: "right"; value: T };

class EitherMonad<E, T> {
  constructor(private either: Either<E, T>) {}

  static right<E, T>(value: T): EitherMonad<E, T> {
    return new EitherMonad({ type: "right", value });
  }

  static left<E, T>(error: E): EitherMonad<E, T> {
    return new EitherMonad({ type: "left", error });
  }

  map<U>(fn: (x: T) => U): EitherMonad<E, U> {
    return this.either.type === "left"
      ? new EitherMonad({ type: "left", error: this.either.error })
      : new EitherMonad({ type: "right", value: fn(this.either.value) });
  }
}
```

---

## Interview Questions

**Q1: Explain pure functions and their benefits in TypeScript.**

Pure functions return the same output for the same inputs without side effects. Benefits: easier testing (deterministic), easier reasoning (no hidden dependencies), optimization opportunities (memoization), easier composition. TypeScript's static types help enforce purity.

**Q2: What's function composition and how does it differ from chaining?**

Composition combines functions into pipelines: `compose(f, g)(x) = f(g(x))`. Chaining calls methods on objects: `obj.method1().method2()`. Composition is more flexible for reuse; chaining is more idiomatic for certain patterns. Both enable clear data transformations.

**Q3: Explain the monad pattern and give an example.**

Monads provide composable error handling and operation sequencing. Example: `Option<T>` wraps nullable values; `map` and `flatMap` chain operations safely. If any step fails, the chain short-circuits. Enables null-safe operations without null checks.

**Q4: When should you use partial application vs currying?**

Partial application fixes some arguments, returning a function with remaining arguments. Currying transforms `f(a, b) → f(a)(b)`. Partial application is more practical; currying enables automatic currying libraries. Use partial application for practical cases; currying for lambda calculus-style composition.

---

## Key Takeaways

1. **Pure functions eliminate bugs** - Same input always produces same output
2. **Immutability prevents unexpected mutations** - Use `readonly` and spread operators
3. **Function composition creates reusable pipelines** - Combine simple functions into complex operations
4. **Higher-order functions enable abstraction** - Decorators, middleware, composition
5. **Monads enable composable error handling** - Option/Either monads short-circuit on failure
6. **Discriminated unions enable exhaustiveness** - Type system ensures all cases handled
7. **Functional patterns scale to large systems** - Predictable, testable, maintainable code