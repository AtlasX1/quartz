# Type Guards & Narrowing: L4 Engineering Guide

## Part 1: Type Narrowing Techniques

### 1.1 Typeof & Instanceof Guards

**Type narrowing** refines variable types in specific code branches, enabling type-safe operations. The type checker tracks control flow to narrow types.

**`typeof` operator** narrows to primitive types: `"string"`, `"number"`, `"boolean"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`.

**`instanceof` operator** narrows to class types by checking prototype chain.

```typescript
// Typeof narrowing
function process(value: string | number | boolean) {
  if (typeof value === "string") {
    console.log(value.toUpperCase()); // value: string
  } else if (typeof value === "number") {
    console.log(value.toFixed(2)); // value: number
  } else {
    console.log(!value); // value: boolean
  }
}

// Instanceof narrowing
class Cat {
  meow() { }
}

class Dog {
  bark() { }
}

function petSound(pet: Cat | Dog) {
  if (pet instanceof Cat) {
    pet.meow(); // pet: Cat
  } else {
    pet.bark(); // pet: Dog
  }
}
```

### 1.2 Type Predicates

**Type predicate functions** are boolean functions that return `value is T`, enabling custom narrowing logic. The type checker trusts the predicate and narrows the type accordingly.

```typescript
// Type predicate
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isNumber(value: unknown): value is number {
  return typeof value === "number";
}

const input: unknown = "hello";
if (isString(input)) {
  console.log(input.toUpperCase()); // input: string (narrowed)
}

// Predicate with object property check
interface User {
  name: string;
  email: string;
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "name" in value &&
    "email" in value &&
    typeof (value as any).name === "string" &&
    typeof (value as any).email === "string"
  );
}

const data: unknown = { name: "John", email: "john@example.com" };
if (isUser(data)) {
  console.log(data.name); // data: User (narrowed)
}
```

### 1.3 Discriminated Unions

**Discriminated unions** combine a common property with different type branches, enabling exhaustive narrowing. The common property determines the type.

```typescript
// Discriminated union
type Success<T> = { status: "success"; data: T };
type Error = { status: "error"; message: string };
type Result<T> = Success<T> | Error;

function handleResult<T>(result: Result<T>) {
  if (result.status === "success") {
    console.log(result.data); // result: Success<T> (narrowed)
  } else {
    console.log(result.message); // result: Error (narrowed)
  }
}

// Complex discriminated union
type Square = { kind: "square"; side: number };
type Circle = { kind: "circle"; radius: number };
type Rectangle = { kind: "rectangle"; width: number; height: number };
type Shape = Square | Circle | Rectangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "square":
      return shape.side ** 2; // shape: Square
    case "circle":
      return Math.PI * shape.radius ** 2; // shape: Circle
    case "rectangle":
      return shape.width * shape.height; // shape: Rectangle
  }
}
```

---

## Part 2: Control Flow & Assertion Functions

### 2.1 Exhaustiveness Checking

**Exhaustiveness checking** ensures all union cases are handled. The `never` type represents impossible states.

```typescript
// Exhaustive type narrowing
type Status = "pending" | "success" | "error";

function handleStatus(status: Status): string {
  switch (status) {
    case "pending":
      return "Loading...";
    case "success":
      return "Done";
    case "error":
      return "Failed";
    // No default: if new Status value added, error here
  }
}

// Using never for exhaustiveness
function assertNever(value: never): never {
  throw new Error(`Unhandled case: ${value}`);
}

function handleStatus2(status: Status): string {
  switch (status) {
    case "pending":
      return "Loading...";
    case "success":
      return "Done";
    // Forgot "error": TypeScript errors (missing case)
    default:
      return assertNever(status); // status: never (all cases handled)
  }
}
```

### 2.2 Assertion Functions

**Assertion functions** throw if a condition is false, narrowing the type afterward. Return type `asserts condition is Type` narrows the type.

```typescript
// Basic assertion function
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

const data: unknown = "test";
assertIsString(data);
console.log(data.toUpperCase()); // data: string (narrowed)

// Assertion with condition check
function assertDefined<T>(value: T | null | undefined): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error("Value is not defined");
  }
}

const optional: string | undefined = "hello";
assertDefined(optional);
console.log(optional.toUpperCase()); // optional: string (narrowed)

// DOM element assertion
function assertElement(element: unknown): asserts element is HTMLElement {
  if (!(element instanceof HTMLElement)) {
    throw new Error("Expected HTMLElement");
  }
}

const el = document.querySelector(".missing") as unknown;
assertElement(el); // Type narrows to HTMLElement
el.innerText = "Hello"; // Type-safe
```

---

## Part 3: Advanced Narrowing Patterns

### 3.1 In Operator & Property Narrowing

The **`in` operator** checks if a property exists on an object, enabling type narrowing for objects with optional properties.

```typescript
type Cat = { meow: () => void };
type Dog = { bark: () => void };
type Animal = Cat | Dog;

function speak(animal: Animal) {
  if ("meow" in animal) {
    animal.meow(); // animal: Cat
  } else {
    animal.bark(); // animal: Dog
  }
}

// Nested property narrowing
type Response =
  | { success: true; data: string }
  | { success: false; error: Error };

function handle(response: Response) {
  if ("data" in response) {
    console.log(response.data); // response: success branch
  } else {
    console.log(response.error); // response: error branch
  }
}
```

### 3.2 Logical Operators & Truthiness

Logical operators narrow types: `&&` narrows on truthiness, `||` narrows negatively, `??` narrows for nullish.

```typescript
// && narrows to truthy
function logLength(str: string | null) {
  if (str && str.length > 0) {
    console.log(str.length); // str: string (not null, truthy)
  }
}

// || narrows to falsy/truthy alternative
function getName(user: { name?: string } | null): string {
  return user?.name || "Anonymous"; // Returns string or "Anonymous"
}

// ?? narrows for nullish
function getValue(value: string | null | undefined): string {
  return value ?? "default"; // Nullish returns "default"
}
```

---

## Interview Questions

**Q1: Explain type narrowing and how `typeof`/`instanceof` enable it.**

Type narrowing refines variable types in specific code branches. `typeof` checks primitive types (`string`, `number`); `instanceof` checks class prototypes. Type checker tracks control flow and narrows types in each branch.

**Q2: What are type predicates and how do they enable custom narrowing?**

Type predicates return `value is Type`, enabling custom narrowing logic. Example: `function isUser(value): value is User { ... }`. Type checker trusts the predicate and narrows accordingly, enabling type-safe operations on narrowed types.

**Q3: Explain discriminated unions and why they're superior to conditional narrowing.**

Discriminated unions have a common property determining the type. This enables exhaustive narrowing: the type checker ensures all cases are handled. More readable than nested conditionals; prevents bugs from missing cases.

**Q4: How do assertion functions differ from type predicates?**

Type predicates return `value is Type` for narrowing in conditionals. Assertion functions return `asserts condition is Type`, throwing if false and narrowing permanently afterward. Use predicates for conditionals; use assertions for preconditions.

---

## Key Takeaways

1. **Type narrowing refines types in branches** - Type checker tracks control flow
2. **Typeof narrows primitives; instanceof narrows classes** - Use `typeof` for primitive checks, `instanceof` for class checks
3. **Type predicates enable custom narrowing** - Return `value is Type` for type-safe narrowing
4. **Discriminated unions enable exhaustive narrowing** - Common property determines type
5. **Assertion functions narrow permanently** - Throw if false; type narrowed afterward
6. **Exhaustiveness checking prevents missed cases** - Use `never` to catch unhandled types
7. **In operator enables property-based narrowing** - Check property existence to distinguish types