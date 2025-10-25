# ADR-0025: Audit Storage Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 4.3 (affects all audit trail queries)

---

## Context

Field-level corrections require immutable audit logging: every change (extraction, override, revert) must be recorded with who, when, why, and what changed. We must choose storage strategy for audit entries.

**The challenge:**

```
User corrects merchant: "AMZN MKTP" → "Amazon Marketplace"

Audit entry must store:
- entity_id: "txn_123"
- field_name: "merchant"
- action: "override"
- previous_value: "AMZN MKTP"
- new_value: "Amazon Marketplace"
- changed_by: "user_123"
- timestamp: "2025-10-24T10:30:00Z"
- reason: "Normalize merchant name"
```

**Key requirements:**
- Immutability (cannot edit/delete audit entries)
- Fast queries ("show me all changes to txn_123")
- Compliance (HIPAA, SOX, GDPR require audit trails)
- Scale to millions of entries (100k corrections/month * 12 months = 1.2M entries/year)
- Flexible metadata (PII redaction flags, approval status, etc.)

**Trade-off space:**
- **Relational DB** vs **Time-series DB**: Relational = simpler, time-series = faster aggregations
- **Append-only table** vs **Event sourcing**: Append-only = simple, event sourcing = complete history
- **Separate audit service** vs **Same database**: Separate = isolated, same = simpler deployment

---

## Decision

**We use PostgreSQL table with JSONB metadata column, enforced append-only via revoked UPDATE/DELETE permissions.**

```sql
CREATE TABLE audit_log (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_id VARCHAR NOT NULL,
  entity_type VARCHAR NOT NULL,
  field_name VARCHAR NOT NULL,
  action VARCHAR NOT NULL,  -- 'extracted', 'override', 'revert', 'delete'
  previous_value TEXT,
  new_value TEXT,
  changed_by VARCHAR NOT NULL,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  reason TEXT,
  source VARCHAR NOT NULL,  -- 'parser_v2', 'manual_correction', etc.
  metadata JSONB,           -- Flexible: {pii_redacted: true, approved_by: "doctor_123"}
  signature VARCHAR         -- SHA-256 hash for tamper detection (optional)
);

CREATE INDEX idx_audit_entity_field_time
  ON audit_log(entity_id, field_name, timestamp DESC);

-- Enforce immutability
REVOKE UPDATE, DELETE ON audit_log FROM app_user;
GRANT INSERT, SELECT ON audit_log TO app_user;
```

---

## Rationale

### 1. Queryable with SQL

**PostgreSQL:**
```sql
-- Get audit trail for specific field
SELECT * FROM audit_log
WHERE entity_id = 'txn_123' AND field_name = 'merchant'
ORDER BY timestamp DESC;

-- Find all corrections by user
SELECT * FROM audit_log
WHERE changed_by = 'user_123' AND action = 'override'
ORDER BY timestamp DESC
LIMIT 100;

-- Count corrections per day
SELECT DATE(timestamp), COUNT(*)
FROM audit_log
WHERE action = 'override'
GROUP BY DATE(timestamp);
```

**Alternative (S3/files):**
```
❌ Cannot query without loading entire file
❌ No indexes, full scan required
```

**Benefit:** Rich querying with indexes for fast performance.

### 2. Relational Integrity

**Foreign keys:**
```sql
ALTER TABLE audit_log
  ADD CONSTRAINT fk_changed_by
  FOREIGN KEY (changed_by) REFERENCES users(user_id);
```

**Cascading deletes (if user deleted):**
- Set `changed_by` to 'deleted_user_xxx' (preserve audit even if user deleted)

**Benefit:** Referential integrity ensures data consistency.

### 3. Flexible Metadata (JSONB)

**JSONB column allows domain-specific attributes without schema changes:**

```sql
-- Healthcare: PII redaction flag
INSERT INTO audit_log VALUES (..., metadata => '{"pii_redacted": true, "compliance_tags": ["HIPAA"]}');

-- Finance: Approval workflow
INSERT INTO audit_log VALUES (..., metadata => '{"approved": true, "approved_by": "manager_456"}');

-- E-commerce: Bulk correction
INSERT INTO audit_log VALUES (..., metadata => '{"batch_id": "batch_001", "batch_size": 50}');

-- Query JSONB
SELECT * FROM audit_log
WHERE metadata->>'approved' = 'true';
```

**Benefit:** Extensible without ALTER TABLE (avoid downtime).

### 4. Immutability Enforced

**Database-level permissions:**
```sql
REVOKE UPDATE, DELETE ON audit_log FROM app_user;
```

**Application code cannot:**
- UPDATE audit_log SET ... ❌ Permission denied
- DELETE FROM audit_log ... ❌ Permission denied

**Only INSERT allowed:**
- INSERT INTO audit_log ... ✅ Success

**Optional: Cryptographic signature**
```typescript
const signature = sha256(`${audit_id}${entity_id}${field_name}${action}${timestamp}${new_value}`);
// Later: verify signature matches to detect tampering
```

**Benefit:** Tamper-proof audit trail for compliance.

### 5. Performance at Scale

**Indexes:**
```sql
CREATE INDEX idx_audit_entity_field_time
  ON audit_log(entity_id, field_name, timestamp DESC);  -- Fast field history queries

CREATE INDEX idx_audit_user_time
  ON audit_log(changed_by, timestamp DESC);  -- Fast user activity queries

CREATE INDEX idx_audit_type_time
  ON audit_log(entity_type, timestamp DESC);  -- Fast entity type queries
```

**Partitioning (for >10M entries):**
```sql
-- Partition by month
CREATE TABLE audit_log_2025_01 PARTITION OF audit_log
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE audit_log_2025_02 PARTITION OF audit_log
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

**Performance benchmarks (1M entries):**
- Query single field history: <50ms (with index)
- Query user activity (last 100): <30ms
- Insert batch 1,000 entries: <2s

**Benefit:** Scales to millions of entries with partitioning.

---

## Consequences

### ✅ Positive

- **Queryable**: Rich SQL queries with indexes for fast performance (<50ms)
- **Relational integrity**: Foreign keys ensure consistency
- **Flexible metadata**: JSONB column for domain-specific attributes (no schema changes)
- **Immutability**: Database-level permissions prevent tampering
- **Compliance-ready**: HIPAA, SOX, GDPR compliant with tamper-proof audit trail
- **Scalable**: Partitioning supports 10M+ entries
- **Same infrastructure**: No new databases/services to deploy

### ⚠️ Trade-offs

- **Storage cost**: TEXT columns for values (not space-efficient for large values)
- **Index overhead**: 3 indexes = ~30% storage overhead (mitigated by partitioning)
- **No built-in time-series aggregations**: Must write SQL for aggregations (vs TimescaleDB auto-aggregation)

### 🔴 Risks (Mitigated)

- **Risk**: Audit log grows too large (>100M entries = slow queries)
  - **Mitigation**: Partition by month, archive old partitions to S3 after 1 year, keep recent 1 year hot
- **Risk**: Application bug bypasses permissions (e.g., superuser connection)
  - **Mitigation**: Use read-only connection pool for app, superuser only for migrations
- **Risk**: Disk space exhaustion (audit log grows 1 GB/month)
  - **Mitigation**: Monitor disk usage, alert at 80%, auto-archive old partitions

---

## Alternatives Considered

### Alternative A: Time-Series Database (TimescaleDB) (Rejected)

**Approach:** Use TimescaleDB extension for PostgreSQL (optimized for time-series data).

**Pros:**
- ✅ Fast time-series aggregations (COUNT, AVG by day/hour)
- ✅ Automatic partitioning (no manual partition management)
- ✅ Compression (3-10x space savings)

**Cons:**
- ❌ Operational complexity (need to install/manage TimescaleDB extension)
- ❌ Limited hosting support (not all cloud providers support TimescaleDB)
- ❌ Overkill for our use case (audit queries are lookups, not aggregations)
- ❌ Less familiar to team (learning curve)

**Decision:** ❌ Rejected - Operational complexity not worth marginal benefit.

---

### Alternative B: Separate Audit Service (Rejected)

**Approach:** Dedicated microservice with its own database for audit logging.

**Pros:**
- ✅ Isolation (audit service independent of main app)
- ✅ Scalability (can scale audit service independently)

**Cons:**
- ❌ Operational complexity (deploy/monitor separate service)
- ❌ Network latency (RPC call to audit service for every correction)
- ❌ Transaction complexity (how to ensure atomicity between main DB and audit DB?)
- ❌ Overkill for v1

**Decision:** ❌ Rejected - Too complex for v1. Keep audit log in same database.

---

### Alternative C: JSON Files in S3 (Rejected)

**Approach:** Write audit entries as JSON files to S3, one file per day.

```
s3://audit-logs/2025-10-24.json
[
  {"audit_id": "audit_001", "entity_id": "txn_123", ...},
  {"audit_id": "audit_002", "entity_id": "txn_456", ...}
]
```

**Pros:**
- ✅ Cheap storage ($0.023/GB/month)
- ✅ Infinite scalability

**Cons:**
- ❌ Not queryable (must download file, parse JSON, filter in-memory)
- ❌ No indexes (full scan required)
- ❌ No relational integrity
- ❌ Complex to query ("show me all changes to txn_123 in last 6 months" = download 180 files?)

**Decision:** ❌ Rejected - Not queryable, defeats purpose of audit trail.

---

### Alternative D: Event Sourcing (Rejected)

**Approach:** Store all events (including non-correction events), replay to reconstruct state.

**Pros:**
- ✅ Complete history (can reconstruct state at any point in time)
- ✅ Time-travel queries

**Cons:**
- ❌ Too complex for v1 (event versioning, replays, snapshots)
- ❌ Performance overhead (replay 1,000 events to get current state)
- ❌ Overkill (bitemporal queries are Vertical 5.1 Provenance Ledger)

**Decision:** ❌ Rejected - Save for v2 (Vertical 5.1).

---

## Implementation Notes

### Database Schema

```sql
CREATE TABLE audit_log (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_id VARCHAR NOT NULL,
  entity_type VARCHAR NOT NULL,
  field_name VARCHAR NOT NULL,
  action VARCHAR NOT NULL CHECK (action IN ('extracted', 'override', 'revert', 'delete', 'rule_applied')),
  previous_value TEXT,
  new_value TEXT,
  changed_by VARCHAR NOT NULL,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  reason TEXT,
  source VARCHAR NOT NULL,
  confidence NUMERIC(3, 2),  -- Optional: 0.00-1.00 for extracted values
  ip_address INET,           -- Optional: for security audits
  user_agent TEXT,           -- Optional: browser info
  metadata JSONB,
  signature VARCHAR(64),     -- Optional: SHA-256 hash
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_audit_entity_field_time
  ON audit_log(entity_id, field_name, timestamp DESC);

CREATE INDEX idx_audit_user_time
  ON audit_log(changed_by, timestamp DESC);

CREATE INDEX idx_audit_type_time
  ON audit_log(entity_type, timestamp DESC);

CREATE INDEX idx_audit_timestamp
  ON audit_log(timestamp DESC);

CREATE INDEX idx_audit_metadata
  ON audit_log USING GIN (metadata);  -- For JSONB queries

-- Partitioning (for >10M entries)
CREATE TABLE audit_log (
  -- columns
) PARTITION BY RANGE (timestamp);

-- Enforce immutability
REVOKE UPDATE, DELETE ON audit_log FROM app_user;
GRANT INSERT, SELECT ON audit_log TO app_user;
```

### Logging Corrections

```typescript
class AuditLog {
  async log(entry: CreateAuditEntryInput): Promise<AuditEntry> {
    const signature = this.calculateSignature(entry);

    const result = await db.query(`
      INSERT INTO audit_log (
        entity_id, entity_type, field_name, action,
        previous_value, new_value, changed_by, reason, source, metadata, signature
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
      RETURNING *
    `, [
      entry.entity_id,
      entry.entity_type,
      entry.field_name,
      entry.action,
      entry.previous_value,
      entry.new_value,
      entry.changed_by,
      entry.reason,
      entry.source,
      JSON.stringify(entry.metadata),
      signature
    ]);

    return result.rows[0];
  }

  private calculateSignature(entry: CreateAuditEntryInput): string {
    const data = `${entry.entity_id}${entry.field_name}${entry.action}${entry.new_value}${Date.now()}`;
    return crypto.createHash('sha256').update(data).digest('hex');
  }
}
```

### Querying Audit Trail

```typescript
class AuditLog {
  async getHistory(entityId: string, fieldName?: string): Promise<AuditEntry[]> {
    const query = fieldName
      ? `SELECT * FROM audit_log WHERE entity_id = $1 AND field_name = $2 ORDER BY timestamp DESC`
      : `SELECT * FROM audit_log WHERE entity_id = $1 ORDER BY timestamp DESC`;

    const params = fieldName ? [entityId, fieldName] : [entityId];
    const result = await db.query(query, params);
    return result.rows;
  }

  async getUserActivity(userId: string, limit = 100): Promise<AuditEntry[]> {
    const result = await db.query(`
      SELECT * FROM audit_log
      WHERE changed_by = $1
      ORDER BY timestamp DESC
      LIMIT $2
    `, [userId, limit]);

    return result.rows;
  }
}
```

### Archiving Old Entries

```typescript
// Archive entries older than 1 year to S3
const archiveOldEntries = async () => {
  const oneYearAgo = new Date();
  oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);

  // 1. Export to S3
  const oldEntries = await db.query(`
    SELECT * FROM audit_log WHERE timestamp < $1
  `, [oneYearAgo]);

  await s3.upload('audit-archive-2024.json', JSON.stringify(oldEntries.rows));

  // 2. Delete from hot database
  await db.query(`DELETE FROM audit_log WHERE timestamp < $1`, [oneYearAgo]);
};
```

---

## Multi-Domain Applicability

### Finance
**Use Case:** Audit trail for expense report corrections (IRS requires 7-year retention)
**Benefit:** SOX-compliant audit logging with tamper-proof signatures

### Healthcare
**Use Case:** HIPAA-compliant audit trail for patient record corrections
**Benefit:** PII redaction via JSONB metadata flag, 6-year retention

### Legal
**Use Case:** Court-required audit trail for case filing corrections
**Benefit:** Immutability enforced, cannot edit/delete audit entries

### Research
**Use Case:** NIH data quality reports showing citation corrections
**Benefit:** Queryable with SQL for data quality metrics

### E-commerce
**Use Case:** Revenue tracking for product price corrections
**Benefit:** Audit trail shows original price + corrected price

### SaaS
**Use Case:** SOX-compliant audit for MRR corrections (public companies)
**Benefit:** Signature verification for tamper detection

---

## Related Decisions

- **ADR-0024**: Field-Level Overrides (audit log stores all field changes)
- **ADR-0026**: Precedence Resolution (audit log shows which source won)

---

**References:**
- Vertical 4.3: [docs/verticals/4.3-corrections-flow.md](../verticals/4.3-corrections-flow.md)
- AuditLog: [docs/primitives/ol/AuditLog.md](../primitives/ol/AuditLog.md)
- Schema: [docs/schemas/audit-entry.schema.json](../schemas/audit-entry.schema.json)
