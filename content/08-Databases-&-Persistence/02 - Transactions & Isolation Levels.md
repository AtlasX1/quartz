# Transactions & Isolation Levels: ACID, Locking & MVCC

## Purpose & Problem Space

Transactions are the mechanism by which databases ensure data consistency and reliability in concurrent environments. A transaction is a logical unit of work consisting of one or more SQL operations that must either all succeed or all fail as an atomic unit.

**Core problems addressed:**
- How to ensure multiple operations succeed or fail together despite hardware/software failures
- How to prevent concurrent transactions from corrupting data through interleaved operations
- How to recover from failures without losing committed data
- How to maintain data consistency while maximizing concurrent access
- How to detect and resolve conflicts when multiple transactions compete for the same data

Without transactions, concurrent operations would create chaos: partial updates, lost changes, corrupted referential integrity. Transactions formalize guarantees (ACID) that make databases reliable systems for critical data.

---

## Core Concepts & Internal Architecture

### ACID Properties: The Consistency Contract

ACID defines the reliability guarantees a database provides for transactions. Understanding ACID deeply requires understanding not just what each letter means, but the internal mechanisms that enforce it.

**Atomicity: All-or-Nothing Semantics**

Atomicity ensures that a transaction's effects are indivisible—either all operations within the transaction are applied, or none are.

*Internal Mechanism:*
Achieving atomicity requires:
1. **Write-ahead Logging (WAL):** Before modifying data, the database writes transaction operations to a log on stable storage. If the system crashes mid-transaction, the log survives; upon recovery, the database can either replay or undo operations.

2. **Undo/Redo Buffers:** For each operation, the database maintains the old value (for rollback) and new value (for recovery). If a transaction aborts (user commits ROLLBACK or system forces abort), the undo information restores the database to its previous state.

3. **Commit Coordination:** Only after all operations succeed and their state is safely logged does the database issue a commit acknowledgment. The commit point is irreversible—once committed, the transaction's effects persist.

Most modern databases use WAL for atomicity. The transaction log is the single source of truth; the in-memory cache is reconstructed from the log upon restart.

**Consistency: Invariant Preservation**

Consistency means that a transaction brings the database from one valid state to another valid state. The database maintains all defined constraints (primary keys, foreign keys, CHECK constraints, NOT NULL).

*Internal Mechanism:*
Consistency is enforced through:
1. **Constraint Checking:** Before committing, the database verifies all constraints. If violated, the transaction rolls back automatically.

2. **Referential Integrity:** Foreign key constraints are checked. Operations that would violate referential integrity (deleting a parent record with existing children) are blocked or handled via CASCADE rules.

3. **Application-Level Constraints:** Complex business rules (e.g., "account balance cannot be negative") are enforced through stored procedures, triggers, or application logic.

Crucially, consistency assumes the transaction's logic is correct. A database cannot prevent a programmer from executing illogical operations—it can only enforce schema constraints.

**Isolation: Concurrency Control**

Isolation ensures that concurrent transactions do not interfere with each other. Each transaction should see a consistent view of data as if no other transactions are running concurrently.

This is the most complex guarantee, with multiple implementation strategies (discussed below).

**Durability: Persistence**

Durability guarantees that once a transaction is committed, its effects persist despite failures (disk corruption, server crash, power loss).

*Internal Mechanism:*
1. **Persistent Storage:** Data is written to non-volatile storage (disk, SSD).

2. **Fsync Operations:** Database writes data to the OS cache, then issues fsync to force the OS to write to actual hardware. Without fsync, a power failure during OS buffering loses data.

3. **Multiple Copies:** Many databases replicate committed data across multiple disks or servers. Even if one disk fails, other copies exist.

4. **Checkpointing:** Periodically, the database writes a consistent snapshot to disk. Between checkpoints, only log records need storage.

The durability guarantee has a cost: fsync operations are slow. High-throughput systems often group multiple transactions' commits before fsyncing, or use asynchronous replication for durability across servers rather than local disk.

### Isolation Levels: A Spectrum of Guarantees

SQL standard defines four isolation levels, forming a hierarchy of strictness. Understanding them requires understanding the anomalies they prevent.

**Read Uncommitted (Isolation Level 0)**

Transactions can read uncommitted changes from other transactions. This allows the anomaly:

- **Dirty Read:** Reading data changed by another transaction that later rolls back, causing your transaction to operate on data that "never really happened."

*When used:* Rarely, only for approximate reads where accuracy doesn't matter (e.g., dashboard counters).

**Read Committed (Isolation Level 1)**

Transactions cannot read uncommitted data, but within a transaction, multiple reads of the same row can return different values (if another transaction modifies and commits between reads). This allows:

- **Non-Repeatable Read:** Reading a row, having another transaction modify it, then reading again within the same transaction returns different data. The same query executed twice in one transaction produces different results.

*Internal Mechanism:* 
Each query sees a snapshot of committed data at the moment the query executes (not the moment the transaction begins). This allows reading committed data but introduces non-repeatable reads.

*When used:* Default for many databases (PostgreSQL, Oracle). Good balance for OLTP: prevents dirty reads without excessive locking.

**Repeatable Read (Isolation Level 2)**

Transactions see a consistent snapshot from the transaction's start time. All reads within the transaction return identical data. However, phantom reads are possible:

- **Phantom Read:** A query that filters rows returns different results if another transaction inserts/deletes matching rows after the initial query. The rows "appear" or "disappear" between queries.

*Example:*
```sql
Transaction A:
1. SELECT COUNT(*) FROM orders WHERE user_id = 5;  -- Returns 3
2. -- (Transaction B inserts a new order for user_id = 5 and commits)
3. SELECT COUNT(*) FROM orders WHERE user_id = 5;  -- Returns 4
```

*Internal Mechanism:*
The database takes a snapshot at transaction start. All queries operate on this snapshot. For range queries, the snapshot prevents seeing uncommitted changes but allows seeing newly committed rows matching the filter criteria.

*When used:* Default for InnoDB (MySQL). Stronger consistency than Read Committed without full serialization overhead.

**Serializable (Isolation Level 3)**

The strictest level. Transactions execute as if they were serial (one after another), despite running concurrently. Phantom reads are prevented.

*Internal Mechanism:*
Two approaches:
1. **Pessimistic Locking:** Acquire locks on all rows a transaction might read or write. This prevents other transactions from modifying those rows.
2. **Optimistic Locking:** Allow concurrent execution but track conflicts. If a transaction's read set (rows read) is modified by concurrent transactions, the transaction aborts and retries.

*When used:* Systems requiring absolute consistency (financial systems, high-value transactions). Cost is significant: high contention, many aborts and retries.

**The Isolation Level Spectrum:**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Cost |
|-------|-----------|-------------------|--------------|------|
| Read Uncommitted | Yes | Yes | Yes | Very Low |
| Read Committed | No | Yes | Yes | Low |
| Repeatable Read | No | No | Yes | Medium |
| Serializable | No | No | No | High |

### Locking Mechanisms: Pessimistic Concurrency Control

Locking is the traditional approach to maintaining isolation. The database grants locks to transactions, preventing concurrent access to locked resources.

**Lock Types:**

1. **Shared Locks (Read Locks):** Multiple transactions can hold shared locks on the same row. Used for SELECT operations. Prevents exclusive locks (writes).

2. **Exclusive Locks (Write Locks):** Only one transaction can hold an exclusive lock on a row. Used for INSERT, UPDATE, DELETE. Prevents both shared and exclusive locks from other transactions.

3. **Intent Locks:** Databases often lock at multiple granularities (row, page, table). Intent locks signal intentions at higher levels. An exclusive lock on a row implies an intent lock on the containing page and table.

**Lock Granularity:**
- **Row-Level Locks:** Fine-grained, allowing high concurrency but high overhead. Most modern systems default here.
- **Page-Level Locks:** Intermediate granularity, balancing overhead and concurrency.
- **Table-Level Locks:** Coarse-grained, preventing all concurrent access to a table. Used when scanning entire tables or when row-level locking overhead is prohibitive.

**Lock Protocol:**

The Two-Phase Locking (2PL) protocol ensures serializability:
1. **Growing Phase:** Transactions acquire locks as needed but never release any.
2. **Shrinking Phase:** Once a lock is released, no new locks are acquired.

This protocol guarantees that if two transactions conflict, their lock ordering determines execution order, preventing anomalies.

**SELECT ... FOR UPDATE:**

Explicit locking in SQL:
```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

This acquires an exclusive lock on the selected rows, preventing other transactions from reading or modifying them. Useful when you need to read-then-modify atomically without relying on isolation level.

### MVCC: Optimistic Concurrency Control

**Multi-Version Concurrency Control** is an alternative to locking that optimizes for read-heavy workloads. Instead of blocking readers with writers, MVCC maintains multiple versions of each row.

**Core Mechanism:**

1. **Row Versioning:** Each row modification creates a new version. Old versions are retained for in-flight transactions.

2. **Visibility Rules:** Each transaction has an associated snapshot ID. When reading a row, the database finds the newest version visible to that snapshot (typically, versions committed before the transaction started).

3. **Garbage Collection:** Old versions are deleted once no in-flight transaction needs them. This is expensive if long-running transactions prevent garbage collection.

*Example Timeline:*
```
Time 1: Transaction A starts (snapshot ID = 100)
Time 2: Row X has value "A" (committed at snapshot 90)
Time 3: Transaction B modifies Row X to "B" (commits at snapshot 105)
Time 4: Transaction A reads Row X → sees "A" (last version before its snapshot)
Time 5: Transaction C starts (snapshot ID = 110)
Time 6: Transaction C reads Row X → sees "B" (last version before its snapshot)
```

**Advantages of MVCC:**
- **Read-Write Parallelism:** Readers never block writers, and writers never block readers. Highly beneficial for OLTP workloads with high read ratios.
- **Snapshot Isolation:** Readers see a consistent snapshot from their start time.

**Disadvantages:**
- **Storage Overhead:** Multiple versions consume disk space. A heavily updated row accumulates versions requiring garbage collection.
- **Write Conflicts:** Concurrent writes to the same row still conflict; second writer must fail or wait.
- **Garbage Collection Complexity:** Determining when a version is safe to delete requires tracking all in-flight transactions.

**MVCC Examples:**
- **PostgreSQL:** Uses transaction IDs (XID) and visibility maps for versioning.
- **InnoDB (MySQL 5.7+):** Implements MVCC through undo logs maintaining old row versions.

### Deadlock: Circular Wait Scenarios

A deadlock occurs when multiple transactions hold locks and wait for locks held by each other, creating a cycle that prevents progress.

*Classic Example:*
```
Transaction A: Locks Row 1, wants Row 2
Transaction B: Locks Row 2, wants Row 1
```

Neither can proceed; both wait indefinitely.

**Deadlock Detection:**

Databases use wait graphs: nodes are transactions, edges represent waits. A cycle indicates deadlock. Detected either periodically (scan all locks, build graph, detect cycles) or on each lock acquisition attempt.

**Deadlock Resolution:**

Once detected, the database chooses a victim transaction to abort. The victim typically:
- Has made least progress (fewest writes)
- Is youngest (most recent start)

Aborting the victim releases its locks, allowing others to proceed. The application must retry the aborted transaction.

**Deadlock Prevention (Lock Ordering):**

Instead of detecting deadlocks, prevent them by requiring all transactions to acquire locks in the same order. If every transaction locks Row 1 before Row 2, the cycle cannot form.

Lock ordering is harder in complex applications (many rows, unclear dependencies) but is the most efficient prevention mechanism.

---

## Consistency, Performance & Reliability Challenges

### Transaction Anomalies and Inconsistency

Despite ACID guarantees, subtle anomalies can occur at non-serializable isolation levels:

**Dirty Read Risk:**
At Read Uncommitted, transaction A reads changes by transaction B before B commits. If B rolls back, A operates on phantom data, potentially causing errors or incorrect results.

**Lost Update Problem:**
Two transactions read the same value, modify it independently, then write back:
```
Account balance: 100
Transaction A: reads 100, adds 50 → writes 150
Transaction B: reads 100, subtracts 30 → writes 70
Final: 70 (one transaction's update lost)
```

Both locking and MVCC can experience lost updates if not carefully handled. Read Committed isolation is particularly vulnerable.

**Write Skew:**
A subtle anomaly at Snapshot Isolation (Repeatable Read in MVCC databases):
```
Transaction A: reads X and Y, writes X based on Y's value
Transaction B: reads X and Y, writes Y based on X's value
```

Both transactions see old values, make decisions based on them, and commit. The result violates invariants that would be checked if transactions were serialized.

*Example:* Two doctors on-call. Each checks if the other is on-call before deciding whether to go home. Both see the other as on-call, so both go home—nobody is on-call.

### Performance Trade-offs

**Locking Contention:**
Heavy locking causes:
- **Lock Waits:** Transactions block waiting for locks, reducing throughput.
- **Deadlocks:** Increased complexity from multiple lock acquisitions increases deadlock probability.
- **Context Switching:** Threads blocked on locks consume CPU cycles through context switching.

High-contention workloads (many concurrent transactions accessing the same rows) suffer dramatically under pessimistic locking.

**MVCC Storage Overhead:**
MVCC eliminates blocking but requires storing multiple versions:
- Disk space increases with update frequency
- Garbage collection runs regularly, consuming I/O and CPU
- Long-running transactions prevent garbage collection, allowing versions to accumulate

Heavy write workloads create many versions, increasing scan costs (each scan must skip invisible versions).

**Snapshot Age:**
In MVCC, old transactions see old snapshots. Very old snapshots may be on disk (not cache), causing slowdowns. Queries see all rows modified since snapshot creation even if unrelated to the query.

### Failure Scenarios and Recovery

**Server Crash Mid-Transaction:**
The write-ahead log ensures consistency. Upon restart:
1. Database reads the log from the last checkpoint
2. Transactions in the log are replayed (Redo) to reconstruct state
3. Transactions not fully logged (no commit record) are rolled back (Undo)

The log must be written to stable storage before acknowledging commits.

**Disk Corruption:**
If the data file is corrupted but the log survives, the database rebuilds from the log. If the log is also lost, committed data is lost (violating durability).

This is why critical systems maintain:
- Multiple disk copies
- Remote replication
- Regular backups

**Replication Lag:**
When durability is achieved via replication, committed data is asynchronously replicated. During replication lag, the data exists on the primary but not replicas. If the primary fails, data between last replica update and failure is lost.

---

## Design Decisions & Trade-offs

### Choosing Isolation Levels

**Read Committed (Default for OLTP):**
- Prevents dirty reads
- Allows non-repeatable and phantom reads
- Minimal locking overhead
- Suitable for most applications if business logic handles potential inconsistencies
- Requires application-level handling of lost updates (optimistic locking, application retries)

**Repeatable Read (Strong Consistency):**
- Prevents dirty reads and non-repeatable reads
- Still allows phantom reads (usually acceptable)
- Higher overhead than Read Committed but avoids serialization cost
- Good for applications needing consistent snapshots without full serialization

**Serializable (Highest Consistency):**
- Prevents all anomalies, ensuring absolute correctness
- Very high overhead; contention causes frequent aborts and retries
- Reduces effective throughput in high-concurrency scenarios
- Necessary for critical operations (financial transactions, medical dosage calculations)

### Explicit Locking vs. Optimistic Strategies

**Pessimistic Locking (SELECT ... FOR UPDATE):**
- Assume conflicts are likely; lock immediately
- Prevents conflicts but blocks other transactions
- When used: Frequent conflicts, critical consistency, acceptable blocking

**Optimistic Locking (Version Numbers/Timestamps):**
- Assume conflicts are rare; detect and retry on conflict
- Application maintains a version column; on UPDATE, verify version hasn't changed
- When used: Rare conflicts, unacceptable blocking, eventual consistency acceptable

Most modern applications use optimistic strategies because:
- Reduced blocking improves throughput in low-conflict scenarios
- Better cache locality (no lock objects)
- Easier horizontal scaling (no distributed lock coordination)

### Write-Ahead Log Strategies

**Synchronous fsync:**
- Safest: every commit waits for disk write
- Slowest: fsync is a bottleneck
- Necessary for: durability-critical systems

**Asynchronous/Group Commits:**
- Batch multiple commits before fsync
- Much faster but slight durability risk (commits in batch can be lost together)
- Acceptable for: non-critical data, high-throughput systems

**Remote Replication:**
- Durability achieved via replication, not local disk
- Commit waits for remote acknowledgment
- Good for: distributed systems, disaster recovery
- Risk: replication lag loses data between primary failure and replication

---

## Alternative Models & Comparisons

### ACID vs. BASE (NoSQL Consistency Models)

**BASE (Basically Available, Soft-state, Eventually consistent):**
- Rejects ACID's strong consistency for availability and partition tolerance (CAP theorem)
- Reads may return stale data
- Consistency is eventual: all nodes eventually agree
- Better for: distributed systems, high availability, AP systems (choose Availability and Partition tolerance over Consistency)

**When BASE is Appropriate:**
- Distributed systems (consistency across replicas is expensive)
- High-traffic services (eventual consistency allows higher throughput)
- Non-critical data (user preferences, counters)

**When ACID is Required:**
- Financial systems (transactions must be exact)
- Inventory systems (cannot oversell)
- Medical records (strict accuracy)

### Distributed Transactions

In distributed systems (multiple databases), maintaining ACID becomes complex:

**Two-Phase Commit (2PC):**
Coordinates commits across multiple databases:
1. Prepare phase: All participants prepare to commit (acquire locks, validate)
2. Commit phase: All commit or all rollback

2PC is a blocking protocol; participants wait during coordination, reducing availability. Network partitions cause indefinite waits.

**Modern Alternative - Saga Pattern:**
Break transaction into compensatable steps executed against different systems. If a step fails, compensate previous steps (reverse them). No distributed locks, higher availability.

---

## When to Use and When to Avoid

### Require Strict ACID Transactions:

- **Financial Systems:** Payment processing, transfers, accounting
- **Inventory Management:** Stock levels must be exact; overselling is catastrophic
- **E-commerce Orders:** Order status, payment, fulfillment are interdependent
- **Medical Records:** Dosage, treatment history must be consistent
- **Critical Business Logic:** Where inconsistency causes significant loss

### Serializable Isolation is Necessary:

- High-value transactions
- Operations with complex invariants
- Regulatory requirements (financial, healthcare)

### Accept Weaker Isolation:

- Social media (eventual consistency of likes, comments acceptable)
- Counters/metrics (approximate values acceptable)
- User sessions (stale state acceptable for brief periods)
- Caches (eventual consistency with source acceptable)

### Avoid Strict Locking:

- High-concurrency scenarios (thousands of concurrent operations)
- Distributed systems (lock coordination overhead)
- Workloads with rare conflicts (optimistic strategies better)

---

## Summary

Transactions are fundamental to database reliability. ACID guarantees—enforced through write-ahead logs, constraint checking, isolation mechanisms, and persistent storage—ensure data integrity despite failures and concurrency. Multiple isolation levels and concurrency control strategies (pessimistic locking, MVCC) provide trade-offs between consistency and performance. Understanding these mechanisms allows architects to choose appropriate systems and isolation levels for their specific workloads, balancing the consistency guarantees against performance costs.
