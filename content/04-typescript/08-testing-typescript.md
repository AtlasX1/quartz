# Testing TypeScript: L4 Engineering Guide

## Part 1: TypeScript-Specific Testing Challenges

### 1.1 Type Safety in Tests

**Type safety extends to tests.** Testing libraries provide generics to ensure assertions match actual types. Incorrect test assertions are caught at compile time, not runtime.

**Test structure:** TypeScript tests follow JavaScript patterns (Jest, Vitest, Mocha) but with additional type safety.

```typescript
// Jest with TypeScript
describe("UserService", () => {
  it("should find user by id", async () => {
    const service = new UserService();
    const user = await service.findById(1);
    
    expect(user).toBeDefined();
    expect(user?.name).toBe("John"); // user is typed
    // expect(user?.invalidProp); // Error: invalidProp doesn't exist
  });

  it("should throw for invalid id", async () => {
    const service = new UserService();
    await expect(service.findById(-1)).rejects.toThrow("Invalid ID");
  });
});

// Typed mocking
type MockUserService = jest.Mocked<UserService>;

const mockService: MockUserService = jest.mocked(userService);
mockService.findById.mockResolvedValue({ id: 1, name: "John" });
```

### 1.2 Mocking Complex Types

**Mocking TypeScript interfaces and generic types** requires precise type definitions to maintain type safety through the mock.

```typescript
// Interface mocking
interface Database<T> {
  find(id: string): Promise<T | null>;
  save(item: T): Promise<void>;
  delete(id: string): Promise<void>;
}

// Typed mock factory
function createMockDatabase<T>(): jest.Mocked<Database<T>> {
  return {
    find: jest.fn(),
    save: jest.fn(),
    delete: jest.fn()
  };
}

// Usage with type safety
interface User {
  id: string;
  name: string;
}

const mockDb = createMockDatabase<User>();
mockDb.find("1").mockResolvedValue({ id: "1", name: "John" });

// Type errors caught
// mockDb.find("1").mockResolvedValue({ id: "1" }); // Error: missing name
```

---

## Part 2: Testing Patterns & Techniques

### 2.1 Generics in Tests

**Generic test utilities** enable reusable test helpers that work with any type while maintaining type safety.

```typescript
// Generic repository test helper
async function testRepositoryFind<T extends { id: string | number }>(
  repo: Repository<T>,
  testItem: T,
  newItem: T
) {
  // Save test item
  await repo.save(testItem);

  // Find should return the saved item
  const found = await repo.find(testItem.id);
  expect(found).toEqual(testItem);

  // Find non-existent should return null
  const notFound = await repo.find(newItem.id);
  expect(notFound).toBeNull();
}

// Usage with specific types
interface User {
  id: number;
  name: string;
}

const userRepo = new Repository<User>();
await testRepositoryFind(userRepo, { id: 1, name: "John" }, { id: 2, name: "Jane" });
// Type-safe: properties must match User type
```

### 2.2 Testing Utility Types

Utility types enable creating typed test scenarios and ensuring test data matches contracts.

```typescript
// Test factory with partial types
function createTestUser(overrides?: Partial<User>): User {
  return {
    id: 1,
    name: "John",
    email: "john@example.com",
    ...overrides
  };
}

const user = createTestUser({ name: "Jane" }); // Partial override

// Readonly types in tests
type ReadonlyUser = Readonly<User>;
const immutableUser: ReadonlyUser = createTestUser();
// immutableUser.name = "Bob"; // Error: readonly

// Pick specific properties for testing
type UserPreview = Pick<User, "id" | "name">;
const preview: UserPreview = { id: 1, name: "John" };
// const invalid: UserPreview = { id: 1, name: "John", email: "..." }; // Error

// Record type for test data
type UserFixtures = Record<"admin" | "user" | "guest", User>;
const fixtures: UserFixtures = {
  admin: createTestUser({ role: "admin" }),
  user: createTestUser({ role: "user" }),
  guest: createTestUser({ role: "guest" })
};
```

---

## Part 3: Advanced Testing Patterns

### 3.1 Type Predicates in Tests

Type predicates enable custom assertions that narrow types in test conditions.

```typescript
// Custom type assertion
function assertIsUser(value: unknown): asserts value is User {
  if (
    typeof value !== "object" ||
    value === null ||
    !("id" in value) ||
    !("name" in value)
  ) {
    throw new Error("Not a valid User");
  }
}

test("API returns valid user", async () => {
  const response = await fetch("/api/user/1");
  const data = await response.json();

  assertIsUser(data); // Type narrowed to User
  expect(data.name).toBe("John"); // Type-safe access
});

// Custom predicate for collections
function isUserArray(value: unknown): value is User[] {
  return Array.isArray(value) && value.every(v => "id" in v && "name" in v);
}

test("API returns user list", async () => {
  const response = await fetch("/api/users");
  const data = await response.json();

  assertIsUserArray(data);
  expect(data.length).toBeGreaterThan(0); // data: User[] (narrowed)
});
```

### 3.2 Testing Discriminated Unions

Discriminated unions enable exhaustive testing of all cases.

```typescript
// Discriminated union for results
type ApiResponse<T> =
  | { status: "success"; data: T }
  | { status: "error"; error: Error }
  | { status: "loading" };

// Exhaustive test for all cases
function testApiResponse<T>(response: ApiResponse<T>) {
  if (response.status === "success") {
    expect(response.data).toBeDefined(); // response: success branch
  } else if (response.status === "error") {
    expect(response.error).toBeInstanceOf(Error); // response: error branch
  } else {
    expect(response.status).toBe("loading"); // response: loading branch
  }
}

// Parameterized tests with discriminated unions
const testCases: Array<ApiResponse<string>> = [
  { status: "success", data: "Hello" },
  { status: "error", error: new Error("Failed") },
  { status: "loading" }
];

testCases.forEach(testCase => {
  test(`handles ${testCase.status}`, () => {
    testApiResponse(testCase);
  });
});
```

---

## Interview Questions

**Q1: How does TypeScript improve test reliability?**

TypeScript catches type errors at compile time, preventing entire classes of bugs. Test assertions are type-checked; incorrect assertions fail before running. Type safety extends to mocks and test fixtures, ensuring test data matches contracts.

**Q2: Explain typed mocking in TypeScript. Why is it important?**

Typed mocking ensures mock implementations match interface contracts. `jest.Mocked<T>` provides type-safe mock creation. Importance: prevents mock drift (mocks diverging from real implementations), catches regressions at compile time.

**Q3: How would you test a generic repository with multiple types?**

Create generic test utilities that work with any type: `async testRepositoryFind<T>(repo: Repository<T>, testItem: T)`. This enables reusable test logic with type safety per type. Alternative: parameterized tests with test data arrays.

**Q4: When should you use type predicates vs type assertions in tests?**

Type predicates return `value is T` for conditional narrowing; assertions return `asserts value is T` for permanent narrowing. Use predicates when testing function behavior; use assertions for preconditions (validate test data is correct before running assertions).

---

## Key Takeaways

1. **Type safety extends to test assertions** - Assertions are type-checked at compile time
2. **Generic test utilities enable reuse** - Parameterized tests work with multiple types
3. **Typed mocking prevents drift** - Mock implementations must match interface contracts
4. **Utility types enhance test data** - `Partial`, `Pick`, `Record` simplify test fixtures
5. **Type predicates enable custom assertions** - Exhaustive testing of all discriminated union cases
6. **Discriminated unions enable exhaustive tests** - All cases must be handled
7. **Test data should match contracts** - Use same interfaces as production code