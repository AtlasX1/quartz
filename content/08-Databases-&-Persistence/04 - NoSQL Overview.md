# NoSQL Overview: Models, Theory & Architecture Comparisons

## Purpose & Problem Space

NoSQL (Not Only SQL) databases emerged to address limitations of traditional relational databases at massive scale and with diverse data shapes. Rather than a single model, NoSQL encompasses multiple specialized data models optimized for different problem spaces.

**Core problems addressed:**
- Horizontal scalability (distributed databases with thousands of nodes)
- High write throughput (billions of events, immutable logs)
- Schema flexibility (rapidly evolving data structures)
- Specialized access patterns (key-value lookups, graph traversal, full-text search)
- Availability over consistency (always-on systems accepting eventual consistency)

Relational databases scale vertically well (bigger hardware) but struggle to scale horizontally (distributed). NoSQL systems sacrifice properties relational databases guarantee (strong consistency, complex queries, multi-row transactions) to gain horizontal scalability and handle diverse data shapes.

---

## Core Concepts & Internal Architecture

### CAP Theorem: The Fundamental Trade-off

The CAP theorem states that distributed systems can guarantee at most two of three properties:

**Consistency (C):** Every read returns the latest write. All nodes agree on data state.

**Availability (A):** System responds to all requests, even during failures. No request is rejected due to unavailable nodes.

**Partition Tolerance (P):** System continues operating despite network partitions (losing communication between nodes).

In distributed systems, network partitions are inevitable (hardware fails, networks disconnect). Thus, P is non-negotiable. The choice is between C and A.

**Consistency-First (CP Systems):**
- Sacrifices availability during partitions
- When network divides the system, some partitions refuse requests to avoid inconsistency
- Examples: Traditional consensus-based systems (etcd, Zookeeper), some databases with synchronous replication
- Suitable for: Systems where consistency is critical (financial records), where brief unavailability is acceptable

**Availability-First (AP Systems):**
- Sacrifices consistency during partitions
- When network divides the system, each partition continues serving requests
- Data diverges; consistency is eventual (when partition heals, nodes synchronize)
- Examples: NoSQL systems (DynamoDB, Cassandra, MongoDB with eventual consistency), distributed caches
- Suitable for: Systems where availability is critical (social media, user presence), where eventual consistency acceptable

**In Practice:**
The binary CP/AP choice is oversimplified. Real systems make trade-offs:
- **Tunable Consistency:** DynamoDB offers eventual and strong consistency options per request
- **Multi-Tier Systems:** Critical data in CP system, non-critical in AP system
- **Time-Bounded Consistency:** Accept eventual consistency with bounded delay (synchronize every N seconds)

### BASE: The Antithesis to ACID

While relational databases guarantee ACID (strong consistency), NoSQL systems often embrace BASE:

**Basically Available:** System responds to requests, possibly returning incomplete or stale data.

**Soft State:** State is soft (may change even without new writes) due to asynchronous propagation and eventual consistency.

**Eventually Consistent:** All replicas eventually agree. System is consistent eventually, not immediately.

**Implications:**
- Reads may return stale data
- Writes succeed even if not replicated to all nodes
- Conflicts resolved asynchronously (last-write-wins, application logic, manual resolution)
- Higher availability and throughput at the cost of consistency guarantees

This represents a profound shift: accepting temporary inconsistency for operational resilience.

### NoSQL Data Models

Rather than the single relational model, NoSQL encompasses multiple specialized models:

**Key-Value Stores**
- **Data Structure:** Simple key → value mapping
- **Access Pattern:** Exact key lookups, no range queries
- **Examples:** Redis, Memcached, DynamoDB
- **Strengths:** Extreme performance (O(1) lookups), simple model, easy to scale
- **Weaknesses:** No complex queries, no relationships
- **Suitable For:** Sessions, caches, counters, leaderboards

**Document Stores**
- **Data Structure:** Semi-structured JSON/BSON documents, grouped in collections
- **Access Pattern:** Query by document fields, no joins required
- **Examples:** MongoDB, CouchDB, Firestore
- **Strengths:** Flexible schema, denormalized data eliminates joins, complex nested structures
- **Weaknesses:** Redundancy, update anomalies, large document sizes
- **Suitable For:** User profiles, content management, rapid prototyping

**Column-Family (Wide-Column) Stores**
- **Data Structure:** Rows with arbitrary columns; columns grouped in families
- **Access Pattern:** Range queries on row keys, column filtering
- **Examples:** Cassandra, HBase, DynamoDB (secondary indexes)
- **Strengths:** Compression (similar column values), massive scale, time-series data
- **Weaknesses:** Complex model, limited query flexibility, complex joins
- **Suitable For:** Time-series, logs, analytics, massive write throughput

**Graph Databases**
- **Data Structure:** Nodes (entities) and edges (relationships) with properties
- **Access Pattern:** Traversals (friend-of-friend), path finding, connected components
- **Examples:** Neo4j, Amazon Neptune, Dgraph
- **Strengths:** Natural representation of relationships, efficient traversal
- **Weaknesses:** Difficult to distribute, not optimized for full-dataset scans
- **Suitable For:** Social networks, recommendation engines, knowledge graphs

**Search/Analytical**
- **Data Structure:** Inverted indexes, denormalized for aggregation
- **Access Pattern:** Full-text search, aggregations, analytics
- **Examples:** Elasticsearch, Solr, ClickHouse
- **Strengths:** Fast full-text search, complex aggregations, analytics
- **Weaknesses:** Limited transactionality, denormalization challenges
- **Suitable For:** Logging, search, business intelligence

### Replication Models in Distributed NoSQL

**Master-Slave (Primary-Replica):**
- Single primary (master) accepts writes
- Replicas replicate changes from primary
- Reads can hit replicas (eventual consistency possible)
- Failover: promote replica to primary if primary fails

Trade-offs:
- Simple consistency model
- Write bottleneck at primary (doesn't scale writes)
- Read scalability (add replicas)
- Single point of failure (primary)

**Multi-Master (Peer-to-Peer):**
- Multiple nodes accept writes independently
- Changes propagate between masters (eventual consistency)
- High availability (no single point of failure)
- Complexity: concurrent writes to same data cause conflicts

Conflict Resolution:
- **Last-Write-Wins:** Keep the write with latest timestamp (simple, loses data)
- **Vector Clocks:** Detect causality; conflicts only when truly concurrent
- **Application Logic:** Merge logic defined by application

Suitable for: Geographic distribution (local writes), offline-first systems.

### Consistency Models and Tuning

**Strong Consistency:**
- All replicas updated before acknowledging write
- Reads always return latest state
- Performance cost: must wait for all replicas

**Eventual Consistency:**
- Acknowledge write once primary is updated
- Replicas updated asynchronously
- Reads may return stale data (for brief period)
- Performance benefit: no waiting for replicas

**Read-Your-Writes Consistency:**
- Client always reads writes it issued (but not others' writes)
- Weaker than strong, stronger than eventual
- Sufficient for many applications (user sees their own changes, others' changes eventual)

**Causal Consistency:**
- Reads return data causally related to previous operations
- More expensive than eventual, less expensive than strong
- Prevents counter-intuitive scenarios in eventually consistent systems

Modern NoSQL systems offer tuning options:
- **Quorum Reads/Writes:** Read from N of R replicas (tunable consistency)
  - Read from all replicas = strong consistency
  - Read from 1 replica = eventual consistency
  - Read from majority = middle ground with durability

---

## Consistency, Performance & Reliability Challenges

### Write Amplification in Distributed Systems

Every write must be replicated to survive failures. Write amplification:
- Write hits primary (1 write)
- Replicates to N replicas (N more writes)
- Total: 1 + N writes for a single logical write

At massive scale (thousands of replicas), write amplification becomes prohibitive. Solutions:
- **Replication Factor ≤ 3:** Most systems replicate to 3 nodes (typical availability guarantee)
- **Quorum Writes:** Write to majority (N/2 + 1) before acknowledging

### Eventual Consistency Anomalies

**Read-After-Write Inconsistency:**
1. Client writes data to node A
2. Client immediately reads from node B (not yet replicated)
3. Reads old value

**Stale Reads:**
In geographically distributed systems, reading from local replicas may return old data if replication is slow. Updates at other data centers haven't propagated yet.

**Conflict Anomalies:**
Multi-master systems with concurrent writes to same data:
```
Node A: Update x = 10
Node B: Update x = 20
Concurrent: Both committed locally

Merge: Which value is correct? Last-write-wins is arbitrary.
```

### Scalability Challenges

**Single-Node Bottlenecks:**
Even distributed systems have limits:
- Primary node in master-slave is write bottleneck
- Hotkeys (frequently accessed keys) can't scale beyond single node performance
- Coordinator nodes in some systems become bottleneck

**Distributed Transactions:**
Transactions spanning multiple partitions are expensive and slow. Most NoSQL systems:
- Abandon multi-document transactions
- Require application-level consistency (orchestrate logic across systems)
- Or provide weak local transactions (single partition)

**Network Partition Challenges:**
When network partitions the cluster:
- Each partition continues operating (AP systems) → divergence
- Some partitions refuse requests (CP systems) → unavailability
- Detecting partition is non-trivial (slow networks vs. partitions)

### Hot Spot and Skew

**Hot Keys:**
If data distribution is uneven, some keys accessed much more frequently:
- All requests hit same partition
- Single node becomes bottleneck regardless of cluster size
- Cannot distribute request load

*Example:* A celebrity's profile in social network. Millions of reads, single partition.

Solutions:
- Replication of hot keys
- Client-side caching
- Hierarchical sharding (shard hot data further)

---

## Design Decisions & Trade-offs

### SQL vs. NoSQL: Decision Framework

**Choose Relational (SQL) When:**
- Schema is well-defined and stable
- Strong consistency and ACID required
- Complex queries across multiple entities
- Many-to-many relationships common
- Data correctness critical (financial, medical)
- Small-to-medium scale (single database viable)

**Choose NoSQL When:**
- Schema undefined or rapidly evolving
- Horizontal scaling critical
- Simple access patterns (lookups, range scans)
- Availability more important than consistency
- Denormalization acceptable for performance
- Massive scale (terabytes, billions of operations)

### Denormalization Strategy

NoSQL systems intentionally denormalize (store redundant data) to avoid joins:

**Embedding:**
Store related data in a single document:
```json
{
  "user_id": 5,
  "name": "Alice",
  "address": {
    "street": "123 Main",
    "city": "Springfield"
  },
  "orders": [
    {"order_id": 101, "total": 150},
    {"order_id": 102, "total": 200}
  ]
}
```

Pros: No joins required, atomic operations on related data
Cons: Redundancy (orders duplicated if also stored separately), update anomalies

**Referencing:**
Store references like foreign keys, require application joins:
```json
{
  "user_id": 5,
  "name": "Alice",
  "address_id": 42
}
```

Pros: Avoid redundancy
Cons: Requires application logic for joins, consistency harder to maintain

Most NoSQL systems mix these: critical relationships embed, others reference.

### Replication Factor Selection

**Replication Factor 1:** No replication
- Fastest writes (no replication overhead)
- Zero durability; any failure loses data
- Never used for persistent data

**Replication Factor 3:** Most common
- Survives any 1 node failure
- Good balance of durability and write overhead
- Industry standard for persistent systems

**Replication Factor 5+:** High-durability systems
- Survives 2+ node failures simultaneously
- Expensive (5x write amplification)
- Used for critical data or high-failure-rate environments

### Sharding Strategy

**Range-Based Sharding:**
Assign key ranges to partitions (user_id 1-1000 → shard 1, 1001-2000 → shard 2).

Pros:
- Range queries efficient (all data in range on same shard)
- Simple logic

Cons:
- Hot spots (uneven data distribution if ranges unequal)
- Rebalancing expensive (range boundaries must shift)

**Hash-Based Sharding:**
Hash key, use hash mod number-of-shards to assign.

Pros:
- Even distribution (good hash spreads uniformly)
- Simple logic

Cons:
- Range queries inefficient (range spans all shards)
- Rebalancing very expensive (adding shards changes hash mapping)

**Directory-Based Sharding:**
Maintain directory mapping keys to shards. Can change mapping without rehashing.

Pros:
- Flexible rebalancing (update directory)
- Supports any sharding strategy

Cons:
- Directory is bottleneck/single point of failure
- Consistency challenges

---

## Alternative Models & Comparisons

### Consistency Spectrum Across Systems

| System | Model | Consistency | CAP |
|--------|-------|-------------|-----|
| PostgreSQL | Relational | Strong | CA (partition intolerant) |
| Spanner | Relational | Strong | PA (CP with latency) |
| MongoDB | Document | Tunable | CP/AP |
| Cassandra | Column-Family | Eventual | AP |
| Redis | Key-Value | Varies | AP |
| Neo4j | Graph | Varies | CP |

### Eventual Consistency vs. Immediate Consistency

**Immediate Consistency (ACID Databases):**
- All nodes agree before acknowledging
- Strong correctness guarantees
- Limited horizontal scalability
- Higher latency (wait for all replicas)

**Eventual Consistency (BASE Systems):**
- Nodes agree eventually
- High availability and scalability
- Weaker guarantees (must handle stale data)
- Lower latency (acknowledge before replicating)

The choice depends on application: high-consistency needs → ACID, high-availability needs → eventual consistency.

### Single-Model vs. Multi-Model Databases

**Single-Model:** Optimized for one model (MongoDB for documents, Cassandra for columns).

Pros: Optimal performance for that model
Cons: Inefficient for other models, application needs multiple databases

**Multi-Model:** Support multiple models in one system (MongoDB supports documents and sub-documents, PostgreSQL supports JSON, Spanner is relational but distributed).

Pros: One system, flexibility
Cons: Not optimal for any single model, higher complexity

---

## When to Use and When to Avoid

### Key-Value Stores:
**Use When:** Sessions, caches, leaderboards, simple key lookups
**Avoid When:** Complex queries, relationships, data integrity critical

### Document Databases:
**Use When:** User profiles, content (blogs, products), flexible schema, rapid iteration
**Avoid When:** Complex multi-document transactions, ad-hoc analytical queries, normalized data

### Column-Family Stores:
**Use When:** Time-series, logs, massive write throughput, analytics
**Avoid When:** Complex queries, small data, latency-sensitive reads

### Graph Databases:
**Use When:** Social networks, recommendations, knowledge graphs, shortest-path queries
**Avoid When:** Full-dataset scans, simple key-value access, transaction-heavy workloads

### Search Systems:
**Use When:** Full-text search, logging, analytics, fuzzy matching
**Avoid When:** Transactional consistency required, simple key lookups

### Scaling Decisions:

**Single Database Still Viable:** <10GB data, <1000 QPS, eventual consistency acceptable
**Replication Needed:** >10GB, >1000 QPS, high availability required
**Sharding Required:** >1TB, >10K QPS, write-heavy workload, geographic distribution

---

## Summary

NoSQL systems represent a paradigm shift from relational databases, sacrificing strong consistency and complex queries for horizontal scalability and schema flexibility. The CAP theorem mandates choosing between consistency (C) and availability (A) when partitions occur; NoSQL systems typically choose availability, accepting eventual consistency. Multiple models (key-value, document, column-family, graph) optimize for different access patterns. Understanding the trade-offs—denormalization challenges, replication models, sharding strategies, consistency anomalies—is essential for choosing appropriate systems and designing robust distributed architectures.
