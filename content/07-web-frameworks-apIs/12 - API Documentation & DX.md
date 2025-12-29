# API Documentation & Developer Experience: Building Usable APIs

## Introduction & Purpose

An API is only useful if developers can understand it, use it, and integrate it into their systems. **API documentation and developer experience determine whether an API succeeds or fails**, regardless of underlying technical quality.

This document explores how to design APIs for usability: documentation generation, client libraries, API discovery, and the infrastructure that enables developers to work effectively.

## Core Concepts & Internal Architecture

### OpenAPI/Swagger Specification

OpenAPI is a **language-agnostic specification** that describes REST APIs:

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
  description: API for managing users

servers:
  - url: https://api.example.com
    description: Production

paths:
  /users:
    get:
      summary: List users
      operationId: listUsers
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 10
      responses:
        '200':
          description: List of users
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
    post:
      summary: Create user
      operationId: createUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: Invalid input

  /users/{id}:
    get:
      summary: Get user
      operationId: getUser
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: User details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found

components:
  schemas:
    User:
      type: object
      required:
        - id
        - name
        - email
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email
        created_at:
          type: string
          format: date-time

    CreateUserRequest:
      type: object
      required:
        - name
        - email
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        email:
          type: string
          format: email
```

**Architectural consequence:** OpenAPI becomes **single source of truth** for API contract.

### Code Generation from Specifications

From OpenAPI spec, tools generate **client libraries and server stubs**:

**OpenAPI Generator:**
```bash
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o generated/

# Generates TypeScript Axios client
```

**Generated Client (TypeScript):**
```typescript
import { UsersApi } from './generated'

const api = new UsersApi()

// Fully typed; IDE knows all methods and types
const user = await api.getUser(123)
// TypeScript knows: user has properties id, name, email, etc.

const newUser = await api.createUser({
    name: 'Alice',
    email: 'alice@example.com'
})
```

**Generated Server (Node.js):**
```typescript
import { UsersApiController } from './generated'

// Must implement these methods
class UserController extends UsersApiController {
    async listUsers(limit?: number) {
        // Implementation
    }
    
    async createUser(createUserRequest: CreateUserRequest) {
        // Implementation
    }
}
```

**Architectural benefit:** **Type safety across client and server**. Changes to spec automatically update clients.

### Interactive API Documentation

**Swagger UI** generates interactive documentation from OpenAPI:

```html
<!-- Auto-generated from openapi.yaml -->
<swagger-ui data-url="openapi.yaml"></swagger-ui>
```

**User experience:**
1. Browse all endpoints
2. See request schema with examples
3. Try endpoints directly in browser (send requests, see responses)
4. View response schemas
5. Understand error responses

**Alternative: ReDoc**

```html
<redoc spec-url='openapi.yaml'></redoc>
```

**ReDoc advantages:**
- Better mobile experience
- Cleaner, more readable format
- Good for public-facing documentation

### API Sandboxes and Mock Servers

**Postman Collections** for manual testing:

```json
{
  "info": {
    "name": "User API Collection",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "List Users",
      "request": {
        "method": "GET",
        "url": "{{baseUrl}}/users"
      }
    }
  ],
  "variable": [
    {
      "key": "baseUrl",
      "value": "https://api.example.com"
    }
  ]
}
```

**Mock servers** (Prism, Mockito) generate sample responses:

```bash
# Start mock server from OpenAPI spec
prism mock openapi.yaml

# Mock server runs on :4010
curl http://localhost:4010/users
# Returns sample response from spec
```

**Architectural benefit:** Developers can test API before implementation.

### Typed API Clients

**tRPC**: TypeScript-first RPC framework

```typescript
// Server defines schema
export const appRouter = router({
    users: router({
        get: publicProcedure
            .input(z.object({ id: z.number() }))
            .query(async ({ input }) => {
                return await db.users.findById(input.id)
            })
    })
})

// Client has full type safety
const user = await trpc.users.get.query({ id: 1 })
// TypeScript knows user's shape; no schema files needed
```

**Orval**: Generate React Query hooks from OpenAPI

```typescript
// Input: OpenAPI spec
// Output: Type-safe React hooks

const { data: user } = useGetUser(123)
// user is fully typed; IDE autocomplete works
```

**Advantages:**
- No schema/types to maintain separately
- Single source of truth
- Full type safety end-to-end

### Versioning Strategies

APIs evolve. How do you handle breaking changes?

#### URI Versioning

```
/v1/users
/v2/users

GET /v1/users → Returns 2020-era schema
GET /v2/users → Returns 2024-era schema
```

**Pros:** Explicit, clear in URLs, easy routing
**Cons:** Duplicates code, clutters API namespace

#### Header Versioning

```
GET /users
Accept-Version: 1
→ Returns v1 response

GET /users
Accept-Version: 2
→ Returns v2 response
```

**Pros:** Clean URLs, modern
**Cons:** Version invisible, harder to debug

#### Semantic Versioning

```
v1.0.0 → v1.0.1 (patch: bug fix, backward compatible)
v1.0.0 → v1.1.0 (minor: new features, backward compatible)
v1.0.0 → v2.0.0 (major: breaking changes)
```

**Strategy:** Only new major versions are breaking. Minor/patch are backward compatible.

### Deprecation and Migration

**Mark deprecated fields/endpoints:**

```openapi
paths:
  /users:
    get:
      deprecated: true  # Mark as deprecated
      description: |
        **DEPRECATED** Use `/v2/users` instead.
        This endpoint will be removed on 2024-12-31.
```

**Timeline:**

```
2024-01-01: Announce deprecation (3 months notice)
2024-04-01: Stop accepting new API keys for v1
2024-07-01: Return deprecation warnings in headers
2024-12-31: Shut down endpoint; return 410 Gone
```

**Architectural consequence:** Gradual deprecation prevents breaking clients suddenly.

### Changelogs and Release Notes

**Semantic versioning enables clear changelogs:**

```markdown
# Changelog

## v2.1.0 (2024-01-15)

### Added
- New `include` parameter for embedding related resources
- GraphQL endpoint at `/graphql`

### Changed
- Response timestamps now use ISO8601 format (was Unix timestamps)
- Rate limit headers renamed (X-RateLimit-* → RateLimit-*)

### Deprecated
- `/v1/users` endpoint (use `/v2/users`)

### Fixed
- Fixed 500 error when posting with null body
```

**Architectural benefit:** Developers know what changed and whether they need to update.

## Common Problems & Failure Scenarios

### Problem 1: Documentation Out of Sync

**Scenario:**

```
OpenAPI spec says: POST /users returns 201
Code actually returns: 200

Developer relies on spec
    ↓
Gets status 200 instead of expected 201
    ↓
Code breaks
```

**Root cause:** Documentation updated separately from code; easy to diverge.

**Solution: Generate docs from code**

```typescript
// Use decorators to document
@Post('/users')
@HttpCode(201)
@ApiResponse({ status: 201, type: User })
async createUser(@Body() dto: CreateUserDto) {
    return this.userService.create(dto)
}

// Tool reads decorators, generates OpenAPI spec
```

**Benefits:** Single source of truth; impossible to diverge.

### Problem 2: Unclear Error Responses

**Scenario:**

```
Error response:
{
    "error": "invalid"
}

Developer confused: What was invalid? Email? Name? Format?
```

**Root cause:** Vague error messages.

**Better response:**

```json
{
    "error": "validation_error",
    "message": "Request validation failed",
    "details": [
        {
            "field": "email",
            "message": "Email must be valid",
            "code": "INVALID_EMAIL"
        }
    ]
}
```

**Architectural consequence:** Detailed errors enable client error handling.

### Problem 3: Missing Rate Limit Information

**Scenario:**

```
Developer makes requests
    ↓
Suddenly gets 429 Too Many Requests
    ↓
No information about when limits reset
    ↓
"How do I know when I can retry?"
```

**Solution: RateLimit headers**

```
HTTP/1.1 429 Too Many Requests
RateLimit-Limit: 1000
RateLimit-Remaining: 0
RateLimit-Reset: 1234567890
```

Tells developer:
- Limit: 1000 requests per period
- Remaining: 0 requests available
- Reset: Unix timestamp when limit resets

### Problem 4: Undocumented Pagination Format

**Scenario:**

```
Response:
{
    "data": [...],
    "next": "abc123"
}

Developer confused: What is "next"? How do I use it?
```

**Better: Standard pagination**

```json
{
    "data": [...],
    "pagination": {
        "total": 1000,
        "limit": 50,
        "offset": 0,
        "nextCursor": "abc123"
    },
    "links": {
        "self": "/users?offset=0&limit=50",
        "next": "/users?cursor=abc123&limit=50"
    }
}
```

Clear, standard format.

## Design Decisions & Trade-offs

### Trade-off 1: Automatic Generation vs. Hand-Written Docs

**Automatic (from code):**
- Pros: Always in sync, less work
- Cons: Requires framework support, less flexibility

**Hand-written:**
- Pros: Fully customizable, can add examples/tutorials
- Cons: Diverges from code easily, extra work

**Best practice:** Automatic + hand-written
```
Auto-generated: API reference (endpoints, schemas)
Hand-written: Guides, tutorials, use cases
```

### Trade-off 2: URI Versioning vs. Header Versioning

**URI (v1/, v2/):**
- Pros: Explicit, easy to route, good for caching
- Cons: Duplicated code, cluttered URLs

**Header:**
- Pros: Clean URLs, industry-modern
- Cons: Version invisible, harder to debug

**Decision:** URI versioning for public APIs (easier for developers); header versioning for internal APIs.

## When to Implement

### For Public APIs

- ✅ OpenAPI spec (code-generated)
- ✅ Swagger UI + ReDoc
- ✅ Postman collection
- ✅ Code examples in multiple languages
- ✅ Detailed changelogs
- ✅ Deprecation warnings
- ✅ Mock server for testing

### For Internal APIs (within organization)

- ✅ OpenAPI spec (code-generated)
- ⚠️ Basic Swagger UI (may skip if small team)
- ✅ Postman collection
- ✅ Basic changelogs

### For Private APIs (B2B)

- ✅ OpenAPI spec
- ✅ Swagger UI
- ✅ Code examples
- ✅ Interactive sandbox (Postman Pro)
- ✅ Detailed error documentation

---

## Conclusion

**Developer Experience determines API adoption.** An API with poor documentation will be abandoned regardless of technical quality.

Key principles:

1. **Single source of truth** — Generate docs from code
2. **Interactive documentation** — Let developers try endpoints
3. **Clear error messages** — Help developers debug quickly
4. **Code examples** — Show common patterns
5. **Semantic versioning** — Clear, predictable evolution
6. **Gradual deprecation** — Don't break clients suddenly
7. **Standard formats** — Use OpenAPI, follow REST conventions

The best API is not the most powerful; it's the easiest to use.
