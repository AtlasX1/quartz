# Document Databases: Structure, Schema, and Distributed Architecture

## Purpose & Problem Space

Document databases store data as self-describing documents (typically JSON/BSON) organized into collections, eliminating the need for predefined schemas and joins. They address problems that relational databases handle awkwardly: highly nested structures, polymorphic data, and rapid schema evolution.

**Core problems addressed:**
- Storing hierarchical/nested data without normalization
- Supporting schema-less or schema-on-read models
- Eliminating expensive joins for related data
- Handling polymorphic data (documents with different structures)
- Scaling horizontally through sharding while maintaining strong consistency
- Providing multi-document transactions without distributed locking

Document databases like MongoDB combine some relational properties (transactions, consistency) with schema flexibility and distributed scalability, occupying a middle ground between strict relational and loosely-consistent NoSQL systems.

---

## Core Concepts & Internal Architecture

### Documents and Collections

**Documents:**
The fundamental unit of storage. Unlike relational tables with fixed columns, documents are self-describing JSON/BSON objects:

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "user_id": 42,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "address": {
    "street": "123 Main St",
    "city": "Springfield",
    "zip": "12345"
  },
  "orders": [
    { "order_id": 101, "total": 150.00, "status": "shipped" },
    { "order_id": 102, "total": 200.00, "status": "pending" }
  ],
  "tags": ["vip", "verified"],
  "created_at": ISODate("2023-01-15T10:30:00Z"),
  "metadata": null
}
```

Key characteristics:
- **Atomic:** A document is the atomic unit of storage; updates to a document are atomic
- **Self-Describing:** Structure defined within the document; no separate schema definition required
- **Nested:** Can contain nested objects and arrays arbitrarily deep
- **Flexible:** Different documents in the same collection can have different structures
- **Typed:** Fields have types (string, number, date, ObjectId, array, etc.)

**Collections:**
Grouping of documents (roughly equivalent to relational tables). Collections are:
- **Schema-Flexible:** No enforced schema (default)
- **Unordered:** Documents not guaranteed in any order (use indexes for ordering)
- **Indexed:** Indexes on collection fields accelerate queries

**The _id Field:**
Every document must have a unique `_id` field (primary key). Default behavior:
- If not provided, MongoDB generates a 12-byte ObjectId (timestamp + machine + process + counter)
- ObjectIds are sortable and include embedded timestamps
- Can be any unique value (GUID, integer, compound)

### Schema Models: Schema-On-Read vs. Schema-On-Write

**Schema-On-Read (Document Default):**
No enforced schema; applications interpret document structure at read time.

Pros:
- Flexibility: documents can have different structures
- Rapid iteration: schema changes require no migration
- Handles polymorphism naturally: discount documents and regular documents in same collection

Cons:
- Application responsibility for structure: code must handle optional fields, variations
- No database-level validation: invalid documents possible
- Query complexity: must account for structure variations

**Schema-On-Write (Optional Validation):**
Define schema validation in MongoDB; documents must conform before insertion:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["user_id", "email"],
      properties: {
        user_id: { bsonType: "int" },
        email: { bsonType: "string", pattern: "^.+@.+$" },
        age: { bsonType: "int", minimum: 0, maximum: 150 }
      }
    }
  }
})
```

Pros:
- Database enforces constraints
- Similar to relational databases (familiar model)
- Prevents invalid documents

Cons:
- Schema changes still require validation updates
- Reduces flexibility

Most MongoDB applications use a hybrid: schema-on-read in application (documented, enforced by code) without database-level validation.

### Data Modeling Strategies

**Embedding (Denormalization):**
Store related data in a single document:

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "user": {
    "name": "Alice",
    "email": "alice@example.com"
  },
  "orders": [
    { "order_id": 101, "total": 150 },
    { "order_id": 102, "total": 200 }
  ]
}
```

Pros:
- Single document contains all related data
- Single query retrieves complete information
- Atomic operations (order and user information consistent)
- No JOINs required

Cons:
- Data duplication if referenced elsewhere
- Document size grows (large arrays problematic)
- Update anomalies (order data duplicated if also in orders collection)
- Cannot search across embedded fields efficiently without special indexes

**Document Referencing (Normalization):**
Store relationships using references (document IDs):

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "user": { "id": ObjectId("..."), "name": "Alice" },
  "order_ids": [ObjectId("..."), ObjectId("...")]
}
```

Then look up referenced documents separately:

```javascript
db.orders.find({ "_id": { $in: user.order_ids } })
```

Pros:
- Avoid redundancy
- Smaller documents
- Easier to update (modify order once)

Cons:
- Requires multiple queries (application-level joins)
- Complex operations (transactions needed for consistency)
- No atomic updates across documents (now with multi-document transactions)

**The One-to-Many Dilemma:**
- One user, many orders
- If orders always accessed with user (user's order history), embed
- If orders independently queried (all orders in date range), reference

**The Many-to-Many Complexity:**
- Many students, many courses
- Options:
  1. **Array of IDs:** `students: [student_id1, student_id2, ...]` in course document
  2. **Lookup Collection:** Separate enrollment documents linking students to courses
  3. **Application Joins:** Query separately

Choice depends on query patterns and data size.

### Indexing in Document Databases

MongoDB indexes are similar to relational databases but tailored to document queries:

**Single-Field Indexes:**
```javascript
db.users.createIndex({ "email": 1 })
```

Indexes documents by email field, supporting queries like:
```javascript
db.users.find({ "email": "alice@example.com" })
```

**Compound Indexes:**
```javascript
db.orders.createIndex({ "user_id": 1, "created_at": -1 })
```

Indexes by user_id ascending, then created_at descending within each user. Effective for:
```javascript
db.orders.find({ "user_id": 42, "created_at": { $gt: ISODate("2023-01-01") } })
```

**Embedded Field Indexes:**
```javascript
db.users.createIndex({ "address.city": 1 })
```

Indexes nested fields within embedded documents.

**Array Indexes (Multikey Indexes):**
```javascript
db.posts.createIndex({ "tags": 1 })
```

When a document has an array field, the index includes one entry per array element. Query `{ "tags": "mongodb" }` efficiently finds documents with that tag.

**Text Indexes:**
```javascript
db.articles.createIndex({ "content": "text", "title": "text" })
```

For full-text search supporting substring matching and stemming.

**Geospatial Indexes:**
```javascript
db.locations.createIndex({ "coordinates": "2dsphere" })
```

For geographic queries (find documents near coordinates, within polygon, etc.).

### Query Language and CRUD Operations

**INSERT (Write Operations):**
```javascript
db.users.insertOne({ name: "Alice", email: "alice@example.com" })
db.users.insertMany([...])
```

- Single document insertion is atomic
- Multiple inserts (insertMany) are not atomic (subsequent inserts fail if collision)

**FIND (Read Operations):**
```javascript
db.users.find({ "email": "alice@example.com" })  // Equality filter
db.users.find({ "age": { $gt: 18, $lt: 65 } })   // Range filter
db.users.find({ "tags": { $in: ["vip", "premium"] } })  // Array match
```

Supports complex filters with operators:
- Comparison: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`
- Logical: `$and`, `$or`, `$not`, `$nor`
- Array: `$in`, `$nin`, `$all`, `$elemMatch`
- Regular expressions: `$regex` for pattern matching

**UPDATE Operations:**
```javascript
db.users.updateOne(
  { "_id": ObjectId("...") },
  { $set: { "email": "newemail@example.com" } }
)

db.users.updateMany(
  { "status": "inactive" },
  { $set: { "last_active": null } }
)
```

Operators for complex updates:
- `$set`: Set field value
- `$unset`: Remove field
- `$inc`: Increment numeric field
- `$push`: Add to array
- `$pull`: Remove from array
- `$rename`: Rename field

**DELETE Operations:**
```javascript
db.users.deleteOne({ "_id": ObjectId("...") })
db.users.deleteMany({ "status": "deleted" })
```

### Aggregation Pipeline

The aggregation pipeline is MongoDB's data transformation framework, conceptually similar to SQL GROUP BY and joins but more powerful:

```javascript
db.orders.aggregate([
  { $match: { "user_id": 42, "created_at": { $gte: ISODate("2023-01-01") } } },
  { $group: { "_id": "$status", "total": { $sum: "$amount" }, "count": { $sum: 1 } } },
  { $sort: { "total": -1 } },
  { $limit: 10 }
])
```

**Pipeline Stages:**

- **$match:** Filter documents (equivalent to WHERE)
- **$project:** Select/transform fields
- **$group:** Group by specified fields and compute aggregations
- **$sort:** Order documents
- **$limit:** Take first N documents
- **$skip:** Skip first N documents
- **$lookup:** Join with another collection (performs LEFT join)
- **$unwind:** Flatten arrays (expand each array element into separate document)
- **$facet:** Create multiple pipelines
- **$redact:** Conditional document filtering based on field values
- **$out:** Write results to another collection

**Key Properties:**

- **Pipelined Execution:** Each stage outputs what next stage inputs; optimization moves $match early to filter before grouping
- **Lazy Evaluation:** Stages executed in optimal order (planner reorders for efficiency)
- **Expressive:** Arbitrary transformations, conditional logic, window functions

The aggregation pipeline is MongoDB's strength for analytical queries; while individual stages are less optimized than relational SQL, the pipelined model is powerful for complex transformations.

### Transactions

**Single-Document Transactions (Always Atomic):**
Operations on a single document are atomic:
```javascript
db.users.updateOne(
  { "_id": user_id },
  { $inc: { "balance": -100 }, $push: { "transactions": transaction } }
)
```

Updates to balance and transaction history happen together, or not at all.

**Multi-Document Transactions (MongoDB 4.0+):**
Transactions can span multiple documents:

```javascript
const session = db.getMongo().startSession()
session.startTransaction()

try {
  db.users.updateOne({ "_id": user_id }, { $inc: { "balance": -100 } }, { session })
  db.transactions.insertOne({ user_id, amount: 100, ... }, { session })
  session.commitTransaction()
} catch (e) {
  session.abortTransaction()
  throw e
}
```

Pros:
- Multiple operations guaranteed to succeed or fail together
- Stronger consistency than eventual consistency

Cons:
- Performance cost (overhead, latency)
- Blocking (transaction holds locks)
- Limited to single replica set (can't span sharded clusters across partitions)
- Application must handle rollback and retries

Most MongoDB applications avoid multi-document transactions when possible, preferring single-document atomic operations or designing documents to be self-sufficient units.

---

## Consistency, Performance & Reliability Challenges

### Data Duplication and Update Anomalies

Document embedding leads to natural denormalization:

**Update Anomaly Example:**
If user address is embedded in multiple documents (user profile, order shipping):
```json
// User collection
{ "user_id": 5, "address": { "city": "Springfield" } }

// Orders collection
{ "order_id": 101, "user_id": 5, "shipping_address": { "city": "Springfield" } }
```

When user moves, updating address in user collection doesn't update orders. Consistency maintained only through application logic or periodic batch updates.

**Causes:**
- Schema flexibility permits redundancy
- Joins discouraged (data kept together)
- Application logic responsible for consistency

**Mitigation:**
- Design documents as atomic units (don't embed same data in multiple places)
- Use transactions for multi-document consistency
- Denormalize only truly independent copies (caching)
- Document data ownership rules

### Large Documents and Array Growth

Documents have size limits (16MB in MongoDB) and unbounded arrays cause problems:

**Array Growth Problem:**
```json
{
  "user_id": 5,
  "orders": [
    // Array grows over time; old orders stay
    // After years, thousands of orders embedded
  ]
}
```

Fetching user requires reading entire document (all orders) even if only basic user info needed. Queries slow as array grows.

**Solutions:**
- Cap array size using $slice operator
- Move to referenced collection (separate orders collection)
- Archive old data

### Sharding Challenges

Unlike relational databases, MongoDB's distributed architecture introduces consistency challenges:

**Shard Key Selection:**
Shards are chosen by hash/range of shard key. Poor shard key choice causes:

```javascript
// Bad shard key: only 2 distinct values
db.users.shardCollection("users", { "is_premium": 1 })
// All non-premium users → shard 1, all premium → shard 2
// Load imbalanced, hotspot on shard 1
```

Good shard keys are:
- High cardinality (many distinct values)
- Evenly distributed (no values with many documents)
- Immutable (don't change after insertion)

**Write Concerns and Durability:**
By default, MongoDB acknowledges writes after applying in memory (not disk). Crash loses unflushed writes.

Write concerns:
- **0:** No acknowledgment
- **1:** Acknowledgment from primary (default)
- **W(N):** Acknowledgment from N replicas
- **"majority":** Acknowledgment from majority of replicas (slow but durable)

High durability requires waiting for replication, increasing latency.

### Distributed Transactions Limitations

MongoDB multi-document transactions have limitations:

- **Single Replica Set Only:** Cannot span across shards (distributed transaction is impossible)
- **No Distributed Locks:** Serializable isolation not guaranteed across shards
- **Performance Overhead:** Transactions slow; blocking other operations

For large-scale systems, multi-document transactions become bottlenecks. Applications prefer:
- Single-document atomicity (design documents atomically complete)
- Application-level consistency (saga pattern, eventual consistency)

### Eventual Consistency in Replicated Systems

When replication is asynchronous, replicas lag:

**Read from Secondary Risk:**
```javascript
db.users.find({}, { readPreference: "secondary" })
```

May return stale data if secondary hasn't replicated latest writes. Causes inconsistencies:
- Insert user, read from secondary immediately, see old data
- Create order, read user from secondary, user doesn't exist yet

**Causal Consistency:**
MongoDB offers causal consistency: if you write, then read from any server, you see your write. Other users' concurrent writes may not be visible.

Requires application to track causal token and pass to next operation—complex for web applications with multiple requests per user.

---

## Design Decisions & Trade-offs

### Embedding vs. Referencing Strategy

**Embed When:**
- Data access is hierarchical (always retrieve together)
- Relationship is one-to-few (one user with few addresses)
- Array size is bounded (emails array has max 5 items)
- Updates are atomic (address and user updated together)

**Reference When:**
- Data accessed independently (orders queried separately from users)
- Many-to-many relationships (students in many courses)
- Array unbounded (potentially thousands of related documents)
- Updates separate (order status changes independently of user info)

**Hybrid Strategy:**
```json
{
  "user_id": 5,
  "name": "Alice",
  "address": { "street": "123 Main", "city": "Springfield" },  // Embedded
  "order_count": 42,
  "order_ids": [ObjectId(...), ObjectId(...), ...]  // Referenced
}
```

Embed commonly-accessed data; reference infrequently-accessed. Use denormalization for fast reads (order_count) but maintain order_ids for access.

### Schema Flexibility vs. Validation

**Permissive (No Validation):**
- Development speed: no schema migrations
- Flexibility: fields can vary
- Risk: inconsistent data, application complexity

**Strict (Full Validation):**
- Data quality: enforced consistency
- Maintenance: schema changes require planning
- Overhead: slower writes, validation complexity

**Pragmatic Middle Ground:**
- Use schema-on-read (document structure in application code)
- Add database validation after schema stabilizes
- Generate schema migrations for major changes
- Document implicit schemas

### Replication and Consistency Trade-offs

**Asynchronous Replication (High Availability):**
- Write acknowledged immediately (fast)
- Replicas lag behind primary (eventual consistency)
- Primary crash loses data not yet replicated
- Better for: user-facing services, high throughput

**Synchronous Replication (High Durability):**
- Write acknowledged after replication (slower)
- Strong durability (data replicated before acknowledging)
- Replication lag = write latency (slower)
- Better for: financial systems, critical data

Most systems use asynchronous replication for user data (acceptable eventual consistency) and synchronous for critical operations.

### Sharding Strategy Selection

**Range-Based Sharding:**
```javascript
// Shard key: user_id range
// Shard 1: user_id 1-1000000
// Shard 2: user_id 1000001-2000000
```

Pros: Range queries on shard key efficient
Cons: Hot shards (uneven distribution), rebalancing expensive

**Hash-Based Sharding:**
```javascript
// Shard key: hash(user_id)
```

Pros: Even distribution
Cons: Range queries inefficient, rebalancing very expensive

**Hashed Sharding + Prefix:**
Modern approach: hash key but preserve some ordering information for range optimization.

---

## Alternative Models & Comparisons

### Document Databases vs. Relational

| Aspect | Relational | Document |
|--------|-----------|----------|
| Schema | Fixed, enforced | Flexible, optional |
| Joins | Normalized, explicit | Embedded, implicit |
| Consistency | ACID transactions | Single-doc atomicity + multi-doc trans |
| Scalability | Vertical | Horizontal (sharding) |
| Queries | Complex SQL | Simple document access |
| Data Shape | Uniform rows | Varied documents |
| Denormalization | Anomalies risk | Natural |

### Document vs. Other NoSQL Models

**Document vs. Key-Value:**
Documents support complex queries (filters, aggregations); key-value only supports lookups. Documents slower but more queryable.

**Document vs. Column-Family:**
Column families optimized for time-series and analytics; documents for operational data.

**Document vs. Graph:**
Graph databases optimize for relationships; documents require application logic for joins.

---

## When to Use and When to Avoid

### Ideal for Document Databases:

- **User Profiles:** Polymorphic data (different user types, different fields)
- **Product Catalogs:** Variable attributes across categories
- **Content Management:** Nested content, rapid schema changes
- **API Responses:** Caching API responses as documents
- **Mobile Apps:** Offline-first with eventual sync
- **Microservices:** Each service manages its own collection

### Avoid Document Databases for:

- **Strict Consistency Required:** Financial transactions, inventory (multi-document transactions slow)
- **Complex Joins:** Analytical queries across many entities (relational better)
- **Highly Normalized Data:** Heavily related data benefits from relational design
- **Real-Time Analytics:** Time-series databases more efficient
- **Simple Key-Value Access:** Redis/Memcached faster

### Scalability Profile:

- **< 10GB:** Single instance, relational or document fine
- **10GB - 100GB:** Replication recommended, document scaling starts
- **100GB+:** Sharding necessary; document databases excel, relational sharding difficult
- **Massive Scale (>1TB):** Document databases with sharding

---

## Summary

Document databases occupy a middle ground between strict relational systems and loosely-consistent NoSQL. They sacrifice some relational properties (complex queries, strong consistency across many documents) but gain schema flexibility, horizontal scalability, and performance through denormalization. Understanding data modeling strategies (embedding vs. referencing), indexing, aggregation pipelines, and distributed consistency trade-offs is essential for effective MongoDB usage. The fundamental design principle: treat documents as atomic units, minimize cross-document dependencies, and accept the consistency challenges inherent in distributed systems.
