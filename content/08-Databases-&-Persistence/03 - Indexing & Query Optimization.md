# Indexing & Query Optimization: Internal Mechanisms & Performance

## Purpose & Problem Space

Indexing is the primary mechanism for accelerating data retrieval in databases. Without indexes, every query requires scanning every row in a table (full table scan), an O(n) operation that becomes prohibitively expensive for large datasets.

**Core problems addressed:**
- How to locate specific rows without scanning the entire table
- How to efficiently filter rows based on WHERE conditions
- How to accelerate range queries and sorting operations
- How to support both point lookups and complex filtering
- How to balance query speed against write performance (index maintenance cost)

A database with no indexes can scan millions of rows per second but must scan all of them. A well-indexed database can retrieve specific rows in milliseconds regardless of table size. However, every INSERT, UPDATE, DELETE must update all affected indexes, creating a fundamental trade-off: indexes speed reads but slow writes.

---

## Core Concepts & Internal Architecture

### Index Data Structures

**B-Tree Indexes: The Default**

B-Trees are the most common index structure because they provide good balance across multiple requirements: efficient point lookups, range queries, and insertions without requiring complete restructuring.

*Internal Structure:*
A B-Tree is a balanced tree where:
- **Leaf Nodes:** Contain actual keys and pointers to data rows (or row IDs)
- **Internal Nodes:** Contain separator keys directing searches to child nodes
- **Branching Factor:** Each node has multiple children (typically 50-1000 depending on key/pointer sizes), reducing tree depth
- **Balance Property:** All leaf nodes are at the same depth, guaranteeing O(log n) search complexity regardless of insertion order

*Insertion into a B-Tree:*
When a node becomes full, it splits. Split propagates up the tree, potentially increasing tree height. The tree grows from leaves upward, maintaining balance.

*Why B-Trees Dominate:*
- **Efficient Range Queries:** Leaf nodes are linked; finding all rows between X and Y requires minimal traversal
- **Sorted Output:** B-Tree leaf nodes are ordered; scanning them provides sorted results without explicit sort
- **Cache Efficient:** Nodes typically fit within a memory page; tree structure aligns with storage I/O patterns
- **Adaptable:** B-Trees perform similarly for random and sequential insertions (unlike binary search trees, which degrade with order)

**Hash Indexes**

Hash indexes compute a hash of the key, then look up the hash value in a table:
- **O(1) Average Case:** Point lookups are extremely fast (compute hash, direct lookup)
- **No Range Queries:** Hash doesn't preserve order; range queries require scanning all buckets
- **Perfect for:** Equality checks, session storage lookups

When used: Point lookups where range queries are irrelevant (exact user ID lookups). Rarely used as primary index because most workloads include range queries.

**Bitmap Indexes**

For low-cardinality columns (few distinct values), bitmap indexes store one bitmap per distinct value:
- Row with value A: bitmap A has 1 in corresponding position
- Row with value B: bitmap B has 1 in corresponding position

Bitwise operations (AND, OR) efficiently combine bitmaps for complex filters.

When used: Data warehouses, OLAP systems with many low-cardinality dimensions. Expensive for high-cardinality columns (bitmap per value requires significant memory).

**GiST (Generalized Search Tree) & GIN (Generalized Inverted Index)**

These are generalized index structures supporting specialized data types:

- **GiST:** Supports geometric data (polygons, circles), full-text search, and custom data types. Predicates return approximate results; post-filter validates.
- **GIN:** Optimized for inverted indexing (many keys map to few values). Excellent for full-text search, JSON arrays, and document search.

When used: PostgreSQL extensions for specialized data types.

**BRIN (Block Range Index)**

For very large tables with sequential data:
- Divides table into blocks
- Stores min/max value for each block
- Queries eliminate entire blocks if values are out of range

When used: Time-series data (log entries, events), large sequential tables where locality is preserved.

### Index Types and Strategies

**Single-Column Indexes**

The simplest index: one column, one B-Tree. Effective for:
- WHERE column = value (point lookups)
- WHERE column > value (range queries)
- Joining on indexed columns

Limitations: Compound conditions require careful planning.

**Composite (Multi-Column) Indexes**

A single index on multiple columns, ordered:
```
Index on (user_id, created_at):
- Row 1: user_id=5, created_at=2023-01-01
- Row 2: user_id=5, created_at=2023-06-15
- Row 3: user_id=7, created_at=2022-05-20
```

The index is ordered first by user_id, then within each user_id by created_at.

*Query Effectiveness:*
- `WHERE user_id = 5 AND created_at > '2023-01-01'`: Perfect match, very efficient
- `WHERE user_id = 5`: Efficient (index is ordered by user_id)
- `WHERE created_at > '2023-01-01'`: **Not efficient** (index not ordered by created_at globally)

**The Leftmost Prefix Rule:**
A composite index is useful for queries on the leftmost columns. Queries starting with later columns can't use the index efficiently.

Design Principle: Order composite indexes by:
1. Equality conditions first (user_id = X)
2. Range conditions next (created_at > Y)
3. High-selectivity columns before low-selectivity

**Partial Indexes**

Indexes only a subset of rows matching a WHERE condition:
```sql
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
```

Benefits:
- Smaller index (fewer rows)
- Faster inserts (inactive users don't update this index)
- Useful for skewed data (e.g., most rows are inactive)

When used: Filtering on low-cardinality columns with highly skewed distribution.

**Covering Indexes (INCLUDE Clause)**

Include extra columns in the index (leaf nodes) without ordering by them:
```sql
CREATE INDEX idx_users ON users(email) INCLUDE (name, phone);
```

Allows satisfying queries entirely from the index without accessing the table:
```sql
SELECT name, phone FROM users WHERE email = 'alice@example.com';
```

The index scan returns all required columns, avoiding table lookup (index-only scan).

Cost: Larger index because non-key columns consume space in every leaf node.

### Query Planner: Choosing Index Strategy

The **Query Planner** evaluates multiple execution strategies and chooses the least expensive:

*Cost Estimation:*
For each strategy, the planner estimates:
- **CPU Cost:** Operations performed (key comparisons, row evaluations)
- **I/O Cost:** Disk reads required

Cost is typically represented as:
```
Total Cost = (sequential_io_pages * seq_page_cost) + 
             (random_io_pages * random_page_cost) +
             (rows_processed * cpu_cost_per_row)
```

*Execution Strategies Evaluated:*
1. **Full Table Scan:** Read all rows, filter in memory
2. **Index Scan:** Use index to locate rows, fetch from table
3. **Index-Only Scan:** Satisfy query entirely from index (covering index)
4. **Bitmap Index Scan:** Combine multiple low-selectivity indexes using bitmaps

*Index Selection:*
The planner chooses an index if:
- Index selectivity (fraction of rows matching) is low
- Index cost + table fetch cost < full scan cost

For queries returning large fractions of the table, full scan is often faster (sequential I/O beats random I/O).

**Example Planning Decision:**
```sql
SELECT * FROM users WHERE is_premium = true;
```

- 95% of users are premium
- Index on is_premium exists

Planner chooses **full table scan** because the index would require accessing 95% of rows anyway, and random I/O is slower than sequential scan.

### Execution Plans: EXPLAIN and EXPLAIN ANALYZE

**EXPLAIN Output:**
Shows the planner's chosen strategy without executing:
```
Seq Scan on users (cost=0.00..35.50 rows=100000)
  Filter: (is_active = true)
```

*Metrics:*
- **cost=0.00..35.50**: Estimated startup cost to first row (0.00) and total cost (35.50)
- **rows=100000**: Estimated row count
- **Filter:** Applied after scanning (row-by-row check)

**EXPLAIN ANALYZE:**
Executes the query and shows actual results:
```
Seq Scan on users (cost=0.00..35.50 rows=100000) (actual time=0.05..15.23 rows=95000)
  Filter: (is_active = true)
```

*Actual vs. Estimated:*
- Estimated rows: 100000
- Actual rows: 95000
- Estimates being wrong indicate stale statistics

**Key Indicators of Problems:**

1. **Filter Applied After Scan:** Indicates predicate not pushed to index; many unnecessary rows processed
2. **Nested Loop Join:** Slow for large tables; often indicates missing index
3. **Sort Operation:** Expensive; indicates index on sort key missing
4. **Seq Scan of Large Table:** Acceptable only if returning large fraction; otherwise indicates missing index

### Indexes in Write Operations

**INSERT Performance:**
Each index on the table must be updated. Inserting a row:
1. Insert into table (write to data file)
2. Insert into every index on that table

A table with 5 indexes requires 6 writes (1 table + 5 indexes). Index maintenance cost scales with index count and key size.

**UPDATE Performance:**
If the update modifies an indexed column:
1. Remove old key from index
2. Insert new key into index

This is equivalent to the insert cost. Modifying non-indexed columns only updates the table, bypassing indexes.

**DELETE Performance:**
Similar to insert: remove key from all indexes.

**The Write Amplification Problem:**
Heavy insert/update workloads become bottlenecked by index maintenance. Every index helps some queries but hurts all writes.

Trade-off: Create indexes for queries, but avoid unnecessary indexes.

---

## Consistency, Performance & Reliability Challenges

### Common Bottlenecks and Anomalies

**Full Table Scans on Large Tables**

Symptom: Query takes minutes to return results; EXPLAIN shows Seq Scan on a large table without an Index condition.

Causes:
- Missing index on WHERE clause column
- Index exists but planner considers it slower than full scan (incorrect statistics or highly selective filter)
- Optimizer chose different strategy (using compound index for range filter)

Solutions:
- Add index on filtered column
- Analyze/update statistics (ANALYZE command)
- Hint the optimizer (use index hints if necessary)

**Sort Operations**

Symptom: Query includes "Sort" in execution plan and takes unexpectedly long.

Causes:
- ORDER BY on unindexed column
- Result set so large that sort spills to disk (slower than in-memory sort)
- Joining tables and sorting on joined columns (expensive)

Solutions:
- Add index on ORDER BY column (B-Tree naturally sorts)
- Limit result set before sorting
- Use covering index including ORDER BY columns

**Inefficient Joins**

Symptom: JOIN queries slow; EXPLAIN shows nested loop with large inner table.

Causes:
- No index on join column in inner table
- Join condition not sargable (can't be used by index, e.g., `f(column) = value` where f is a function)
- Wrong join order (outer/inner tables reversed)

Solutions:
- Index the join column
- Rewrite non-sargable conditions (`f(column) = value` → `column = value_pre_computed`)
- Hint join order if planner is wrong

**Index Underutilization**

Symptom: Index exists but EXPLAIN shows full table scan.

Causes:
- Data type mismatch (comparing int column to string '123')
- Operator not supported by index (some operators can't use indexes)
- WHERE condition contains function calls (`WHERE YEAR(created_at) = 2024` can't use index on created_at)
- Column is nullable and condition doesn't filter NULLs

Solutions:
- Rewrite conditions to be sargable (push computations outside WHERE)
- Cast values to correct type
- Create partial index excluding NULLs

### Index Bloat and Maintenance

**Index Growth**

Indexes grow as data grows:
- Insert new row → new key added to all indexes
- Each key occupies space in leaf node

For a table with 1 million rows and 10 indexes on 8-byte integer keys, index space overhead is significant.

**Dead Index Entries**

When rows are deleted, index entries aren't always immediately removed. Many databases mark entries as deleted (dead tuples) but leave them in the index structure until vacuum/reorg operations.

**REINDEX and Maintenance:**
Periodic index maintenance (REINDEX in PostgreSQL, OPTIMIZE in MySQL) removes dead entries and reorganizes the index structure, improving query performance and freeing space.

### Statistics and Query Planner Misestimation

**Stale Statistics**

The planner's cost estimation depends on statistics:
- Row counts per table
- Column value distribution
- Index effectiveness

If table size changes significantly without updating statistics, the planner makes poor decisions:
- Estimate 100 rows when actually 1 million → chooses index when full scan better
- Estimate 100 rows in join result when actually 1 million → chooses nested loop when hash join better

**Skewed Data**

For a column with skewed distribution (e.g., is_premium: 99% false, 1% true), a generic statistic (avg selectivity) is wrong:
- `WHERE is_premium = true`: Very selective, index good
- `WHERE is_premium = false`: Not selective, index bad

Modern optimizers use histogram statistics per value to handle skew.

### Join Strategy Trade-offs

**Nested Loop Join:**
For each row from outer table, scan inner table:
- Cost: O(outer_rows × inner_rows)
- Good when inner table is small or highly filtered
- Bad when both tables are large

**Hash Join:**
Build hash table of inner table, probe with outer table:
- Cost: O(outer_rows + inner_rows)
- Good when memory available and both tables large
- Bad when memory constrained (spilling to disk expensive)

**Sort-Merge Join:**
Sort both tables on join key, merge:
- Cost: O(n log n + m log m) where n, m are table sizes
- Good when data already sorted or join key is sort key
- Good for distributed systems (each node sorts local data, then merge)

The planner chooses based on estimated costs and available resources.

---

## Design Decisions & Trade-offs

### Index Selection Strategy

**Indexing for OLTP (Online Transaction Processing):**
- Many small transactions, frequent updates
- Queries typically have specific filters
- Strategy: Index columns used in WHERE and JOIN clauses, avoid unnecessary indexes

**Indexing for OLAP (Online Analytical Processing):**
- Few large queries, infrequent updates
- Queries scan large result sets
- Strategy: Create many indexes for specific workload queries; maintenance cost less important

**Composite Index Design:**

Given query: `SELECT * FROM orders WHERE user_id = 5 AND status = 'shipped' AND created_at > '2023-01-01';`

Indexes to consider:
1. `(user_id)` - filters to one user
2. `(status)` - filters to shipped orders (lower cardinality than user_id)
3. `(created_at)` - range query
4. `(user_id, status, created_at)` - composite

The composite index is likely best:
- Filter by user_id (equality, highly selective)
- Then status (equality, further filters)
- Then created_at (range, within remaining set)

Order matters: swapping columns changes effectiveness.

### Write-Heavy vs. Read-Heavy Optimization

**Write-Heavy Systems:**
- Index cost is critical; every write hits all indexes
- Minimize index count: create indexes only for absolutely necessary queries
- Wider columns (composite keys) cost more
- Consider denormalization to reduce join queries requiring new indexes

**Read-Heavy Systems:**
- Write cost less important; read performance critical
- Create indexes liberally for expected queries
- Covering indexes acceptable (extra columns increase index size)
- Materialized views useful for complex queries

### Partial Index Strategy

**When Beneficial:**
- Skewed column (e.g., 99% is_active = true, 1% is_active = false)
- Index only active records: `CREATE INDEX idx ON users(email) WHERE is_active = true;`
- 100x smaller index, much faster inserts for inactive users

**When Not Beneficial:**
- Balanced distribution: partial index isn't much smaller
- Queries filter on both values: need two indexes anyway

### Materialized Views for Complex Queries

Instead of computing aggregations repeatedly, materialize the result:
```sql
CREATE MATERIALIZED VIEW daily_sales AS
SELECT date, SUM(amount) as total, COUNT(*) as count
FROM sales
GROUP BY date;
```

Queries now use the materialized view (fast aggregation) but it must be refreshed after data changes (expensive maintenance).

Trade-off: Fast queries, expensive refreshes, stale data between refreshes.

---

## Alternative Models & Comparisons

### B-Tree vs. Other Index Structures

**B-Tree Dominance:**
B-Trees excel at general-purpose indexing. Alternatives optimize for specific scenarios:

| Structure | Strength | Weakness |
|-----------|----------|----------|
| B-Tree | Range queries, sorted output, general purpose | Not optimal for any specific case |
| Hash | Point lookups (O(1)) | No range queries, no ordering |
| Bitmap | Low-cardinality columns, complex filters | Large for high-cardinality, INSERT heavy |
| GiST | Complex data types, spatial | Slower than B-Tree for simple types |
| Full-Text | Text search, substring matches | Not for numeric data |

**Inverted Indexes (for Full-Text Search):**
Maps words to documents containing them:
- Forward index: document → words in it
- Inverted index: word → documents containing it

Inverted indexes excel at text search but are complex to maintain for full ACID compliance.

### Column Stores and Compression

**Row-Oriented (Traditional):**
Each row stored together:
```
Row 1: user_id=5, name='Alice', email='alice@...', created_at='2023-01-01'
Row 2: user_id=6, name='Bob', email='bob@...', created_at='2023-01-02'
```

Good for: OLTP queries touching all columns, small result sets

**Column-Oriented:**
Each column stored separately:
```
Column user_id: 5, 6, 7, 8, ...
Column name: 'Alice', 'Bob', ...
Column email: 'alice@...', 'bob@...', ...
```

Good for: OLAP queries touching few columns, large result sets. Compression benefits (similar values in a column compress well).

Analytic databases (ClickHouse, Vertica) use column storage. General-purpose databases (PostgreSQL, MySQL) use row storage.

---

## When to Use and When to Avoid

### Create Index When:

- Column frequently appears in WHERE clauses
- Column used in JOIN conditions
- Column in ORDER BY and many rows returned
- Table has many rows (>1000) and queries are slow
- Column is foreign key (improves join performance and referential integrity checks)

### Avoid Indexing:

- Columns with very low cardinality (few distinct values, e.g., boolean) - full scan often faster
- Columns rarely used in queries
- Tables with few rows (< 100) - scan is fast anyway
- High write workload and column rarely read
- Volatile data where indexes constantly updated

### Performance Troubleshooting:

1. Run EXPLAIN ANALYZE to understand actual execution
2. Look for full table scans on large tables
3. Check missing indexes on filtered/joined columns
4. Verify statistics are up-to-date
5. Consider query rewrite to be more sargable
6. Profile to identify most expensive queries
7. Index high-impact queries first

---

## Summary

Indexing is the primary lever for query performance, but it's not simple. B-Trees dominate because they support diverse queries efficiently. The query planner chooses execution strategies based on estimated costs; misestimations occur when statistics are stale or data is skewed. Composite index design, selectivity, and join strategy choices significantly impact performance. The fundamental trade-off—faster reads versus slower writes—requires careful balancing based on workload characteristics. Understanding index structure, planner behavior, and query optimization techniques is essential for building performant databases.
