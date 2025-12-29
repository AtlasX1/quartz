# Database Scalability: Replication, Sharding & Distributed Consistency

## Purpose & Problem Space

Single-server databases hit scalability walls: storage limits, write throughput bottlenecks, and lack of geographic distribution. Distributed databases solve these through replication (copies across servers) and sharding (data partitioning).

**Core problems addressed:**
- Single server becomes bottleneck for write throughput
- Data loss if single server fails
- Latency for geographically distant users
- Storage capacity limited to single machine
- High availability requires instant failover
- Consistency maintenance across replicas is complex

Distributed systems scale horizontally (add more servers) but introduce complexity: ensuring consistency across replicas, handling failures, preventing data loss, managing partial failures when network partitions servers.

---

## Core Concepts & Internal Architecture

### Replication: Copies Across Servers

Replication maintains multiple copies of data on different servers. Synchronization strategies determine consistency and performance trade-offs.

**Primary-Replica (Master-Slave) Architecture:**

One primary (master) accepts all writes. Replicas (slaves) copy data from primary, serve reads.

*Write Flow:*
```
Write Request → Primary receives write
            → Primary writes locally
            → Primary sends change to replicas
            → Replicas write locally
            → (Optional: wait for replicas to acknowledge)
            → Primary sends write acknowledgment to client
```

*Replication Methods:*

1. **Statement-Based Replication:**
   Primary sends SQL statements to replicas; replicas execute them.
   
   Pros: Compact (statements are small)
   Cons: Nondeterministic statements (NOW(), RAND()) execute differently on primary and replicas. Causes divergence.

2. **WAL (Write-Ahead Log) Shipping:**
   Primary sends transaction log entries to replicas; replicas replay.
   
   Pros: Accurate replication (same operations as primary)
   Cons: Low-level (implementation-specific), large volume of log data

3. **Row-Based Replication:**
   Primary sends actual row changes (new values for modified rows).
   
   Pros: Works with nondeterministic statements, clear what changed
   Cons: Larger volume for bulk operations, cannot replicate non-SQL operations

**Synchronous vs. Asynchronous Replication:**

*Synchronous:*
Primary waits for replicas to acknowledge write before returning to client.
```
Client: Write
Primary: Write locally, send to replicas, wait for ack
Replicas: Receive, write, send ack
Primary: All replicas acknowledged, return success to client
```

Pros: Strong durability (replicated before acknowledging), consistency (all replicas in sync)
Cons: Slow (wait for replicas), blocking if replica is slow/offline

*Asynchronous:*
Primary acknowledges write immediately, replicates in background.
```
Client: Write
Primary: Write locally, return success, queue replication
Background: Send to replicas asynchronously
```

Pros: Fast (don't wait for replication), doesn't block on replica slowness
Cons: Weak durability (crash before replication loses data), eventual consistency (replicas lag)

**Semi-Synchronous (Tunable):**
Wait for at least one replica to acknowledge (not all). Balances durability and performance.

**Failover (Replica Promotion):**
If primary fails, promote replica to primary:
1. Detect primary failure (heartbeat timeout, connection failure)
2. Choose replica to promote (usually most up-to-date)
3. Promote replica (write elections, repoint reads)
4. Handle divergence (old primary had unreplicated writes, new primary is behind)

**Failover Challenges:**
- **Split-Brain:** Network partition causes both primary and replica to think other is dead. Both accept writes independently, diverging.
- **Data Loss:** Asynchronous replicas lose unreplicated writes
- **Detection Latency:** How long before detecting primary is actually dead vs. slow?
- **Replica Lag:** Which replica is "most up-to-date"? Promoting replica missing writes causes data loss.

**Cascade Replication (Chain Topology):**
```
Primary → Replica1 → Replica2 → Replica3
```

Primary replicates to Replica1, which replicates to Replica2, etc.

Pros: Reduces load on primary
Cons: Replication latency increases (each hop adds delay), failures cascade

### Sharding: Horizontal Partitioning

Instead of copying entire data, sharding partitions data across servers (shard 1 has users 1-1000000, shard 2 has users 1000001-2000000).

*Shard Selection Strategies:*

**Range-Based Sharding:**
Partition key ranges assigned to shards (user_id 1-1M → shard 1).

```
user_id 1-1M → Shard 1
user_id 1M-2M → Shard 2
user_id 2M-3M → Shard 3
```

Pros:
- Range queries efficient (all data in range on same shard)
- Easy to understand

Cons:
- Hot shards if distribution uneven (e.g., all active users in certain ID range)
- Rebalancing expensive (boundaries shift, data moves)

**Hash-Based Sharding:**
Hash shard key, use hash % shard_count to select shard.

```
shard_id = hash(user_id) % num_shards
```

Pros:
- Even distribution (good hash spreads uniformly)
- Simple logic

Cons:
- Range queries inefficient (user_ids in range across all shards)
- Rebalancing very expensive (adding shard changes hash mapping for all keys)

**Directory-Based Sharding:**
Maintain mapping table: shard_key → shard_id.

```
Mapping: user_id=5 → Shard 2
         user_id=6 → Shard 1
         user_id=7 → Shard 3
```

Pros:
- Flexible (any assignment logic)
- Rebalancing simple (update mapping)

Cons:
- Lookup overhead (check mapping for every operation)
- Single point of failure (mapping service must be highly available)
- Consistency challenge (mapping changes must propagate)

**Consistent Hashing:**
Hash function where adding servers minimally affects distribution:

```
Hash ring: 0-100 with server positions
Servers: A@10, B@50, C@80
Key k: hash(k)=35 → assigned to B (next server clockwise)

Add server D@70: Keys 50-70 migrate from C to D
                 Keys 70-80 stay at C
                 Other keys unaffected (only 30 keys move)
```

Pros:
- Adding/removing servers only rehashes subset of keys
- Natural for caching layers

Cons:
- Complex implementation
- Still requires rebalancing for even distribution

### Distributed Consistency Models

**Strong Consistency (Strict):**
All replicas always agree; reads always return latest write.

*Implementation:*
- **Synchronous replication:** Wait for all replicas before acknowledging
- **Quorum reads/writes:** Write to N/2+1 replicas, read from N/2+1 replicas

*Trade-offs:*
- Correctness: All clients see same data
- Performance: Slow (wait for slowest replica or majority)
- Availability: Reduced if replicas unavailable

**Eventual Consistency (BASE):**
Replicas eventually agree; reads may return stale data.

*Implementation:*
- **Asynchronous replication:** Replicate in background
- **Read from any replica:** Accept stale reads

*Trade-offs:*
- Performance: Fast (don't wait for replication)
- Availability: High (can read from any working replica)
- Correctness: Weak (temporary inconsistency, read-after-write anomalies)

**Causal Consistency (Middle Ground):**
Causally related operations are consistent; concurrent operations may diverge.

*Example:*
```
Write "Hello" → Read gets "Hello"
Later: Write "World" → Read gets "World"
But concurrent write by another process → may or may not be visible
```

*Implementation:*
Track causality through timestamps or vector clocks. More complex than eventual, weaker than strong.

**Quorum-Based Consistency (Tunable):**

Define:
- W = replicas that must acknowledge write
- R = replicas read from

Trade-offs:
- R=1, W=N: Eventually consistent (read may be stale, write waits for all)
- R=N, W=1: Strong for reads (read all), eventual for writes (write one)
- R=(N/2+1), W=(N/2+1): Majority quorum (strong consistency, high availability)
- R=W=N: Strongly consistent (all must agree)

**RAMP (Read-After-Write Prepared):**
Special protocol ensuring client sees its own writes even with eventual consistency.

---

## Consistency, Performance & Reliability Challenges

### Replication Lag and Its Anomalies

**Problem:** Asynchronous replication causes replicas to lag behind primary. During lag window, stale data visible.

**Anomalies:**

1. **Read-After-Write Inconsistency:**
```
Client A: Writes to primary
Client A: Immediately reads from replica (not yet replicated)
Result: Reads old value
```

2. **Monotonic Read Anomaly:**
```
Client: Reads from primary at time T1 → value = X
Client: Reads from replica at time T2 (T2 > T1) → value = old X
(Replica lag caused newer read to return older value)
```

3. **Causal Anomaly:**
```
Alice writes "I'm at the cafe"
Bob reads Alice's post → sees it
Bob writes "I'll be there soon"
Charlie reads Bob's post first → sees it
Charlie reads Alice's post → doesn't see it (lag)
(Causal ordering violated)
```

**Mitigation:**
- **Synchronous replication:** Eliminate lag but slower
- **Session guarantees:** Track session's writes, always read from replica with those writes
- **Eventual consistency acceptance:** Document that stale reads possible

### Write Amplification in Distributed Systems

Every write duplicated to replicas. Write amplification:
- Write to primary: 1 operation
- Replicate to N replicas: N operations
- Total: N+1 operations for 1 logical write

At scale (thousands of replicas), amplification becomes prohibitive. Solutions:
- **Replication factor ≤ 3:** Most systems replicate to 3 servers
- **Quorum writes:** Write to majority (N/2+1), not all

### Sharding Hotspots

**Problem:** Uneven data distribution causes some shards to handle most traffic.

*Example:*
```
Shard by user_id with range sharding
Recently created users: IDs 90000000-100000000
All active new users concentrated in this range
Shard 5 handles all traffic; other shards idle
```

**Solutions:**
- **Better sharding key:** Choose key with even distribution
- **Secondary sharding:** Shard the hotspot further (sub-shards)
- **Replication:** Replicate hotspot for read scaling (doesn't help writes)
- **Caching:** Cache hotspot data

### Distributed Transactions Across Shards

**Problem:** Transaction spanning multiple shards requires coordination.

*Example:*
```
Transfer money: Deduct from user A (shard 1), add to user B (shard 2)
If fails mid-transfer, inconsistency results
```

**Solutions:**

1. **Two-Phase Commit (2PC):**
   ```
   Prepare Phase: Each shard locks row, confirms can commit
   Commit Phase: All commit or all rollback
   ```
   
   Pros: Atomicity across shards
   Cons: Blocking (locks held during coordination), slow, network partition causes indefinite wait

2. **Saga Pattern:**
   Break transaction into compensatable steps:
   ```
   Step 1: Deduct from user A (shard 1)
   Step 2: Add to user B (shard 2)
   If step 2 fails: Compensate step 1 (add back to user A)
   ```
   
   Pros: No blocking, high availability
   Cons: Compensation logic complex, temporary inconsistency

3. **Avoid Cross-Shard Transactions:**
   Design schema so related data sharded together.

### Shard Rebalancing

**Problem:** Adding shards or uneven growth requires moving data between shards.

*Example:*
```
Original: 3 shards, 10M users each
Add shard 4: Should have 7.5M users each
Rebalancing: Move 2.5M users from each of 1-3 to shard 4
```

**Challenges:**
- **Downtime Risk:** Moving data causes temporary unavailability
- **Consistency During Move:** Reads/writes during migration to either old or new location?
- **Capacity:** Temporary double capacity (source + destination)
- **Fallback Complexity:** If migration fails, reverting changes complex

**Strategies:**
- **Offline Rebalancing:** Stop writes, move data, resume (downtime)
- **Online Rebalancing:** Redirect writes during migration, maintain consistency
- **Incremental:** Move data in batches, not all at once

### Network Partition (Split-Brain)

**Problem:** Network divides cluster. Each partition thinks other is dead, both continue operating.

```
Primary-Replica setup with network partition:
Primary + Replica1 (partition A)
Replica2 + Replica3 (partition B)

Partition A: Primary still online, accepts writes
Partition B: Replicas suspect primary dead, promote one to primary, accept writes
Both partitions have "primary", both accept writes
Data diverges
```

**Solutions:**

1. **Quorum-Based Leader Election:**
   Partition with majority of nodes continues; minority stops.
   
   ```
   3 nodes: 2 form majority, 1 stops
   5 nodes: 3 form majority, 2 stop
   ```
   
   Prevents split-brain if servers ≥ 3.

2. **External Arbiter:**
   Separate service determines which partition is valid. AWS Elastic Cluster uses this.

3. **Partition Tolerance Trade-off (CAP):**
   If partition occurs, choose: Consistency (stop serving) or Availability (inconsistency risk).

---

## Design Decisions & Trade-offs

### Replication Factor Selection

**Factor 1:** No replication.
- Pro: No overhead
- Con: Any failure = data loss

**Factor 3:** Standard.
- Pro: Survives 1 node failure, common practice
- Con: 3x write amplification

**Factor 5+:** High durability.
- Pro: Survives 2+ failures simultaneously
- Con: Expensive, significant write amplification

**Decision:** Balance durability requirements against write cost.

### Synchronous vs. Asynchronous

**Synchronous:** For critical data (financial, medical).
- Strong durability
- Performance cost acceptable

**Asynchronous:** For user-facing data (social, UGC).
- High throughput
- Durability risk acceptable (cache-like attitude)

**Hybrid:** Critical operations (transfers) synchronous, non-critical (profile views) asynchronous.

### Read-Only Replicas vs. Sharding

**Read-Only Replicas (Scaling Reads):**
```
Primary handles writes, N replicas handle reads
Good for: Read-heavy (90% reads, 10% writes)
Bad for: Write-heavy (50% reads, 50% writes)
```

**Sharding (Scaling Reads + Writes):**
```
Data partitioned; each shard handles its data
Good for: Both reads and writes scale
Bad for: Cross-shard queries expensive
```

**Hybrid:** Some shards replicated for read scaling.

### Sharding Key Selection

**Good Shard Keys:**
- High cardinality (many distinct values; user_id better than gender)
- Evenly distributed (no value with excessive data)
- Immutable (don't change after insertion)
- Queryable (natural to filter by)

**Bad Shard Keys:**
- Low cardinality (few values; gender has ~2 distinct values)
- Skewed (some values dominate)
- Mutable (changes require resharding)
- Rarely queried

**Example:**
- **Good:** user_id (high cardinality, even distribution)
- **Bad:** region (few regions, hotspots, low cardinality)

### Single Database vs. Replicated vs. Sharded

**Single Database:**
- Size < 10GB, throughput < 1K QPS
- Cost-effective, simple

**Replicated (Master-Slave):**
- Size 10-100GB, read-heavy (reads > 10x writes)
- High availability, read scaling
- Write bottleneck at master

**Sharded:**
- Size > 100GB, high write throughput
- Write scaling, geographic distribution
- Cross-shard queries expensive, operational complexity

---

## Alternative Models & Comparisons

### Consistency Models Spectrum

| Model | Consistency | Performance | Availability | Typical Systems |
|-------|-------------|-------------|--------------|-----------------|
| Strong | Immediate | Slow | Lower | PostgreSQL, MySQL primary |
| Quorum | Tunable | Tunable | Tunable | Cassandra, DynamoDB |
| Causal | Moderate | Fast | High | Riak |
| Eventual | Weak | Very Fast | Very High | DynamoDB eventual, Memcached |

### Centralized vs. Decentralized Replication

**Centralized (Primary-Replica):**
- Primary single source of truth
- Simple, no conflicts
- Single point of failure

**Decentralized (Multi-Master):**
- Multiple primaries accept writes
- High availability, no SPOF
- Conflicts require resolution

**Use Centralized:** When single authority acceptable, simpler operations needed.

**Use Decentralized:** When availability critical, geographic distribution needed.

### Consensus Algorithms

**Raft:**
Algorithm for leader election and log replication. One node elected leader; leader replicates log. On leader failure, election promotes new leader.

Used in: etcd, Consul, TiDB.

**Paxos:**
Older consensus algorithm, more complex than Raft. Ensures agreement among replicas despite failures.

Used in: Google Spanner, Chubby.

---

## When to Use and When to Avoid

### Add Replication When:

- High availability needed (99.9%+ uptime)
- Durable backups required
- Read scaling needed (reads > 5x writes)
- Geographic distribution required
- Failover acceptable (seconds to minutes)

### Avoid Replication When:

- Size < 10GB (single instance sufficient)
- Writes completely dominate (replication doesn't help)
- Complexity unacceptable for value gained
- Consistency more important than availability

### Add Sharding When:

- Data > 1TB or write throughput > 10K QPS
- Scaling beyond single server necessary
- Geographic distribution required
- Data naturally partitionable

### Avoid Sharding When:

- Data < 100GB and QPS < 1K (replication sufficient)
- Many cross-shard queries (expensive)
- Complex transactions across shards
- Operational complexity unacceptable

### Tools and Frameworks

**Vitess (MySQL):**
Middleware for sharding and scaling MySQL. Handles rebalancing, failover.

**Citus (PostgreSQL):**
Extension making PostgreSQL distributed. Built-in sharding.

**MongoDB Atlas:**
Managed MongoDB with automatic sharding and replication.

**Spanner (Google Cloud):**
Globally distributed ACID database. Automatic replication, strong consistency despite distribution.

---

## Summary

Scaling distributed databases requires balancing consistency, performance, and availability (CAP theorem). Replication provides redundancy and read scaling but introduces complexity through replication lag and failover coordination. Sharding enables write scaling and geographic distribution but makes cross-shard queries expensive and requires careful shard key selection. Understanding replication modes (synchronous/asynchronous), consistency models (strong/eventual), and failure scenarios (split-brain, hotspots) enables architects to design systems appropriate for their scalability and durability requirements. Trade-offs are unavoidable: choose based on application needs and acceptable compromises.
