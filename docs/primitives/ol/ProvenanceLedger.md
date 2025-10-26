# ProvenanceLedger OL Primitive

**Domain:** Provenance Ledger (Vertical 5.1)
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
8. [Bitemporal Architecture](#bitemporal-architecture)
9. [Integrity & Verification](#integrity--verification)
10. [Query Patterns](#query-patterns)
11. [Edge Cases](#edge-cases)
12. [Performance Characteristics](#performance-characteristics)
13. [Implementation Notes](#implementation-notes)
14. [Security Considerations](#security-considerations)
15. [Integration Patterns](#integration-patterns)
16. [Multi-Domain Examples](#multi-domain-examples)
17. [Testing Strategy](#testing-strategy)
18. [Migration Guide](#migration-guide)
19. [Related Primitives](#related-primitives)
20. [References](#references)

---

## Overview

The **ProvenanceLedger** is an append-only bitemporal event storage primitive that maintains a complete, immutable history of all changes to entities in the system. It serves as the authoritative source of truth for understanding "what happened when" and "what we knew when" across all domain entities.

### Key Capabilities

- **Append-Only Storage**: Events can only be added, never modified or deleted
- **Bitemporal Tracking**: Captures both transaction time (when recorded) and valid time (when effective)
- **Cryptographic Integrity**: SHA-256 hashing ensures tampering detection
- **Complete Audit Trail**: Every field change, correction, and retroactive update is logged
- **High-Performance Queries**: Optimized indexes for common temporal queries
- **Export Capabilities**: Generate compliance reports in JSON or CSV formats

### Design Philosophy

The ProvenanceLedger follows four core principles:

1. **Immutability**: Once written, records cannot be changed (enforced at database level)
2. **Completeness**: Every state change is captured with full context
3. **Verifiability**: Cryptographic hashes enable integrity verification
4. **Queryability**: Efficient temporal queries for audit and analysis

---

## Purpose & Scope

### Problem Statement

In data-intensive systems, understanding the provenance of information is critical:

- **Compliance**: Regulatory requirements demand complete audit trails (SOX, HIPAA, GDPR)
- **Debugging**: When data is wrong, we need to know when and why it changed
- **Analytics**: Historical analysis requires accurate "point-in-time" snapshots
- **Trust**: Users need confidence that the system accurately reflects reality

Traditional approaches have significant limitations:

**Approach 1: Update-in-Place**
- ❌ Loses historical values
- ❌ No audit trail
- ❌ Can't answer "what did we know on date X?"

**Approach 2: Versioned Entities**
- ❌ Complex version management
- ❌ Difficult to query across entity types
- ❌ No distinction between transaction time and valid time

**Approach 3: Append-Only Event Log (Provenance Ledger)**
- ✅ Complete history preserved
- ✅ Bitemporal queries supported
- ✅ Tamper-evident with cryptographic hashing
- ✅ Scales to billions of events

### Solution

The ProvenanceLedger implements an **append-only bitemporal event log** with:

1. **Immutable Storage**: PostgreSQL table with REVOKE UPDATE/DELETE privileges
2. **Bitemporal Timestamps**: Both `transaction_time` (when recorded) and `valid_time` (when effective)
3. **Event Sourcing**: Every change captured as an event, not a state mutation
4. **Cryptographic Integrity**: SHA-256 hash chain for tamper detection
5. **Efficient Indexing**: B-tree and GiST indexes for temporal queries

---

## Multi-Domain Applicability

The ProvenanceLedger is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Track every transaction, correction, and reconciliation with complete audit trail.

**Events Tracked**:
- Transaction extraction from bank statement
- Merchant name normalization
- Category classification
- Amount corrections
- Transfer linking
- Reconciliation status changes

**Compliance**: SOX (Sarbanes-Oxley) requires complete audit trails for financial data.

**Example**:
```typescript
// Original transaction extracted
await ledger.append({
  entity_id: "txn_001",
  entity_type: "transaction",
  event_type: "extracted",
  field_name: "merchant",
  old_value: null,
  new_value: "AMZN MKTP US*1234",
  valid_time: "2025-01-15T10:00:00Z",
  user_id: "system",
  reason: "Extracted from Chase bank statement"
});

// User corrects merchant name
await ledger.append({
  entity_id: "txn_001",
  entity_type: "transaction",
  event_type: "corrected",
  field_name: "merchant",
  old_value: "AMZN MKTP US*1234",
  new_value: "Amazon.com",
  valid_time: "2025-01-15T10:00:00Z", // Still effective on original date
  user_id: "user_jane_doe",
  reason: "Normalized merchant name for reporting"
});
```

### 2. Healthcare

**Use Case**: HIPAA-compliant audit trail for patient records, diagnoses, and treatments.

**Events Tracked**:
- Diagnosis code assignments
- Treatment plan updates
- Lab result corrections
- Provider assignments
- Insurance claim status changes
- Patient consent updates

**Compliance**: HIPAA requires tracking who accessed/modified patient data and when.

**Example**:
```typescript
// Initial diagnosis from ER visit
await ledger.append({
  entity_id: "enc_001",
  entity_type: "encounter",
  event_type: "diagnosed",
  field_name: "diagnosis_code",
  old_value: null,
  new_value: "R07.9", // Chest pain, unspecified
  valid_time: "2025-01-15T14:30:00Z",
  user_id: "dr_smith",
  reason: "Initial ER diagnosis"
});

// Updated diagnosis after further testing
await ledger.append({
  entity_id: "enc_001",
  entity_type: "encounter",
  event_type: "corrected",
  field_name: "diagnosis_code",
  old_value: "R07.9",
  new_value: "I20.9", // Angina pectoris
  valid_time: "2025-01-15T14:30:00Z", // Retroactively effective at encounter time
  user_id: "dr_jones",
  reason: "Updated after EKG results"
});
```

### 3. Legal

**Use Case**: Document chain of custody, case status changes, and evidence handling.

**Events Tracked**:
- Document filed timestamps
- Case status transitions
- Attorney assignments
- Evidence logged
- Settlement negotiations
- Court ruling records

**Compliance**: Legal discovery requires proving when documents were created/modified.

**Example**:
```typescript
// Case filed
await ledger.append({
  entity_id: "case_001",
  entity_type: "legal_case",
  event_type: "filed",
  field_name: "status",
  old_value: null,
  new_value: "filed",
  valid_time: "2024-12-01T09:00:00Z",
  user_id: "attorney_sarah_lee",
  reason: "Case filed in district court"
});

// Discovery phase begins
await ledger.append({
  entity_id: "case_001",
  entity_type: "legal_case",
  event_type: "status_change",
  field_name: "status",
  old_value: "filed",
  new_value: "discovery",
  valid_time: "2024-12-15T10:00:00Z",
  user_id: "system",
  reason: "Discovery phase initiated by court order"
});
```

### 4. Research (RSRCH - Utilitario)

**Use Case**: Track fact extraction, enrichment, and corrections from multiple sources (web, interviews, tweets, podcasts).

**Context**: RSRCH collects facts about founders/companies/entities. Same fact evolves as new sources are discovered. Provenance ledger tracks complete fact lifecycle with multi-source attribution.

**Events Tracked**:
- Fact extracted from source (TechCrunch, interview, tweet)
- Fact enriched (vague → specific)
- Entity name normalized (@sama → Sam Altman)
- Source added (multi-source confirmation)
- Contradiction detected (conflicting values from different sources)
- Fact merged (duplicate facts unified)

**Compliance**: Provenance transparency for truth construction - track which sources contributed to each fact.

**Example (Investment Fact Lifecycle)**:
```typescript
// Step 1: Initial fact extracted from TechCrunch article (Jan 15)
await ledger.append({
  entity_id: "fact_sama_helion_001",
  entity_type: "fact",
  event_type: "extracted",
  field_name: "claim",
  old_value: null,
  new_value: "Sam Altman invested in Helion Energy",
  valid_time: "2025-01-15T00:00:00Z", // Article publication date
  user_id: "web_scraper_techcrunch",
  reason: "Extracted from TechCrunch article",
  metadata: {
    source_url: "https://techcrunch.com/2025/01/15/sama-helion",
    source_type: "web_article",
    source_credibility: 0.9
  }
});

// Step 2: Fact enriched with specific amount from podcast (Feb 20, retroactive to Jan 15)
await ledger.append({
  entity_id: "fact_sama_helion_001",
  entity_type: "fact",
  event_type: "enriched",
  field_name: "investment_amount",
  old_value: null,
  new_value: 375000000, // $375 million
  valid_time: "2025-01-15T00:00:00Z", // Investment happened on Jan 15
  user_id: "podcast_parser_lex_fridman",
  reason: "Amount revealed in Lex Fridman podcast interview",
  metadata: {
    source_url: "https://youtube.com/watch?v=xyz",
    source_type: "podcast_transcript",
    source_credibility: 0.95
  }
});

// Step 3: Entity name normalized (retroactive to Jan 15)
await ledger.append({
  entity_id: "fact_sama_helion_001",
  entity_type: "fact",
  event_type: "normalized",
  field_name: "subject_entity",
  old_value: "@sama", // Twitter handle from tweet
  new_value: "Sam Altman", // Canonical name
  valid_time: "2025-01-15T00:00:00Z",
  user_id: "entity_normalizer_v2",
  reason: "Normalized Twitter handle to canonical name"
});

// Step 4: Multi-source confirmation (Bloomberg article confirms TechCrunch)
await ledger.append({
  entity_id: "fact_sama_helion_001",
  entity_type: "fact",
  event_type: "source_added",
  field_name: "sources",
  old_value: ["techcrunch"],
  new_value: ["techcrunch", "bloomberg", "lex_fridman_podcast"],
  valid_time: "2025-01-15T00:00:00Z",
  user_id: "fact_consolidator",
  reason: "Multi-source confirmation increases confidence",
  metadata: {
    combined_confidence: 0.98 // (0.9 + 0.9 + 0.95) / 3
  }
});
```

**RSRCH-Specific Patterns**:
- **Retroactive Enrichment**: Initial fact from TechCrunch (vague) → Enriched by podcast (specific) → Effective date retroactive to original article date
- **Multi-Source Provenance**: Track WHICH sources contributed WHICH fields to the complete fact
- **Entity Normalization Trail**: @sama → sama → Sam Altman (complete provenance of normalization steps)

### 5. E-commerce

**Use Case**: Track product catalog changes, price updates, and inventory corrections.

**Events Tracked**:
- Product created
- Price changes
- Inventory adjustments
- Category reassignments
- Description updates
- SKU corrections

**Compliance**: Consumer protection laws require tracking price history.

**Example**:
```typescript
// Product created
await ledger.append({
  entity_id: "prod_001",
  entity_type: "product",
  event_type: "created",
  field_name: "price",
  old_value: null,
  new_value: 29.99,
  valid_time: "2025-01-01T00:00:00Z",
  user_id: "catalog_manager_tom",
  reason: "New product added to catalog"
});

// Sale price applied
await ledger.append({
  entity_id: "prod_001",
  entity_type: "product",
  event_type: "price_change",
  field_name: "price",
  old_value: 29.99,
  new_value: 24.99,
  valid_time: "2025-01-15T00:00:00Z", // Sale starts Jan 15
  user_id: "pricing_automation",
  reason: "Winter sale - 20% off"
});
```

### 6. SaaS

**Use Case**: Track subscription changes, feature flag toggles, and billing adjustments.

**Events Tracked**:
- Subscription plan changes
- Feature flag toggles
- Usage limit adjustments
- Billing corrections
- Trial conversions
- Cancellations/reactivations

**Compliance**: Revenue recognition (ASC 606) requires tracking subscription changes.

**Example**:
```typescript
// Subscription created
await ledger.append({
  entity_id: "sub_001",
  entity_type: "subscription",
  event_type: "created",
  field_name: "plan",
  old_value: null,
  new_value: "starter_monthly",
  valid_time: "2024-12-01T00:00:00Z",
  user_id: "cust_abc123",
  reason: "Customer signed up for Starter plan"
});

// Upgraded to Pro plan
await ledger.append({
  entity_id: "sub_001",
  entity_type: "subscription",
  event_type: "upgraded",
  field_name: "plan",
  old_value: "starter_monthly",
  new_value: "pro_monthly",
  valid_time: "2025-01-15T00:00:00Z",
  user_id: "cust_abc123",
  reason: "Customer upgraded to Pro plan"
});
```

### 7. Insurance

**Use Case**: Track policy changes, claim status, and premium adjustments.

**Events Tracked**:
- Policy issued
- Coverage changes
- Premium adjustments
- Claim filed
- Claim status updates
- Payout recorded

**Compliance**: Insurance regulations require complete audit trail of policy changes.

**Example**:
```typescript
// Policy issued
await ledger.append({
  entity_id: "pol_001",
  entity_type: "insurance_policy",
  event_type: "issued",
  field_name: "premium",
  old_value: null,
  new_value: 1200.00,
  valid_time: "2025-01-01T00:00:00Z",
  user_id: "underwriter_mike",
  reason: "Annual auto insurance policy issued"
});

// Premium adjusted after claim
await ledger.append({
  entity_id: "pol_001",
  entity_type: "insurance_policy",
  event_type: "premium_adjustment",
  field_name: "premium",
  old_value: 1200.00,
  new_value: 1440.00, // 20% increase
  valid_time: "2025-06-01T00:00:00Z", // Effective at renewal
  user_id: "system",
  reason: "Premium increase due to at-fault accident claim"
});
```

### Cross-Domain Benefits

**Consistent Audit Trail**: All domains benefit from:
- Complete history of changes
- Who made each change and why
- When changes were effective vs. when recorded
- Ability to reconstruct state at any point in time

**Regulatory Compliance**: Supports:
- SOX (Finance)
- HIPAA (Healthcare)
- GDPR (Privacy/Right to access)
- ASC 606 (Revenue Recognition)
- Legal discovery requirements
- Academic integrity standards

---

## Core Concepts

### Bitemporal Events

Every event in the ProvenanceLedger has **two timestamps**:

1. **Transaction Time (`transaction_time`)**: When the event was recorded in the ledger
2. **Valid Time (`valid_time`)**: When the change was effective in reality

**Example**:
```typescript
{
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,

  // Change was effective on Jan 15 (when transaction occurred)
  valid_time: "2025-01-15T10:00:00Z",

  // But we didn't discover error until Jan 20 (when we recorded correction)
  transaction_time: "2025-01-20T14:30:00Z",

  reason: "Corrected amount from receipt - original extraction was wrong"
}
```

### Event Types

The ledger tracks various event types:

- `created`: Entity first created
- `updated`: Field value changed
- `corrected`: Retroactive correction (valid_time in past)
- `deleted`: Entity soft-deleted
- `restored`: Entity restored from deletion
- `reconciled`: Field reconciled across sources
- `normalized`: Field normalized (e.g., merchant name)
- `classified`: Field classified (e.g., category assigned)
- `linked`: Relationship established (e.g., transfer linked)
- `unlinked`: Relationship removed

### Immutability

**Database-Level Enforcement**:

```sql
-- Revoke UPDATE and DELETE privileges
REVOKE UPDATE, DELETE ON provenance_ledger FROM application_user;

-- Only INSERT allowed
GRANT INSERT, SELECT ON provenance_ledger TO application_user;
```

**Application-Level**: The ProvenanceLedger interface provides only `append()` - no update or delete methods.

### Cryptographic Integrity

Each record includes a SHA-256 hash of:
- Previous record's hash (forming a chain)
- Current record's data (entity_id, field_name, old_value, new_value, timestamps)

**Hash Chain**:
```
Record 1: hash = SHA256(data1)
Record 2: hash = SHA256(hash1 + data2)
Record 3: hash = SHA256(hash2 + data3)
```

This creates a **tamper-evident** ledger - any modification breaks the hash chain.

---

## Interface Definition

### TypeScript Interface

```typescript
interface ProvenanceLedger {
  /**
   * Append a new event to the provenance ledger
   *
   * @param record - Bitemporal event record
   * @returns Promise<void> when record is durably stored
   * @throws ValidationError if record is invalid
   * @throws DatabaseError if append fails
   */
  append(record: BitemporalRecord): Promise<void>;

  /**
   * Get complete history for an entity (all fields)
   *
   * @param entityId - Entity identifier
   * @param fieldName - Optional: filter to specific field
   * @returns Array of events in chronological order (by transaction_time)
   */
  getHistory(
    entityId: string,
    fieldName?: string
  ): Promise<BitemporalRecord[]>;

  /**
   * Verify cryptographic integrity of a record
   *
   * @param recordId - Record to verify
   * @returns true if hash chain is valid, false if tampered
   */
  verifyIntegrity(recordId: string): Promise<boolean>;

  /**
   * Export provenance data matching filters
   *
   * @param filters - Query filters
   * @param format - Output format (json or csv)
   * @returns Serialized data as string
   */
  export(
    filters: ProvenanceQueryFilters,
    format: 'json' | 'csv'
  ): Promise<string>;

  /**
   * Get events within transaction time range
   *
   * @param startTime - Start of time range (transaction_time)
   * @param endTime - End of time range (transaction_time)
   * @param filters - Additional filters
   * @returns Array of events
   */
  getEventsByTransactionTime(
    startTime: string,
    endTime: string,
    filters?: ProvenanceQueryFilters
  ): Promise<BitemporalRecord[]>;

  /**
   * Get events within valid time range
   *
   * @param startTime - Start of time range (valid_time)
   * @param endTime - End of time range (valid_time)
   * @param filters - Additional filters
   * @returns Array of events
   */
  getEventsByValidTime(
    startTime: string,
    endTime: string,
    filters?: ProvenanceQueryFilters
  ): Promise<BitemporalRecord[]>;

  /**
   * Count events matching filters
   *
   * @param filters - Query filters
   * @returns Number of matching events
   */
  count(filters: ProvenanceQueryFilters): Promise<number>;

  /**
   * Get the last N events (most recent by transaction_time)
   *
   * @param limit - Maximum number of events to return
   * @param filters - Optional filters
   * @returns Array of most recent events
   */
  getRecentEvents(
    limit: number,
    filters?: ProvenanceQueryFilters
  ): Promise<BitemporalRecord[]>;
}
```

---

## Data Model

### BitemporalRecord Type

```typescript
interface BitemporalRecord {
  // Record Identity
  record_id: string;              // Unique ID (e.g., "prov_1234567890")
  sequence_number: number;        // Auto-incrementing sequence for ordering

  // Entity Context
  entity_id: string;              // Entity being tracked (e.g., "txn_001")
  entity_type: string;            // Type of entity (e.g., "transaction", "quote")

  // Event Details
  event_type: string;             // Type of change (e.g., "created", "updated", "corrected")
  field_name: string;             // Field that changed (e.g., "merchant", "category")

  // Values
  old_value: any;                 // Value before change (JSONB, null if created)
  new_value: any;                 // Value after change (JSONB)

  // Bitemporal Timestamps
  transaction_time: string;       // ISO timestamp: when recorded in ledger
  valid_time: string;             // ISO timestamp: when effective in reality

  // Metadata
  user_id: string;                // Who made the change
  reason?: string;                // Optional: why the change was made
  source_system?: string;         // Optional: which system originated the change
  correlation_id?: string;        // Optional: group related changes

  // Integrity
  hash: string;                   // SHA-256 hash of this record + previous hash
  previous_hash: string;          // Hash of previous record (forms chain)

  // System
  created_at: string;             // ISO timestamp: when row was inserted (same as transaction_time)
}
```

### ProvenanceQueryFilters Type

```typescript
interface ProvenanceQueryFilters {
  // Entity Filters
  entity_ids?: string[];          // Filter by entity IDs
  entity_types?: string[];        // Filter by entity types

  // Event Filters
  event_types?: string[];         // Filter by event types
  field_names?: string[];         // Filter by field names

  // User Filters
  user_ids?: string[];            // Filter by user IDs

  // Time Range Filters
  transaction_time_start?: string; // Transaction time >= this
  transaction_time_end?: string;   // Transaction time <= this
  valid_time_start?: string;       // Valid time >= this
  valid_time_end?: string;         // Valid time <= this

  // Pagination
  limit?: number;                 // Max results (default: 1000)
  offset?: number;                // Skip N results (default: 0)

  // Sorting
  sort_by?: 'transaction_time' | 'valid_time' | 'sequence_number';
  sort_order?: 'asc' | 'desc';    // Default: 'asc'
}
```

### Database Schema (PostgreSQL)

```sql
CREATE TABLE provenance_ledger (
  -- Primary Key
  record_id VARCHAR(64) PRIMARY KEY,
  sequence_number BIGSERIAL UNIQUE NOT NULL, -- Auto-incrementing sequence

  -- Entity Context
  entity_id VARCHAR(128) NOT NULL,
  entity_type VARCHAR(64) NOT NULL,

  -- Event Details
  event_type VARCHAR(64) NOT NULL,
  field_name VARCHAR(128) NOT NULL,

  -- Values (JSONB for flexibility)
  old_value JSONB,
  new_value JSONB NOT NULL,

  -- Bitemporal Timestamps
  transaction_time TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  valid_time TIMESTAMP WITH TIME ZONE NOT NULL,

  -- Metadata
  user_id VARCHAR(128) NOT NULL,
  reason TEXT,
  source_system VARCHAR(128),
  correlation_id VARCHAR(128),

  -- Cryptographic Integrity
  hash VARCHAR(64) NOT NULL,
  previous_hash VARCHAR(64),

  -- System Timestamp
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes for Performance
CREATE INDEX idx_provenance_entity_id ON provenance_ledger(entity_id);
CREATE INDEX idx_provenance_entity_type ON provenance_ledger(entity_type);
CREATE INDEX idx_provenance_event_type ON provenance_ledger(event_type);
CREATE INDEX idx_provenance_field_name ON provenance_ledger(field_name);
CREATE INDEX idx_provenance_user_id ON provenance_ledger(user_id);
CREATE INDEX idx_provenance_transaction_time ON provenance_ledger(transaction_time DESC);
CREATE INDEX idx_provenance_valid_time ON provenance_ledger(valid_time DESC);
CREATE INDEX idx_provenance_sequence ON provenance_ledger(sequence_number DESC);

-- GiST Index for Bitemporal Range Queries
CREATE INDEX idx_provenance_bitemporal ON provenance_ledger
  USING GIST (
    tstzrange(transaction_time, transaction_time + INTERVAL '1 microsecond'),
    tstzrange(valid_time, valid_time + INTERVAL '1 microsecond')
  );

-- Revoke UPDATE and DELETE to enforce immutability
REVOKE UPDATE, DELETE ON provenance_ledger FROM application_user;
GRANT INSERT, SELECT ON provenance_ledger TO application_user;

-- Trigger to set created_at = transaction_time
CREATE OR REPLACE FUNCTION set_created_at_from_transaction_time()
RETURNS TRIGGER AS $$
BEGIN
  NEW.created_at = NEW.transaction_time;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_provenance_created_at
BEFORE INSERT ON provenance_ledger
FOR EACH ROW
EXECUTE FUNCTION set_created_at_from_transaction_time();
```

---

## Core Functionality

### 1. append()

Append a new event to the provenance ledger.

#### Signature

```typescript
append(record: BitemporalRecord): Promise<void>
```

#### Behavior

1. **Validation**:
   - All required fields present
   - `transaction_time` defaults to NOW() if not provided
   - `valid_time` must be valid ISO timestamp
   - `entity_id` and `field_name` are non-empty strings

2. **ID Generation**:
   - Generate unique `record_id` (e.g., `prov_` + timestamp + random)
   - `sequence_number` auto-generated by database

3. **Hash Calculation**:
   - Get `previous_hash` from most recent record in ledger
   - Calculate `hash = SHA256(previous_hash + record_data)`
   - Store both `hash` and `previous_hash`

4. **Insert**:
   - INSERT into `provenance_ledger` table
   - Transaction ensures atomicity with hash chain

5. **Return**:
   - Return void (success) or throw error

#### Example

```typescript
await ledger.append({
  entity_id: "txn_001",
  entity_type: "transaction",
  event_type: "corrected",
  field_name: "merchant",
  old_value: "AMZN MKTP US*1234",
  new_value: "Amazon.com",
  valid_time: "2025-01-15T10:00:00Z",
  transaction_time: "2025-01-20T14:30:00Z", // Optional, defaults to NOW()
  user_id: "user_jane_doe",
  reason: "Normalized merchant name for reporting"
});
```

#### Error Cases

```typescript
// Case 1: Missing required field
try {
  await ledger.append({
    entity_id: "txn_001",
    // missing entity_type
    event_type: "updated",
    field_name: "merchant",
    old_value: "AMZN",
    new_value: "Amazon.com",
    valid_time: "2025-01-15T10:00:00Z",
    user_id: "user_jane"
  });
} catch (error) {
  console.log(error.code); // "VALIDATION_ERROR"
  console.log(error.message); // "entity_type is required"
}

// Case 2: Invalid timestamp
try {
  await ledger.append({
    entity_id: "txn_001",
    entity_type: "transaction",
    event_type: "updated",
    field_name: "merchant",
    old_value: "AMZN",
    new_value: "Amazon.com",
    valid_time: "invalid-timestamp",
    user_id: "user_jane"
  });
} catch (error) {
  console.log(error.code); // "VALIDATION_ERROR"
  console.log(error.message); // "valid_time must be valid ISO timestamp"
}
```

---

### 2. getHistory()

Get complete history for an entity, optionally filtered by field.

#### Signature

```typescript
getHistory(entityId: string, fieldName?: string): Promise<BitemporalRecord[]>
```

#### Behavior

1. Query `provenance_ledger` WHERE `entity_id = $1`
2. Optionally filter by `field_name = $2` if provided
3. Sort by `transaction_time ASC` (chronological order)
4. Return array of events

#### Example

```typescript
// Get all events for entity
const history = await ledger.getHistory("txn_001");

console.log(`${history.length} events for txn_001`);

for (const event of history) {
  console.log(`${event.transaction_time}: ${event.event_type} ${event.field_name}`);
  console.log(`  ${event.old_value} -> ${event.new_value}`);
}

// Output:
// 2025-01-15T10:00:00Z: created merchant
//   null -> "AMZN MKTP US*1234"
// 2025-01-15T10:00:00Z: created category
//   null -> "Uncategorized"
// 2025-01-20T14:30:00Z: corrected merchant
//   "AMZN MKTP US*1234" -> "Amazon.com"
```

#### Get History for Single Field

```typescript
// Get history for specific field
const merchantHistory = await ledger.getHistory("txn_001", "merchant");

console.log(`${merchantHistory.length} changes to merchant field`);

for (const event of merchantHistory) {
  console.log(`${event.transaction_time}: ${event.old_value} -> ${event.new_value}`);
  console.log(`  Reason: ${event.reason}`);
}
```

---

### 3. verifyIntegrity()

Verify cryptographic integrity of a record by validating the hash chain.

#### Signature

```typescript
verifyIntegrity(recordId: string): Promise<boolean>
```

#### Behavior

1. Fetch record by `record_id`
2. Fetch previous record by `sequence_number - 1`
3. Verify `record.previous_hash === previous_record.hash`
4. Re-calculate hash from record data
5. Verify calculated hash matches stored hash
6. Return `true` if valid, `false` if tampered

#### Example

```typescript
const isValid = await ledger.verifyIntegrity("prov_1234567890");

if (isValid) {
  console.log("Record integrity verified ✓");
} else {
  console.error("Record integrity FAILED - possible tampering!");
}
```

#### Batch Verification

```typescript
async function verifyLedgerIntegrity(
  startSequence: number,
  endSequence: number
): Promise<boolean> {
  for (let seq = startSequence; seq <= endSequence; seq++) {
    const record = await getRecordBySequence(seq);
    const isValid = await ledger.verifyIntegrity(record.record_id);

    if (!isValid) {
      console.error(`Integrity check failed at sequence ${seq}`);
      return false;
    }
  }

  console.log(`Verified ${endSequence - startSequence + 1} records ✓`);
  return true;
}
```

---

### 4. export()

Export provenance data matching filters in JSON or CSV format.

#### Signature

```typescript
export(
  filters: ProvenanceQueryFilters,
  format: 'json' | 'csv'
): Promise<string>
```

#### Behavior

1. Query provenance_ledger with filters
2. Serialize results to requested format
3. Return string (JSON array or CSV text)

#### Example: JSON Export

```typescript
const jsonData = await ledger.export(
  {
    entity_ids: ["txn_001", "txn_002"],
    event_types: ["corrected"],
    transaction_time_start: "2025-01-01T00:00:00Z",
    transaction_time_end: "2025-01-31T23:59:59Z"
  },
  'json'
);

// Write to file
fs.writeFileSync('provenance-export.json', jsonData);
```

#### Example: CSV Export

```typescript
const csvData = await ledger.export(
  {
    entity_type: ["transaction"],
    field_names: ["merchant", "category"],
    limit: 10000
  },
  'csv'
);

// Headers: record_id,entity_id,entity_type,event_type,field_name,old_value,new_value,transaction_time,valid_time,user_id,reason
// Row: prov_001,txn_001,transaction,corrected,merchant,"AMZN MKTP US*1234","Amazon.com",2025-01-20T14:30:00Z,2025-01-15T10:00:00Z,user_jane_doe,"Normalized merchant name"
```

---

### 5. getEventsByTransactionTime()

Get events within a transaction time range (when recorded).

#### Signature

```typescript
getEventsByTransactionTime(
  startTime: string,
  endTime: string,
  filters?: ProvenanceQueryFilters
): Promise<BitemporalRecord[]>
```

#### Behavior

Query events where `transaction_time BETWEEN startTime AND endTime`.

#### Example

```typescript
// Get all changes recorded in January 2025
const events = await ledger.getEventsByTransactionTime(
  "2025-01-01T00:00:00Z",
  "2025-01-31T23:59:59Z"
);

console.log(`${events.length} changes recorded in January 2025`);

// Group by event type
const byType = events.reduce((acc, event) => {
  acc[event.event_type] = (acc[event.event_type] || 0) + 1;
  return acc;
}, {} as Record<string, number>);

console.log("Changes by type:", byType);
// { created: 150, updated: 75, corrected: 25 }
```

---

### 6. getEventsByValidTime()

Get events within a valid time range (when effective).

#### Signature

```typescript
getEventsByValidTime(
  startTime: string,
  endTime: string,
  filters?: ProvenanceQueryFilters
): Promise<BitemporalRecord[]>
```

#### Behavior

Query events where `valid_time BETWEEN startTime AND endTime`.

#### Example

```typescript
// Get all changes effective in January 2025
// (includes retroactive corrections recorded later)
const events = await ledger.getEventsByValidTime(
  "2025-01-01T00:00:00Z",
  "2025-01-31T23:59:59Z"
);

console.log(`${events.length} changes effective in January 2025`);

// Find retroactive corrections
const retroactive = events.filter(
  e => e.transaction_time > e.valid_time
);

console.log(`${retroactive.length} retroactive corrections`);
```

---

## Bitemporal Architecture

### Two Time Dimensions

**Transaction Time**: When we recorded the information
**Valid Time**: When the information was true in reality

### Query Types

#### 1. Current State Query

"What is the current state of entity X?"

```typescript
async function getCurrentState(entityId: string): Promise<Record<string, any>> {
  // Get all events for entity
  const history = await ledger.getHistory(entityId);

  // Build current state by applying events in order
  const state: Record<string, any> = {};

  for (const event of history) {
    state[event.field_name] = event.new_value;
  }

  return state;
}
```

#### 2. Transaction Time Query

"What did we know on date X?" (System's knowledge at that time)

```typescript
async function getKnownStateAt(
  entityId: string,
  asOfTransactionTime: string
): Promise<Record<string, any>> {
  // Get events recorded on or before the transaction time
  const history = await ledger.getHistory(entityId);
  const relevantEvents = history.filter(
    e => e.transaction_time <= asOfTransactionTime
  );

  // Build state as of that transaction time
  const state: Record<string, any> = {};
  for (const event of relevantEvents) {
    state[event.field_name] = event.new_value;
  }

  return state;
}

// Example: What did we know on Jan 18?
const stateOnJan18 = await getKnownStateAt(
  "txn_001",
  "2025-01-18T23:59:59Z"
);
// Returns: { merchant: "AMZN MKTP US*1234" }
// (Correction to "Amazon.com" wasn't recorded until Jan 20)
```

#### 3. Valid Time Query

"What was true on date X?" (Reality at that time)

```typescript
async function getTrueStateAt(
  entityId: string,
  asOfValidTime: string
): Promise<Record<string, any>> {
  // Get events effective on or before the valid time
  const history = await ledger.getHistory(entityId);
  const relevantEvents = history.filter(
    e => e.valid_time <= asOfValidTime
  );

  // Sort by transaction time to get latest knowledge
  relevantEvents.sort(
    (a, b) => a.transaction_time.localeCompare(b.transaction_time)
  );

  // Build state as of that valid time
  const state: Record<string, any> = {};
  for (const event of relevantEvents) {
    state[event.field_name] = event.new_value;
  }

  return state;
}

// Example: What was true on Jan 15?
const stateOnJan15 = await getTrueStateAt(
  "txn_001",
  "2025-01-15T23:59:59Z"
);
// Returns: { merchant: "Amazon.com" }
// (Retroactive correction applied, so we now know it was Amazon.com all along)
```

#### 4. Bitemporal Query

"What did we know on date X about what was true on date Y?"

```typescript
async function getBitemporalState(
  entityId: string,
  asOfTransactionTime: string,
  asOfValidTime: string
): Promise<Record<string, any>> {
  // Get events recorded before transaction time AND effective before valid time
  const history = await ledger.getHistory(entityId);
  const relevantEvents = history.filter(
    e => e.transaction_time <= asOfTransactionTime
      && e.valid_time <= asOfValidTime
  );

  const state: Record<string, any> = {};
  for (const event of relevantEvents) {
    state[event.field_name] = event.new_value;
  }

  return state;
}

// Example: What did we know on Jan 18 about what was true on Jan 15?
const state = await getBitemporalState(
  "txn_001",
  "2025-01-18T23:59:59Z", // Transaction time
  "2025-01-15T23:59:59Z"  // Valid time
);
// Returns: { merchant: "AMZN MKTP US*1234" }
// (We knew AMZN on Jan 18, didn't learn about correction until Jan 20)
```

---

## Integrity & Verification

### Hash Chain Implementation

```typescript
function calculateHash(
  record: BitemporalRecord,
  previousHash: string
): string {
  // Create deterministic string from record data
  const data = [
    record.entity_id,
    record.entity_type,
    record.event_type,
    record.field_name,
    JSON.stringify(record.old_value),
    JSON.stringify(record.new_value),
    record.transaction_time,
    record.valid_time,
    record.user_id,
    record.reason || '',
    previousHash
  ].join('|');

  // Calculate SHA-256 hash
  const hash = crypto.createHash('sha256');
  hash.update(data);
  return hash.digest('hex');
}
```

### Tamper Detection

```typescript
async function detectTampering(
  startSequence: number,
  endSequence: number
): Promise<TamperReport> {
  const tamperedRecords: string[] = [];
  const brokenChains: { sequence: number; reason: string }[] = [];

  for (let seq = startSequence; seq <= endSequence; seq++) {
    const record = await getRecordBySequence(seq);

    if (!record) {
      brokenChains.push({ sequence: seq, reason: 'Record missing' });
      continue;
    }

    // Verify hash chain
    if (seq > 1) {
      const prevRecord = await getRecordBySequence(seq - 1);
      if (record.previous_hash !== prevRecord.hash) {
        brokenChains.push({
          sequence: seq,
          reason: 'Previous hash mismatch'
        });
      }
    }

    // Verify record hash
    const expectedHash = calculateHash(record, record.previous_hash);
    if (record.hash !== expectedHash) {
      tamperedRecords.push(record.record_id);
    }
  }

  return {
    total_checked: endSequence - startSequence + 1,
    tampered_records: tamperedRecords,
    broken_chains: brokenChains,
    is_valid: tamperedRecords.length === 0 && brokenChains.length === 0
  };
}
```

---

## Query Patterns

### Pattern 1: Entity Timeline

```typescript
async function getEntityTimeline(entityId: string): Promise<Timeline> {
  const history = await ledger.getHistory(entityId);

  return {
    entity_id: entityId,
    total_events: history.length,
    first_event: history[0]?.transaction_time,
    last_event: history[history.length - 1]?.transaction_time,
    events: history.map(e => ({
      timestamp: e.transaction_time,
      event_type: e.event_type,
      field: e.field_name,
      change: `${e.old_value} → ${e.new_value}`,
      user: e.user_id,
      reason: e.reason
    }))
  };
}
```

### Pattern 2: Field History

```typescript
async function getFieldHistory(
  entityId: string,
  fieldName: string
): Promise<FieldTimeline> {
  const history = await ledger.getHistory(entityId, fieldName);

  return {
    entity_id: entityId,
    field_name: fieldName,
    total_changes: history.length,
    changes: history.map(e => ({
      from: e.old_value,
      to: e.new_value,
      when_recorded: e.transaction_time,
      when_effective: e.valid_time,
      by: e.user_id,
      reason: e.reason
    }))
  };
}
```

### Pattern 3: User Activity Report

```typescript
async function getUserActivityReport(
  userId: string,
  startDate: string,
  endDate: string
): Promise<ActivityReport> {
  const events = await ledger.getEventsByTransactionTime(
    startDate,
    endDate,
    { user_ids: [userId] }
  );

  const byEventType = events.reduce((acc, e) => {
    acc[e.event_type] = (acc[e.event_type] || 0) + 1;
    return acc;
  }, {} as Record<string, number>);

  const byEntityType = events.reduce((acc, e) => {
    acc[e.entity_type] = (acc[e.entity_type] || 0) + 1;
    return acc;
  }, {} as Record<string, number>);

  return {
    user_id: userId,
    period: { start: startDate, end: endDate },
    total_changes: events.length,
    by_event_type: byEventType,
    by_entity_type: byEntityType,
    most_edited_entities: findMostEditedEntities(events)
  };
}
```

---

## Edge Cases

### Edge Case 1: Simultaneous Events for Same Entity

**Scenario**: Two events for same entity recorded at exactly the same transaction_time.

**Handling**: Use `sequence_number` for deterministic ordering:

```typescript
// Events with same transaction_time are ordered by sequence_number
const history = await ledger.getHistory("txn_001");

// Even if transaction_time is identical, sequence_number guarantees order
history.sort((a, b) => a.sequence_number - b.sequence_number);
```

**Database Guarantee**: `sequence_number` is UNIQUE and auto-incrementing, ensuring total order.

---

### Edge Case 2: Future Valid Time

**Scenario**: Event recorded today but effective in the future (scheduled change).

**Handling**: Allowed - valid_time can be in the future:

```typescript
// Schedule price change for next month
await ledger.append({
  entity_id: "prod_001",
  entity_type: "product",
  event_type: "scheduled_price_change",
  field_name: "price",
  old_value: 29.99,
  new_value: 24.99,
  transaction_time: "2025-01-20T10:00:00Z", // Recorded today
  valid_time: "2025-02-01T00:00:00Z",       // Effective next month
  user_id: "pricing_manager",
  reason: "February sale - scheduled price drop"
});
```

**Query Impact**: Current state queries should filter `valid_time <= NOW()` to exclude future events.

---

### Edge Case 3: Out-of-Order Transaction Times

**Scenario**: Events inserted with non-monotonic transaction_times (e.g., backfilling historical data).

**Handling**: Allowed, but use `sequence_number` for actual insertion order:

```typescript
// Backfill: insert event from 2024 today
await ledger.append({
  entity_id: "txn_001",
  entity_type: "transaction",
  event_type: "created",
  field_name: "merchant",
  old_value: null,
  new_value: "Walmart",
  transaction_time: "2024-12-01T10:00:00Z", // Past transaction time
  valid_time: "2024-12-01T10:00:00Z",
  user_id: "import_script",
  reason: "Backfilled from legacy system"
});

// Query by transaction_time: gets events from 2024
// Query by sequence_number: shows this was inserted recently
```

**Best Practice**: Set `transaction_time = NOW()` unless explicitly backfilling.

---

### Edge Case 4: Null vs. Undefined Values

**Scenario**: How to represent "no value" vs. "unknown value"?

**Handling**:
- `null`: Explicit absence of value (e.g., field was empty)
- `undefined`: Not applicable (use null in JSONB)

```typescript
// Field was explicitly empty
await ledger.append({
  entity_id: "txn_001",
  field_name: "memo",
  old_value: null,      // Was empty
  new_value: "Lunch",   // Now has value
  ...
});

// Field being deleted
await ledger.append({
  entity_id: "txn_001",
  field_name: "memo",
  old_value: "Lunch",
  new_value: null,      // Cleared to empty
  event_type: "cleared",
  ...
});
```

---

### Edge Case 5: Very Large Field Values

**Scenario**: Field value exceeds reasonable size (e.g., 10MB JSON document).

**Handling**: Store reference instead of value:

```typescript
// Instead of storing huge JSON directly
await ledger.append({
  entity_id: "doc_001",
  field_name: "content",
  old_value: { type: "reference", url: "s3://bucket/old-content.json" },
  new_value: { type: "reference", url: "s3://bucket/new-content.json" },
  ...
});
```

**Limit**: Consider limiting `old_value`/`new_value` JSONB to 1MB per record.

---

### Edge Case 6: Hash Chain Break on First Record

**Scenario**: First record has no previous hash.

**Handling**: Use genesis hash (all zeros):

```typescript
const GENESIS_HASH = '0'.repeat(64); // 64 zeros

// First record in ledger
await ledger.append({
  entity_id: "txn_001",
  entity_type: "transaction",
  event_type: "created",
  field_name: "merchant",
  old_value: null,
  new_value: "Walmart",
  valid_time: "2025-01-01T00:00:00Z",
  user_id: "system",
  // previous_hash will be GENESIS_HASH
});
```

---

### Edge Case 7: Retroactive Deletion

**Scenario**: Entity should be marked deleted retroactively.

**Handling**: Use `event_type: "deleted"` with past `valid_time`:

```typescript
// Discover transaction should have been deleted on Jan 15
await ledger.append({
  entity_id: "txn_001",
  entity_type: "transaction",
  event_type: "deleted",
  field_name: "_status", // Special field for entity status
  old_value: "active",
  new_value: "deleted",
  transaction_time: "2025-01-20T10:00:00Z", // Recorded today
  valid_time: "2025-01-15T10:00:00Z",       // Effective retroactively
  user_id: "user_jane_doe",
  reason: "Transaction was duplicate, retroactively deleted"
});
```

**Query Impact**: Queries at `valid_time >= 2025-01-15` should exclude this entity.

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `append()` | 1 record | < 10ms | Single INSERT |
| `getHistory()` | 1 entity | < 50ms | Indexed by entity_id |
| `getHistory()` | 1 entity, 1 field | < 20ms | Indexed by entity_id + field_name |
| `verifyIntegrity()` | 1 record | < 30ms | Fetch + hash calculation |
| `getEventsByTransactionTime()` | 1 day | < 100ms | Indexed range query |
| `getEventsByValidTime()` | 1 day | < 100ms | Indexed range query |
| `export()` | 10,000 records | < 5s | Bulk query + serialization |
| `count()` | Any filters | < 100ms | COUNT query with indexes |

### Throughput

- **Writes**: 1,000-5,000 appends/sec (single node)
- **Reads**: 10,000-50,000 queries/sec (with indexes)

### Scalability

| Total Records | Query Performance | Notes |
|--------------|------------------|-------|
| < 1M | Excellent (< 50ms) | All queries fast |
| 1M - 10M | Good (< 100ms) | Indexed queries fast |
| 10M - 100M | Moderate (< 500ms) | Consider partitioning |
| > 100M | Slower (< 2s) | Partition by time range |

### Index Strategy

```sql
-- B-tree indexes for exact matches
CREATE INDEX idx_provenance_entity_id ON provenance_ledger(entity_id);
CREATE INDEX idx_provenance_entity_type ON provenance_ledger(entity_type);

-- B-tree indexes for time range queries (DESC for recent-first queries)
CREATE INDEX idx_provenance_transaction_time ON provenance_ledger(transaction_time DESC);
CREATE INDEX idx_provenance_valid_time ON provenance_ledger(valid_time DESC);

-- GiST index for bitemporal range queries
CREATE INDEX idx_provenance_bitemporal ON provenance_ledger
  USING GIST (
    tstzrange(transaction_time, transaction_time + INTERVAL '1 microsecond'),
    tstzrange(valid_time, valid_time + INTERVAL '1 microsecond')
  );
```

### Optimization Tips

**1. Partition by Time Range**

For very large ledgers (>100M records), partition by transaction_time:

```sql
CREATE TABLE provenance_ledger_2025_01 PARTITION OF provenance_ledger
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE provenance_ledger_2025_02 PARTITION OF provenance_ledger
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

**2. Batch Appends**

```typescript
// Bad: N individual inserts
for (const record of records) {
  await ledger.append(record); // N round trips
}

// Good: Batch insert (if ledger supports it)
await ledger.appendBatch(records); // 1 round trip
```

**3. Limit History Queries**

```typescript
// Bad: Fetch all history (could be millions of events)
const history = await ledger.getHistory(entityId);

// Good: Limit to recent history
const recentHistory = await ledger.getHistory(entityId, undefined, { limit: 100 });
```

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';
import crypto from 'crypto';

export class PostgresProvenanceLedger implements ProvenanceLedger {
  constructor(private pool: Pool) {}

  async append(record: BitemporalRecord): Promise<void> {
    // Validate
    this.validate(record);

    // Generate ID and timestamps
    const record_id = this.generateId();
    const transaction_time = record.transaction_time || new Date().toISOString();
    const created_at = transaction_time;

    // Get previous hash
    const previousHash = await this.getLatestHash();

    // Calculate hash
    const hash = this.calculateHash(record, previousHash);

    // Insert
    const query = `
      INSERT INTO provenance_ledger (
        record_id, entity_id, entity_type, event_type, field_name,
        old_value, new_value, transaction_time, valid_time,
        user_id, reason, source_system, correlation_id,
        hash, previous_hash, created_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16)
    `;

    const values = [
      record_id,
      record.entity_id,
      record.entity_type,
      record.event_type,
      record.field_name,
      JSON.stringify(record.old_value),
      JSON.stringify(record.new_value),
      transaction_time,
      record.valid_time,
      record.user_id,
      record.reason || null,
      record.source_system || null,
      record.correlation_id || null,
      hash,
      previousHash,
      created_at
    ];

    await this.pool.query(query, values);
  }

  async getHistory(
    entityId: string,
    fieldName?: string
  ): Promise<BitemporalRecord[]> {
    let query = `
      SELECT * FROM provenance_ledger
      WHERE entity_id = $1
    `;
    const values: any[] = [entityId];

    if (fieldName) {
      query += ` AND field_name = $2`;
      values.push(fieldName);
    }

    query += ` ORDER BY transaction_time ASC, sequence_number ASC`;

    const result = await this.pool.query(query, values);
    return result.rows.map(row => this.rowToRecord(row));
  }

  async verifyIntegrity(recordId: string): Promise<boolean> {
    // Fetch record
    const record = await this.getByRecordId(recordId);
    if (!record) return false;

    // Fetch previous record
    const prevQuery = `
      SELECT * FROM provenance_ledger
      WHERE sequence_number = $1
    `;
    const prevResult = await this.pool.query(prevQuery, [record.sequence_number - 1]);

    if (record.sequence_number > 1 && prevResult.rows.length === 0) {
      return false; // Missing previous record
    }

    const prevRecord = prevResult.rows[0];

    // Verify hash chain
    if (record.sequence_number > 1 && record.previous_hash !== prevRecord.hash) {
      return false; // Chain broken
    }

    // Verify record hash
    const expectedHash = this.calculateHash(record, record.previous_hash);
    return record.hash === expectedHash;
  }

  async export(
    filters: ProvenanceQueryFilters,
    format: 'json' | 'csv'
  ): Promise<string> {
    const records = await this.query(filters);

    if (format === 'json') {
      return JSON.stringify(records, null, 2);
    } else {
      return this.toCsv(records);
    }
  }

  private calculateHash(record: BitemporalRecord, previousHash: string): string {
    const data = [
      record.entity_id,
      record.entity_type,
      record.event_type,
      record.field_name,
      JSON.stringify(record.old_value),
      JSON.stringify(record.new_value),
      record.transaction_time,
      record.valid_time,
      record.user_id,
      record.reason || '',
      previousHash
    ].join('|');

    const hash = crypto.createHash('sha256');
    hash.update(data);
    return hash.digest('hex');
  }

  private async getLatestHash(): Promise<string> {
    const query = `
      SELECT hash FROM provenance_ledger
      ORDER BY sequence_number DESC
      LIMIT 1
    `;
    const result = await this.pool.query(query);

    if (result.rows.length === 0) {
      return '0'.repeat(64); // Genesis hash
    }

    return result.rows[0].hash;
  }

  private generateId(): string {
    return `prov_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private validate(record: BitemporalRecord): void {
    if (!record.entity_id) throw new Error('entity_id is required');
    if (!record.entity_type) throw new Error('entity_type is required');
    if (!record.event_type) throw new Error('event_type is required');
    if (!record.field_name) throw new Error('field_name is required');
    if (!record.valid_time) throw new Error('valid_time is required');
    if (!record.user_id) throw new Error('user_id is required');
  }

  private rowToRecord(row: any): BitemporalRecord {
    return {
      record_id: row.record_id,
      sequence_number: parseInt(row.sequence_number, 10),
      entity_id: row.entity_id,
      entity_type: row.entity_type,
      event_type: row.event_type,
      field_name: row.field_name,
      old_value: JSON.parse(row.old_value),
      new_value: JSON.parse(row.new_value),
      transaction_time: row.transaction_time,
      valid_time: row.valid_time,
      user_id: row.user_id,
      reason: row.reason,
      source_system: row.source_system,
      correlation_id: row.correlation_id,
      hash: row.hash,
      previous_hash: row.previous_hash,
      created_at: row.created_at
    };
  }

  private toCsv(records: BitemporalRecord[]): string {
    const headers = [
      'record_id', 'sequence_number', 'entity_id', 'entity_type', 'event_type',
      'field_name', 'old_value', 'new_value', 'transaction_time', 'valid_time',
      'user_id', 'reason', 'hash', 'previous_hash'
    ];

    const rows = records.map(r => [
      r.record_id,
      r.sequence_number,
      r.entity_id,
      r.entity_type,
      r.event_type,
      r.field_name,
      JSON.stringify(r.old_value),
      JSON.stringify(r.new_value),
      r.transaction_time,
      r.valid_time,
      r.user_id,
      r.reason || '',
      r.hash,
      r.previous_hash
    ]);

    return [headers, ...rows].map(row => row.join(',')).join('\n');
  }
}
```

---

## Security Considerations

### 1. Immutability Enforcement

**Database Level**:
```sql
REVOKE UPDATE, DELETE ON provenance_ledger FROM application_user;
```

**Application Level**: No update/delete methods in interface.

### 2. Hash Chain Verification

Run periodic integrity checks:

```typescript
// Daily cron job
async function dailyIntegrityCheck() {
  const report = await detectTampering(1, await getMaxSequenceNumber());

  if (!report.is_valid) {
    await alertSecurityTeam(report);
  }
}
```

### 3. Access Control

Restrict who can append events:

```typescript
async function appendWithAuth(
  record: BitemporalRecord,
  currentUser: User
): Promise<void> {
  // Verify user has permission to modify this entity type
  if (!currentUser.canModify(record.entity_type)) {
    throw new Error('Unauthorized');
  }

  // Set user_id to authenticated user (prevent spoofing)
  record.user_id = currentUser.id;

  await ledger.append(record);
}
```

### 4. Sensitive Data

For PII/PHI, consider encryption:

```typescript
await ledger.append({
  entity_id: "patient_001",
  field_name: "ssn",
  old_value: encrypt("111-11-1111"),
  new_value: encrypt("222-22-2222"),
  ...
});
```

---

## Integration Patterns

### Pattern 1: Automatic Event Logging

```typescript
class EntityService {
  async updateEntity(
    entityId: string,
    updates: Record<string, any>,
    userId: string
  ): Promise<void> {
    const entity = await this.getEntity(entityId);

    // Apply updates
    await this.entityStore.update(entityId, updates);

    // Log to provenance ledger
    for (const [field, newValue] of Object.entries(updates)) {
      await ledger.append({
        entity_id: entityId,
        entity_type: entity.type,
        event_type: 'updated',
        field_name: field,
        old_value: entity[field],
        new_value: newValue,
        valid_time: new Date().toISOString(),
        user_id: userId,
        reason: 'User update'
      });
    }
  }
}
```

### Pattern 2: Event Sourcing

```typescript
class EventSourcedEntity {
  async replay(entityId: string): Promise<Entity> {
    const history = await ledger.getHistory(entityId);

    const entity: any = { id: entityId };

    for (const event of history) {
      if (event.event_type === 'created' || event.event_type === 'updated') {
        entity[event.field_name] = event.new_value;
      } else if (event.event_type === 'deleted') {
        entity._deleted = true;
      }
    }

    return entity;
  }
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section above with 7 domains)

---

## Testing Strategy

### Unit Tests

```typescript
describe('ProvenanceLedger', () => {
  let ledger: ProvenanceLedger;

  beforeEach(async () => {
    ledger = new PostgresProvenanceLedger(pool);
    await cleanDatabase();
  });

  describe('append()', () => {
    it('should append event with generated ID', async () => {
      await ledger.append({
        entity_id: 'txn_001',
        entity_type: 'transaction',
        event_type: 'created',
        field_name: 'merchant',
        old_value: null,
        new_value: 'Walmart',
        valid_time: '2025-01-15T10:00:00Z',
        user_id: 'user_jane'
      });

      const history = await ledger.getHistory('txn_001');
      expect(history).toHaveLength(1);
      expect(history[0].entity_id).toBe('txn_001');
      expect(history[0].new_value).toBe('Walmart');
    });

    it('should maintain hash chain', async () => {
      await ledger.append({
        entity_id: 'txn_001',
        entity_type: 'transaction',
        event_type: 'created',
        field_name: 'merchant',
        old_value: null,
        new_value: 'Walmart',
        valid_time: '2025-01-15T10:00:00Z',
        user_id: 'user_jane'
      });

      await ledger.append({
        entity_id: 'txn_001',
        entity_type: 'transaction',
        event_type: 'updated',
        field_name: 'merchant',
        old_value: 'Walmart',
        new_value: 'Walmart Supercenter',
        valid_time: '2025-01-15T10:00:00Z',
        user_id: 'user_jane'
      });

      const history = await ledger.getHistory('txn_001');
      expect(history[1].previous_hash).toBe(history[0].hash);
    });
  });

  describe('verifyIntegrity()', () => {
    it('should verify valid record', async () => {
      await ledger.append({
        entity_id: 'txn_001',
        entity_type: 'transaction',
        event_type: 'created',
        field_name: 'merchant',
        old_value: null,
        new_value: 'Walmart',
        valid_time: '2025-01-15T10:00:00Z',
        user_id: 'user_jane'
      });

      const history = await ledger.getHistory('txn_001');
      const isValid = await ledger.verifyIntegrity(history[0].record_id);

      expect(isValid).toBe(true);
    });
  });
});
```

---

## Migration Guide

### From No Provenance System

```typescript
// Before: Update without history
await entityStore.update('txn_001', { merchant: 'Amazon.com' });

// After: Update with provenance
const entity = await entityStore.getById('txn_001');

await ledger.append({
  entity_id: 'txn_001',
  entity_type: 'transaction',
  event_type: 'updated',
  field_name: 'merchant',
  old_value: entity.merchant,
  new_value: 'Amazon.com',
  valid_time: new Date().toISOString(),
  user_id: currentUserId,
  reason: 'User correction'
});

await entityStore.update('txn_001', { merchant: 'Amazon.com' });
```

---

## Related Primitives

- **BitemporalQuery**: Query provenance ledger with temporal filters
- **TimelineReconstructor**: Reconstruct entity state at any point in time
- **RetroactiveCorrector**: Handle retroactive corrections with validation
- **AuditLog**: System-level audit trail (complements provenance ledger)

---

## References

- [Objective Layer Architecture](../architecture/objective-layer.md)
- [Provenance Ledger Vertical 5.1](../verticals/provenance-ledger.md)
- [Bitemporal Data Management](https://en.wikipedia.org/wiki/Temporal_database)
- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)

---

**End of ProvenanceLedger OL Primitive Specification**
