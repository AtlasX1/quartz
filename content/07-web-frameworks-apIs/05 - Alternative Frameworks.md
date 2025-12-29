# Alternative Web Frameworks: Koa, Hapi, and AdonisJS

## Introduction & Purpose

While Express, Fastify, and NestJS dominate the Node.js ecosystem, alternative frameworks offer distinct architectural approaches and design philosophies that solve specific problems or appeal to particular development styles.

This document explores three significant alternatives: **Koa** (minimal, async-first), **Hapi** (configuration-heavy, enterprise-grade), and **AdonisJS** (full-stack MVC). Understanding these frameworks reveals how different design principles lead to different trade-offs, and when each becomes the optimal choice.

## Koa: Async/Await Native Minimalism

### Introduction and Design Philosophy

Koa emerged in 2013 as a **lightweight successor to Express**, built from the ground up for async/await composition. While Express predates JavaScript Promises, Koa was designed when async/await was already a language feature (or imminent).

Koa's core insight: "Middleware should compose naturally with async/await, and the framework should be minimal enough not to impose constraints."

### Architecture and Core Concepts

#### Context Objects as Primary Abstraction

Koa wraps Node.js req/res in a single **context object**:

```javascript
const Koa = require('koa')
const app = new Koa()

app.use(async (ctx) => {
    ctx.body = { message: 'Hello' }
})
```

The context object contains:
- `ctx.request` — Koa request wrapper
- `ctx.response` — Koa response wrapper
- `ctx.req` — Raw Node.js request
- `ctx.res` — Raw Node.js response
- Properties dynamically added by middleware

**Architectural consequence:** A single `ctx` parameter replaces Express's separate `req, res, next`. This is simpler and enables convenient property attachment:

```javascript
app.use(async (ctx, next) => {
    ctx.userId = extractUserId(ctx.headers)
    await next()
})

app.use(async (ctx) => {
    const userId = ctx.userId // Available from previous middleware
    ctx.body = { userId }
})
```

#### Middleware as Functions Returning Promises

Koa middleware is fundamentally different from Express:

```javascript
// Express middleware
app.use((req, res, next) => {
    // Synchronous callback pattern
    next()
})

// Koa middleware
app.use(async (ctx, next) => {
    // Returns a Promise
    await next()
})
```

In Koa, middleware **returns immediately** after calling `next()`. The framework awaits the Promise:

```
Middleware 1
    ↓
    await next() → Middleware 2 (awaited)
    ↓ (returns after Middleware 2 completes)
Post-processing
```

**Architectural advantage:** No need for callbacks or manual state tracking. Errors thrown in async handlers automatically propagate:

```javascript
app.use(async (ctx, next) => {
    try {
        await next()
    } catch (err) {
        ctx.status = 500
        ctx.body = { error: err.message }
    }
})

app.use(async (ctx) => {
    const data = await database.query() // If fails, caught by error middleware
})
```

#### Routing and Middleware Stacking

Koa has **no built-in routing**. Routing requires third-party libraries (`@koa/router`, `koa-route`):

```javascript
const Router = require('@koa/router')
const router = new Router()

router.get('/users/:id', async (ctx) => {
    const userId = ctx.params.id
    ctx.body = { id: userId }
})

app.use(router.routes())
```

**Architectural consequence:** The framework is intentionally unopinionated. You can choose routing style, validation library, and everything else.

### Common Problems with Koa

**Problem 1: No standard error handling**

Koa provides minimal error handling. Developers must implement centralized error middleware:

```javascript
app.on('error', (err, ctx) => {
    // Global error handler
    console.error('Error:', err)
})
```

But without clear conventions, error handling becomes inconsistent across projects.

**Problem 2: Ecosystem fragmentation**

Unlike Express (which has standard middleware for everything), Koa's ecosystem is fragmented. Validation, body parsing, authentication—all require choosing between multiple libraries.

**Problem 3: Context mutation**

The shared context object can be mutated by any middleware, making it hard to reason about state:

```javascript
app.use(async (ctx, next) => {
    ctx.arbitraryData = { ... }
    await next()
})

// Later middlewares see this data; dependencies implicit
```

### When to Use Koa

- **Learning async/await composition** — Koa's minimalism teaches fundamental concepts
- **Specific architectural needs** — When standard frameworks don't fit
- **Small, focused APIs** — Where minimalism is an advantage
- **Teams that value flexibility** — Over enforced structure

### When NOT to Use Koa

- **Large enterprise teams** — Lack of standard practices causes inconsistency
- **Projects requiring stability** — Express and Fastify have larger ecosystems
- **Quick prototyping** — Express or Fastify is faster for standard patterns
- **Complex validation** — No built-in schema system

---

## Hapi: Configuration-Driven Enterprise Framework

### Introduction and Design Philosophy

Hapi (originally created by Walmart for Black Friday traffic) is built on a **configuration-first philosophy**: "Every behavior should be configurable, and configuration should be data-driven, not code-driven."

This contrasts sharply with Express (code-driven) and NestJS (decorator-driven).

### Architecture and Core Concepts

#### Server and Plugin System

Hapi organizes code through **plugins**, each with configuration:

```javascript
const Hapi = require('@hapi/hapi')

// Define a plugin
const usersPlugin = {
    name: 'users',
    version: '1.0.0',
    register: async (server, options) => {
        server.route({
            method: 'GET',
            path: '/users/{id}',
            handler: async (request, h) => {
                return { id: request.params.id }
            }
        })
    }
}

// Register plugin
const server = new Hapi.server({ port: 3000 })
await server.register(usersPlugin, { /* options */ })
await server.start()
```

**Architectural consequence:** Configuration is **first-class**. Every plugin receives `options` that modify behavior without code changes.

#### Route Configuration as Data

Routes in Hapi are data-driven:

```javascript
server.route({
    method: 'POST',
    path: '/users',
    config: {
        auth: 'jwt',  // Requires JWT authentication
        validate: {
            payload: Joi.object({  // Request body validation
                name: Joi.string().required(),
                email: Joi.string().email().required()
            })
        },
        tags: ['api', 'users'],
        description: 'Create a new user'
    },
    handler: async (request, h) => {
        // request.payload is guaranteed valid
        return h.response(user).code(201)
    }
})
```

The `config` object declares **all route metadata in one place**: auth requirements, validation rules, documentation, lifecycle events.

#### Lifecycle Events

Hapi defines **request lifecycle events** that plugins can hook into:

```javascript
server.ext('onRequest', async (request, h) => {
    // Runs before routing
    request.app.startTime = Date.now()
    return h.continue
})

server.ext('onPreAuth', async (request, h) => {
    // Before authentication
})

server.ext('onPostAuth', async (request, h) => {
    // After authentication, before handler
})

server.ext('onPreResponse', async (request, h) => {
    // Before response sent; can modify response
})
```

**Architectural advantage:** Clear, ordered lifecycle with extension points at specific phases.

#### Built-in Validation with Joi

Hapi includes **Joi validation** as first-class citizen:

```javascript
const schema = Joi.object({
    username: Joi.string().alphanum().min(3).max(30).required(),
    email: Joi.string().email().required(),
    age: Joi.number().integer().min(0).max(150)
})

server.route({
    method: 'POST',
    path: '/users',
    config: {
        validate: { payload: schema }
    },
    handler: async (request, h) => {
        // request.payload validated against schema
    }
})
```

Validation happens before handler execution; invalid requests are rejected automatically.

### Common Problems with Hapi

**Problem 1: Boilerplate in configuration**

Even simple routes require substantial configuration:

```javascript
server.route({
    method: 'GET',
    path: '/health',
    config: {
        auth: false,  // Explicitly disable auth
        cache: { expiresIn: 60000 },
        tags: ['monitoring']
    },
    handler: async (request, h) => {
        return { status: 'ok' }
    }
})
```

For simple endpoints, this feels verbose.

**Problem 2: Learning curve for advanced features**

Hapi is feature-rich (caching, authentication, validation). Using these features effectively requires deep documentation reading.

**Problem 3: Smaller ecosystem than Express**

While Hapi has excellent built-in features, the third-party plugin ecosystem is smaller. Specialized integrations may not exist.

### When to Use Hapi

- **Configuration-heavy applications** — Where behavior varies by environment
- **Large-scale enterprise systems** — Built for high-traffic scenarios
- **Strong validation requirements** — Joi integration is excellent
- **Complex authentication** — Built-in auth plugin is comprehensive
- **Teams that value consistency** — Configuration-driven approach enforces patterns

### When NOT to Use Hapi

- **Simple APIs** — Configuration overhead not justified
- **Rapid prototyping** — Slower to set up than Express/Fastify
- **Teams preferring minimal structure** — Koa or Express better fits
- **Learning Node.js** — Complexity obscures HTTP fundamentals

---

## AdonisJS: Full-Stack MVC Framework

### Introduction and Design Philosophy

AdonisJS is a **full-stack MVC framework** inspired by Laravel. It's comprehensive, opinionated, and designed for building complete web applications (not just APIs).

AdonisJS bundles: routing, ORM, migrations, authentication, session management, validation, templating—everything needed for a traditional web application.

### Architecture and Core Concepts

#### MVC Organization

AdonisJS enforces standard MVC structure:

```
app/
  Models/
    User.ts
  Controllers/
    UserController.ts
  Middleware/
    AuthMiddleware.ts
  Validators/
    CreateUserValidator.ts
database/
  migrations/
  seeders/
resources/
  views/
  css/
routes.ts
```

Each layer has explicit responsibility:
- **Models** — Database abstraction (ORM)
- **Controllers** — Request handlers
- **Middleware** — Cross-cutting concerns
- **Validators** — Input validation

#### Lucid ORM Integration

AdonisJS includes **Lucid**, a powerful ORM:

```typescript
import { BaseModel, column } from '@ioc:Adonis/Lucid/Orm'

export default class User extends BaseModel {
    @column({ isPrimary: true })
    public id: number

    @column()
    public name: string

    @column()
    public email: string
}

// Usage in controller
const user = await User.findOrFail(id)
await user.delete()
```

**Architectural consequence:** Database interactions are **object-oriented and type-safe**. Relations are declared as associations:

```typescript
export default class User extends BaseModel {
    @hasMany(() => Post)
    public posts: HasMany<typeof Post>
}

// Load user with posts
const user = await User.query().preload('posts')
```

#### Routing with Type Safety

AdonisJS provides **type-safe routing**:

```typescript
// routes.ts
Route.post('/users', 'UserController.store').validate(CreateUserValidator)

// UserController
export default class UserController {
    public async store({ request, response }: HttpContext) {
        const data = request.all() // Already validated
        const user = await User.create(data)
        return response.status(201).json(user)
    }
}
```

Routes can enforce validation at the routing layer, and controllers receive validated data.

#### Validators

AdonisJS provides **declarative validation**:

```typescript
export default class CreateUserValidator {
    public schema = schema.create({
        name: schema.string([rules.required(), rules.minLength(3)]),
        email: schema.string([rules.required(), rules.email()]),
        password: schema.string([rules.required(), rules.minLength(8)])
    })
}
```

### Common Problems with AdonisJS

**Problem 1: Opinionated structure limits flexibility**

If you need a non-standard architecture, AdonisJS's enforced MVC structure becomes a limitation.

**Problem 2: Full-stack complexity for API-only projects**

AdonisJS includes templating, session management, and other full-stack features unnecessary for REST APIs. This adds overhead.

**Problem 3: Smaller ecosystem than Express**

While AdonisJS is mature, the plugin ecosystem is smaller. Some integrations require custom implementation.

**Problem 4: Database-heavy design**

Everything flows through Lucid ORM. If you need raw SQL or non-relational databases, integration feels forced.

### When to Use AdonisJS

- **Full-stack web applications** — Traditional MVC applications with server-rendered HTML
- **Rapid application development** — All scaffolding included (migrations, seeders, ORM)
- **Teams from Rails/Laravel backgrounds** — Familiar patterns and conventions
- **Projects needing strict structure** — Enforced MVC prevents architectural inconsistency
- **Database-centric applications** — Lucid ORM simplifies data layer

### When NOT to Use AdonisJS

- **API-only services** — Unnecessary full-stack features add overhead
- **Microservices** — Overkill for focused, single-responsibility services
- **Rapid prototyping** — Setup overhead (still need database, migrations)
- **Non-relational applications** — Lucid is SQL-focused; NoSQL integration feels secondary
- **Teams needing flexibility** — Strict MVC can be constraining

---

## Comparative Analysis

### Philosophical Differences

| Framework | Philosophy | Target |
|-----------|-----------|--------|
| **Koa** | Minimal, composable | Developers who want control |
| **Hapi** | Configuration-driven, enterprise | Large teams, high-traffic systems |
| **AdonisJS** | MVC, batteries-included | Full-stack developers, rapid dev |

### Feature Comparison

| Feature | Koa | Hapi | AdonisJS |
|---------|-----|------|----------|
| **Built-in Validation** | No | Joi (excellent) | Yes (schema-based) |
| **ORM** | No | No | Lucid (included) |
| **Authentication** | Manual | Plugin | Built-in |
| **Middleware** | Simple | Complex | Standard |
| **Routing** | Third-party | Built-in | Built-in |
| **Type Safety** | Optional | Optional | Excellent (TypeScript) |
| **Performance** | Very fast (~30k req/s) | Fast (~25k req/s) | Good (~15k req/s) |
| **Learning Curve** | Gentle | Steep | Moderate |

### When to Choose Each

```
Simple API?
├─ Koa ✅ (minimal, composable)
└─ Fastify ✅ (faster, schema validation)

Complex API, large team?
├─ Hapi ✅ (configuration-driven reliability)
└─ NestJS ✅ (DI, modules, enterprise patterns)

Full-stack web application?
└─ AdonisJS ✅ (batteries-included MVC)

Rapid prototyping?
├─ Express ✅ (ecosystem)
└─ Fastify ✅ (speed + features)
```

---

## Conclusion

The Node.js framework ecosystem offers **distinct architectural philosophies**:

- **Koa** represents **minimal, composable design** for developers who value control and understanding
- **Hapi** represents **configuration-driven reliability** for teams that need predictability and complex features
- **AdonisJS** represents **full-stack conventions** for building complete applications quickly

The choice depends on **project scope, team structure, and priorities**:

- **MVPs and simple APIs** → Express or Fastify
- **Complex, configuration-heavy systems** → Hapi
- **Full-stack web applications** → AdonisJS
- **Learning or specialized needs** → Koa
- **Enterprise, large teams** → NestJS or Hapi

Understanding these alternatives clarifies the trade-offs inherent in any framework choice: minimalism vs. features, flexibility vs. structure, learning curve vs. productivity.
