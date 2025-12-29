# Query Design & Data Analysis: Advanced Techniques & Analytical Databases

## Purpose & Problem Space

Analytical queries—aggregations, rankings, complex filtering across multiple attributes—are fundamentally different from operational OLTP queries. OLTP queries typically touch small row counts (user profile, single order). Analytical queries scan millions of rows, compute aggregations, compare across dimensions.

**Core problems addressed:**
- OLTP databases pessimized for analytical queries (row-oriented storage, heavy indexing)
- Reporting requires querying production database (performance impact)
- Complex multi-step aggregations require application-level logic or complex SQL
- Time-window analyses (moving averages, trends) difficult in SQL
- Ranking and relative comparisons require complex self-joins
- Fast analytical performance requires different storage and indexing (column-oriented)

Analytical queries require different approaches: optimized aggregation query patterns (CTEs, window functions), materialized views for pre-computed results, specialized analytical databases for large-scale analytics, and different storage formats (column-oriented) for compression and cache efficiency.

---

## Core Concepts & Internal Architecture

### Window Functions: Analytic Without Collapsing

Window functions compute values across a subset of rows without aggregating them into single row (unlike GROUP BY).

**Window Specification Syntax:**
```sql
aggregate_function(expression) OVER (
  PARTITION BY partition_column
  ORDER BY order_column
  ROWS/RANGE frame_specification
)
```

**Components:**

- **PARTITION BY:** Divide rows into groups; window functions operate independently per partition
- **ORDER BY:** Order rows within partition; determines which rows are "previous", "next", etc.
- **ROWS/RANGE:** Frame specification defining which rows included in window

**Frame Specifications:**
```sql
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
  -- Include current row and 2 rows before

ROWS UNBOUNDED PRECEDING
  -- Include all rows from partition start to current

RANGE BETWEEN INTERVAL '1 day' PRECEDING AND CURRENT ROW
  -- Include rows within 1 day before current row (by ORDER BY value)
```

**Ranking Functions:**
```sql
ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at)
  -- 1, 2, 3, 4, ... (unique, no ties)

RANK() OVER (PARTITION BY user_id ORDER BY score DESC)
  -- 1, 1, 3, 4, ... (same rank for ties)

DENSE_RANK() OVER (PARTITION BY user_id ORDER BY score DESC)
  -- 1, 1, 2, 3, ... (no gaps for ties)

PERCENT_RANK() OVER (PARTITION BY user_id ORDER BY score DESC)
  -- (rank - 1) / (rows - 1), between 0 and 1
```

**Offset Functions:**
```sql
LAG(column, offset, default) OVER (ORDER BY ...)
  -- Access previous row(s)

LEAD(column, offset, default) OVER (ORDER BY ...)
  -- Access next row(s)

FIRST_VALUE(column) OVER (ORDER BY ...)
  -- First value in window

LAST_VALUE(column) OVER (ORDER BY ... ROWS BETWEEN ... AND UNBOUNDED FOLLOWING)
  -- Last value in window (requires UNBOUNDED FOLLOWING frame)
```

**Use Cases:**
```sql
-- Running total
SELECT user_id, amount, 
       SUM(amount) OVER (PARTITION BY user_id ORDER BY date) as running_total
FROM transactions

-- Month-over-month growth
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
       (revenue - LAG(revenue) OVER (ORDER BY month)) / LAG(revenue) OVER (...) as growth_rate
FROM monthly_sales

-- Rank within partition
SELECT user_id, score, ROW_NUMBER() OVER (PARTITION BY game_id ORDER BY score DESC) as rank
FROM leaderboard
```

**Advantages over GROUP BY:**
- Rows retain identity (don't collapse)
- Can access neighboring rows (LAG, LEAD)
- Complex ranking/numbering
- Moving aggregates (e.g., 7-day moving average)

### Common Table Expressions (WITH): Query Decomposition

CTEs (WITH clause) provide named temporary result sets within a query, improving readability and enabling recursion.

**Simple CTE:**
```sql
WITH user_orders AS (
  SELECT user_id, COUNT(*) as order_count, SUM(total) as total_spent
  FROM orders
  GROUP BY user_id
)
SELECT u.name, uo.order_count, uo.total_spent
FROM users u
JOIN user_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 5
```

Benefits:
- Readability: Named intermediate results clarify logic
- Reusability: CTE referenced multiple times in main query
- Testability: Can run CTE independently to verify intermediate results

**Recursive CTE (Hierarchical Data):**
```sql
WITH RECURSIVE employee_hierarchy AS (
  -- Base case: top-level managers
  SELECT id, name, manager_id, 1 as level
  FROM employees
  WHERE manager_id IS NULL
  
  UNION ALL
  
  -- Recursive case: find direct reports
  SELECT e.id, e.name, e.manager_id, eh.level + 1
  FROM employees e
  INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
  WHERE eh.level < 10  -- Prevent infinite recursion
)
SELECT * FROM employee_hierarchy
ORDER BY level, name
```

*Execution Model:*
1. Execute base case (managers with no superior)
2. Execute recursive case using previous result
3. Union results
4. Repeat until recursive case returns no rows

**Use Cases:**
- Organizational hierarchies (employees, managers)
- Category trees (product categories, subcategories)
- Graphs (shortest path, connected components)
- Bill of materials (component hierarchies)

### Materialized Views: Pre-Computed Results

Materialized views store query results in a table, trading freshness for performance.

**Standard View (Virtual):**
```sql
CREATE VIEW monthly_sales AS
SELECT DATE_TRUNC('month', created_at) as month,
       SUM(total) as revenue,
       COUNT(*) as order_count
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
```

Query executes the underlying SELECT each time the view is accessed.

**Materialized View (Stored):**
```sql
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT DATE_TRUNC('month', created_at) as month,
       SUM(total) as revenue,
       COUNT(*) as order_count
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
```

Results stored in a table. Queries execute instantly but require refreshing when underlying data changes.

**Refresh Strategies:**

1. **Full Refresh:** Recompute entire view.
   ```sql
   REFRESH MATERIALIZED VIEW monthly_sales
   ```
   Pros: Simple
   Cons: Slow for large datasets, temporary unavailability

2. **Incremental Refresh:** Update only affected rows.
   - Track changes to underlying tables
   - Recompute only aggregations with new/modified data
   - Faster but complex

3. **Schedule-Based:** Refresh at fixed intervals (nightly).
   - Acceptable lag between real data and view
   - Fits batch processes

**Trade-offs:**
- **Performance:** Materialized views extremely fast (pre-computed)
- **Freshness:** Stale until refreshed
- **Maintenance:** Refresh cost and complexity
- **Storage:** Copy of data stored

**When to Use:**
- Complex aggregations queried frequently
- Acceptable staleness (data from last night fine)
- Query performance critical

### Temporal Queries and Time Windows

**Time-Window Aggregations:**
```sql
SELECT 
  DATE_TRUNC('hour', created_at) as hour,
  COUNT(*) as event_count,
  AVG(duration) as avg_duration,
  MAX(duration) as max_duration
FROM events
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY DATE_TRUNC('hour', created_at)
ORDER BY hour DESC
```

Aggregates events by hour across time window.

**Moving/Sliding Window:**
```sql
SELECT 
  created_at,
  COUNT(*) OVER (ORDER BY created_at ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_7day_count,
  AVG(value) OVER (ORDER BY created_at ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as moving_avg
FROM daily_metrics
```

Computes aggregation over rolling window of rows (7 preceding + current).

**Cumulative Aggregates:**
```sql
SELECT 
  created_at,
  revenue,
  SUM(revenue) OVER (ORDER BY created_at) as cumulative_revenue,
  SUM(revenue) OVER (ORDER BY created_at) * 100.0 / SUM(revenue) OVER (ORDER BY DATE_TRUNC('month', created_at)) as pct_of_month
FROM daily_sales
```

Shows running totals and percentages.

### Complex Filtering and Conditional Logic

**Correlated Subqueries:**
Subquery references outer query's rows:
```sql
SELECT u.id, u.name, 
       (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) as order_count,
       (SELECT SUM(total) FROM orders o WHERE o.user_id = u.id) as total_spent
FROM users u
```

Executed once per outer row (potentially slow). Rewriting with window functions or LEFT JOIN often faster.

**EXISTS for Existence Testing:**
```sql
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 1000
)
```

More efficient than COUNT subquery for "does any matching row exist" questions.

**CASE for Conditional Logic:**
```sql
SELECT 
  user_id,
  COUNT(*) as total_orders,
  COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
  COUNT(CASE WHEN status = 'cancelled' THEN 1 END) as cancelled,
  SUM(CASE WHEN discount > 0 THEN 1 ELSE 0 END) as discounted_orders
FROM orders
GROUP BY user_id
```

Conditional aggregation without requiring multiple queries.

### Specialized Query Patterns

**Pivot/Crosstab Queries:**
Transform rows into columns:
```sql
SELECT 
  DATE_TRUNC('month', created_at) as month,
  SUM(CASE WHEN status = 'completed' THEN total ELSE 0 END) as completed,
  SUM(CASE WHEN status = 'pending' THEN total ELSE 0 END) as pending,
  SUM(CASE WHEN status = 'cancelled' THEN total ELSE 0 END) as cancelled
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
```

Shows metrics broken down by category across rows.

**Funnel Analysis:**
```sql
SELECT 
  DATE_TRUNC('day', created_at) as day,
  COUNT(DISTINCT user_id) as users_viewed_product,
  COUNT(DISTINCT CASE WHEN action = 'add_to_cart' THEN user_id END) as users_added_cart,
  COUNT(DISTINCT CASE WHEN action = 'checkout' THEN user_id END) as users_checked_out,
  COUNT(DISTINCT CASE WHEN action = 'completed' THEN user_id END) as users_purchased
FROM events
GROUP BY DATE_TRUNC('day', created_at)
```

Tracks progression through stages.

**Cohort Analysis:**
```sql
WITH cohort_data AS (
  SELECT 
    user_id,
    DATE_TRUNC('month', first_purchase_date) as cohort_month,
    DATE_TRUNC('month', purchase_date) as purchase_month,
    SUM(total) as revenue
  FROM orders
  GROUP BY user_id, cohort_month, purchase_month
)
SELECT 
  cohort_month,
  (purchase_month - cohort_month) / INTERVAL '1 month' as months_since_cohort,
  COUNT(DISTINCT user_id) as user_count,
  SUM(revenue) as cohort_revenue
FROM cohort_data
GROUP BY cohort_month, months_since_cohort
```

Groups users by acquisition time, tracks behavior over time.

---

## Consistency, Performance & Reliability Challenges

### Analytical Query Performance

**Full Table Scans:**
Analytical queries often scan entire tables (aggregating across all data). Indexes don't help.

*Traditional Solution:* Vectorized processing, columnar storage, compression.

**Intermediate Result Size:**
Joins create large intermediate result sets:
```sql
SELECT * FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
```

3M orders × 5 items average × 500 products = 7.5B intermediate rows. Memory exhaustion, disk spilling.

**Solutions:**
- Denormalize (store data together, avoid joins)
- Pre-aggregate (materialized views)
- Distribute (analytical database)

### Stale Materialized Views

**Problem:** View refreshes lag behind underlying data changes.

```
15:00 - Data changes
15:30 - View refreshed
User sees data from 15:30, thinks it's current, makes decision on stale information
```

**Severity:** Depends on use case. Financial reports need accuracy; dashboard rough estimates acceptable.

**Mitigation:**
- Document refresh schedule
- Show refresh timestamp in UI
- Real-time views for critical data (accept slower performance)

### Analytical Database Consistency

Separate analytical database (data warehouse) requires ETL (Extract, Transform, Load) from operational database.

*Consistency Challenge:*
```
Operational DB: Order complete at 15:00:00.123
ETL: Extracts at 15:05 (orders complete)
DW: Shows order at 15:05

Operational DB: User inquires "where is my order?" at 15:01
System queries DW: "no order" (not yet extracted)
Data latency causes incorrect response
```

**Typical ETL Lag:** Minutes to hours (daily batch ETL common). For critical queries, must check operational database.

---

## Design Decisions & Trade-offs

### Materialized View vs. Real-Time Query

**Materialized View:**
- Response: <1ms (pre-computed)
- Freshness: Stale (updates delayed)
- Storage: Duplicate data

**Real-Time Query:**
- Response: 1-10s (computed on demand)
- Freshness: Current
- Storage: Single source of truth

**Decision Framework:**
- **High-frequency queries, acceptable staleness:** Materialized view
- **Infrequent queries, freshness critical:** Real-time query
- **Middle ground:** Hybrid (materialized for most, real-time for critical)

### Window Functions vs. Self-Joins

**Window Functions:**
```sql
SELECT user_id, score, 
       ROW_NUMBER() OVER (PARTITION BY game_id ORDER BY score DESC) as rank
FROM leaderboard
```

Simple, efficient, readable.

**Self-Joins (Pre-window Function):**
```sql
SELECT l1.user_id, l1.score,
       COUNT(DISTINCT l2.user_id) + 1 as rank
FROM leaderboard l1
LEFT JOIN leaderboard l2 ON l1.game_id = l2.game_id AND l2.score > l1.score
GROUP BY l1.user_id, l1.score
```

Complex, potentially slow (O(n²) comparisons).

**Always prefer window functions** when available (most modern databases support them).

### Normalization vs. Denormalization for Analytics

**Normalized Schema (Operational):**
- Clean structure
- Single source of truth
- Joins required for analytics (complex queries)

**Denormalized Schema (Analytical):**
- Redundant data
- Update anomalies
- Simple queries, fast aggregations

**Typical Architecture:**
- Operational DB: Normalized (OLTP)
- Analytical DB: Denormalized (OLAP)
- ETL: Transforms normalized → denormalized

### Dedicated Analytical Database vs. Operational Database

**Shared Database:**
- Pros: Single source of truth, simpler operations
- Cons: Analytical queries slow down operational database, resources contended

**Separate Analytical Database:**
- Pros: Analytical optimization (columnar, denormalized), no impact on operations
- Cons: Replication complexity, consistency challenges, data duplication

**Decision:** At scale (>1TB data, >10K analytical queries/day), separate analytics database justified.

---

## Alternative Models & Comparisons

### Analytical Database Systems

**ClickHouse (Column-Oriented OLAP):**
- Optimized for read-heavy aggregations
- Column-wise storage (highly compressible)
- Missing features: transactions, updates difficult
- Use: Logs, metrics, time-series at massive scale

**Redshift (AWS Data Warehouse):**
- Managed Postgres-based analytical database
- Columnar storage
- Integrated with AWS ecosystem
- Use: Corporate BI, analytics

**Snowflake (Cloud Data Warehouse):**
- Managed, cloud-native
- Separates storage and compute
- Good for: Organizations with data in cloud

**DuckDB (In-Process OLAP):**
- Embedded analytical database
- No server required
- Good for: Analytics on files, small-scale analytics

**TimescaleDB (Time-Series):**
- PostgreSQL extension
- Optimized for time-series (large volume events)
- Good for: Metrics, logs, sensor data

### Row vs. Column Storage

**Row-Oriented (Traditional):**
```
Disk: [user_id=1, name=Alice, score=100] [user_id=2, name=Bob, score=85] ...
```

- Pro: INSERT/UPDATE efficient (single row write)
- Pro: Transaction friendly
- Con: Aggregation requires reading all columns
- Con: Compression poor (dissimilar types in row)

**Column-Oriented:**
```
Disk: [user_id column: 1, 2, 3, ...] [name column: Alice, Bob, ...] [score column: 100, 85, ...]
```

- Pro: Aggregation reads only needed columns
- Pro: Compression excellent (similar types compress well)
- Pro: Vectorization (CPU processes multiple rows at once)
- Con: INSERT/UPDATE requires modifying multiple columns
- Con: Transaction complexity

**Typical System:** Operational DB row-oriented, analytical DB column-oriented.

---

## When to Use and When to Avoid

### Use Materialized Views For:

- Frequently queried, computationally expensive aggregations
- Acceptable staleness (data changes infrequently or can be delayed)
- Report generation (run once nightly)
- Multi-table aggregations reducing massive result sets

### Avoid Materialized Views For:

- Real-time dashboards (freshness critical)
- Data changes frequently (refresh overhead high)
- Simple queries (no computational benefit)

### Use Separate Analytical Database For:

- Data > 1TB
- Frequent analytical queries (>100/day)
- Analytical workload completely different from operational
- Strong consistency between OLTP and OLAP not required

### Use Window Functions For:

- Ranking, ranking gaps
- Running aggregates (cumulative totals)
- Comparative analysis (previous row, next row)
- Multiple aggregations without GROUP BY collapse

### Use Recursive CTEs For:

- Hierarchical data traversal
- Graph algorithms (shortest path)
- Bill of materials, category trees
- Org charts

### Avoid Recursive CTEs For:

- Simple queries (clarity cost)
- Very deep hierarchies (performance issues)
- Real-time systems (computation cost)

---

## Summary

Analytical queries require different approaches than operational OLTP. Window functions provide powerful ranking and time-window analysis without GROUP BY's row collapsing. CTEs decompose complex queries into readable steps and enable hierarchical traversals. Materialized views pre-compute expensive aggregations, trading freshness for performance. Specialized analytical databases (column-oriented storage, denormalized schemas) provide superior performance for analytics at scale. Understanding these techniques—when to apply each and their trade-offs—enables efficient analytical query design and reporting systems that serve both operational and analytical needs.
