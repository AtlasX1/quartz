# Relational Databases: Structure, Theory & SQL

## Purpose & Problem Space

Relational databases solve a fundamental problem in data management: how to store, organize, and retrieve structured data efficiently while maintaining integrity and consistency across concurrent operations. The relational model, pioneered by Edgar Codd in 1970, provides a mathematical foundation for organizing data into tables with well-defined relationships.

**Core problems addressed:**
- Organizing complex data without redundancy (normalization)
- Querying across multiple related entities efficiently
- Maintaining referential integrity across related tables
- Supporting complex analytical queries and aggregations
- Handling schema changes while preserving existing data

The relational model became the dominant paradigm because it balances theoretical rigor with practical expressiveness. Unlike hierarchical or network databases, the relational model uses a declarative query language (SQL) that separates the logical description of what data is needed from the physical implementation of how to retrieve it.

---

## Core Concepts & Internal Architecture

### The Relational Model Foundation

**Relations, Attributes, and Tuples:**
A relation in database theory is a set of tuples (rows) with identical structure. Each tuple is an unordered collection of attribute-value pairs. This mathematical foundation is crucial: relations are sets, meaning:
- Order of rows is logically irrelevant (though storage and retrieval order may differ)
- Duplicate tuples cannot exist (unless explicitly allowed via MULTISET extensions)
- Each attribute has a well-defined domain (data type)

**Keys and Uniqueness:**
- **Primary Key:** Minimal set of attributes that uniquely identifies each tuple. The database enforces uniqueness and NOT NULL constraints on the primary key.
- **Candidate Key:** Any set of attributes with uniqueness property; one is designated as primary.
- **Foreign Key:** Set of attributes referencing the primary key of another relation, establishing a logical relationship.
- **Composite Key:** Primary key composed of multiple attributes, often necessary in many-to-many relationships.

Keys are not merely indexing constructs—they define the logical schema. The primary key constraint ensures entity integrity; foreign keys implement referential integrity.

**Domains and Data Types:**
Each column is restricted to a specific domain (INT, VARCHAR, DATE, etc.). The domain defines:
- Valid values for the attribute
- Operations applicable to the attribute
- Storage requirements and indexing strategies

### Normalization Theory

Normalization is a process of decomposing relations to eliminate data anomalies and redundancy. It's grounded in functional dependencies—constraints where certain attributes determine others.

**Functional Dependencies (FDs):**
An attribute B is functionally dependent on attribute A if each A value corresponds to exactly one B value. Formally: if two tuples have the same A value, they must have the same B value.

**Normal Forms (Progressive Decomposition):**

1. **First Normal Form (1NF):** Every attribute value is atomic (no repeating groups or nested relations). This is foundational—without 1NF, traditional relational theory doesn't apply.

2. **Second Normal Form (2NF):** Remove partial dependencies. No non-key attribute is dependent on only part of a composite primary key. This prevents anomalies when updating partial keys.

3. **Third Normal Form (3NF):** Remove transitive dependencies. No non-key attribute depends on another non-key attribute. This ensures attributes describe only the entity identified by the primary key.

4. **Boyce-Codd Normal Form (BCNF):** Stronger than 3NF. Every determinant (attribute set determining others) is a candidate key. Eliminates anomalies in cases where 3NF allows them.

**Why Normalization Matters:**
- **Update anomalies:** Inserting, updating, or deleting data becomes complex when attributes are unnecessarily duplicated across multiple rows
- **Insertion anomalies:** Cannot insert partial information without violating constraints
- **Deletion anomalies:** Deleting one fact unintentionally removes other facts
- **Storage efficiency:** Normalized schemas eliminate redundancy, reducing disk I/O

### JOIN Operations: Reconstructing Relationships

Since normalization decomposes data across tables, JOINs reconstruct relationships at query time. This is computationally expensive but necessary for maintaining consistency.

**JOIN Types:**

- **INNER JOIN:** Returns tuples present in both relations, matching on join condition. The most common join type in normalized databases.
- **LEFT (OUTER) JOIN:** Returns all tuples from the left relation plus matching tuples from right. Introduces NULL for unmatched attributes.
- **RIGHT (OUTER) JOIN:** Symmetric to LEFT; all right tuples plus matches from left.
- **FULL OUTER JOIN:** Union of LEFT and RIGHT joins.
- **CROSS JOIN:** Cartesian product—every tuple from left paired with every tuple from right.

**Join Implementation Strategies (handled by query planner):**
- **Nested Loop Join:** For each tuple in outer relation, scan inner relation. O(m*n) complexity; used when one table is small.
- **Hash Join:** Build hash table of inner relation, probe with outer. O(m+n) average case; effective for large joins.
- **Sort-Merge Join:** Sort both relations on join key, merge. O(m log m + n log n); good for pre-sorted data or joining large tables.

### Aggregation and Window Functions

Aggregations summarize groups of rows:

**GROUP BY Logic:**
The GROUP BY clause partitions rows into groups sharing identical values for specified attributes. Aggregate functions (COUNT, SUM, AVG, MAX, MIN) operate on each group independently. The HAVING clause filters groups after aggregation (unlike WHERE, which filters before).

Conceptually, GROUP BY groups the relation, then applies the aggregation function to each group. Attributes in SELECT must either be in GROUP BY or be part of an aggregation—this constraint prevents ambiguous aggregations.

**Window Functions (Analytic Functions):**
Window functions compute values across a subset (window) of rows without collapsing groups like GROUP BY does. Each row retains its individual identity while having access to window-based computations.

Window functions include:
- **Ranking:** ROW_NUMBER, RANK, DENSE_RANK—assign order based on ordering criteria
- **Offset:** LAG, LEAD—access previous/next row's values
- **Aggregation:** SUM, AVG, COUNT over windows
- **Distribution:** PERCENT_RANK, CUME_DIST, NTILE

Window frames (ROWS BETWEEN) define which rows participate in computation: from N preceding rows to M following rows. This enables moving averages, running totals, and comparative analysis.

### Complex Query Constructs

**Subqueries & Derived Tables:**
A subquery is a SELECT within another SELECT. Subqueries can appear in:
- **WHERE clause:** Scalar subqueries compare a value; IN/EXISTS check membership
- **FROM clause (derived tables/CTEs):** Create virtual relations
- **SELECT clause:** Scalar subqueries returning single values

Correlation—where inner query references outer query's attributes—introduces dependence and often requires row-by-row execution.

**Common Table Expressions (WITH):**
CTEs provide named temporary result sets within a query. Benefits include:
- Readability: Complex logic decomposed into named steps
- Reusability: CTE can be referenced multiple times in main query
- Recursion: CTEs can reference themselves (for hierarchies, graphs)

Recursive CTEs enable traversal of hierarchical data (org charts, categories, bill-of-materials) without joins at each level.

**Set Operations:**
- **UNION:** Combines result sets, removing duplicates (UNION ALL preserves duplicates)
- **INTERSECT:** Returns rows present in both result sets
- **EXCEPT:** Returns rows in first set but not second

These operations treat result sets as mathematical sets, requiring compatible column types and order.

---

## Consistency, Performance & Reliability Challenges

### Anomalies in Unnormalized Data

**Denormalization Pitfalls:**
Intentionally violating normalization (storing derived data, duplicating attributes) for performance creates anomalies:

- **Update Anomaly:** Changing a single fact requires updates in multiple places. Inconsistency occurs if some updates are missed. Example: storing customer address in both customers and orders tables—updating the address must happen twice.

- **Insertion Anomaly:** Cannot insert information without inserting related information. Example: cannot add a department without assigning an employee to it.

- **Deletion Anomaly:** Deleting a record removes unintended information. Example: deleting the last employee in a department also removes department information.

These anomalies cause data inconsistency and demand application-level logic to maintain consistency, increasing bug risk.

### Join Performance Degradation

**The Cost of Reassembly:**
Every JOIN operation involves:
1. Reading two (or more) tables from storage
2. Finding matching rows based on join condition
3. Combining matching rows into result

Costs scale with table sizes and distribution. A query joining 5 normalized tables may require 4 join operations; each incurs I/O and CPU overhead.

**Join Selectivity Issues:**
Join selectivity—the fraction of potential row combinations that match the join condition—determines result set size. Poor selectivity (matching many rows) can create large intermediate result sets, causing memory pressure and disk spilling.

### Query Complexity Growth

Normalized schemas often require complex queries:
- Multiple JOINs for seemingly simple questions
- Complex WHERE conditions filtering across many tables
- Nested subqueries reducing readability

Complex queries are harder to optimize, more prone to mistakes, and consume more resources.

### Concurrent Query Conflicts

Multiple concurrent queries operating on the same tables create conflicts:
- **Reader-Writer Conflicts:** A transaction inserting rows while another reads the same table may produce inconsistent snapshots
- **Writer-Writer Conflicts:** Two transactions updating the same row must coordinate
- **Lock Contention:** Locking for isolation introduces waits and deadlocks (discussed in detail in Transactions topic)

---

## Design Decisions & Trade-offs

### Normalization vs. Denormalization

**When to Normalize:**
- Online Transaction Processing (OLTP) systems with frequent inserts/updates
- Limited storage
- Referential integrity critical
- Consistency more important than read speed

**When to Denormalize:**
- Read-heavy analytical workloads (OLAP)
- High query complexity from many JOINs
- Performance-critical paths where consistency managed at application level
- Data warehouses where updates are infrequent (batch loads)

The correct answer is often **selective denormalization**: maintain normalized core for operational data, materialize views or aggregate tables for analytical access.

### Primary Key Strategy

**Natural vs. Surrogate Keys:**
- **Natural Key:** Meaningful business attributes (email, username, ISBN). Advantages: meaning, no extra storage. Disadvantages: size, difficulty changing business rules, joining overhead.
- **Surrogate Key:** System-generated unique identifier (auto-increment INT, UUID). Advantages: small, stable, fast joins. Disadvantages: meaningless, requires application awareness.

**Impact on Storage and Indexing:**
A UUID primary key is 16 bytes; a composite natural key might be much larger. Since foreign keys reference the primary key, choosing a smaller primary key reduces foreign key storage overhead. However, UUIDs don't auto-increment, affecting index structure and requiring special UUID generation strategies to maintain locality.

### Index Strategy Trade-offs

Indexes accelerate lookups but slow writes. Every INSERT, UPDATE, DELETE must maintain all indexes on that table. Design decisions:
- **Selective indexing:** Index only high-selectivity columns used in WHERE clauses or joins
- **Composite indexes:** Combine multiple columns for compound queries, but ordering matters significantly
- **Covering indexes:** Include additional columns to avoid table lookups, trading storage for read performance

---

## Alternative Models & Comparisons

### Relational vs. Other Data Models

**Relational vs. Hierarchical:**
Hierarchical models (XML, some NoSQL) organize data in tree structures. The relational model flattens structure, treating all entities equally. This makes relational superior for many-to-many relationships and ad-hoc queries but may require multiple joins where hierarchies would navigate directly.

**Relational vs. Document:**
Document databases store nested structures (JSON) without normalization. Benefits: storing related entities together, schema flexibility. Costs: potential redundancy, difficulty querying across documents, update anomalies if documents aren't treated as atomic units.

**Relational vs. Graph:**
Graph databases optimize for relationship queries (shortest path, connected components) through native traversal. Relational requires JOINs for each relationship level. However, relational is superior for aggregations and complex filtering across properties.

### ACID Guarantees

The relational model's consistency guarantee (ACID—discussed in detail later) is mathematically enforced:
- **Atomicity:** Transactions ensure all-or-nothing semantics
- **Consistency:** Constraints (primary keys, foreign keys, CHECK) prevent invalid states
- **Isolation:** Concurrent transactions don't interfere
- **Durability:** Once committed, persists despite failures

This strict consistency contrasts with NoSQL's eventual consistency, where all databases eventually agree but may temporarily diverge.

---

## When to Use and When to Avoid

### Ideal for Relational Databases:

- **OLTP Systems:** Financial transactions, e-commerce orders, inventory management—frequent small writes with strong consistency requirements
- **Structured Data:** When schema is well-defined and stable
- **Complex Queries:** Multi-table analysis, aggregations, comparisons
- **Integrity Critical:** Systems where data corruption is unacceptable (banking, healthcare)
- **Complex Relationships:** Many-to-many relationships efficiently modeled via junction tables
- **Compliance:** Regulations (GDPR, HIPAA) often require audit trails and precise data tracking that relational databases support well

### Avoid Relational for:

- **Highly Unstructured Data:** Logging, images, videos—relational adds overhead without benefit
- **Massive Scale with High Write Throughput:** Single database instance becomes bottleneck; sharding breaks relational integrity
- **Extreme Read Performance:** Analytical queries on massive datasets benefit from denormalization and specialized data structures
- **Schema Flexibility:** Rapidly evolving schemas require frequent migrations
- **Hierarchical/Nested Data:** Document databases handle naturally nested structures more efficiently
- **Real-Time Streaming:** Relational databases aren't designed for streaming inserts and aggregations

### Scalability Limitations:

A single relational database scales vertically (more powerful hardware) until approaching hardware limits. Horizontal scaling (sharding across servers) breaks fundamental assumptions:
- Foreign key relationships span multiple shards (expensive to enforce)
- Transactions spanning shards are complex and slow (distributed transactions)
- Joins across shards require data movement

This is why relational databases dominate OLTP but give way to specialized systems (data warehouses, NoSQL) at massive scale.

---

## Summary

The relational model provides a theoretically sound, practical approach to structured data management. Normalization eliminates anomalies at the cost of join overhead; SQL provides a powerful, declarative query language; ACID guarantees ensure consistency. However, scalability challenges and performance costs in highly demanding workloads have led to proliferation of alternative models. Understanding relational databases deeply—their strengths and limitations—is essential for choosing appropriate systems and designing schemas that balance consistency, performance, and maintainability.
