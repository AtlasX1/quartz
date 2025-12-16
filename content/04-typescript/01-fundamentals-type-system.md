# TypeScript Fundamentals & Type System: L4 Engineering Guide

## Part 1: Type System Foundations

### 1.1 Static Typing & Type Inference

TypeScript adds **static typing** to JavaScript—type checking occurs at compile time, not runtime. The compiler verifies that operations are type-safe before code executes, preventing entire classes of bugs.

**Type inference** (bidirectional type flow) determines types automatically when explicit annotations aren't provided. The compiler infers types from:
- Initial assignment: `let x = 5` → `x: number`
- Function returns: `return "hello"` → return type `string`
- Context: parameter types propagate to expression types

**Key distinction:** TypeScript types exist only during compilation; they're erased in JavaScript output. Types have zero runtime overhead.

```typescript
// Explicit annotation
let name: string = "John";
const age: number = 30;

// Type inference
let count = 5; // number (inferred)
const isActive = true; // boolean (inferred)

// Function type inference
function add(a: number, b: number) {
  return a + b; // return type inferred as number
}

// Contextual typing
const numbers = [1, 2, 3];
numbers.forEach(n => console.log(n * 2)); // n: number (inferred from array)
```

### 1.2 Primitive Types & Special Types

**Primitives:** `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`.

**Special types:**
- **`any`**: Disables type checking (escape hatch, use sparingly)
- **`unknown`**: Type-safe `any`; must narrow before use
- **`never`**: Type with no possible values; represents unreachable code
- **`void`**: Function returns nothing; variable of type `void` can only be `undefined`

**Type assignments:** TypeScript's type system is **structural** (based on shape, not name). Objects are compatible if their properties match, regardless of declared type.

```typescript
// Primitive types
const text: string = "hello";
const count: number = 42;
const isComplete: boolean = true;

// Special types
let something: any = "string"; // Disables checking
something.unknownMethod(); // No error (unsafe)

let unknown: unknown = "string";
if (typeof unknown === "string") {
  console.log(unknown.toUpperCase()); // Safe; narrowed to string
}

// Never type
function throwError(msg: string): never {
  throw new Error(msg);
}

function infiniteLoop(): never {
  while (true) {}
}

// Structural typing
interface Point { x: number; y: number; }
type Coord = { x: number; y: number; };
const point: Point = { x: 1, y: 2 }; // Point and Coord are compatible
```

### 1.3 Union & Intersection Types

**Union types** represent a value that could be one of several types: `string | number`. The type system requires checking which type it actually is before using type-specific operations.

**Intersection types** combine multiple types: `string & number` (rarely used; more common in object combinations). Every property from both types is required.

```typescript
// Union types
type ID = string | number;
function printID(id: ID) {
  if (typeof id === "string") {
    console.log(id.toUpperCase()); // string methods available
  } else {
    console.log(id.toFixed(2)); // number methods available
  }
}

// Intersection types
interface Named { name: string; }
interface Aged { age: number; }
type Person = Named & Aged; // Both properties required

const person: Person = { name: "John", age: 30 };

// Union of objects
type User = { id: number; name: string } | { email: string };
// User can be either structure, but not necessarily both
```

---

## Part 2: Interfaces & Type Aliases

### 2.1 Interfaces: Contracts & Evolution

**Interfaces** define object shapes and contracts. They're the primary tool for abstraction in TypeScript.

**Key properties:**
- Declaration merging: Multiple interface declarations with the same name merge properties
- Extensibility: `extends` keyword enables interface inheritance
- Optional properties: `property?: type` makes property optional
- Readonly properties: `readonly property: type` prevents modification

```typescript
// Basic interface
interface User {
  id: number;
  name: string;
  email?: string; // Optional
  readonly createdAt: Date; // Read-only
}

// Declaration merging
interface User {
  role?: string;
}
// Equivalent to:
interface User {
  id: number;
  name: string;
  email?: string;
  readonly createdAt: Date;
  role?: string;
}

// Interface inheritance
interface Admin extends User {
  permissions: string[];
}

// Method signatures
interface Repository {
  find(id: number): Promise<User>;
  save(user: User): Promise<void>;
}
```

### 2.2 Type Aliases vs Interfaces

**Type aliases** create a name for any type using `type` keyword. **Interfaces** are specifically for object shapes.

**Differences:**
- Interfaces: Declaration merging (multiple declarations combine); extensible
- Type aliases: No merging; immutable after declaration
- Type aliases: Can represent unions, tuples, primitives; interfaces cannot
- Performance: Negligible difference for most use cases

```typescript
// Type alias: flexible, can be anything
type ID = string | number;
type Callback = (data: unknown) => void;
type Tuple = [string, number];

// Interface: specialized for objects
interface User {
  name: string;
}

// Practical choice
type Status = "active" | "inactive"; // Only type alias works for unions
interface UserData {
  name: string;
  email: string;
}
```

---

## Part 3: Function Types & Overloads

### 3.1 Function Signatures

**Function types** specify parameter and return types. **Type inference** often determines return types automatically.

```typescript
// Function declaration
function add(a: number, b: number): number {
  return a + b;
}

// Function expression
const multiply: (a: number, b: number) => number = (a, b) => a * b;

// Function type alias
type MathOp = (a: number, b: number) => number;

// Optional and default parameters
function greet(name: string, greeting: string = "Hello"): string {
  return `${greeting}, ${name}`;
}

// Rest parameters
function sum(...numbers: number[]): number {
  return numbers.reduce((a, b) => a + b, 0);
}
```

### 3.2 Function Overloads

**Overloads** allow functions to be called with different argument combinations, with different return types for each.

```typescript
// Overload signatures (no implementation)
function process(input: string): string;
function process(input: number): number;

// Implementation signature (must be compatible with all overloads)
function process(input: string | number): string | number {
  if (typeof input === "string") {
    return input.toUpperCase();
  } else {
    return input * 2;
  }
}

// Usage: type checker enforces correct argument/return types
const strResult = process("hello"); // strResult: string
const numResult = process(5); // numResult: number
```

---

## Interview Questions

**Q1: Explain structural typing in TypeScript.**

TypeScript uses structural typing: two types are compatible if their properties match, regardless of declared type names. This differs from nominal typing (class-based languages) where names matter. Structural typing enables flexibility but can create unexpected type compatibility.

**Q2: What's the difference between `any` and `unknown`?**

`any` disables all type checking; `unknown` is type-safe and requires narrowing. Variables of type `unknown` must be checked before use (with `typeof`, `instanceof`, etc.), preventing unsafe operations. Prefer `unknown` for maximum safety.

**Q3: When should you use type aliases vs interfaces?**

Use interfaces for object contracts and extensibility via declaration merging. Use type aliases for everything else: unions, tuples, primitives, functions. When you need just an object shape, either works; interfaces are more conventional for APIs.

**Q4: Explain function overloads. Why not just use union types?**

Overloads enable different return types for different input combinations. Unions (type-safe but less precise) would require the caller to check the return type manually. Overloads provide caller-friendly APIs with precise return types.

---

## Key Takeaways

1. **TypeScript types are compile-time only** - Zero runtime overhead; erased to JavaScript
2. **Type inference reduces annotation burden** - Let compiler determine types when obvious
3. **Structural typing enables flexibility** - Objects are compatible by shape, not name
4. **`unknown` is safer than `any`** - Requires narrowing; prevents unsafe operations
5. **Interfaces are for object contracts** - Extensible via declaration merging
6. **Type aliases for unions and complex types** - Functions, tuples, primitives
7. **Function overloads provide precise types** - Different return types per call signature