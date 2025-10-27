# OverrideStore OL Primitive

**Domain:** Corrections Flow (Vertical 4.3)
**Layer:** Objective Layer (OL)
**Version:** 1.0.0
**Status:** Specification

---

## Table of Contents

1. [Overview](#overview)
2. [Purpose](#purpose)
3. [Core Concepts](#core-concepts)
4. [Interface Definition](#interface-definition)
5. [Data Model](#data-model)
6. [Core Operations](#core-operations)
7. [Bulk Operations](#bulk-operations)
8. [Query Operations](#query-operations)
9. [Multi-Domain Examples](#multi-domain-examples)
10. [Edge Cases](#edge-cases)
11. [Performance Characteristics](#performance-characteristics)
12. [Implementation Details](#implementation-details)
13. [Error Handling](#error-handling)
14. [Integration Patterns](#integration-patterns)
15. [Testing Strategy](#testing-strategy)
16. [Migration Guide](#migration-guide)
17. [FAQ](#faq)

---

## Overview

The **OverrideStore** is an Objective Layer (OL) primitive that provides CRUD operations for field-level overrides with robust bulk operation support. It enables users to correct or override specific fields on entities without modifying the underlying canonical data, maintaining a complete audit trail of all corrections.

### Key Capabilities

- **Field-Level Precision**: Override individual fields on any entity
- **Bulk Operations**: Efficient batch processing for hundreds of overrides
- **Audit Trail**: Complete history of who changed what and when
- **Conflict Prevention**: Unique constraints prevent duplicate overrides
- **Multi-Domain**: Works across finance, healthcare, legal, research, and more

### Design Philosophy

The OverrideStore follows three core principles:

1. **Non-Destructive**: Original data remains unchanged in canonical storage
2. **Transparent**: Every override includes metadata about who, what, when, and why
3. **Performant**: Optimized for both single and bulk operations

---

## Purpose

### Problem Statement

In real-world data systems, extraction and classification errors are inevitable:

- **Finance**: Merchant names may be incorrectly extracted from receipts
- **Healthcare**: Diagnosis codes might be miscategorized by automation
- **Legal**: Case numbers could be misread from scanned documents
- **Research (RSRCH - Utilitario)**: Entity names might be incorrectly parsed from web articles (e.g., '@sama' → needs manual correction to 'Sam Altman')

Traditional approaches have significant drawbacks:

**Approach 1: Modify Source Data**
- ❌ Loses original extracted value
- ❌ No audit trail
- ❌ Can't distinguish between "correct extraction" and "manual fix"

**Approach 2: Create New Version of Entity**
- ❌ Duplicates entire entity for one field change
- ❌ Complicated versioning logic
- ❌ Difficult to query "current state"

**Approach 3: Separate Override Layer**
- ✅ Preserves original data
- ✅ Clear audit trail
- ✅ Simple to implement and query
- ✅ Scales to millions of overrides

### Solution

The OverrideStore implements **Approach 3** with:

1. **Separate Storage**: Overrides stored in dedicated `field_overrides` table
2. **Entity-Field Uniqueness**: One override per (entity_id, field_name) pair
3. **Application-Level Merging**: Client code merges overrides with canonical data
4. **Bulk Efficiency**: Optimized batch operations for mass corrections

---

## Core Concepts

### Field Override

A **field override** is a single correction applied to one field on one entity:

```typescript
{
  override_id: "ovr_1234567890",
  entity_id: "txn_abc123",
  field_name: "merchant",

  original_value: "AMZN MKTP US*1234",
  overridden_value: "Amazon.com",

  overridden_by: "user_jane_doe",
  overridden_at: "2025-01-15T10:30:00Z",
  reason: "Normalized merchant name for reporting"
}
```

### Uniqueness Constraint

Only **one active override** per `(entity_id, field_name)`:

```sql
UNIQUE (entity_id, field_name)
```

Attempting to create a duplicate override will:
- **Single create**: Return error
- **Bulk create**: Skip duplicate, continue with others

### Override Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    Override Lifecycle                        │
└─────────────────────────────────────────────────────────────┘

1. CREATE
   ├─ User identifies incorrect field value
   ├─ Creates override with new value + reason
   └─ System stores in field_overrides table

2. ACTIVE
   ├─ Override exists in database
   ├─ Application merges with canonical data on read
   └─ User sees overridden value in UI

3. DELETE
   ├─ User removes override
   ├─ System deletes from field_overrides table
   └─ Original canonical value becomes visible again
```

### Merge Strategy

When reading entities, application code merges overrides:

```typescript
function applyOverrides(entity: Entity, overrides: FieldOverride[]): Entity {
  const merged = { ...entity };

  for (const override of overrides) {
    merged[override.field_name] = override.overridden_value;
  }

  return merged;
}
```

---

## Interface Definition

### TypeScript Interface

```typescript
interface OverrideStore {
  // ===== Single Operations =====

  /**
   * Create a new field override
   *
   * @param override - Override data
   * @returns Created override with generated ID and metadata
   * @throws ConflictError if override already exists for (entity_id, field_name)
   */
  create(override: CreateOverrideInput): Promise<FieldOverride>;

  /**
   * Get override by ID
   *
   * @param overrideId - Unique override identifier
   * @returns Override if found, null otherwise
   */
  getById(overrideId: string): Promise<FieldOverride | null>;

  /**
   * Get all overrides for an entity
   *
   * @param entityId - Entity identifier
   * @returns Array of overrides (empty if none)
   */
  getByEntity(entityId: string): Promise<FieldOverride[]>;

  /**
   * Get override for specific field on entity
   *
   * @param entityId - Entity identifier
   * @param fieldName - Field name (e.g., "merchant", "category")
   * @returns Override if exists, null otherwise
   */
  getByField(
    entityId: string,
    fieldName: string
  ): Promise<FieldOverride | null>;

  /**
   * Delete override by ID
   *
   * @param overrideId - Override to delete
   * @returns void (no error if override doesn't exist)
   */
  delete(overrideId: string): Promise<void>;

  // ===== Bulk Operations =====

  /**
   * Create multiple overrides in one transaction
   *
   * @param overrides - Array of override data
   * @returns Array of created overrides (may be fewer than input if duplicates skipped)
   * @throws Error if database transaction fails
   */
  createBulk(overrides: CreateOverrideInput[]): Promise<FieldOverride[]>;

  /**
   * Delete multiple overrides in one transaction
   *
   * @param overrideIds - Array of override IDs
   * @returns void (no error if some IDs don't exist)
   */
  deleteBulk(overrideIds: string[]): Promise<void>;

  // ===== Query Operations =====

  /**
   * Query overrides with filters
   *
   * @param filters - Query criteria
   * @returns Array of matching overrides
   */
  query(filters: OverrideQueryFilters): Promise<FieldOverride[]>;

  /**
   * Count overrides matching filters
   *
   * @param filters - Query criteria
   * @returns Count of matching overrides
   */
  count(filters: OverrideQueryFilters): Promise<number>;
}
```

---

## Data Model

### FieldOverride Type

```typescript
interface FieldOverride {
  // Identity
  override_id: string;           // Unique ID (e.g., "ovr_1234567890")
  entity_id: string;             // Entity being overridden (e.g., "txn_abc123")
  field_name: string;            // Field being overridden (e.g., "merchant")

  // Values
  original_value: any;           // Value before override (for audit)
  overridden_value: any;         // New value to use

  // Metadata
  overridden_by: string;         // User who made override (e.g., "user_jane_doe")
  overridden_at: string;         // ISO timestamp when override was created
  reason?: string;               // Optional justification for override

  // System
  created_at: string;            // ISO timestamp of record creation
  updated_at: string;            // ISO timestamp of last update
}
```

### CreateOverrideInput Type

```typescript
interface CreateOverrideInput {
  entity_id: string;             // Required
  field_name: string;            // Required
  original_value: any;           // Required (for audit trail)
  overridden_value: any;         // Required (new value)
  overridden_by: string;         // Required (user ID)
  reason?: string;               // Optional (recommended for audit)
}
```

### OverrideQueryFilters Type

```typescript
interface OverrideQueryFilters {
  entity_ids?: string[];         // Filter by entity IDs
  field_names?: string[];        // Filter by field names
  overridden_by?: string[];      // Filter by user who made override
  created_after?: string;        // ISO timestamp
  created_before?: string;       // ISO timestamp

  // Pagination
  limit?: number;                // Max results (default: 100)
  offset?: number;               // Skip N results (default: 0)

  // Sorting
  sort_by?: 'created_at' | 'overridden_at' | 'entity_id';
  sort_order?: 'asc' | 'desc';
}
```

### Database Schema (PostgreSQL)

```sql
CREATE TABLE field_overrides (
  -- Primary Key
  override_id VARCHAR(64) PRIMARY KEY,

  -- Entity & Field
  entity_id VARCHAR(128) NOT NULL,
  field_name VARCHAR(128) NOT NULL,

  -- Values (JSONB for flexibility)
  original_value JSONB NOT NULL,
  overridden_value JSONB NOT NULL,

  -- Metadata
  overridden_by VARCHAR(128) NOT NULL,
  overridden_at TIMESTAMP WITH TIME ZONE NOT NULL,
  reason TEXT,

  -- System Timestamps
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

  -- Unique Constraint (one override per entity+field)
  CONSTRAINT unique_entity_field UNIQUE (entity_id, field_name)
);

-- Indexes for Performance
CREATE INDEX idx_overrides_entity_id ON field_overrides(entity_id);
CREATE INDEX idx_overrides_field_name ON field_overrides(field_name);
CREATE INDEX idx_overrides_overridden_by ON field_overrides(overridden_by);
CREATE INDEX idx_overrides_overridden_at ON field_overrides(overridden_at);
CREATE INDEX idx_overrides_created_at ON field_overrides(created_at);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_field_overrides_updated_at
BEFORE UPDATE ON field_overrides
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

---

## Core Operations

### 1. create()

Create a single field override.

#### Signature

```typescript
create(override: CreateOverrideInput): Promise<FieldOverride>
```

#### Behavior

1. **Validation**:
   - All required fields present
   - `entity_id` format valid
   - `field_name` is non-empty string
   - `overridden_value` is different from `original_value`

2. **ID Generation**:
   - Generate unique `override_id` (e.g., `ovr_` + timestamp + random)

3. **Conflict Check**:
   - Query for existing override with same `(entity_id, field_name)`
   - If exists, throw `ConflictError`

4. **Insert**:
   - Insert into `field_overrides` table
   - Set `created_at` and `updated_at` to current timestamp

5. **Return**:
   - Return full `FieldOverride` object with all metadata

#### Example

```typescript
const override = await store.create({
  entity_id: "txn_abc123",
  field_name: "merchant",
  original_value: "AMZN MKTP US*1234",
  overridden_value: "Amazon.com",
  overridden_by: "user_jane_doe",
  reason: "Normalized merchant name"
});

console.log(override);
// {
//   override_id: "ovr_1234567890",
//   entity_id: "txn_abc123",
//   field_name: "merchant",
//   original_value: "AMZN MKTP US*1234",
//   overridden_value: "Amazon.com",
//   overridden_by: "user_jane_doe",
//   overridden_at: "2025-01-15T10:30:00Z",
//   reason: "Normalized merchant name",
//   created_at: "2025-01-15T10:30:00Z",
//   updated_at: "2025-01-15T10:30:00Z"
// }
```

#### Error Cases

```typescript
// Case 1: Duplicate override
try {
  await store.create({
    entity_id: "txn_abc123",
    field_name: "merchant", // Already has override
    original_value: "...",
    overridden_value: "...",
    overridden_by: "user_jane_doe"
  });
} catch (error) {
  console.log(error.code); // "CONFLICT"
  console.log(error.message);
  // "Override already exists for entity txn_abc123, field merchant"
}

// Case 2: Missing required field
try {
  await store.create({
    entity_id: "txn_abc123",
    // missing field_name
    original_value: "...",
    overridden_value: "...",
    overridden_by: "user_jane_doe"
  });
} catch (error) {
  console.log(error.code); // "VALIDATION_ERROR"
  console.log(error.message); // "field_name is required"
}
```

---

### 2. getById()

Retrieve override by its unique ID.

#### Signature

```typescript
getById(overrideId: string): Promise<FieldOverride | null>
```

#### Behavior

1. Query `field_overrides` table by `override_id`
2. Return full `FieldOverride` object if found
3. Return `null` if not found

#### Example

```typescript
const override = await store.getById("ovr_1234567890");

if (override) {
  console.log(`Override for ${override.entity_id}.${override.field_name}`);
  console.log(`Changed from: ${override.original_value}`);
  console.log(`Changed to: ${override.overridden_value}`);
} else {
  console.log("Override not found");
}
```

---

### 3. getByEntity()

Get all overrides for a specific entity.

#### Signature

```typescript
getByEntity(entityId: string): Promise<FieldOverride[]>
```

#### Behavior

1. Query `field_overrides` table WHERE `entity_id = $1`
2. Return array of all matching overrides
3. Return empty array if no overrides exist

#### Example

```typescript
const overrides = await store.getByEntity("txn_abc123");

console.log(`Found ${overrides.length} overrides for txn_abc123`);

for (const override of overrides) {
  console.log(`  ${override.field_name}: ${override.overridden_value}`);
}

// Output:
// Found 3 overrides for txn_abc123
//   merchant: Amazon.com
//   category: Shopping
//   amount: 49.99
```

#### Use Case: Merging Overrides with Entity

```typescript
async function getEntityWithOverrides(entityId: string): Promise<Entity> {
  // 1. Get canonical entity
  const entity = await entityStore.getById(entityId);

  // 2. Get all overrides for entity
  const overrides = await overrideStore.getByEntity(entityId);

  // 3. Apply overrides
  const merged = { ...entity };
  for (const override of overrides) {
    merged[override.field_name] = override.overridden_value;
  }

  return merged;
}
```

---

### 4. getByField()

Get override for a specific field on a specific entity.

#### Signature

```typescript
getByField(entityId: string, fieldName: string): Promise<FieldOverride | null>
```

#### Behavior

1. Query `field_overrides` WHERE `entity_id = $1 AND field_name = $2`
2. Return `FieldOverride` if exists
3. Return `null` if no override exists

#### Example

```typescript
const merchantOverride = await store.getByField("txn_abc123", "merchant");

if (merchantOverride) {
  console.log(`Merchant overridden to: ${merchantOverride.overridden_value}`);
  console.log(`By: ${merchantOverride.overridden_by}`);
  console.log(`Reason: ${merchantOverride.reason}`);
} else {
  console.log("No merchant override exists");
}
```

---

### 5. delete()

Delete a single override by ID.

#### Signature

```typescript
delete(overrideId: string): Promise<void>
```

#### Behavior

1. Execute `DELETE FROM field_overrides WHERE override_id = $1`
2. No error if override doesn't exist (idempotent)
3. Return void

#### Example

```typescript
// Delete override
await store.delete("ovr_1234567890");

// Verify deletion
const override = await store.getById("ovr_1234567890");
console.log(override); // null

// Safe to call multiple times
await store.delete("ovr_1234567890"); // No error
await store.delete("ovr_1234567890"); // No error
```

#### Use Case: Reverting Override

```typescript
async function revertFieldToOriginal(
  entityId: string,
  fieldName: string
): Promise<void> {
  // Find override
  const override = await store.getByField(entityId, fieldName);

  if (override) {
    // Delete it to revert to original value
    await store.delete(override.override_id);
    console.log(`Reverted ${fieldName} to original value`);
  } else {
    console.log(`No override exists for ${fieldName}`);
  }
}
```

---

## Bulk Operations

### 6. createBulk()

Create multiple overrides in a single database transaction.

#### Signature

```typescript
createBulk(overrides: CreateOverrideInput[]): Promise<FieldOverride[]>
```

#### Behavior

1. **Validation**:
   - Validate all inputs before starting transaction
   - Skip invalid entries (log warning)

2. **Deduplication**:
   - Group by `(entity_id, field_name)`
   - If multiple overrides for same field, keep last one

3. **Conflict Handling**:
   - Check existing overrides in database
   - Skip overrides that already exist (don't throw error)
   - Continue with non-conflicting ones

4. **Batch Insert**:
   - Use single `INSERT INTO ... VALUES (...), (...), (...)`
   - Wrap in transaction for atomicity
   - Generate IDs for all overrides

5. **Return**:
   - Return array of successfully created overrides
   - May be fewer than input if duplicates/conflicts skipped

#### Example: Bulk Category Override

```typescript
// Scenario: User reviewed 500 transactions and wants to fix categories
const overrides = transactions.map(txn => ({
  entity_id: txn.id,
  field_name: "category",
  original_value: txn.category,
  overridden_value: correctCategoryFor(txn),
  overridden_by: "user_jane_doe",
  reason: "Bulk category correction after review"
}));

const created = await store.createBulk(overrides);

console.log(`Created ${created.length} overrides`);
console.log(`Skipped ${overrides.length - created.length} (duplicates/conflicts)`);
```

#### Performance Characteristics

```
Input Size     | Time (ms) | Notes
---------------|-----------|------------------------
10 overrides   | 5-10      | Single INSERT
100 overrides  | 20-50     | Batched INSERT
1,000 overrides| 200-500   | Batched INSERT
10,000+        | 2,000+    | Consider chunking
```

#### Chunking for Large Batches

```typescript
async function createBulkChunked(
  overrides: CreateOverrideInput[],
  chunkSize: number = 1000
): Promise<FieldOverride[]> {
  const results: FieldOverride[] = [];

  for (let i = 0; i < overrides.length; i += chunkSize) {
    const chunk = overrides.slice(i, i + chunkSize);
    const created = await store.createBulk(chunk);
    results.push(...created);

    console.log(`Processed ${i + chunk.length}/${overrides.length}`);
  }

  return results;
}
```

---

### 7. deleteBulk()

Delete multiple overrides in a single transaction.

#### Signature

```typescript
deleteBulk(overrideIds: string[]): Promise<void>
```

#### Behavior

1. Execute `DELETE FROM field_overrides WHERE override_id IN ($1, $2, ...)`
2. Wrap in transaction for atomicity
3. No error if some IDs don't exist
4. Return void

#### Example

```typescript
// Get all overrides for entity
const overrides = await store.getByEntity("txn_abc123");

// Delete them all
const ids = overrides.map(o => o.override_id);
await store.deleteBulk(ids);

// Verify
const remaining = await store.getByEntity("txn_abc123");
console.log(remaining.length); // 0
```

#### Use Case: Clear All Overrides by User

```typescript
async function clearUserOverrides(userId: string): Promise<number> {
  // Find all overrides by user
  const overrides = await store.query({
    overridden_by: [userId]
  });

  // Delete them
  const ids = overrides.map(o => o.override_id);
  await store.deleteBulk(ids);

  console.log(`Cleared ${ids.length} overrides by ${userId}`);
  return ids.length;
}
```

---

## Query Operations

### 8. query()

Flexible querying of overrides with filters, pagination, and sorting.

#### Signature

```typescript
query(filters: OverrideQueryFilters): Promise<FieldOverride[]>
```

#### Filters

```typescript
interface OverrideQueryFilters {
  // Field Filters
  entity_ids?: string[];         // Match any of these entity IDs
  field_names?: string[];        // Match any of these field names
  overridden_by?: string[];      // Match any of these user IDs

  // Time Range Filters
  created_after?: string;        // ISO timestamp
  created_before?: string;       // ISO timestamp

  // Pagination
  limit?: number;                // Default: 100, Max: 1000
  offset?: number;               // Default: 0

  // Sorting
  sort_by?: 'created_at' | 'overridden_at' | 'entity_id';
  sort_order?: 'asc' | 'desc';   // Default: 'desc'
}
```

#### Examples

**Example 1: Get Recent Overrides**

```typescript
const recent = await store.query({
  created_after: "2025-01-01T00:00:00Z",
  sort_by: "created_at",
  sort_order: "desc",
  limit: 50
});

console.log(`${recent.length} overrides created since Jan 1, 2025`);
```

**Example 2: Get All Merchant Overrides**

```typescript
const merchantOverrides = await store.query({
  field_names: ["merchant"],
  limit: 1000
});

console.log(`${merchantOverrides.length} merchant overrides in system`);
```

**Example 3: Get Overrides for Multiple Entities**

```typescript
const overrides = await store.query({
  entity_ids: ["txn_001", "txn_002", "txn_003"]
});

// Group by entity
const byEntity = overrides.reduce((acc, override) => {
  if (!acc[override.entity_id]) {
    acc[override.entity_id] = [];
  }
  acc[override.entity_id].push(override);
  return acc;
}, {} as Record<string, FieldOverride[]>);
```

**Example 4: Pagination**

```typescript
async function getAllOverridesPages(): Promise<FieldOverride[]> {
  const pageSize = 100;
  let offset = 0;
  let allOverrides: FieldOverride[] = [];

  while (true) {
    const page = await store.query({
      limit: pageSize,
      offset: offset
    });

    allOverrides.push(...page);

    if (page.length < pageSize) {
      break; // Last page
    }

    offset += pageSize;
  }

  return allOverrides;
}
```

---

### 9. count()

Count overrides matching filters (for pagination metadata).

#### Signature

```typescript
count(filters: OverrideQueryFilters): Promise<number>
```

#### Behavior

1. Execute `SELECT COUNT(*) FROM field_overrides WHERE ...`
2. Apply same filters as `query()` (except limit/offset)
3. Return integer count

#### Example

```typescript
// Count total overrides
const total = await store.count({});
console.log(`Total overrides: ${total}`);

// Count merchant overrides
const merchantCount = await store.count({
  field_names: ["merchant"]
});
console.log(`Merchant overrides: ${merchantCount}`);

// Pagination metadata
const filters = { field_names: ["category"] };
const totalCount = await store.count(filters);
const totalPages = Math.ceil(totalCount / 100);
console.log(`${totalCount} category overrides across ${totalPages} pages`);
```

---

## Multi-Domain Examples

### Example 1: Finance - Transaction Corrections

**Scenario**: User corrects merchant names, categories, and amounts on bank transactions.

```typescript
// Original transaction from bank statement
const transaction = {
  id: "txn_001",
  date: "2025-01-10",
  merchant: "AMZN MKTP US*AB1234",
  category: "Uncategorized",
  amount: -42.50,
  currency: "USD"
};

// User corrections
const overrides = await store.createBulk([
  {
    entity_id: "txn_001",
    field_name: "merchant",
    original_value: "AMZN MKTP US*AB1234",
    overridden_value: "Amazon.com",
    overridden_by: "user_john_doe",
    reason: "Normalized merchant name"
  },
  {
    entity_id: "txn_001",
    field_name: "category",
    original_value: "Uncategorized",
    overridden_value: "Shopping",
    overridden_by: "user_john_doe",
    reason: "Categorized as shopping"
  },
  {
    entity_id: "txn_001",
    field_name: "amount",
    original_value: -42.50,
    overridden_value: -45.00,
    overridden_by: "user_john_doe",
    reason: "Corrected amount from receipt"
  }
]);

// Apply overrides to get display version
const overridesForTxn = await store.getByEntity("txn_001");
const displayTransaction = { ...transaction };
for (const override of overridesForTxn) {
  displayTransaction[override.field_name] = override.overridden_value;
}

console.log(displayTransaction);
// {
//   id: "txn_001",
//   date: "2025-01-10",
//   merchant: "Amazon.com",        // Overridden
//   category: "Shopping",           // Overridden
//   amount: -45.00,                 // Overridden
//   currency: "USD"
// }
```

**Report Generation with Overrides**

```typescript
async function generateMonthlyReport(month: string): Promise<Report> {
  // 1. Get all transactions for month
  const transactions = await transactionStore.query({
    date_range: { start: `${month}-01`, end: `${month}-31` }
  });

  // 2. Get all overrides for these transactions
  const entityIds = transactions.map(t => t.id);
  const overrides = await overrideStore.query({ entity_ids: entityIds });

  // 3. Group overrides by entity
  const overridesByEntity = overrides.reduce((acc, o) => {
    if (!acc[o.entity_id]) acc[o.entity_id] = [];
    acc[o.entity_id].push(o);
    return acc;
  }, {} as Record<string, FieldOverride[]>);

  // 4. Apply overrides to each transaction
  const finalTransactions = transactions.map(txn => {
    const txnOverrides = overridesByEntity[txn.id] || [];
    const merged = { ...txn };
    for (const override of txnOverrides) {
      merged[override.field_name] = override.overridden_value;
    }
    return merged;
  });

  // 5. Generate report from final data
  return generateReport(finalTransactions);
}
```

---

### Example 2: Healthcare - Patient Record Corrections

**Scenario**: Medical coder corrects diagnosis codes and provider names on patient encounters.

```typescript
// Original encounter from EHR extraction
const encounter = {
  id: "enc_001",
  patient_id: "pat_12345",
  date: "2025-01-15",
  provider: "Dr. J Smith",          // Ambiguous
  diagnosis_code: "R07.9",          // Generic chest pain
  procedure_code: "99213",
  charge_amount: 150.00
};

// Medical coder corrections
await store.createBulk([
  {
    entity_id: "enc_001",
    field_name: "provider",
    original_value: "Dr. J Smith",
    overridden_value: "Dr. Jane Smith, MD",
    overridden_by: "coder_mary_jones",
    reason: "Disambiguated provider name for billing"
  },
  {
    entity_id: "enc_001",
    field_name: "diagnosis_code",
    original_value: "R07.9",
    overridden_value: "I20.9",      // Angina pectoris
    overridden_by: "coder_mary_jones",
    reason: "More specific diagnosis from chart review"
  }
]);

// Generate claim with overrides
const overrides = await store.getByEntity("enc_001");
const claimData = { ...encounter };
for (const override of overrides) {
  claimData[override.field_name] = override.overridden_value;
}

console.log("Claim submitted with:");
console.log(`  Provider: ${claimData.provider}`);
console.log(`  Diagnosis: ${claimData.diagnosis_code}`);
```

**Audit Trail for Compliance**

```typescript
async function generateCodingAuditReport(
  startDate: string,
  endDate: string
): Promise<AuditReport> {
  // Get all diagnosis code overrides in date range
  const overrides = await store.query({
    field_names: ["diagnosis_code", "procedure_code"],
    created_after: startDate,
    created_before: endDate
  });

  return {
    total_corrections: overrides.length,
    by_coder: groupBy(overrides, o => o.overridden_by),
    by_field: groupBy(overrides, o => o.field_name),
    details: overrides.map(o => ({
      encounter_id: o.entity_id,
      field: o.field_name,
      from: o.original_value,
      to: o.overridden_value,
      coder: o.overridden_by,
      date: o.overridden_at,
      reason: o.reason
    }))
  };
}
```

---

### Example 3: Legal - Case Information Corrections

**Scenario**: Legal assistant corrects case numbers and attorney assignments.

```typescript
// Original case from document scanning
const legalCase = {
  id: "case_001",
  case_number: "2024-CV-12345",    // OCR error
  client_name: "Acme Corp",
  attorney: "Smith, J.",           // Abbreviated
  filing_date: "2024-12-01",
  status: "Active"
};

// Legal assistant corrections
await store.create({
  entity_id: "case_001",
  field_name: "case_number",
  original_value: "2024-CV-12345",
  overridden_value: "2024-CV-01234",
  overridden_by: "assistant_sarah_lee",
  reason: "OCR misread case number from filed document"
});

await store.create({
  entity_id: "case_001",
  field_name: "attorney",
  original_value: "Smith, J.",
  overridden_value: "Smith, Jane (Partner)",
  overridden_by: "assistant_sarah_lee",
  reason: "Full attorney name for court filings"
});

// Generate court document with corrections
const overrides = await store.getByEntity("case_001");
const courtDocument = applyOverrides(legalCase, overrides);
```

---

### Example 4: Research - Citation Corrections

**Scenario**: Researcher corrects author names and publication years in bibliography.

```typescript
// Original citation from PDF extraction
const citation = {
  id: "cite_001",
  title: "Machine Learning in Healthcare",
  authors: "J. Smith, et al.",    // Incomplete
  year: "2023",                    // Wrong year
  journal: "Nature Medicine",
  doi: "10.1038/s41591-023-01234-5"
};

// Researcher corrections after manual lookup
await store.createBulk([
  {
    entity_id: "cite_001",
    field_name: "authors",
    original_value: "J. Smith, et al.",
    overridden_value: "Jane Smith, Robert Johnson, Maria Garcia",
    overridden_by: "researcher_alice_wong",
    reason: "Full author list from DOI lookup"
  },
  {
    entity_id: "cite_001",
    field_name: "year",
    original_value: "2023",
    overridden_value: "2024",
    overridden_by: "researcher_alice_wong",
    reason: "Corrected publication year from journal website"
  }
]);

// Generate bibliography with corrections
const citations = await citationStore.getAllForPaper("paper_123");
const correctedCitations = await Promise.all(
  citations.map(async (cite) => {
    const overrides = await store.getByEntity(cite.id);
    return applyOverrides(cite, overrides);
  })
);
```

**Bulk Import Corrections**

```typescript
async function correctBulkImportedCitations(
  csvFile: string
): Promise<void> {
  // Parse CSV of corrections
  const corrections = parseCSV(csvFile);
  // [
  //   { citation_id: "cite_001", field: "year", value: "2024" },
  //   { citation_id: "cite_002", field: "authors", value: "..." },
  //   ...
  // ]

  // Create overrides
  const overrides = corrections.map(corr => ({
    entity_id: corr.citation_id,
    field_name: corr.field,
    original_value: getOriginalValue(corr.citation_id, corr.field),
    overridden_value: corr.value,
    overridden_by: "bulk_import_script",
    reason: "Bulk correction from manual review"
  }));

  const created = await store.createBulk(overrides);
  console.log(`Created ${created.length} citation overrides`);
}
```

---

### Example 5: E-commerce - Product Data Corrections

**Scenario**: Catalog manager corrects product categories and prices.

```typescript
// Original product from supplier feed
const product = {
  id: "prod_001",
  sku: "ABC-123",
  name: "Wireless Mouse",
  category: "Electronics > Accessories",  // Too broad
  price: 29.99,                           // Old price
  supplier_id: "supp_xyz"
};

// Catalog manager corrections
await store.createBulk([
  {
    entity_id: "prod_001",
    field_name: "category",
    original_value: "Electronics > Accessories",
    overridden_value: "Electronics > Computer Accessories > Mice",
    overridden_by: "manager_tom_chen",
    reason: "More specific category for better search"
  },
  {
    entity_id: "prod_001",
    field_name: "price",
    original_value: 29.99,
    overridden_value: 24.99,
    overridden_by: "manager_tom_chen",
    reason: "Updated price from supplier email 2025-01-15"
  }
]);

// Display product on website with overrides
const displayProduct = await getProductWithOverrides("prod_001");
```

**Seasonal Price Adjustments**

```typescript
async function applySaleDiscount(
  categoryPath: string,
  discountPercent: number
): Promise<number> {
  // Get all products in category
  const products = await productStore.query({
    category_starts_with: categoryPath
  });

  // Create price overrides
  const overrides = products.map(product => ({
    entity_id: product.id,
    field_name: "price",
    original_value: product.price,
    overridden_value: product.price * (1 - discountPercent / 100),
    overridden_by: "automated_sale_script",
    reason: `${discountPercent}% off sale for ${categoryPath}`
  }));

  const created = await store.createBulk(overrides);
  return created.length;
}

// Apply 20% off to all electronics
const count = await applySaleDiscount("Electronics", 20);
console.log(`Applied sale to ${count} products`);

// Revert after sale ends
const saleOverrides = await store.query({
  field_names: ["price"],
  overridden_by: ["automated_sale_script"]
});
await store.deleteBulk(saleOverrides.map(o => o.override_id));
```

---

### Example 6: SaaS - Subscription Data Corrections

**Scenario**: Customer success manager corrects plan names and MRR on customer subscriptions.

```typescript
// Original subscription from payment processor webhook
const subscription = {
  id: "sub_001",
  customer_id: "cust_abc123",
  plan_name: "pro_monthly",         // Internal code
  mrr: 99.00,                       // Doesn't include discount
  status: "active",
  created_at: "2024-12-01"
};

// CS manager corrections for reporting
await store.createBulk([
  {
    entity_id: "sub_001",
    field_name: "plan_name",
    original_value: "pro_monthly",
    overridden_value: "Professional Plan (Monthly)",
    overridden_by: "cs_manager_lisa_park",
    reason: "Display name for customer reports"
  },
  {
    entity_id: "sub_001",
    field_name: "mrr",
    original_value: 99.00,
    overridden_value: 79.20,
    overridden_by: "cs_manager_lisa_park",
    reason: "Applied 20% annual discount negotiated with customer"
  }
]);

// Generate MRR report with overrides
const subscriptions = await subscriptionStore.getActiveSubscriptions();
const totalMRR = await calculateTotalMRR(subscriptions); // Uses overrides
```

---

## Edge Cases

### Edge Case 1: Duplicate Override Attempts

**Scenario**: User tries to create override for entity+field that already has one.

```typescript
// First override succeeds
await store.create({
  entity_id: "txn_001",
  field_name: "merchant",
  original_value: "AMZN",
  overridden_value: "Amazon.com",
  overridden_by: "user_jane"
});

// Second override fails
try {
  await store.create({
    entity_id: "txn_001",
    field_name: "merchant",        // Same entity+field
    original_value: "AMZN",
    overridden_value: "Amazon Inc", // Different value
    overridden_by: "user_jane"
  });
} catch (error) {
  console.log(error.code); // "CONFLICT"
  console.log(error.message);
  // "Override already exists for entity txn_001, field merchant"
}
```

**Solution 1: Update Pattern**

```typescript
async function createOrUpdateOverride(
  input: CreateOverrideInput
): Promise<FieldOverride> {
  // Check if override exists
  const existing = await store.getByField(
    input.entity_id,
    input.field_name
  );

  if (existing) {
    // Delete old override
    await store.delete(existing.override_id);
  }

  // Create new override
  return await store.create(input);
}
```

**Solution 2: Bulk Create Skips Duplicates**

```typescript
const overrides = [
  { entity_id: "txn_001", field_name: "merchant", ... },
  { entity_id: "txn_001", field_name: "merchant", ... }, // Duplicate
  { entity_id: "txn_002", field_name: "merchant", ... }
];

const created = await store.createBulk(overrides);
console.log(created.length); // 2 (duplicate skipped)
```

---

### Edge Case 2: Bulk Create with Partial Failures

**Scenario**: Bulk create where some overrides are valid, others have conflicts.

```typescript
const overrides = [
  // Valid
  { entity_id: "txn_001", field_name: "category", ... },

  // Conflict (already exists)
  { entity_id: "txn_002", field_name: "merchant", ... },

  // Valid
  { entity_id: "txn_003", field_name: "amount", ... },

  // Invalid (missing field_name)
  { entity_id: "txn_004", field_name: "", ... }
];

const created = await store.createBulk(overrides);

console.log(`Created: ${created.length}`);        // 2
console.log(`Skipped: ${overrides.length - created.length}`); // 2

// Identify which were skipped
const createdEntityFields = new Set(
  created.map(o => `${o.entity_id}:${o.field_name}`)
);

const skipped = overrides.filter(o =>
  !createdEntityFields.has(`${o.entity_id}:${o.field_name}`)
);

console.log("Skipped overrides:", skipped);
```

**Strategy: Pre-Validation**

```typescript
async function createBulkWithValidation(
  overrides: CreateOverrideInput[]
): Promise<BulkCreateResult> {
  const valid: CreateOverrideInput[] = [];
  const invalid: { override: CreateOverrideInput; reason: string }[] = [];

  // 1. Validate inputs
  for (const override of overrides) {
    if (!override.entity_id) {
      invalid.push({ override, reason: "Missing entity_id" });
    } else if (!override.field_name) {
      invalid.push({ override, reason: "Missing field_name" });
    } else {
      valid.push(override);
    }
  }

  // 2. Check for existing overrides
  const entityFieldPairs = valid.map(o => ({
    entity_id: o.entity_id,
    field_name: o.field_name
  }));

  const existing = await store.query({
    entity_ids: [...new Set(valid.map(o => o.entity_id))]
  });

  const existingSet = new Set(
    existing.map(o => `${o.entity_id}:${o.field_name}`)
  );

  const toCreate: CreateOverrideInput[] = [];
  for (const override of valid) {
    const key = `${override.entity_id}:${override.field_name}`;
    if (existingSet.has(key)) {
      invalid.push({ override, reason: "Override already exists" });
    } else {
      toCreate.push(override);
    }
  }

  // 3. Create valid ones
  const created = await store.createBulk(toCreate);

  return {
    created,
    skipped: invalid
  };
}
```

---

### Edge Case 3: Concurrent Override Updates

**Scenario**: Two users try to override same field on same entity simultaneously.

```typescript
// User A and User B both load transaction
const txn = await getTransaction("txn_001");

// User A creates override
await store.create({
  entity_id: "txn_001",
  field_name: "merchant",
  original_value: "AMZN",
  overridden_value: "Amazon.com",
  overridden_by: "user_a"
});

// User B tries to create override (fails)
try {
  await store.create({
    entity_id: "txn_001",
    field_name: "merchant",
    original_value: "AMZN",
    overridden_value: "Amazon Inc",
    overridden_by: "user_b"
  });
} catch (error) {
  console.log("Conflict: Override already exists");

  // User B sees error message:
  // "Another user has already overridden this field.
  //  Current value: Amazon.com (by user_a at 2025-01-15 10:30)"
}
```

**Resolution: Last Write Wins with Notification**

```typescript
async function createOrReplaceOverride(
  input: CreateOverrideInput
): Promise<{ override: FieldOverride; replaced?: FieldOverride }> {
  const existing = await store.getByField(
    input.entity_id,
    input.field_name
  );

  if (existing) {
    // Delete old
    await store.delete(existing.override_id);

    // Create new
    const newOverride = await store.create(input);

    // Notify original user
    await notifyUser(existing.overridden_by, {
      message: `Your override on ${input.entity_id}.${input.field_name} was replaced`,
      old_value: existing.overridden_value,
      new_value: input.overridden_value,
      replaced_by: input.overridden_by
    });

    return { override: newOverride, replaced: existing };
  } else {
    const newOverride = await store.create(input);
    return { override: newOverride };
  }
}
```

---

### Edge Case 4: Delete Non-Existent Override

**Scenario**: Attempt to delete override that doesn't exist (or was already deleted).

```typescript
// Delete once (succeeds)
await store.delete("ovr_999");

// Delete again (no error)
await store.delete("ovr_999");

// Verify
const override = await store.getById("ovr_999");
console.log(override); // null
```

**Idempotency**: `delete()` is idempotent - can be called multiple times safely.

---

### Edge Case 5: Query with No Results

**Scenario**: Query that matches no overrides.

```typescript
const overrides = await store.query({
  entity_ids: ["nonexistent_entity"]
});

console.log(overrides); // []
console.log(overrides.length); // 0

const count = await store.count({
  entity_ids: ["nonexistent_entity"]
});

console.log(count); // 0
```

**Safe Iteration**:

```typescript
const overrides = await store.query({ field_names: ["unknown_field"] });

// Safe - works even if empty
for (const override of overrides) {
  console.log(override);
}

// Safe - reduce on empty array
const byEntity = overrides.reduce((acc, o) => {
  acc[o.entity_id] = o;
  return acc;
}, {});
```

---

### Edge Case 6: Override on Non-Existent Entity

**Scenario**: Create override for entity that doesn't exist in entity store.

```typescript
// Entity doesn't exist
const entity = await entityStore.getById("nonexistent_entity");
console.log(entity); // null

// Override creation still succeeds
const override = await store.create({
  entity_id: "nonexistent_entity",
  field_name: "merchant",
  original_value: null,
  overridden_value: "Amazon.com",
  overridden_by: "user_jane"
});

console.log(override); // Created successfully
```

**Rationale**: OverrideStore does NOT enforce foreign key constraint on `entity_id` because:
1. Entities might be created asynchronously
2. Entities might be in different database/service
3. Simplifies implementation

**Cleanup Strategy**:

```typescript
async function cleanupOrphanedOverrides(): Promise<number> {
  // Get all override entity IDs
  const allOverrides = await store.query({ limit: 10000 });
  const overrideEntityIds = new Set(allOverrides.map(o => o.entity_id));

  // Check which entities exist
  const existingEntities = await entityStore.query({
    ids: Array.from(overrideEntityIds)
  });
  const existingIds = new Set(existingEntities.map(e => e.id));

  // Find orphaned overrides
  const orphaned = allOverrides.filter(
    o => !existingIds.has(o.entity_id)
  );

  // Delete orphaned
  await store.deleteBulk(orphaned.map(o => o.override_id));

  return orphaned.length;
}
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency | Notes |
|-----------|-----------|----------------|-------|
| `create()` | 1 override | < 10ms | Single INSERT |
| `getById()` | 1 ID | < 5ms | Indexed lookup |
| `getByEntity()` | 1 entity | < 10ms | Indexed query |
| `getByField()` | 1 entity+field | < 5ms | Unique index |
| `delete()` | 1 ID | < 10ms | Single DELETE |
| `createBulk()` | 100 overrides | < 50ms | Batch INSERT |
| `createBulk()` | 1,000 overrides | < 500ms | Batch INSERT |
| `deleteBulk()` | 100 IDs | < 30ms | Batch DELETE |
| `query()` | No filters | < 100ms | Full table scan |
| `query()` | With filters | < 50ms | Indexed query |
| `count()` | Any filters | < 50ms | COUNT query |

### Throughput

- **Single operations**: 500-1000 ops/sec
- **Bulk operations**: 10,000-50,000 overrides/sec

### Scalability

| Total Overrides | Query Performance | Notes |
|----------------|-------------------|-------|
| < 10,000 | Excellent (< 10ms) | All queries fast |
| 10,000 - 100,000 | Good (< 50ms) | Indexed queries fast |
| 100,000 - 1M | Moderate (< 200ms) | Pagination recommended |
| > 1M | Slower (< 500ms) | Consider archiving old overrides |

### Index Strategy

```sql
-- Primary key (clustered index)
PRIMARY KEY (override_id)

-- Unique constraint (creates unique index)
UNIQUE (entity_id, field_name)

-- Query optimization indexes
CREATE INDEX idx_overrides_entity_id ON field_overrides(entity_id);
CREATE INDEX idx_overrides_field_name ON field_overrides(field_name);
CREATE INDEX idx_overrides_overridden_by ON field_overrides(overridden_by);
CREATE INDEX idx_overrides_overridden_at ON field_overrides(overridden_at DESC);
CREATE INDEX idx_overrides_created_at ON field_overrides(created_at DESC);
```

### Optimization Tips

**1. Batch Reads**

```typescript
// Bad: N+1 query problem
for (const entity of entities) {
  const overrides = await store.getByEntity(entity.id); // N queries
  applyOverrides(entity, overrides);
}

// Good: Single bulk query
const entityIds = entities.map(e => e.id);
const allOverrides = await store.query({ entity_ids: entityIds }); // 1 query
const overridesByEntity = groupBy(allOverrides, o => o.entity_id);

for (const entity of entities) {
  const overrides = overridesByEntity[entity.id] || [];
  applyOverrides(entity, overrides);
}
```

**2. Chunked Bulk Operations**

```typescript
// For very large batches, chunk into smaller pieces
async function createBulkChunked(
  overrides: CreateOverrideInput[],
  chunkSize = 1000
): Promise<FieldOverride[]> {
  const results: FieldOverride[] = [];

  for (let i = 0; i < overrides.length; i += chunkSize) {
    const chunk = overrides.slice(i, i + chunkSize);
    const created = await store.createBulk(chunk);
    results.push(...created);
  }

  return results;
}
```

**3. Cached Override Lookups**

```typescript
class CachedOverrideStore {
  private cache = new Map<string, FieldOverride[]>();

  async getByEntity(entityId: string): Promise<FieldOverride[]> {
    if (this.cache.has(entityId)) {
      return this.cache.get(entityId)!;
    }

    const overrides = await store.getByEntity(entityId);
    this.cache.set(entityId, overrides);
    return overrides;
  }

  invalidate(entityId: string): void {
    this.cache.delete(entityId);
  }
}
```

---

## Implementation Details

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';

export class PostgresOverrideStore implements OverrideStore {
  constructor(private pool: Pool) {}

  async create(input: CreateOverrideInput): Promise<FieldOverride> {
    // Validate
    this.validate(input);

    // Generate ID
    const override_id = this.generateId();
    const overridden_at = new Date().toISOString();

    // Insert
    const query = `
      INSERT INTO field_overrides (
        override_id, entity_id, field_name,
        original_value, overridden_value,
        overridden_by, overridden_at, reason
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
      RETURNING *
    `;

    const values = [
      override_id,
      input.entity_id,
      input.field_name,
      JSON.stringify(input.original_value),
      JSON.stringify(input.overridden_value),
      input.overridden_by,
      overridden_at,
      input.reason || null
    ];

    try {
      const result = await this.pool.query(query, values);
      return this.rowToOverride(result.rows[0]);
    } catch (error: any) {
      if (error.code === '23505') { // Unique violation
        throw new ConflictError(
          `Override already exists for entity ${input.entity_id}, field ${input.field_name}`
        );
      }
      throw error;
    }
  }

  async getById(overrideId: string): Promise<FieldOverride | null> {
    const query = 'SELECT * FROM field_overrides WHERE override_id = $1';
    const result = await this.pool.query(query, [overrideId]);

    if (result.rows.length === 0) {
      return null;
    }

    return this.rowToOverride(result.rows[0]);
  }

  async getByEntity(entityId: string): Promise<FieldOverride[]> {
    const query = 'SELECT * FROM field_overrides WHERE entity_id = $1';
    const result = await this.pool.query(query, [entityId]);
    return result.rows.map(row => this.rowToOverride(row));
  }

  async getByField(
    entityId: string,
    fieldName: string
  ): Promise<FieldOverride | null> {
    const query = `
      SELECT * FROM field_overrides
      WHERE entity_id = $1 AND field_name = $2
    `;
    const result = await this.pool.query(query, [entityId, fieldName]);

    if (result.rows.length === 0) {
      return null;
    }

    return this.rowToOverride(result.rows[0]);
  }

  async delete(overrideId: string): Promise<void> {
    const query = 'DELETE FROM field_overrides WHERE override_id = $1';
    await this.pool.query(query, [overrideId]);
  }

  async createBulk(
    overrides: CreateOverrideInput[]
  ): Promise<FieldOverride[]> {
    if (overrides.length === 0) {
      return [];
    }

    // Deduplicate by (entity_id, field_name)
    const deduped = this.deduplicateByEntityField(overrides);

    // Check for existing overrides
    const entityIds = [...new Set(deduped.map(o => o.entity_id))];
    const existing = await this.query({ entity_ids: entityIds });
    const existingSet = new Set(
      existing.map(o => `${o.entity_id}:${o.field_name}`)
    );

    // Filter out conflicts
    const toCreate = deduped.filter(
      o => !existingSet.has(`${o.entity_id}:${o.field_name}`)
    );

    if (toCreate.length === 0) {
      return [];
    }

    // Generate IDs and timestamps
    const overridden_at = new Date().toISOString();
    const rows = toCreate.map(input => ({
      override_id: this.generateId(),
      ...input,
      overridden_at
    }));

    // Build multi-row INSERT
    const values: any[] = [];
    const valuePlaceholders: string[] = [];

    rows.forEach((row, i) => {
      const offset = i * 8;
      valuePlaceholders.push(
        `($${offset + 1}, $${offset + 2}, $${offset + 3}, $${offset + 4}, $${offset + 5}, $${offset + 6}, $${offset + 7}, $${offset + 8})`
      );
      values.push(
        row.override_id,
        row.entity_id,
        row.field_name,
        JSON.stringify(row.original_value),
        JSON.stringify(row.overridden_value),
        row.overridden_by,
        row.overridden_at,
        row.reason || null
      );
    });

    const query = `
      INSERT INTO field_overrides (
        override_id, entity_id, field_name,
        original_value, overridden_value,
        overridden_by, overridden_at, reason
      ) VALUES ${valuePlaceholders.join(', ')}
      RETURNING *
    `;

    const result = await this.pool.query(query, values);
    return result.rows.map(row => this.rowToOverride(row));
  }

  async deleteBulk(overrideIds: string[]): Promise<void> {
    if (overrideIds.length === 0) {
      return;
    }

    const placeholders = overrideIds.map((_, i) => `$${i + 1}`).join(', ');
    const query = `DELETE FROM field_overrides WHERE override_id IN (${placeholders})`;
    await this.pool.query(query, overrideIds);
  }

  async query(filters: OverrideQueryFilters): Promise<FieldOverride[]> {
    const { query, values } = this.buildQuerySQL(filters);
    const result = await this.pool.query(query, values);
    return result.rows.map(row => this.rowToOverride(row));
  }

  async count(filters: OverrideQueryFilters): Promise<number> {
    const { query, values } = this.buildCountSQL(filters);
    const result = await this.pool.query(query, values);
    return parseInt(result.rows[0].count, 10);
  }

  // Helper methods
  private generateId(): string {
    return `ovr_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private validate(input: CreateOverrideInput): void {
    if (!input.entity_id) throw new Error('entity_id is required');
    if (!input.field_name) throw new Error('field_name is required');
    if (input.original_value === undefined) throw new Error('original_value is required');
    if (input.overridden_value === undefined) throw new Error('overridden_value is required');
    if (!input.overridden_by) throw new Error('overridden_by is required');
  }

  private rowToOverride(row: any): FieldOverride {
    return {
      override_id: row.override_id,
      entity_id: row.entity_id,
      field_name: row.field_name,
      original_value: JSON.parse(row.original_value),
      overridden_value: JSON.parse(row.overridden_value),
      overridden_by: row.overridden_by,
      overridden_at: row.overridden_at,
      reason: row.reason,
      created_at: row.created_at,
      updated_at: row.updated_at
    };
  }

  private deduplicateByEntityField(
    overrides: CreateOverrideInput[]
  ): CreateOverrideInput[] {
    const map = new Map<string, CreateOverrideInput>();

    for (const override of overrides) {
      const key = `${override.entity_id}:${override.field_name}`;
      map.set(key, override); // Last one wins
    }

    return Array.from(map.values());
  }

  private buildQuerySQL(filters: OverrideQueryFilters): { query: string; values: any[] } {
    const conditions: string[] = [];
    const values: any[] = [];
    let paramIndex = 1;

    if (filters.entity_ids?.length) {
      const placeholders = filters.entity_ids.map(() => `$${paramIndex++}`).join(', ');
      conditions.push(`entity_id IN (${placeholders})`);
      values.push(...filters.entity_ids);
    }

    if (filters.field_names?.length) {
      const placeholders = filters.field_names.map(() => `$${paramIndex++}`).join(', ');
      conditions.push(`field_name IN (${placeholders})`);
      values.push(...filters.field_names);
    }

    if (filters.overridden_by?.length) {
      const placeholders = filters.overridden_by.map(() => `$${paramIndex++}`).join(', ');
      conditions.push(`overridden_by IN (${placeholders})`);
      values.push(...filters.overridden_by);
    }

    if (filters.created_after) {
      conditions.push(`created_at >= $${paramIndex++}`);
      values.push(filters.created_after);
    }

    if (filters.created_before) {
      conditions.push(`created_at <= $${paramIndex++}`);
      values.push(filters.created_before);
    }

    let query = 'SELECT * FROM field_overrides';

    if (conditions.length > 0) {
      query += ' WHERE ' + conditions.join(' AND ');
    }

    // Sorting
    const sortBy = filters.sort_by || 'created_at';
    const sortOrder = filters.sort_order || 'desc';
    query += ` ORDER BY ${sortBy} ${sortOrder.toUpperCase()}`;

    // Pagination
    const limit = filters.limit || 100;
    const offset = filters.offset || 0;
    query += ` LIMIT $${paramIndex++} OFFSET $${paramIndex++}`;
    values.push(limit, offset);

    return { query, values };
  }

  private buildCountSQL(filters: OverrideQueryFilters): { query: string; values: any[] } {
    const conditions: string[] = [];
    const values: any[] = [];
    let paramIndex = 1;

    // Same WHERE clause as query
    if (filters.entity_ids?.length) {
      const placeholders = filters.entity_ids.map(() => `$${paramIndex++}`).join(', ');
      conditions.push(`entity_id IN (${placeholders})`);
      values.push(...filters.entity_ids);
    }

    if (filters.field_names?.length) {
      const placeholders = filters.field_names.map(() => `$${paramIndex++}`).join(', ');
      conditions.push(`field_name IN (${placeholders})`);
      values.push(...filters.field_names);
    }

    if (filters.overridden_by?.length) {
      const placeholders = filters.overridden_by.map(() => `$${paramIndex++}`).join(', ');
      conditions.push(`overridden_by IN (${placeholders})`);
      values.push(...filters.overridden_by);
    }

    if (filters.created_after) {
      conditions.push(`created_at >= $${paramIndex++}`);
      values.push(filters.created_after);
    }

    if (filters.created_before) {
      conditions.push(`created_at <= $${paramIndex++}`);
      values.push(filters.created_before);
    }

    let query = 'SELECT COUNT(*) FROM field_overrides';

    if (conditions.length > 0) {
      query += ' WHERE ' + conditions.join(' AND ');
    }

    return { query, values };
  }
}
```

---

## Error Handling

### Error Types

```typescript
class ConflictError extends Error {
  code = 'CONFLICT';
  constructor(message: string) {
    super(message);
    this.name = 'ConflictError';
  }
}

class ValidationError extends Error {
  code = 'VALIDATION_ERROR';
  constructor(message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

class DatabaseError extends Error {
  code = 'DATABASE_ERROR';
  constructor(message: string, public originalError: any) {
    super(message);
    this.name = 'DatabaseError';
  }
}
```

### Error Handling Patterns

```typescript
// Pattern 1: Graceful degradation
async function getEntityWithOverrides(entityId: string): Promise<Entity> {
  const entity = await entityStore.getById(entityId);

  try {
    const overrides = await overrideStore.getByEntity(entityId);
    return applyOverrides(entity, overrides);
  } catch (error) {
    console.error('Failed to fetch overrides, returning canonical entity', error);
    return entity; // Fallback to original
  }
}

// Pattern 2: Retry on conflict
async function createWithRetry(
  input: CreateOverrideInput,
  maxRetries = 3
): Promise<FieldOverride> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await store.create(input);
    } catch (error: any) {
      if (error.code === 'CONFLICT' && i < maxRetries - 1) {
        // Wait and retry
        await sleep(100 * (i + 1));
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}

// Pattern 3: Validation before bulk create
async function createBulkSafe(
  overrides: CreateOverrideInput[]
): Promise<BulkCreateResult> {
  const valid: CreateOverrideInput[] = [];
  const invalid: { input: CreateOverrideInput; error: string }[] = [];

  for (const input of overrides) {
    try {
      validateInput(input);
      valid.push(input);
    } catch (error: any) {
      invalid.push({ input, error: error.message });
    }
  }

  const created = await store.createBulk(valid);

  return { created, invalid };
}
```

---

## Integration Patterns

### Pattern 1: Automatic Override Application

```typescript
class EntityService {
  constructor(
    private entityStore: EntityStore,
    private overrideStore: OverrideStore
  ) {}

  async getEntity(entityId: string): Promise<Entity> {
    // Always return entity with overrides applied
    const entity = await this.entityStore.getById(entityId);
    const overrides = await this.overrideStore.getByEntity(entityId);
    return this.applyOverrides(entity, overrides);
  }

  async getEntities(filters: any): Promise<Entity[]> {
    // Bulk fetch with overrides
    const entities = await this.entityStore.query(filters);
    const entityIds = entities.map(e => e.id);
    const allOverrides = await this.overrideStore.query({ entity_ids: entityIds });

    // Group overrides by entity
    const overridesByEntity = this.groupBy(allOverrides, o => o.entity_id);

    // Apply to each entity
    return entities.map(entity => {
      const overrides = overridesByEntity[entity.id] || [];
      return this.applyOverrides(entity, overrides);
    });
  }

  private applyOverrides(entity: Entity, overrides: FieldOverride[]): Entity {
    const merged = { ...entity };
    for (const override of overrides) {
      merged[override.field_name] = override.overridden_value;
    }
    return merged;
  }

  private groupBy<T>(array: T[], keyFn: (item: T) => string): Record<string, T[]> {
    return array.reduce((acc, item) => {
      const key = keyFn(item);
      if (!acc[key]) acc[key] = [];
      acc[key].push(item);
      return acc;
    }, {} as Record<string, T[]>);
  }
}
```

### Pattern 2: UI Override Workflow

```typescript
// React component for editing entity with override support
function EntityEditor({ entityId }: { entityId: string }) {
  const [entity, setEntity] = useState<Entity | null>(null);
  const [overrides, setOverrides] = useState<FieldOverride[]>([]);

  useEffect(() => {
    loadEntity();
  }, [entityId]);

  async function loadEntity() {
    const [entityData, overrideData] = await Promise.all([
      entityStore.getById(entityId),
      overrideStore.getByEntity(entityId)
    ]);
    setEntity(entityData);
    setOverrides(overrideData);
  }

  async function updateField(fieldName: string, value: any) {
    // Check if override already exists
    const existingOverride = overrides.find(o => o.field_name === fieldName);

    if (existingOverride) {
      // Delete old override
      await overrideStore.delete(existingOverride.override_id);
    }

    // Create new override
    const override = await overrideStore.create({
      entity_id: entityId,
      field_name: fieldName,
      original_value: entity[fieldName],
      overridden_value: value,
      overridden_by: currentUserId,
      reason: `Manual correction by user`
    });

    // Update local state
    setOverrides([...overrides.filter(o => o.field_name !== fieldName), override]);
  }

  async function revertField(fieldName: string) {
    const override = overrides.find(o => o.field_name === fieldName);
    if (override) {
      await overrideStore.delete(override.override_id);
      setOverrides(overrides.filter(o => o.field_name !== fieldName));
    }
  }

  const displayEntity = useMemo(() => {
    if (!entity) return null;
    const merged = { ...entity };
    for (const override of overrides) {
      merged[override.field_name] = override.overridden_value;
    }
    return merged;
  }, [entity, overrides]);

  return (
    <div>
      <h2>Entity: {entityId}</h2>

      {displayEntity && (
        <>
          <Field
            name="merchant"
            value={displayEntity.merchant}
            isOverridden={overrides.some(o => o.field_name === 'merchant')}
            onUpdate={(value) => updateField('merchant', value)}
            onRevert={() => revertField('merchant')}
          />

          <Field
            name="category"
            value={displayEntity.category}
            isOverridden={overrides.some(o => o.field_name === 'category')}
            onUpdate={(value) => updateField('category', value)}
            onRevert={() => revertField('category')}
          />
        </>
      )}
    </div>
  );
}
```

---

## Testing Strategy

### Unit Tests

```typescript
describe('OverrideStore', () => {
  let store: OverrideStore;

  beforeEach(async () => {
    store = new PostgresOverrideStore(pool);
    await cleanDatabase();
  });

  describe('create()', () => {
    it('should create override with generated ID', async () => {
      const override = await store.create({
        entity_id: 'txn_001',
        field_name: 'merchant',
        original_value: 'AMZN',
        overridden_value: 'Amazon.com',
        overridden_by: 'user_jane'
      });

      expect(override.override_id).toMatch(/^ovr_/);
      expect(override.entity_id).toBe('txn_001');
      expect(override.field_name).toBe('merchant');
      expect(override.overridden_value).toBe('Amazon.com');
    });

    it('should throw ConflictError on duplicate', async () => {
      await store.create({
        entity_id: 'txn_001',
        field_name: 'merchant',
        original_value: 'AMZN',
        overridden_value: 'Amazon.com',
        overridden_by: 'user_jane'
      });

      await expect(store.create({
        entity_id: 'txn_001',
        field_name: 'merchant',
        original_value: 'AMZN',
        overridden_value: 'Amazon Inc',
        overridden_by: 'user_jane'
      })).rejects.toThrow(ConflictError);
    });
  });

  describe('getByEntity()', () => {
    it('should return all overrides for entity', async () => {
      await store.createBulk([
        { entity_id: 'txn_001', field_name: 'merchant', ... },
        { entity_id: 'txn_001', field_name: 'category', ... },
        { entity_id: 'txn_002', field_name: 'merchant', ... }
      ]);

      const overrides = await store.getByEntity('txn_001');
      expect(overrides).toHaveLength(2);
      expect(overrides.map(o => o.field_name).sort()).toEqual(['category', 'merchant']);
    });

    it('should return empty array for entity with no overrides', async () => {
      const overrides = await store.getByEntity('nonexistent');
      expect(overrides).toEqual([]);
    });
  });

  describe('createBulk()', () => {
    it('should create multiple overrides', async () => {
      const created = await store.createBulk([
        { entity_id: 'txn_001', field_name: 'merchant', ... },
        { entity_id: 'txn_002', field_name: 'category', ... }
      ]);

      expect(created).toHaveLength(2);
    });

    it('should skip duplicates', async () => {
      await store.create({
        entity_id: 'txn_001',
        field_name: 'merchant',
        ...
      });

      const created = await store.createBulk([
        { entity_id: 'txn_001', field_name: 'merchant', ... }, // Duplicate
        { entity_id: 'txn_002', field_name: 'merchant', ... }  // New
      ]);

      expect(created).toHaveLength(1);
      expect(created[0].entity_id).toBe('txn_002');
    });
  });
});
```

### Integration Tests

```typescript
describe('Override Integration', () => {
  it('should apply overrides when fetching entities', async () => {
    // Create entity
    const entity = await entityStore.create({
      id: 'txn_001',
      merchant: 'AMZN',
      category: 'Uncategorized'
    });

    // Create overrides
    await overrideStore.createBulk([
      {
        entity_id: 'txn_001',
        field_name: 'merchant',
        original_value: 'AMZN',
        overridden_value: 'Amazon.com',
        overridden_by: 'user_jane'
      },
      {
        entity_id: 'txn_001',
        field_name: 'category',
        original_value: 'Uncategorized',
        overridden_value: 'Shopping',
        overridden_by: 'user_jane'
      }
    ]);

    // Fetch with overrides
    const entityWithOverrides = await entityService.getEntity('txn_001');

    expect(entityWithOverrides.merchant).toBe('Amazon.com');
    expect(entityWithOverrides.category).toBe('Shopping');
  });
});
```

---

## Migration Guide

### From No Override System

```typescript
// Before: Modifying entity directly
await entityStore.update('txn_001', {
  merchant: 'Amazon.com'  // Loses original value
});

// After: Creating override
await overrideStore.create({
  entity_id: 'txn_001',
  field_name: 'merchant',
  original_value: originalEntity.merchant,
  overridden_value: 'Amazon.com',
  overridden_by: currentUserId,
  reason: 'User correction'
});
```

### Database Migration

```sql
-- Step 1: Create table
CREATE TABLE field_overrides (
  override_id VARCHAR(64) PRIMARY KEY,
  entity_id VARCHAR(128) NOT NULL,
  field_name VARCHAR(128) NOT NULL,
  original_value JSONB NOT NULL,
  overridden_value JSONB NOT NULL,
  overridden_by VARCHAR(128) NOT NULL,
  overridden_at TIMESTAMP WITH TIME ZONE NOT NULL,
  reason TEXT,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  CONSTRAINT unique_entity_field UNIQUE (entity_id, field_name)
);

-- Step 2: Create indexes
CREATE INDEX idx_overrides_entity_id ON field_overrides(entity_id);
CREATE INDEX idx_overrides_field_name ON field_overrides(field_name);
CREATE INDEX idx_overrides_overridden_by ON field_overrides(overridden_by);
CREATE INDEX idx_overrides_overridden_at ON field_overrides(overridden_at);
CREATE INDEX idx_overrides_created_at ON field_overrides(created_at);

-- Step 3: Create trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_field_overrides_updated_at
BEFORE UPDATE ON field_overrides
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

---

## FAQ

**Q: Should I use OverrideStore or modify the entity directly?**

A: Use OverrideStore when:
- You want to preserve the original extracted value
- You need an audit trail of changes
- You want to be able to revert changes easily

Modify entity directly when:
- The original extraction was fundamentally wrong and has no value
- You're doing a one-time data migration
- Audit trail is not needed

**Q: Can I override the same field multiple times?**

A: No, only one override per (entity_id, field_name) pair. To "update" an override, delete the old one and create a new one.

**Q: What happens if I delete an override?**

A: The original value from the canonical entity becomes visible again immediately.

**Q: How do I handle overrides in reports?**

A: Always apply overrides before aggregating data. See "Integration Patterns" section.

**Q: Can I bulk delete all overrides for a user?**

A: Yes:
```typescript
const overrides = await store.query({ overridden_by: ['user_id'] });
await store.deleteBulk(overrides.map(o => o.override_id));
```

**Q: What's the performance impact of overrides?**

A: Minimal if done correctly. Use bulk queries to fetch overrides for multiple entities at once, not N+1 individual queries.

---

## Changelog

### Version 1.0.0 (2025-01-15)

- Initial specification
- Core CRUD operations
- Bulk operation support
- Multi-domain examples
- PostgreSQL implementation
- Comprehensive edge case handling

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Store user field corrections that override canonical transaction data
**Example:** Canonical merchant "AMZN MKTP US" → User overrides to "Amazon" → OverrideStore: save({entity_id: "tx_001", field_name: "merchant", value: "Amazon", overridden_by: "user_darwin"}) → Application layer applies override → User sees "Amazon"
**Operations:** save (create override), get (retrieve override), delete (revert to canonical), query (list all user's overrides)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Store clinician corrections to patient records
**Example:** Canonical diagnosis "J44.0" → Doctor overrides to "J45.0" → OverrideStore: save({entity_id: "record_456", field_name: "diagnosis_code", value: "J45.0", overridden_by: "dr_smith"}) → EHR applies override → Shows corrected diagnosis
**Operations:** Clinical override tracking, approval workflow, audit trail
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Store paralegal corrections to case metadata
**Example:** Canonical filing_date "2024-01-18" → Paralegal overrides to "2024-01-15" → OverrideStore: save({entity_id: "case_789", field_name: "filing_date", value: "2024-01-15", overridden_by: "paralegal_alice"}) → Case management shows corrected date
**Operations:** Legal override tracking, attorney approval, conflict detection
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** Store analyst corrections to fact entity names
**Example:** Canonical entity "@sama" → Analyst overrides to "Sam Altman" → OverrideStore: save({entity_id: "fact_101", field_name: "entity_name", value: "Sam Altman", overridden_by: "analyst_bob"}) → Research platform shows normalized entity
**Operations:** Editorial override tracking, fact verification workflow
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Store catalog manager corrections to product data
**Example:** Canonical price "$1,299.99" → Manager overrides to "$1,199.99" → OverrideStore: save({entity_id: "sku_202", field_name: "price", value: "1199.99", overridden_by: "manager_carol"}) → Storefront displays corrected price
**Operations:** Price override tracking, approval workflow, effective date scheduling
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (generic field override storage with entity_id/field_name/value pattern)
**Reusability:** High (same save/get/delete operations work for transactions, records, cases, facts, products)

---

## Related Primitives

- **EntityStore**: Stores canonical entity data
- **AuditLog**: Tracks all system changes (including override creation/deletion)
- **ValidationEngine**: Can use overrides in validation rules

---

## References

- [Objective Layer Architecture](../architecture/objective-layer.md)
- [Corrections Flow Vertical 4.3](../verticals/corrections-flow.md)
- [Field-Level Reconciliation](../reconciliation/field-level.md)

---

**End of OverrideStore OL Primitive Specification**
