# AuditLogger OL Primitive

**Domain:** Security & Access (Vertical 5.4)
**Layer:** Objective Layer (OL)
**Version:** 1.0.0
**Status:** Specification

---

## Table of Contents

1. [Overview](#overview)
2. [Purpose & Scope](#purpose--scope)
3. [Multi-Domain Applicability](#multi-domain-applicability)
4. [Core Concepts](#core-concepts)
5. [Interface Definition](#interface-definition)
6. [Data Model](#data-model)
7. [Core Functionality](#core-functionality)
8. [Event Types](#event-types)
9. [Append-Only Architecture](#append-only-architecture)
10. [Integrity Verification](#integrity-verification)
11. [Edge Cases](#edge-cases)
12. [Performance Characteristics](#performance-characteristics)
13. [Implementation Notes](#implementation-notes)
14. [Security Considerations](#security-considerations)
15. [Integration Patterns](#integration-patterns)
16. [Multi-Domain Examples](#multi-domain-examples)
17. [Testing Strategy](#testing-strategy)
18. [Configuration](#configuration)
19. [Observability](#observability)
20. [Future Enhancements](#future-enhancements)

---

## Overview

The **AuditLogger** provides immutable audit trails for compliance and security. It implements append-only logging with integrity verification (SHA-256 chain), supports 5 event types, and enables 7-year retention with compliance exports (JSON/CSV/PDF).

### Key Capabilities

- **5 Event Types**: data_access, data_modification, user_action, security_event, system_event
- **Append-Only Storage**: PostgreSQL with no UPDATE/DELETE (immutable)
- **Integrity Hashing**: SHA-256 chain for tamper detection
- **7-Year Retention**: Configurable retention policy
- **Compliance Exports**: JSON, CSV, PDF formats
- **High Performance**: <8ms p95 async writes, batching (50 events/100ms)
- **Query API**: Fast queries by user, resource, event type, time range

### Design Philosophy

The AuditLogger follows four core principles:

1. **Immutability**: Audit logs cannot be modified or deleted
2. **Integrity**: Tamper detection via cryptographic hashing
3. **Compliance**: 7-year retention, exportable reports
4. **Performance**: <8ms p95 with async writes and batching

---

## Purpose & Scope

### Problem Statement

In regulated industries, audit logging poses critical challenges:

- **Regulatory Requirements**: HIPAA, SOX, GDPR require audit trails
- **Tamper Detection**: No way to prove logs weren't modified
- **Slow Writes**: Synchronous logging adds 20-50ms per operation
- **Query Performance**: Querying millions of logs is slow
- **Retention Management**: Manual log cleanup is error-prone
- **Export Formats**: Auditors require specific export formats

Traditional approaches have significant limitations:

**Approach 1: Application Logs**
- ❌ Mutable (can be edited)
- ❌ No integrity verification
- ❌ No structured querying

**Approach 2: Database INSERT-only**
- ❌ Still allows UPDATE/DELETE at DB level
- ❌ No integrity verification
- ❌ Slow writes (synchronous)

**Approach 3: AuditLogger Primitive**
- ✅ Append-only (no UPDATE/DELETE)
- ✅ Integrity verification (SHA-256 chain)
- ✅ <8ms p95 (async + batching)
- ✅ 7-year retention
- ✅ Compliance exports (JSON/CSV/PDF)

### Solution

The AuditLogger implements **immutable audit logging** with:

1. **Append-Only Storage**: PostgreSQL with no UPDATE/DELETE triggers
2. **Integrity Chain**: SHA-256 hash of previous event included in current event
3. **Async Batching**: Buffer events for 100ms, batch insert (50 events at once)
4. **Fast Queries**: Indexes on user_id, resource, event_type, timestamp
5. **Retention Policy**: Automatic archival after 7 years (configurable)
6. **Compliance Exports**: Generate JSON/CSV/PDF reports for auditors

---

## Multi-Domain Applicability

The AuditLogger is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Audit all access to financial transactions, statements, and reports.

**Audit Requirements**:
- SOX: Audit all changes to financial data
- Retention: 7 years
- Exports: CSV for external auditors

**Events Logged**:
- `data_access`: User viewed bank statement
- `data_modification`: User categorized transaction
- `user_action`: User exported financial report
- `security_event`: Failed login attempt
- `system_event`: Automatic bank sync completed

**Example**:
```typescript
// Log transaction access
await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "user_123",
  resource: "transaction:txn_456",
  action: "view",
  tenant_id: "tenant_abc",
  metadata: {
    ip_address: "192.168.1.100",
    user_agent: "Mozilla/5.0...",
    access_method: "web_ui"
  }
});

// Log transaction modification
await auditLogger.logEvent({
  event_type: "data_modification",
  user_id: "user_123",
  resource: "transaction:txn_456",
  action: "update",
  tenant_id: "tenant_abc",
  changes: {
    category: { old: "uncategorized", new: "groceries" }
  },
  metadata: {
    reason: "Manual categorization"
  }
});

// Query audit logs for transaction
const logs = await auditLogger.queryLogs({
  resource: "transaction:txn_456",
  tenant_id: "tenant_abc",
  time_range: "30d"
});

console.log(`Found ${logs.length} audit events for transaction`);
```

**Performance Target**: <8ms p95 per log event.

---

### 2. Healthcare

**Use Case**: Audit all PHI access (HIPAA requirement).

**Audit Requirements**:
- HIPAA: Audit all PHI access and modifications
- Retention: 6 years (HIPAA) or longer
- Exports: PDF for compliance audits

**Events Logged**:
- `data_access`: Physician viewed patient record
- `data_modification`: Nurse added clinical note
- `user_action`: Patient exported medical records
- `security_event`: Unauthorized access attempt blocked
- `system_event`: HL7 message processed

**Example**:
```typescript
// Log PHI access
await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "physician_123",
  resource: "patient:pt_456",
  action: "view",
  tenant_id: "hospital_abc",
  metadata: {
    access_reason: "Treatment",
    department: "Emergency",
    ip_address: "10.0.1.50"
  }
});

// Log PHI modification
await auditLogger.logEvent({
  event_type: "data_modification",
  user_id: "nurse_789",
  resource: "patient:pt_456:vitals",
  action: "update",
  tenant_id: "hospital_abc",
  changes: {
    blood_pressure: { old: null, new: "120/80" },
    heart_rate: { old: null, new: 75 }
  },
  metadata: {
    encounter_id: "enc_012"
  }
});

// Export audit logs for compliance
const pdfReport = await auditLogger.exportLogs({
  resource: "patient:pt_456",
  tenant_id: "hospital_abc",
  format: "pdf",
  time_range: "1y"
});
```

**Performance Target**: <8ms p95, HIPAA compliant.

---

### 3. Legal

**Use Case**: Audit all access to case files and privileged documents.

**Audit Requirements**:
- Ethical compliance: Audit who accessed privileged documents
- Retention: Indefinite (case lifecycle)
- Exports: CSV for opposing counsel discovery

**Events Logged**:
- `data_access`: Attorney viewed privileged memo
- `data_modification`: Paralegal redacted document
- `user_action`: Client downloaded case documents
- `security_event`: Unauthorized user blocked
- `system_event`: Document auto-classification completed

**Example**:
```typescript
// Log privileged document access
await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "attorney_123",
  resource: "document:doc_privileged_789",
  action: "view",
  tenant_id: "law_firm_abc",
  metadata: {
    case_id: "case_456",
    privilege_type: "attorney_client",
    access_purpose: "Case preparation"
  }
});

// Query all access to privileged documents
const privilegedAccess = await auditLogger.queryLogs({
  resource_pattern: "document:doc_privileged_*",
  tenant_id: "law_firm_abc",
  time_range: "all"
});

console.log(`${privilegedAccess.length} accesses to privileged documents`);
```

**Performance Target**: <8ms p95 per event.

---

### 4. HR (Human Resources)

**Use Case**: Audit all access to employee records and compensation data.

**Audit Requirements**:
- GDPR: Audit all personal data access
- Retention: 3 years (GDPR)
- Exports: JSON for data subject access requests (DSAR)

**Events Logged**:
- `data_access`: HR manager viewed employee record
- `data_modification`: Manager updated performance review
- `user_action`: Employee downloaded pay stubs
- `security_event`: Failed login to HR system
- `system_event`: Payroll auto-calculated

**Example**:
```typescript
// Log employee record access
await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "hr_manager_123",
  resource: "employee:emp_456",
  action: "view",
  tenant_id: "company_abc",
  metadata: {
    access_reason: "Performance review",
    department: "HR"
  }
});

// Employee requests audit logs (GDPR right)
const employeeAudit = await auditLogger.exportLogs({
  user_id: "emp_456",
  tenant_id: "company_abc",
  format: "json",
  time_range: "all"
});

// Returns all events where employee is subject or actor
```

**Performance Target**: <8ms p95, GDPR compliant.

---

### 5. E-commerce

**Use Case**: Audit all access to customer payment info and order history.

**Audit Requirements**:
- PCI DSS: Audit all access to cardholder data
- Retention: 1 year (PCI DSS minimum)
- Exports: CSV for compliance audits

**Events Logged**:
- `data_access`: CS agent viewed order
- `data_modification`: Customer updated shipping address
- `user_action`: Customer downloaded invoice
- `security_event`: Failed payment attempt
- `system_event`: Order auto-fulfilled

**Example**:
```typescript
// Log payment info access
await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "cs_agent_123",
  resource: "order:ord_456:payment",
  action: "view",
  tenant_id: "store_abc",
  metadata: {
    access_reason: "Customer inquiry",
    ticket_id: "ticket_789",
    pii_masked: true // Payment info was masked
  }
});

// Log payment modification
await auditLogger.logEvent({
  event_type: "data_modification",
  user_id: "customer_456",
  resource: "order:ord_456:payment",
  action: "update",
  tenant_id: "store_abc",
  changes: {
    payment_method: { old: "card_***1111", new: "card_***2222" }
  }
});
```

**Performance Target**: <8ms p95, PCI DSS compliant.

---

### 6. SaaS

**Use Case**: Audit all API key access and account configuration changes.

**Audit Requirements**:
- SOC 2: Audit all security-relevant events
- Retention: 1 year
- Exports: JSON for customer security audits

**Events Logged**:
- `data_access`: User viewed API key
- `data_modification`: User regenerated API key
- `user_action`: User invited team member
- `security_event`: API rate limit exceeded
- `system_event`: Subscription auto-renewed

**Example**:
```typescript
// Log API key regeneration
await auditLogger.logEvent({
  event_type: "data_modification",
  user_id: "user_123",
  resource: "api_key:key_456",
  action: "regenerate",
  tenant_id: "account_abc",
  changes: {
    key_value: { old: "tc_live_old***", new: "tc_live_new***" }
  },
  metadata: {
    ip_address: "203.0.113.50",
    reason: "Security incident response"
  }
});

// Query security events
const securityEvents = await auditLogger.queryLogs({
  event_type: "security_event",
  tenant_id: "account_abc",
  time_range: "7d"
});
```

**Performance Target**: <8ms p95.

---

### 7. Government

**Use Case**: Audit all access to classified documents and citizen records.

**Audit Requirements**:
- FISMA: Audit all access to sensitive data
- Retention: 7 years (NARA)
- Exports: PDF for inspector general audits

**Events Logged**:
- `data_access`: Analyst viewed classified document
- `data_modification`: Originator declassified document
- `user_action`: Citizen requested records (FOIA)
- `security_event`: Clearance level violation blocked
- `system_event`: Classification auto-review completed

**Example**:
```typescript
// Log classified document access
await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "analyst_123",
  resource: "document:doc_classified_456",
  action: "view",
  tenant_id: "agency_abc",
  metadata: {
    classification_level: "SECRET",
    analyst_clearance: "TOP_SECRET",
    access_justification: "Intelligence analysis",
    supervisor_approval: "sup_789"
  }
});

// Verify audit log integrity (tamper detection)
const integrityCheck = await auditLogger.verifyIntegrity({
  tenant_id: "agency_abc",
  time_range: "30d"
});

console.log(`Integrity check: ${integrityCheck.valid ? 'PASS' : 'FAIL'}`);
```

**Performance Target**: <8ms p95, FISMA compliant.

---

### Cross-Domain Benefits

**Consistent Audit Trails**: All domains benefit from:
- Immutable audit logs
- Cryptographic integrity verification
- Fast async writes (<8ms p95)
- Compliance exports (JSON/CSV/PDF)

**Regulatory Compliance**: Supports:
- HIPAA (Healthcare): PHI access audit trail
- PCI DSS (Payments): Cardholder data access audit
- SOX (Finance): Financial data modification audit
- GDPR (Privacy): Personal data processing records
- FISMA (Government): Security audit trail

---

## Core Concepts

### Event Types

The AuditLogger supports 5 primary event types:

1. **data_access**: User viewed data (read-only)
2. **data_modification**: User created/updated/deleted data
3. **user_action**: User performed action (e.g., export, invite)
4. **security_event**: Security-relevant event (e.g., failed login, blocked access)
5. **system_event**: System-generated event (e.g., auto-sync, cron job)

### Append-Only Architecture

**Immutability**:
- Audit logs can only be INSERTed, never UPDATEd or DELETEd
- Database triggers prevent UPDATE/DELETE
- Retention policy archives old logs (but doesn't delete)

**Benefits**:
- Tamper-proof: Logs cannot be modified after creation
- Compliance: Meets regulatory requirements for immutable logs
- Forensics: Complete history of events

### Integrity Verification

**SHA-256 Hash Chain**:
- Each event includes hash of previous event
- Creates chain: Event1 → Event2 → Event3 → ...
- Tampering breaks the chain (detectable)

**Chain Structure**:
```
Event 1: { data: "...", prev_hash: null, hash: "abc123..." }
Event 2: { data: "...", prev_hash: "abc123...", hash: "def456..." }
Event 3: { data: "...", prev_hash: "def456...", hash: "ghi789..." }
```

---

## Interface Definition

### TypeScript Interface

```typescript
interface AuditLogger {
  /**
   * Log audit event
   *
   * @param event - Audit event to log
   * @returns Event ID
   * @throws ValidationError if event is invalid
   */
  logEvent(event: AuditEvent): Promise<string>;

  /**
   * Log multiple events in batch
   *
   * @param events - Array of audit events
   * @returns Array of event IDs
   * @throws ValidationError if any event is invalid
   */
  logBatch(events: AuditEvent[]): Promise<string[]>;

  /**
   * Query audit logs
   *
   * @param query - Query parameters
   * @returns Array of audit events
   */
  queryLogs(query: AuditLogQuery): Promise<AuditEvent[]>;

  /**
   * Export audit logs to file
   *
   * @param request - Export request
   * @returns Export result with download URL
   */
  exportLogs(request: ExportRequest): Promise<ExportResult>;

  /**
   * Verify integrity of audit log chain
   *
   * @param request - Integrity check request
   * @returns Verification result
   */
  verifyIntegrity(request: IntegrityCheckRequest): Promise<IntegrityCheckResult>;

  /**
   * Get audit logger statistics
   *
   * @returns Audit logger stats
   */
  getStats(): AuditLoggerStats;
}
```

---

## Data Model

### AuditEvent Type

```typescript
interface AuditEvent {
  // Event Type
  event_type: EventType;          // "data_access" | "data_modification" | ...

  // Actor
  user_id: string;                // User who triggered event

  // Resource
  resource: string;               // Resource affected (e.g., "transaction:txn_123")
  action: string;                 // Action performed (e.g., "view", "update", "delete")

  // Tenant Context
  tenant_id: string;              // Tenant identifier

  // Changes (for data_modification events)
  changes?: Record<string, { old: any; new: any }>;

  // Metadata
  metadata?: Record<string, any>; // Additional context

  // Timestamp (auto-generated)
  timestamp?: string;             // ISO timestamp
}
```

### AuditLogQuery Type

```typescript
interface AuditLogQuery {
  // Filters
  event_type?: EventType;         // Filter by event type
  user_id?: string;               // Filter by user
  resource?: string;              // Filter by resource (exact match)
  resource_pattern?: string;      // Filter by resource pattern (wildcard)
  action?: string;                // Filter by action
  tenant_id: string;              // Tenant context (required)

  // Time Range
  time_range?: string;            // "7d", "30d", "1y", "all"
  start_time?: string;            // ISO timestamp
  end_time?: string;              // ISO timestamp

  // Pagination
  limit?: number;                 // Max results (default: 100)
  offset?: number;                // Offset for pagination
}
```

### ExportRequest Type

```typescript
interface ExportRequest {
  // Query
  query: AuditLogQuery;           // Query to export

  // Format
  format: ExportFormat;           // "json" | "csv" | "pdf"

  // Options
  include_metadata?: boolean;     // Include full metadata (default: true)
  time_range?: string;            // Time range to export
}
```

### ExportResult Type

```typescript
interface ExportResult {
  export_id: string;              // Export identifier
  format: ExportFormat;
  download_url: string;           // URL to download export
  file_size_bytes: number;
  events_count: number;
  generated_at: string;           // ISO timestamp
  expires_at: string;             // Download URL expiration
}
```

### IntegrityCheckRequest Type

```typescript
interface IntegrityCheckRequest {
  tenant_id: string;              // Tenant context
  time_range?: string;            // Time range to verify (default: "all")
  start_time?: string;            // ISO timestamp
  end_time?: string;              // ISO timestamp
}
```

### IntegrityCheckResult Type

```typescript
interface IntegrityCheckResult {
  valid: boolean;                 // true if chain is intact
  events_checked: number;
  first_invalid_event?: string;   // Event ID of first tampered event
  error_message?: string;         // Error description if invalid
  checked_at: string;             // ISO timestamp
}
```

### EventType Enum

```typescript
enum EventType {
  DATA_ACCESS = "data_access",
  DATA_MODIFICATION = "data_modification",
  USER_ACTION = "user_action",
  SECURITY_EVENT = "security_event",
  SYSTEM_EVENT = "system_event"
}
```

### ExportFormat Enum

```typescript
enum ExportFormat {
  JSON = "json",
  CSV = "csv",
  PDF = "pdf"
}
```

### AuditLoggerStats Type

```typescript
interface AuditLoggerStats {
  // Throughput
  events_logged_total: number;
  events_per_second: number;

  // Performance
  log_latency_p50_ms: number;
  log_latency_p95_ms: number;
  log_latency_p99_ms: number;

  // By Type
  events_by_type: Record<EventType, number>;

  // Storage
  total_events_stored: number;
  oldest_event_timestamp: string;
  newest_event_timestamp: string;

  // Integrity
  last_integrity_check_at?: string;
  last_integrity_check_result?: boolean;

  // Errors
  logging_errors_total: number;
}
```

### Database Schema (PostgreSQL)

```sql
-- Audit events table (append-only)
CREATE TABLE audit_events (
  -- Primary Key
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Event Type
  event_type VARCHAR(32) NOT NULL,

  -- Actor
  user_id VARCHAR(128) NOT NULL,

  -- Resource
  resource VARCHAR(256) NOT NULL,
  action VARCHAR(64) NOT NULL,

  -- Tenant Context
  tenant_id VARCHAR(128) NOT NULL,

  -- Changes (for data_modification)
  changes JSONB,

  -- Metadata
  metadata JSONB,

  -- Integrity Chain
  prev_hash VARCHAR(64),          -- SHA-256 hash of previous event
  current_hash VARCHAR(64) NOT NULL, -- SHA-256 hash of this event

  -- Timestamp
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Prevent UPDATE/DELETE (append-only)
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Audit logs are immutable and cannot be modified or deleted';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER prevent_update_audit_events
BEFORE UPDATE ON audit_events
FOR EACH ROW
EXECUTE FUNCTION prevent_audit_modification();

CREATE TRIGGER prevent_delete_audit_events
BEFORE DELETE ON audit_events
FOR EACH ROW
EXECUTE FUNCTION prevent_audit_modification();

-- Indexes for fast queries
CREATE INDEX idx_audit_events_user_id ON audit_events(user_id);
CREATE INDEX idx_audit_events_resource ON audit_events(resource);
CREATE INDEX idx_audit_events_event_type ON audit_events(event_type);
CREATE INDEX idx_audit_events_tenant_id ON audit_events(tenant_id);
CREATE INDEX idx_audit_events_created_at ON audit_events(created_at DESC);
CREATE INDEX idx_audit_events_tenant_created ON audit_events(tenant_id, created_at DESC);

-- GIN index for JSONB columns
CREATE INDEX idx_audit_events_metadata ON audit_events USING GIN(metadata);
CREATE INDEX idx_audit_events_changes ON audit_events USING GIN(changes);

-- Partitioning by month (for performance)
-- CREATE TABLE audit_events_2025_01 PARTITION OF audit_events
--   FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

---

## Core Functionality

### 1. logEvent()

Log audit event to append-only storage.

#### Signature

```typescript
logEvent(event: AuditEvent): Promise<string>
```

#### Behavior

1. **Validate Event**:
   - Validate required fields
   - Validate tenant_id

2. **Buffer Event**:
   - Add to in-memory buffer
   - Trigger batch insert if buffer reaches threshold (50 events)

3. **Batch Insert** (async):
   - Calculate SHA-256 hash chain
   - INSERT batch into `audit_events` table

4. **Return**:
   - Return event_id immediately (before batch insert completes)

#### Algorithm

```typescript
async logEvent(event: AuditEvent): Promise<string> {
  // 1. Validate
  if (!event.user_id || !event.resource || !event.tenant_id) {
    throw new ValidationError('user_id, resource, and tenant_id are required');
  }

  // 2. Generate event ID
  const event_id = randomUUID();

  // 3. Add timestamp
  event.timestamp = new Date().toISOString();

  // 4. Buffer event
  this.buffer.push({ event_id, event });

  // 5. Trigger batch insert if threshold reached
  if (this.buffer.length >= this.batchSize) {
    this.flushBuffer(); // Async, don't wait
  }

  // 6. Return event ID immediately
  return event_id;
}

private async flushBuffer(): Promise<void> {
  if (this.buffer.length === 0) return;

  const batch = this.buffer.splice(0, this.batchSize);

  // Calculate hash chain
  const lastEvent = await this.getLastEvent(batch[0].event.tenant_id);
  let prevHash = lastEvent?.current_hash || null;

  for (const item of batch) {
    const eventData = JSON.stringify({
      event_type: item.event.event_type,
      user_id: item.event.user_id,
      resource: item.event.resource,
      action: item.event.action,
      tenant_id: item.event.tenant_id,
      changes: item.event.changes,
      metadata: item.event.metadata,
      timestamp: item.event.timestamp
    });

    const currentHash = createHash('sha256')
      .update(prevHash || '')
      .update(eventData)
      .digest('hex');

    item.prev_hash = prevHash;
    item.current_hash = currentHash;

    prevHash = currentHash;
  }

  // Bulk insert
  await this.bulkInsert(batch);
}
```

#### Example

```typescript
const event_id = await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "user_123",
  resource: "transaction:txn_456",
  action: "view",
  tenant_id: "tenant_abc",
  metadata: {
    ip_address: "192.168.1.100"
  }
});

console.log(`Logged event: ${event_id}`);
```

#### Performance

- **Target Latency**: <8ms p95 (async + batching)
- **Throughput**: 5K events/sec

---

### 2. logBatch()

Log multiple events in batch.

#### Signature

```typescript
logBatch(events: AuditEvent[]): Promise<string[]>
```

#### Behavior

1. **Validate All Events**:
   - Validate each event

2. **Buffer All Events**:
   - Add all to buffer

3. **Flush**:
   - Flush buffer if threshold reached

4. **Return**:
   - Return array of event IDs

#### Example

```typescript
const events = [
  {
    event_type: "data_access",
    user_id: "user_123",
    resource: "transaction:txn_1",
    action: "view",
    tenant_id: "tenant_abc"
  },
  {
    event_type: "data_access",
    user_id: "user_123",
    resource: "transaction:txn_2",
    action: "view",
    tenant_id: "tenant_abc"
  }
];

const event_ids = await auditLogger.logBatch(events);
console.log(`Logged ${event_ids.length} events`);
```

#### Performance

- **Target Latency**: <50ms for 50 events

---

### 3. queryLogs()

Query audit logs with filters.

#### Signature

```typescript
queryLogs(query: AuditLogQuery): Promise<AuditEvent[]>
```

#### Behavior

1. **Build SQL Query**:
   - Apply filters (user_id, resource, event_type, time_range)

2. **Execute Query**:
   - Query `audit_events` table

3. **Return**:
   - Return array of events

#### Example

```typescript
// Query all access events for user
const events = await auditLogger.queryLogs({
  user_id: "user_123",
  event_type: "data_access",
  tenant_id: "tenant_abc",
  time_range: "30d",
  limit: 100
});

events.forEach(event => {
  console.log(`${event.timestamp}: ${event.action} on ${event.resource}`);
});
```

#### Performance

- **Target Latency**: <100ms for 1000 results
- **Optimization**: Indexed queries

---

### 4. exportLogs()

Export audit logs to file (JSON/CSV/PDF).

#### Signature

```typescript
exportLogs(request: ExportRequest): Promise<ExportResult>
```

#### Behavior

1. **Query Events**:
   - Execute query

2. **Format**:
   - Convert to requested format (JSON/CSV/PDF)

3. **Store**:
   - Store file in S3/blob storage

4. **Generate URL**:
   - Generate signed download URL (expires in 24h)

5. **Return**:
   - Return export result

#### Example

```typescript
// Export all events for user as CSV
const exportResult = await auditLogger.exportLogs({
  query: {
    user_id: "user_123",
    tenant_id: "tenant_abc",
    time_range: "1y"
  },
  format: "csv"
});

console.log(`Export ready: ${exportResult.download_url}`);
console.log(`File size: ${exportResult.file_size_bytes} bytes`);
console.log(`Events: ${exportResult.events_count}`);
console.log(`Expires: ${exportResult.expires_at}`);
```

#### Performance

- **Target Latency**: <5s for 10K events

---

### 5. verifyIntegrity()

Verify integrity of audit log chain (tamper detection).

#### Signature

```typescript
verifyIntegrity(request: IntegrityCheckRequest): Promise<IntegrityCheckResult>
```

#### Behavior

1. **Query Events**:
   - Query all events in time range (ordered by created_at)

2. **Verify Chain**:
   - For each event, verify: `current_hash = SHA256(prev_hash + event_data)`

3. **Detect Tampering**:
   - If hash mismatch, tampering detected

4. **Return**:
   - Return verification result

#### Algorithm

```typescript
async verifyIntegrity(
  request: IntegrityCheckRequest
): Promise<IntegrityCheckResult> {
  // 1. Query events
  const events = await this.queryLogs({
    tenant_id: request.tenant_id,
    time_range: request.time_range,
    limit: 1000000 // All events
  });

  let prevHash: string | null = null;

  // 2. Verify chain
  for (let i = 0; i < events.length; i++) {
    const event = events[i];

    // Calculate expected hash
    const eventData = JSON.stringify({
      event_type: event.event_type,
      user_id: event.user_id,
      resource: event.resource,
      action: event.action,
      tenant_id: event.tenant_id,
      changes: event.changes,
      metadata: event.metadata,
      timestamp: event.timestamp
    });

    const expectedHash = createHash('sha256')
      .update(prevHash || '')
      .update(eventData)
      .digest('hex');

    // Compare
    if (expectedHash !== event.current_hash) {
      return {
        valid: false,
        events_checked: i + 1,
        first_invalid_event: event.event_id,
        error_message: `Hash mismatch at event ${i + 1}`,
        checked_at: new Date().toISOString()
      };
    }

    prevHash = event.current_hash;
  }

  // 3. All events verified
  return {
    valid: true,
    events_checked: events.length,
    checked_at: new Date().toISOString()
  };
}
```

#### Example

```typescript
// Verify integrity of all logs
const result = await auditLogger.verifyIntegrity({
  tenant_id: "tenant_abc",
  time_range: "all"
});

if (result.valid) {
  console.log(`Integrity check PASSED (${result.events_checked} events)`);
} else {
  console.error(`Integrity check FAILED at event ${result.first_invalid_event}`);
  console.error(result.error_message);
}
```

#### Performance

- **Target Latency**: <10s for 100K events

---

## Event Types

### 1. data_access

**Definition**: User viewed data (read-only operation).

**Required Fields**:
- `user_id`: User who accessed data
- `resource`: Resource accessed
- `action`: "view", "read", "query"

**Example**:
```typescript
await auditLogger.logEvent({
  event_type: "data_access",
  user_id: "user_123",
  resource: "patient:pt_456",
  action: "view",
  tenant_id: "hospital_abc",
  metadata: {
    access_reason: "Treatment",
    ip_address: "10.0.1.50"
  }
});
```

---

### 2. data_modification

**Definition**: User created, updated, or deleted data.

**Required Fields**:
- `user_id`: User who modified data
- `resource`: Resource modified
- `action`: "create", "update", "delete"
- `changes`: Old and new values

**Example**:
```typescript
await auditLogger.logEvent({
  event_type: "data_modification",
  user_id: "user_123",
  resource: "transaction:txn_456",
  action: "update",
  tenant_id: "tenant_abc",
  changes: {
    category: { old: "uncategorized", new: "groceries" }
  }
});
```

---

### 3. user_action

**Definition**: User performed action (not data access/modification).

**Required Fields**:
- `user_id`: User who performed action
- `resource`: Resource affected
- `action`: Action name (e.g., "export", "invite", "share")

**Example**:
```typescript
await auditLogger.logEvent({
  event_type: "user_action",
  user_id: "user_123",
  resource: "report:rpt_789",
  action: "export",
  tenant_id: "tenant_abc",
  metadata: {
    export_format: "csv"
  }
});
```

---

### 4. security_event

**Definition**: Security-relevant event.

**Required Fields**:
- `user_id`: User involved (or "system" for automated events)
- `resource`: Resource affected
- `action`: Security event type

**Example**:
```typescript
await auditLogger.logEvent({
  event_type: "security_event",
  user_id: "user_123",
  resource: "login",
  action: "failed_login",
  tenant_id: "tenant_abc",
  metadata: {
    ip_address: "203.0.113.50",
    reason: "Invalid password",
    attempt_count: 3
  }
});
```

---

### 5. system_event

**Definition**: System-generated event (automated).

**Required Fields**:
- `user_id`: "system"
- `resource`: Resource affected
- `action`: System action

**Example**:
```typescript
await auditLogger.logEvent({
  event_type: "system_event",
  user_id: "system",
  resource: "upload:upload_123",
  action: "auto_sync_completed",
  tenant_id: "tenant_abc",
  metadata: {
    observations_synced: 150,
    duration_ms: 3500
  }
});
```

---

## Append-Only Architecture

### Database Enforcement

**Prevent UPDATE/DELETE**:
```sql
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Audit logs are immutable and cannot be modified or deleted';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER prevent_update_audit_events
BEFORE UPDATE ON audit_events
FOR EACH ROW
EXECUTE FUNCTION prevent_audit_modification();

CREATE TRIGGER prevent_delete_audit_events
BEFORE DELETE ON audit_events
FOR EACH ROW
EXECUTE FUNCTION prevent_audit_modification();
```

### Retention Policy

**Archive Old Logs** (not delete):
```sql
-- Move logs older than 7 years to archive table
INSERT INTO audit_events_archive
SELECT * FROM audit_events
WHERE created_at < NOW() - INTERVAL '7 years';

-- Optional: Drop old partitions (if using partitioning)
DROP TABLE audit_events_2018_01;
```

---

## Integrity Verification

### SHA-256 Hash Chain

**Chain Construction**:
```typescript
function calculateHash(prevHash: string | null, eventData: string): string {
  return createHash('sha256')
    .update(prevHash || '')
    .update(eventData)
    .digest('hex');
}

// Event 1
const event1Hash = calculateHash(null, JSON.stringify(event1));
// "abc123..."

// Event 2
const event2Hash = calculateHash(event1Hash, JSON.stringify(event2));
// "def456..."

// Event 3
const event3Hash = calculateHash(event2Hash, JSON.stringify(event3));
// "ghi789..."
```

### Tampering Detection

**Scenario**: Attacker modifies Event 2.

**Detection**:
1. Verify Event 2: Calculate hash from Event 1's hash + Event 2 data
2. Compare calculated hash with stored hash
3. Mismatch detected → Event 2 was tampered with

**Chain Break**:
```
Event 1: hash = abc123 (valid)
Event 2: hash = TAMPERED (invalid - doesn't match calculated)
Event 3: hash = ghi789 (invalid - depends on Event 2's correct hash)
```

All events after tampered event are invalidated.

---

## Edge Cases

### Edge Case 1: Buffer Overflow

**Scenario**: Events logged faster than they can be flushed.

**Resolution**:
- Increase batch size
- Decrease flush interval
- Apply backpressure (block on flush)

---

### Edge Case 2: Database Unavailable

**Scenario**: PostgreSQL is down during flush.

**Resolution**:
- Retry with exponential backoff
- Persist buffer to disk
- Alert operations team

---

### Edge Case 3: Concurrent Flushes

**Scenario**: Two flush operations triggered simultaneously.

**Resolution**:
- Use mutex lock to ensure single flush at a time

---

### Edge Case 4: Clock Skew

**Scenario**: Server clocks out of sync, incorrect timestamps.

**Resolution**:
- Use database NOW() for timestamp (not application time)

---

### Edge Case 5: Extremely Large Events

**Scenario**: Event metadata >1MB.

**Resolution**:
- Truncate large metadata
- Store large data separately (S3)

---

### Edge Case 6: Export Timeout

**Scenario**: Exporting 1M events times out.

**Resolution**:
- Stream export (don't load all events in memory)
- Paginate export (multiple files)

---

### Edge Case 7: Retention Policy Conflict

**Scenario**: GDPR requires deletion, but audit logs are immutable.

**Resolution**:
- Anonymize logs instead of deleting
- Replace user_id with "[ANONYMIZED]"

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `logEvent()` | 1 event | < 8ms | Async + batching |
| `logBatch()` | 50 events | < 50ms | Batch insert |
| `queryLogs()` | 1000 results | < 100ms | Indexed query |
| `exportLogs()` | 10K events | < 5s | Generate file |
| `verifyIntegrity()` | 100K events | < 10s | Hash verification |

### Throughput

- **Events/sec**: 5K events/sec (with batching)
- **Batch size**: 50 events per batch
- **Flush interval**: 100ms

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';
import { createHash, randomUUID } from 'crypto';

export class PostgresAuditLogger implements AuditLogger {
  private pool: Pool;
  private buffer: Array<{ event_id: string; event: AuditEvent }> = [];
  private batchSize = 50;
  private flushInterval = 100; // ms

  constructor(pool: Pool) {
    this.pool = pool;
    this.startFlushTimer();
  }

  async logEvent(event: AuditEvent): Promise<string> {
    // Implementation shown in Core Functionality section
    // ...
  }

  private async flushBuffer(): Promise<void> {
    // Implementation shown in Core Functionality section
    // ...
  }

  private startFlushTimer(): void {
    setInterval(() => {
      this.flushBuffer().catch(err => {
        console.error('Failed to flush buffer:', err);
      });
    }, this.flushInterval);
  }
}
```

---

## Security Considerations

### 1. Append-Only Enforcement

**Database Triggers**:
- Prevent UPDATE/DELETE at database level
- Cannot be bypassed from application

### 2. Integrity Verification

**Regular Checks**:
- Run integrity verification daily
- Alert if tampering detected

### 3. Access Control

**Restrict Query Access**:
- Only auditors can query audit logs
- Enforce with AccessControl primitive

---

## Integration Patterns

### Pattern 1: Automatic Logging Middleware

```typescript
// Express middleware
function auditMiddleware(auditLogger: AuditLogger) {
  return async (req: Request, res: Response, next: NextFunction) => {
    // Log request
    await auditLogger.logEvent({
      event_type: "data_access",
      user_id: req.user.id,
      resource: `${req.method} ${req.path}`,
      action: "api_request",
      tenant_id: req.user.tenant_id,
      metadata: {
        ip_address: req.ip,
        user_agent: req.headers['user-agent']
      }
    });

    next();
  };
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section with 7 domains)

---

## Testing Strategy

### Unit Tests

```typescript
describe('AuditLogger', () => {
  let auditLogger: AuditLogger;

  beforeEach(() => {
    auditLogger = new PostgresAuditLogger(pool);
  });

  describe('logEvent()', () => {
    it('should log event', async () => {
      const event_id = await auditLogger.logEvent({
        event_type: "data_access",
        user_id: "user_123",
        resource: "transaction:txn_456",
        action: "view",
        tenant_id: "tenant_abc"
      });

      expect(event_id).toBeDefined();
    });
  });

  describe('verifyIntegrity()', () => {
    it('should verify integrity of chain', async () => {
      // Log events
      await auditLogger.logEvent({ ... });
      await auditLogger.logEvent({ ... });

      // Verify
      const result = await auditLogger.verifyIntegrity({
        tenant_id: "tenant_abc"
      });

      expect(result.valid).toBe(true);
    });
  });
});
```

---

## Configuration

### Tunable Parameters

```typescript
interface AuditLoggerConfig {
  // Batching
  batch_size: number;               // Default: 50
  flush_interval_ms: number;        // Default: 100

  // Retention
  retention_days: number;           // Default: 2555 (7 years)

  // Exports
  export_expiration_hours: number;  // Default: 24
}
```

---

## Observability

### Metrics

```typescript
interface AuditLoggerMetrics {
  events_logged_total: Counter;
  log_latency_ms: Histogram;
  buffer_size: Gauge;
  flush_errors_total: Counter;
}
```

---

## Future Enhancements

### Phase 1: Real-Time Alerts (3 months)
- Alert on suspicious patterns
- Anomaly detection

### Phase 2: Blockchain Integration (12 months)
- Store hash chain anchors in blockchain
- Immutable proof of log integrity

### Phase 3: ML-Based Fraud Detection (18 months)
- Detect fraudulent activity from audit logs
- Predictive alerting

---

**End of AuditLogger OL Primitive Specification**
