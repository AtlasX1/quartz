# Data Integrity & Security: Constraints, Encryption, Auditing & Recovery

## Purpose & Problem Space

Data integrity and security protect databases from corruption, unauthorized access, and data loss. These are layered concerns addressing different threats:

**Integrity Threats:**
- Invalid data violates business rules (age = 999, balance = NaN)
- Referential integrity broken (order references non-existent customer)
- Duplicate data causes de-duplication problems
- NULL values where not allowed

**Security Threats:**
- Unauthorized read access (competitor steals customer list)
- Unauthorized modification (hacker alters financial records)
- Data exposure in transit (sniffing unencrypted connections)
- Data exposure at rest (stolen disk leaks information)
- Audit trail loss (cannot prove who changed what)

**Durability Threats:**
- Hardware failure (disk corruption, server crash)
- Software bugs (replication lag, inconsistency)
- Operational errors (accidental deletion)
- Disaster (data center loss)

Addressing these requires defense in depth: constraints prevent invalid data, encryption protects from snooping, access controls prevent unauthorized operations, auditing provides accountability, and backups/replication enable recovery.

---

## Core Concepts & Internal Architecture

### Constraints: Database-Level Validation

Constraints enforce invariants at the database level, preventing invalid data from ever being persisted.

**Primary Key Constraint:**
```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL
)
```

- **Uniqueness:** No duplicate primary key values
- **NOT NULL:** Primary key cannot be NULL
- **Enforcement:** Rejected on INSERT/UPDATE violating constraint
- **Index:** Primary key automatically indexed for O(log n) lookups

**Foreign Key Constraint (Referential Integrity):**
```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT NOT NULL,
  total DECIMAL(10, 2),
  FOREIGN KEY (user_id) REFERENCES users(id)
)
```

Ensures order.user_id always references valid users.id. Prevents:
- Insertion of order with non-existent user_id
- Deletion of user with referencing orders (unless CASCADE configured)

**Cascade Actions:**
```sql
FOREIGN KEY (user_id) REFERENCES users(id) 
  ON DELETE CASCADE
  ON UPDATE CASCADE
```

- **CASCADE:** Delete/update user → automatically delete/update referencing orders
- **RESTRICT:** Prevent deletion/update if referencing data exists
- **SET NULL:** Delete/update user → set user_id to NULL in orders

**CHECK Constraints (Domain Constraints):**
```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  price DECIMAL(10, 2) CHECK (price > 0),
  discount_pct INT CHECK (discount_pct BETWEEN 0 AND 100),
  end_date DATE CHECK (end_date > start_date)
)
```

Enforces logical constraints: price must be positive, discount between 0-100%, end_date after start_date.

**UNIQUE Constraints:**
```sql
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email)
```

Enforces unique values (like primary key but allows NULL and multiple per table).

**NOT NULL Constraints:**
```sql
email VARCHAR(255) NOT NULL
```

Column must have value; NULL insertion rejected.

**Enforcement at Insert/Update:**
When INSERT or UPDATE violates constraint, database rejects operation and returns error. Application must handle constraint violations.

### Data Validation Architecture

**Database Constraints (Structural):**
```sql
CHECK (age >= 0 AND age <= 150)
```

Enforces structural validity (correct types, ranges, relationships).

**Application Validation (Business Rules):**
```python
if email and '@' not in email:
    raise ValidationError("Invalid email")
if password_strength < MINIMUM_STRENGTH:
    raise ValidationError("Password too weak")
```

Enforces business logic (email format, password rules, duplicate prevention at natural keys).

**Layered Approach (Best Practice):**
1. **Database:** Enforce structural integrity (types, ranges, foreign keys)
2. **Application:** Enforce business rules (email format, password rules)
3. **API:** Input validation (prevent obvious garbage from reaching database)

Each layer catches errors the previous layer might miss. Database layer is last resort; earlier layers catch most errors faster.

### Encryption: Protecting Data at Rest and in Transit

**Encryption at Rest (Stored Data):**

Encrypts data on disk, protecting against physical theft or unauthorized access to storage.

**Database-Level Encryption (TDE - Transparent Data Encryption):**
```sql
ALTER TABLE users ENCRYPT WITH algorithm AES256
```

Database automatically encrypts/decrypts data. Application unchanged. Overhead: ~5-10% performance cost.

Pros:
- Transparent (application unaware)
- Protects full database
- Simple to enable

Cons:
- Key management (where to store decryption key?)
- Decryption key needed to query (only slows snooping, not application attacks)

**Field-Level Encryption:**
```python
encrypted_ssn = encrypt(ssn, key)
db.insert(user_id, encrypted_ssn)

# To read:
decrypted_ssn = decrypt(encrypted_ssn, key)
```

Only sensitive fields encrypted, others in plaintext.

Pros:
- Granular (only sensitive data encrypted)
- Searchable encrypted fields possible (deterministic encryption)

Cons:
- Application logic complex (encryption/decryption)
- Cannot search encrypted fields (depends on method)
- Partial protection (other fields visible)

**Encryption in Transit (Network):**

SSL/TLS encrypts database connections:
```
Client → [SSL/TLS encrypted] → Database Server
```

Prevents packet sniffing revealing passwords or data.

**Implementation:**
```sql
-- Database server configured with SSL certificate
-- Client connection with encryption required
psql "postgresql://user:pass@host/db?sslmode=require"
```

Typical setup: All database connections use SSL/TLS (cost: ~2% overhead, standard practice).

### Key Management

Encryption keys must be protected (stealing key defeats encryption).

**Key Storage Options:**

1. **Hardware Security Module (HSM):**
   Dedicated hardware storing keys, performing encryption/decryption. Application never sees key.
   
   Pros: Highest security (key never in software)
   Cons: Expensive, specialized infrastructure

2. **Key Management Service (KMS):**
   Hosted service (AWS KMS, Google Cloud KMS) managing keys. Application requests encryption/decryption.
   
   Pros: Managed, scalable, audit logging
   Cons: Network dependency, vendor lock-in

3. **Application-Level Key Store:**
   Application stores key (in secure file or environment variable).
   
   Pros: Simple, independent
   Cons: Key vulnerable if application compromised, key rotation complex

**Key Rotation:**
Periodically change keys for security (limit breach window). Re-encrypt data with new keys.

Pros: Limits exposure if old key compromised
Cons: Expensive operation (re-encrypt entire dataset)

### Access Control: Authentication & Authorization

**Authentication (Who are you?):**

Verifies user identity:
```sql
-- User provides username + password
SELECT password_hash FROM users WHERE username = 'alice'
-- Compare provided password to stored hash
```

Passwords should be hashed (irreversible) not stored plaintext. Modern algorithms: bcrypt, scrypt, Argon2.

**Authorization (What are you allowed to do?):**

Enforces permissions after authentication.

**Role-Based Access Control (RBAC):**
```sql
CREATE ROLE admin
CREATE ROLE data_analyst
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO admin
GRANT SELECT ON orders TO data_analyst

-- User inherits role permissions
ALTER USER alice WITH ROLE admin
```

- **Admin:** Full database access
- **Data Analyst:** Read-only access to orders
- **Application User:** Restricted subset (only own records)

**Row-Level Security (RLS):**
```sql
-- Only users can see their own data
CREATE POLICY user_isolation ON orders
  USING (user_id = current_user_id)
```

Filters rows based on user identity at query level.

**Column-Level Security:**
```sql
-- Mask sensitive data for non-admins
ALTER TABLE users MODIFY ssn MASKED WITH (FUNCTION = 'partial(1, "XXX-XX", 4)')
```

Hides sensitive columns from non-authorized users.

### Auditing: Accountability & Forensics

Audit logs track what data changed and who changed it, enabling forensics and accountability.

**Database-Level Auditing:**

```sql
-- PostgreSQL with pgaudit extension
CREATE EXTENSION pgaudit
SET pgaudit.log = 'ALL'
```

Logs all queries (executed at database level).

Capture:
- SQL statement
- User executing statement
- Timestamp
- Success/failure

Example log:
```
2024-01-15 10:23:45 alice UPDATE orders SET status='shipped' WHERE id=101 SUCCESS
```

Pros:
- Comprehensive (all access logged)
- Cannot be bypassed (database enforces)

Cons:
- Performance overhead (logging adds latency)
- Log volume (every query logged)

**Application-Level Auditing:**

Application logs specific operations:
```python
def update_order_status(order_id, new_status):
    old_status = get_order_status(order_id)
    db.update(f"UPDATE orders SET status='{new_status}' WHERE id={order_id}")
    audit_log(
        action="order_status_update",
        user_id=current_user.id,
        order_id=order_id,
        old_value=old_status,
        new_value=new_status,
        timestamp=now()
    )
```

Pros:
- Selective (only important operations logged)
- Contextual (application can add meaning)
- Less volume

Cons:
- Incomplete (depends on application implementing logging)
- Can be bypassed (if application compromised)

**Best Practice:** Both database and application logging. Database captures what happened, application captures why.

### Backup & Recovery

Backups protect against data loss. Recovery time objective (RTO) and recovery point objective (RPO) define backup strategy.

**RTO (Recovery Time Objective):** Acceptable downtime. 1 hour RTO = must recover within 1 hour.

**RPO (Recovery Point Objective):** Acceptable data loss. 1 hour RPO = can lose ≤1 hour of data.

**Backup Methods:**

1. **Full Backup:**
   Copy entire database to backup storage.
   
   Schedule: Once daily
   Pros: Complete, self-contained
   Cons: Large (entire database copied), long duration
   
   Recovery: Restore backup directly

2. **Incremental Backup:**
   Copy only data changed since last backup.
   
   Schedule: Hourly after daily full backup
   Pros: Small (only changes), fast
   Cons: Requires previous backup to restore (chain of backups)
   
   Recovery: Restore full + all incrementals since

3. **Write-Ahead Log (WAL) Archiving:**
   Store transaction logs (WAL) separately. Full backup + WAL replay recovers to any point in time.
   
   Pros: Granular recovery (any timestamp), minimal data loss
   Cons: Complex (requires log management)
   
   Recovery: Restore backup + replay logs up to target timestamp (PITR - Point-In-Time Recovery)

**Storage Locations:**

- **Local Disk:** Fast, vulnerable to local failure
- **Remote Storage (S3, GCS):** Slow but durable, survives data center loss
- **Physical Offsite Tape:** Archival, very durable
- **Multiple Replicas:** Read-only copies (survive single replica failure)

**Best Practice:** 3-2-1 rule:
- 3 copies of data (original + 2 backups)
- 2 different storage types (disk + cloud)
- 1 offsite (geographic separation)

### Point-In-Time Recovery (PITR)

Using WAL archiving, recover database to any historical point:

```
Full backup at 00:00: Database state at 00:00
WAL logs 00:00-12:00: All changes during period
Recovery request: Restore to 10:30
Process: Restore backup + replay logs 00:00-10:30 = database at 10:30
```

Enables recovery from accidental deletion or corruption after-the-fact.

---

## Consistency, Performance & Reliability Challenges

### Constraint Enforcement Overhead

Enforcing constraints adds overhead:

**Foreign Key Checks:**
Before DELETE/UPDATE, database checks referencing rows:
```sql
DELETE FROM users WHERE id=5
-- Database checks: any orders.user_id = 5?
-- If yes, rejects or cascades
```

Check overhead scales with number of referencing rows. Heavy references cause slowdowns.

**CHECK Constraints:**
```sql
UPDATE products SET price = ? WHERE id = ?
-- Database evaluates: price > 0?
```

Minor overhead (check evaluation fast) but multiplied across millions of rows.

**Unique Constraints:**
```sql
INSERT INTO users (email, ...) VALUES (?, ...)
-- Database checks: email already exists?
```

Requires index lookup (fast but not free).

### Encryption Performance Impact

**At-Rest Encryption (TDE):**
- Typical overhead: 5-10% (encryption/decryption CPU cost)
- Worth it for sensitive data; acceptable for most systems

**In-Transit Encryption (SSL/TLS):**
- Typical overhead: 2-5% (handshake + encryption/decryption)
- Standard practice, minimal cost

**Field-Level Encryption:**
- Overhead varies (simple XOR: <1%, complex algorithms: 5-20%)
- Application-level complexity high

### Audit Logging Performance

Full database auditing (every statement logged) can cause 10-50% performance degradation depending on:
- Log volume (simple query vs. bulk operations)
- Storage (local disk vs. remote syslog)
- Log parsing/filtering

**Optimization:**
- Log only important operations (DML, not SELECT)
- Batch logging (write logs in bulk)
- Separate audit log system (don't slow production queries)

### Key Rotation Complexity

Rotating encryption keys requires re-encrypting entire dataset:
- Halt/slow application
- Re-encrypt all data with new key
- Update key references
- Delete old key

For large datasets (TB scale), rotation takes hours, requiring maintenance window.

### Backup and Recovery Challenges

**Backup Size:**
Full backup of multi-TB database consumes massive storage. Incremental backups mitigate but require careful chain management.

**Recovery Testing:**
Backups untested often fail when needed. Regular restore tests (practice disaster) catch issues beforehand.

**Retention vs. Compliance:**
Some regulations require retaining backups for years. Storage costs scale. Automated deletion risks data loss if policies misconfigured.

### Split-Brain During Failure

Master fails; replica promotes. But old master recovers later, thinks it's still primary.

```
Master handles writes, replica has old data
Master fails at 10:00
Replica promoted, handles writes from 10:00-10:15
Master recovers at 10:15, still thinks it's primary
Both accept writes simultaneously → divergence
```

Prevention: Stonith (Shoot The Other Node In The Head) - automatically disable old master to prevent split-brain.

---

## Design Decisions & Trade-offs

### Constraint Strictness

**Strict Constraints (All Violations Rejected):**
- Prevents invalid data
- May reject legitimate bulk operations (referential integrity issues)
- Requires careful data loading (disable constraints, load, re-enable)

**Soft Constraints (Validated at Application):**
- Database permits invalid data
- Application responsible for validity
- Faster writes but risk of corruption

**Decision:** Always use database constraints for structural integrity (foreign keys, primary keys). Application constraints for business rules (e.g., discount must be within limits).

### Encryption Scope

**Full Database Encryption:**
- Simplest (transparent)
- Less granular (cannot search encrypted fields)
- All data protected equally

**Field-Level Encryption:**
- Granular (only sensitive fields encrypted)
- Complex (application handles encryption)
- Can enable selective searching (deterministic encryption)

**Decision:** Use database-level TDE for simplicity unless fine-grained encryption needed.

### Synchronous vs. Asynchronous Audit Logging

**Synchronous:**
- Every write waits for audit log write
- Guaranteed accountability
- Performance cost (audit log I/O)

**Asynchronous:**
- Audit log written in background
- No performance impact on application
- Risk: audit log loss if crash

**Decision:** Synchronous for compliance-critical (financial, medical); asynchronous for non-critical.

### Backup Frequency and RPO Trade-off

**Hourly Backups:**
- RPO: 1 hour (lose ≤1 hour of data)
- Cost: High (frequent backups)
- Use: High-value data, limited loss tolerance

**Daily Backups + WAL Archiving:**
- RPO: 1 hour (daily backup + hourly WAL replay)
- Cost: Medium (daily + WAL storage)
- Use: Standard production databases

**Weekly Backups:**
- RPO: 1 week (lose ≤1 week of data)
- Cost: Low
- Use: Development, non-critical data

**Decision:** Based on acceptable data loss and cost tolerance.

### Access Control Granularity

**Coarse (Role-Based):**
- Admin, User, Analyst roles
- Simple to manage
- Inflexible (user sees all data of their type)

**Fine-Grained (Row-Level Security):**
- User sees only their data
- Complex policies
- High security

**Hybrid:** Role-based for broad categories (admin, user), RLS for data isolation.

---

## Alternative Models & Comparisons

### Constraint Types and Their Costs

| Constraint | Type | Overhead | Importance |
|-----------|------|----------|-----------|
| Primary Key | Structural | Low (indexed) | Critical |
| Foreign Key | Referential | Medium (checks references) | Important |
| Unique | Domain | Low (indexed) | Important |
| CHECK | Domain | Negligible | Nice |
| NOT NULL | Structural | Negligible | Important |

Primary keys and foreign keys are non-negotiable. CHECK/NOT NULL relatively cheap.

### Encryption Trade-offs

| Method | Simplicity | Performance | Searchability | Security |
|--------|-----------|-------------|---------------|----------|
| TDE | High | 90% | Yes (transparent) | Good |
| Field Encrypt | Medium | 70% | Limited | Excellent |
| App-Level | Low | 50% | No | Excellent |

TDE is sweet spot for most systems.

### Auditing Scope

**Comprehensive (All Queries):**
- Complete accountability
- Massive log volume
- Performance impact

**Selective (DML Only):**
- Captures important changes
- Moderate log volume
- Acceptable performance

**Minimal (Specific Tables):**
- Focused auditing (high-value tables)
- Small log volume
- Possible gaps

**Decision:** DML-level auditing balances completeness and performance.

---

## When to Use and When to Avoid

### Always Use:

- **Primary Keys:** Every table must have unique identifier
- **Foreign Keys:** Essential for referential integrity
- **Constraints:** Prevent structural corruption
- **Encryption in Transit:** Standard practice for all databases
- **Backups:** Non-negotiable for any persistent data
- **Access Control:** Prevent unauthorized access

### Use When Critical:

- **Encryption at Rest:** Sensitive data (PII, financial, healthcare)
- **Field-Level Encryption:** Regulatory requirements (GDPR, HIPAA)
- **Audit Logging:** Compliance needs, forensics critical
- **PITR:** Quick recovery critical (financial, trading systems)

### Can Skip in Non-Critical Systems:

- **Row-Level Security:** Simple applications without sensitive segmentation
- **Asynchronous Audit Logging:** Non-critical data, loss acceptable

---

## Summary

Data integrity and security require defense in depth. Database constraints prevent structural corruption; encryption protects from snooping; access controls prevent unauthorized access; auditing provides accountability; backups enable recovery. Each layer adds overhead but is essential for reliable, secure systems. Understanding constraints' role (fast, structural), encryption's limitations (doesn't prevent application attacks), RBAC's simplicity (role-based), and backup strategies (RPO/RTO trade-offs) enables architects to design systems appropriate for their threat model and operational requirements.
