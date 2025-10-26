# MigrationEngine OL Primitive

**Domain:** Schema Registry (Vertical 5.2)
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
8. [Migration Planning](#migration-planning)
9. [Execution Strategy](#execution-strategy)
10. [Rollback Mechanism](#rollback-mechanism)
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

The **MigrationEngine** generates and executes data migrations to transform data from one schema version to another. It provides automatic migration plan generation, batch processing, progress tracking, and automatic rollback on failure.

### Key Capabilities

- **Automatic Migration Planning**: Generate migration plans from schema diffs
- **Batch Processing**: Process data in configurable batches (default: 10K records/batch)
- **Progress Tracking**: Real-time progress monitoring with ETA calculation
- **Automatic Rollback**: Rollback changes if migration fails mid-execution
- **Backup Before Migration**: Create backup before executing migration
- **Duration Estimation**: Estimate migration duration based on data volume
- **Dry Run Mode**: Validate migration without executing changes
- **Idempotency**: Safe to re-run failed migrations

### Design Philosophy

The MigrationEngine follows four core principles:

1. **Safety First**: Never lose data - backup before migration, rollback on failure
2. **Predictability**: Dry run and duration estimation before execution
3. **Observability**: Real-time progress tracking and detailed logging
4. **Resilience**: Handle failures gracefully with automatic rollback

---

## Purpose & Scope

### Problem Statement

In schema evolution, data migration poses critical challenges:

- **Manual Migration Scripts**: Error-prone, time-consuming manual SQL/code
- **No Rollback**: Failed migrations leave data in inconsistent state
- **No Progress Tracking**: Long migrations with no visibility into progress
- **Data Loss Risk**: Migrations can delete data without backup
- **No Dry Run**: Can't validate migration before execution
- **Performance Issues**: Migrations lock tables, causing downtime

Traditional approaches have significant limitations:

**Approach 1: Manual SQL Scripts**
- ❌ Time-consuming to write
- ❌ Error-prone (typos, logic errors)
- ❌ No automatic rollback
- ❌ No progress tracking

**Approach 2: ORM Migrations (Alembic, Flyway)**
- ❌ Database-specific
- ❌ Limited schema diff detection
- ❌ No automatic plan generation
- ❌ Manual rollback scripts required

**Approach 3: MigrationEngine**
- ✅ Automatic migration plan generation
- ✅ Batch processing for large datasets
- ✅ Real-time progress tracking
- ✅ Automatic rollback on failure
- ✅ Backup before execution
- ✅ Dry run validation

### Solution

The MigrationEngine implements **automated data migration** with:

1. **Plan Generation**: Analyze schema diff and generate migration steps
2. **Batch Execution**: Process data in batches to avoid table locks
3. **Progress Tracking**: Real-time progress with ETA calculation
4. **Backup & Rollback**: Automatic backup before migration, rollback on failure
5. **Dry Run**: Validate migration plan without executing changes

---

## Multi-Domain Applicability

The MigrationEngine is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Migrate bank transaction data when schema changes (e.g., add `merchant_category_code` field).

**Migration Scenarios**:
- Add optional field with default value
- Change field type (string → number)
- Rename field with data transformation
- Remove deprecated field

**Example**:
```typescript
const engine = new MigrationEngine();

// Scenario: Add merchant_category_code field to 1M transactions
const plan = await engine.generatePlan({
  schema_id: "bank_transaction",
  from_version: "1.0.0",
  to_version: "1.1.0",
  table_name: "transactions",
  total_records: 1000000
});

console.log(plan);
// {
//   steps: [
//     {
//       type: "ADD_FIELD",
//       field: "merchant_category_code",
//       default_value: null,
//       sql: "ALTER TABLE transactions ADD COLUMN merchant_category_code VARCHAR(4)"
//     }
//   ],
//   estimated_duration_ms: 120000, // 2 minutes
//   requires_data_migration: false,
//   breaking: false
// }

// Execute migration with progress tracking
const result = await engine.execute(plan, {
  batch_size: 10000,
  on_progress: (progress) => {
    console.log(`Progress: ${progress.percent}% - ${progress.records_processed}/${progress.total_records}`);
  }
});

console.log(`Migration completed in ${result.duration_ms}ms`);
```

**Compliance**: SOX requires migration audit trail.

---

### 2. Healthcare

**Use Case**: Migrate patient records from FHIR R3 to R4 schema.

**Migration Scenarios**:
- Rename field: `deceasedBoolean` → `deceased` (with type change)
- Add required field: `gender` (backfill from existing data)
- Remove deprecated field: `animal` (FHIR R3 → R4)

**Example**:
```typescript
// Scenario: Migrate 500K patient records from FHIR R3 to R4
const plan = await engine.generatePlan({
  schema_id: "fhir_patient",
  from_version: "3.0.0",
  to_version: "4.0.0",
  table_name: "patients",
  total_records: 500000
});

console.log(plan);
// {
//   steps: [
//     {
//       type: "RENAME_FIELD",
//       from_field: "deceasedBoolean",
//       to_field: "deceased",
//       transform: (value) => ({ boolean: value })
//     },
//     {
//       type: "ADD_REQUIRED_FIELD",
//       field: "gender",
//       backfill: (record) => record.administrativeGender || "unknown"
//     },
//     {
//       type: "REMOVE_FIELD",
//       field: "animal"
//     }
//   ],
//   estimated_duration_ms: 600000, // 10 minutes
//   requires_data_migration: true,
//   breaking: true
// }

// Dry run first to validate
const dryRunResult = await engine.dryRun(plan);
console.log(`Dry run successful: ${dryRunResult.validation_errors.length} errors`);

// Execute if dry run passes
if (dryRunResult.validation_errors.length === 0) {
  const result = await engine.execute(plan);
  console.log(`Migrated ${result.records_migrated} patient records`);
}
```

**Compliance**: HIPAA requires data migration audit trail.

---

### 3. Legal

**Use Case**: Migrate court filing documents when schema changes.

**Migration Scenarios**:
- Add new enum value to `document_type` field
- Change `case_number` format with regex transformation
- Add `jurisdiction` field with backfill logic

**Example**:
```typescript
// Scenario: Add jurisdiction field to 100K court filings
const plan = await engine.generatePlan({
  schema_id: "court_filing",
  from_version: "1.0.0",
  to_version: "1.1.0",
  table_name: "court_filings",
  total_records: 100000
});

console.log(plan);
// {
//   steps: [
//     {
//       type: "ADD_FIELD",
//       field: "jurisdiction",
//       backfill: (record) => inferJurisdiction(record.case_number)
//     }
//   ],
//   estimated_duration_ms: 60000, // 1 minute
//   requires_data_migration: true,
//   breaking: false
// }

// Estimate duration before execution
const estimate = await engine.estimateDuration(plan);
console.log(`Estimated duration: ${estimate.duration_ms}ms (${estimate.batches} batches)`);

// Execute with backup
const result = await engine.execute(plan, {
  create_backup: true,
  backup_table: "court_filings_backup_20250101"
});
```

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Migrate founder fact metadata when schema evolves in RSRCH utilitario research system.

**Migration Scenarios**:
- Add `confidence` field for fact credibility
- Change `subject_entity` from string to structured object
- Add `investment_amount` validation constraint for investment facts

**Example**:
```typescript
// Scenario: Migrate subject_entity from string to object for 1M facts
const plan = await engine.generatePlan({
  schema_id: "founder_fact",
  from_version: "1.0.0",
  to_version: "2.0.0",
  table_name: "facts",
  total_records: 1000000
});

console.log(plan);
// {
//   steps: [
//     {
//       type: "TRANSFORM_FIELD",
//       field: "subject_entity",
//       transform: "string_to_structured_entity",  // Lookup entity_id from knowledge base
//       default_value: null
//     }
//   ],
//   estimated_duration_ms: 300000, // 5 minutes (requires entity resolution lookups)
//   requires_data_migration: true,
//   breaking: true
// }

// Execute with progress tracking
const result = await engine.execute(plan, {
  batch_size: 10000,  // Smaller batches due to entity lookups
  on_progress: (progress) => {
    console.log(`ETA: ${progress.eta_ms}ms - ${progress.percent}%`);
  }
});
```

---

### 5. E-commerce

**Use Case**: Migrate product catalog when schema changes.

**Migration Scenarios**:
- Add `brand` field (required) with backfill
- Change `price` field type (string → number)
- Add `variants` array field

**Example**:
```typescript
// Scenario: Make brand field required in 500K products
const plan = await engine.generatePlan({
  schema_id: "product_listing",
  from_version: "1.0.0",
  to_version: "2.0.0",
  table_name: "products",
  total_records: 500000
});

console.log(plan);
// {
//   steps: [
//     {
//       type: "ADD_REQUIRED_FIELD",
//       field: "brand",
//       backfill: (record) => extractBrandFromName(record.name) || "Unknown"
//     }
//   ],
//   estimated_duration_ms: 300000, // 5 minutes
//   requires_data_migration: true,
//   breaking: true
// }

// Rollback test: Simulate failure
try {
  await engine.execute(plan, {
    simulate_failure_at_percent: 50 // Fail at 50%
  });
} catch (error) {
  console.log("Migration failed - automatic rollback triggered");
  const rollbackResult = await engine.rollback(plan);
  console.log(`Rolled back ${rollbackResult.records_restored} records`);
}
```

---

### 6. SaaS

**Use Case**: Migrate subscription data when schema evolves.

**Migration Scenarios**:
- Add `trial_ends_at` timestamp field
- Change `plan` field from string to object
- Add `features` array field

**Example**:
```typescript
// Scenario: Add trial_ends_at field to 50K subscriptions
const plan = await engine.generatePlan({
  schema_id: "subscription",
  from_version: "1.0.0",
  to_version: "1.1.0",
  table_name: "subscriptions",
  total_records: 50000
});

console.log(plan);
// {
//   steps: [
//     {
//       type: "ADD_FIELD",
//       field: "trial_ends_at",
//       backfill: (record) => {
//         if (record.status === "trial") {
//           return new Date(record.created_at).setDate(+14); // 14-day trial
//         }
//         return null;
//       }
//     }
//   ],
//   estimated_duration_ms: 30000, // 30 seconds
//   requires_data_migration: true,
//   breaking: false
// }

// Execute with monitoring
const result = await engine.execute(plan, {
  on_batch_complete: (batch) => {
    console.log(`Batch ${batch.number} completed: ${batch.records} records`);
  }
});
```

---

### 7. Insurance

**Use Case**: Migrate insurance policy data when schema changes.

**Migration Scenarios**:
- Add `deductible` field with backfill
- Remove deprecated `legacy_policy_number` field
- Change `premium` field precision

**Example**:
```typescript
// Scenario: Remove legacy field from 200K policies
const plan = await engine.generatePlan({
  schema_id: "insurance_policy",
  from_version: "1.1.0",
  to_version: "2.0.0",
  table_name: "policies",
  total_records: 200000
});

console.log(plan);
// {
//   steps: [
//     {
//       type: "REMOVE_FIELD",
//       field: "legacy_policy_number",
//       archive_table: "policies_legacy_archive"
//     }
//   ],
//   estimated_duration_ms: 90000, // 1.5 minutes
//   requires_data_migration: true,
//   breaking: true
// }

// Archive data before removal
const result = await engine.execute(plan, {
  archive_removed_fields: true
});
console.log(`Archived ${result.records_archived} legacy policy numbers`);
```

---

### Cross-Domain Benefits

**Automated Migrations**: All domains benefit from:
- Automatic migration plan generation
- Batch processing for large datasets
- Progress tracking with ETA
- Automatic rollback on failure

**Regulatory Compliance**: Supports:
- SOX (Finance): Migration audit trail
- HIPAA (Healthcare): Data transformation logging
- GDPR (Privacy): Data migration transparency
- Legal Discovery: Migration history preservation

---

## Core Concepts

### Migration Plan

A **migration plan** is a sequence of steps to transform data from one schema version to another.

**Example**:
```typescript
{
  schema_id: "user_profile",
  from_version: "1.0.0",
  to_version: "2.0.0",
  steps: [
    {
      type: "ADD_REQUIRED_FIELD",
      field: "email",
      backfill: (record) => `${record.username}@example.com`
    },
    {
      type: "RENAME_FIELD",
      from_field: "full_name",
      to_field: "name"
    }
  ],
  estimated_duration_ms: 60000,
  requires_data_migration: true,
  breaking: true
}
```

### Migration Steps

**ADD_FIELD**: Add new optional field
```typescript
{
  type: "ADD_FIELD",
  field: "middle_name",
  default_value: null
}
```

**ADD_REQUIRED_FIELD**: Add new required field with backfill
```typescript
{
  type: "ADD_REQUIRED_FIELD",
  field: "email",
  backfill: (record) => `${record.username}@example.com`
}
```

**REMOVE_FIELD**: Remove field (optionally archive data)
```typescript
{
  type: "REMOVE_FIELD",
  field: "legacy_id",
  archive_table: "users_legacy_archive"
}
```

**RENAME_FIELD**: Rename field (with optional transformation)
```typescript
{
  type: "RENAME_FIELD",
  from_field: "full_name",
  to_field: "name",
  transform: (value) => value.trim()
}
```

**CHANGE_TYPE**: Change field type with transformation
```typescript
{
  type: "CHANGE_TYPE",
  field: "age",
  from_type: "string",
  to_type: "integer",
  transform: (value) => parseInt(value, 10)
}
```

### Batch Processing

Migrations process data in batches to avoid table locks:

- **Default batch size**: 10,000 records
- **Configurable**: Adjust based on table size and performance
- **Progress tracking**: Report progress after each batch

### Rollback

If migration fails, automatic rollback restores data to pre-migration state:

1. Detect failure (exception, validation error)
2. Stop processing
3. Restore from backup or reverse migration steps
4. Report rollback status

---

## Interface Definition

### TypeScript Interface

```typescript
interface MigrationEngine {
  /**
   * Generate migration plan from schema diff
   *
   * @param config - Migration configuration
   * @returns Generated migration plan
   */
  generatePlan(config: MigrationConfig): Promise<MigrationPlan>;

  /**
   * Execute migration plan
   *
   * @param plan - Migration plan to execute
   * @param options - Execution options
   * @returns Migration result
   */
  execute(
    plan: MigrationPlan,
    options?: ExecutionOptions
  ): Promise<MigrationResult>;

  /**
   * Dry run migration without executing changes
   *
   * @param plan - Migration plan to validate
   * @returns Dry run result with validation errors
   */
  dryRun(plan: MigrationPlan): Promise<DryRunResult>;

  /**
   * Estimate migration duration
   *
   * @param plan - Migration plan
   * @returns Duration estimate
   */
  estimateDuration(plan: MigrationPlan): Promise<DurationEstimate>;

  /**
   * Rollback failed migration
   *
   * @param plan - Migration plan that failed
   * @returns Rollback result
   */
  rollback(plan: MigrationPlan): Promise<RollbackResult>;

  /**
   * Get migration status
   *
   * @param migrationId - Migration ID
   * @returns Current migration status
   */
  getStatus(migrationId: string): Promise<MigrationStatus>;

  /**
   * Cancel running migration
   *
   * @param migrationId - Migration ID to cancel
   * @returns Cancellation result
   */
  cancel(migrationId: string): Promise<CancellationResult>;

  /**
   * List migration history
   *
   * @param schemaId - Schema ID
   * @param options - Filter options
   * @returns Array of past migrations
   */
  listMigrations(
    schemaId: string,
    options?: ListMigrationsOptions
  ): Promise<MigrationRecord[]>;
}
```

---

## Data Model

### MigrationConfig Type

```typescript
interface MigrationConfig {
  schema_id: string;              // Schema identifier
  from_version: string;           // Source version
  to_version: string;             // Target version
  table_name: string;             // Database table name
  total_records: number;          // Total records to migrate
  custom_transforms?: Record<string, TransformFunction>; // Custom field transforms
}
```

### MigrationPlan Type

```typescript
interface MigrationPlan {
  plan_id: string;                // Unique plan ID
  schema_id: string;              // Schema identifier
  from_version: string;           // Source version
  to_version: string;             // Target version
  steps: MigrationStep[];         // Migration steps
  estimated_duration_ms: number;  // Estimated duration
  requires_data_migration: boolean; // Whether data migration needed
  breaking: boolean;              // Whether migration is breaking
  created_at: string;             // Plan creation timestamp
}
```

### MigrationStep Type

```typescript
interface MigrationStep {
  type: StepType;                 // Step type
  field?: string;                 // Field name (if applicable)
  from_field?: string;            // Source field (for RENAME)
  to_field?: string;              // Target field (for RENAME)
  default_value?: any;            // Default value (for ADD_FIELD)
  backfill?: (record: any) => any; // Backfill function (for ADD_REQUIRED_FIELD)
  transform?: (value: any) => any; // Transform function
  sql?: string;                   // SQL statement (for DDL changes)
  archive_table?: string;         // Archive table (for REMOVE_FIELD)
}

type StepType =
  | "ADD_FIELD"
  | "ADD_REQUIRED_FIELD"
  | "REMOVE_FIELD"
  | "RENAME_FIELD"
  | "CHANGE_TYPE"
  | "ADD_CONSTRAINT"
  | "REMOVE_CONSTRAINT";
```

### ExecutionOptions Type

```typescript
interface ExecutionOptions {
  batch_size?: number;            // Records per batch (default: 10000)
  create_backup?: boolean;        // Create backup before migration (default: true)
  backup_table?: string;          // Backup table name
  on_progress?: (progress: ProgressUpdate) => void; // Progress callback
  on_batch_complete?: (batch: BatchUpdate) => void; // Batch callback
  archive_removed_fields?: boolean; // Archive data from removed fields
  simulate_failure_at_percent?: number; // For testing rollback
}
```

### MigrationResult Type

```typescript
interface MigrationResult {
  migration_id: string;           // Migration ID
  status: 'completed' | 'failed' | 'rolled_back'; // Final status
  records_migrated: number;       // Records successfully migrated
  records_failed: number;         // Records that failed
  duration_ms: number;            // Actual duration
  started_at: string;             // Start timestamp
  completed_at: string;           // Completion timestamp
  backup_table?: string;          // Backup table name (if created)
  error?: string;                 // Error message (if failed)
}
```

### ProgressUpdate Type

```typescript
interface ProgressUpdate {
  migration_id: string;           // Migration ID
  total_records: number;          // Total records
  records_processed: number;      // Records processed so far
  percent: number;                // Progress percentage (0-100)
  eta_ms: number;                 // Estimated time remaining (ms)
  current_step: number;           // Current step index
  total_steps: number;            // Total steps
}
```

---

## Core Functionality

### 1. generatePlan()

Generate migration plan from schema diff.

#### Signature

```typescript
generatePlan(config: MigrationConfig): Promise<MigrationPlan>
```

#### Behavior

1. Fetch schemas for from_version and to_version
2. Compute schema diff using BackwardCompatibilityChecker
3. Generate migration steps from diff
4. Estimate duration based on total_records
5. Return migration plan

#### Example

```typescript
const engine = new MigrationEngine();

const plan = await engine.generatePlan({
  schema_id: "user_profile",
  from_version: "1.0.0",
  to_version: "2.0.0",
  table_name: "users",
  total_records: 100000
});

console.log(plan);
// {
//   plan_id: "mig_abc123",
//   schema_id: "user_profile",
//   from_version: "1.0.0",
//   to_version: "2.0.0",
//   steps: [
//     { type: "ADD_REQUIRED_FIELD", field: "email", backfill: ... },
//     { type: "RENAME_FIELD", from_field: "full_name", to_field: "name" }
//   ],
//   estimated_duration_ms: 60000,
//   requires_data_migration: true,
//   breaking: true,
//   created_at: "2025-01-20T10:00:00Z"
// }
```

---

### 2. execute()

Execute migration plan with batch processing and progress tracking.

#### Signature

```typescript
execute(
  plan: MigrationPlan,
  options?: ExecutionOptions
): Promise<MigrationResult>
```

#### Behavior

1. **Pre-execution**:
   - Create backup table (if `create_backup: true`)
   - Validate plan
   - Initialize progress tracking

2. **Execution**:
   - Process data in batches (default: 10K records/batch)
   - Apply each migration step to batch
   - Report progress after each batch
   - Handle errors and trigger rollback if needed

3. **Post-execution**:
   - Verify migration success
   - Return migration result

#### Example

```typescript
// Execute with progress tracking
const result = await engine.execute(plan, {
  batch_size: 10000,
  create_backup: true,
  on_progress: (progress) => {
    console.log(`Progress: ${progress.percent}%`);
    console.log(`ETA: ${progress.eta_ms}ms`);
    console.log(`Processed: ${progress.records_processed}/${progress.total_records}`);
  },
  on_batch_complete: (batch) => {
    console.log(`Batch ${batch.number} completed: ${batch.records} records in ${batch.duration_ms}ms`);
  }
});

console.log(result);
// {
//   migration_id: "mig_abc123",
//   status: "completed",
//   records_migrated: 100000,
//   records_failed: 0,
//   duration_ms: 58243,
//   started_at: "2025-01-20T10:00:00Z",
//   completed_at: "2025-01-20T10:00:58Z",
//   backup_table: "users_backup_20250120"
// }
```

---

### 3. dryRun()

Validate migration without executing changes.

#### Signature

```typescript
dryRun(plan: MigrationPlan): Promise<DryRunResult>
```

#### Behavior

1. Validate migration steps
2. Check for data consistency issues
3. Estimate duration
4. Return validation result without modifying data

#### Example

```typescript
const dryRunResult = await engine.dryRun(plan);

console.log(dryRunResult);
// {
//   valid: true,
//   validation_errors: [],
//   estimated_duration_ms: 60000,
//   estimated_batches: 10,
//   warnings: [
//     "Field 'email' will be backfilled with generated values"
//   ]
// }

// Check for errors before execution
if (dryRunResult.validation_errors.length > 0) {
  console.error("Migration validation failed:");
  for (const error of dryRunResult.validation_errors) {
    console.error(`- ${error.message}`);
  }
} else {
  // Safe to execute
  await engine.execute(plan);
}
```

---

### 4. estimateDuration()

Estimate migration duration based on data volume.

#### Signature

```typescript
estimateDuration(plan: MigrationPlan): Promise<DurationEstimate>
```

#### Behavior

1. Calculate number of batches
2. Estimate time per batch based on step complexity
3. Add overhead for backup and rollback
4. Return duration estimate

#### Example

```typescript
const estimate = await engine.estimateDuration(plan);

console.log(estimate);
// {
//   duration_ms: 60000,
//   batches: 10,
//   records_per_batch: 10000,
//   time_per_batch_ms: 6000,
//   overhead_ms: 0,
//   confidence: 0.85 // 85% confidence
// }

// Convert to human-readable
const minutes = Math.ceil(estimate.duration_ms / 60000);
console.log(`Estimated duration: ${minutes} minute(s)`);
```

---

### 5. rollback()

Rollback failed migration.

#### Signature

```typescript
rollback(plan: MigrationPlan): Promise<RollbackResult>
```

#### Behavior

1. **Detect failure point**: Determine how many records were migrated
2. **Restore from backup**: Copy data from backup table
3. **Reverse migration steps**: Undo DDL changes
4. **Verify restoration**: Check data integrity
5. **Return rollback result**

#### Example

```typescript
try {
  await engine.execute(plan);
} catch (error) {
  console.error("Migration failed:", error.message);

  // Automatic rollback
  const rollbackResult = await engine.rollback(plan);

  console.log(rollbackResult);
  // {
  //   success: true,
  //   records_restored: 50000, // Restored 50K of 100K records
  //   duration_ms: 12000,
  //   backup_table: "users_backup_20250120",
  //   message: "Rollback completed successfully"
  // }
}
```

---

### 6. getStatus()

Get real-time migration status.

#### Signature

```typescript
getStatus(migrationId: string): Promise<MigrationStatus>
```

#### Behavior

Query migration status from migration log table.

#### Example

```typescript
const status = await engine.getStatus("mig_abc123");

console.log(status);
// {
//   migration_id: "mig_abc123",
//   status: "running",
//   progress: {
//     total_records: 100000,
//     records_processed: 45000,
//     percent: 45,
//     eta_ms: 30000
//   },
//   started_at: "2025-01-20T10:00:00Z"
// }
```

---

### 7. cancel()

Cancel running migration.

#### Signature

```typescript
cancel(migrationId: string): Promise<CancellationResult>
```

#### Behavior

1. Stop batch processing
2. Trigger rollback
3. Return cancellation result

#### Example

```typescript
const cancellation = await engine.cancel("mig_abc123");

console.log(cancellation);
// {
//   migration_id: "mig_abc123",
//   status: "cancelled",
//   records_processed: 45000,
//   rollback_completed: true,
//   message: "Migration cancelled and rolled back"
// }
```

---

## Migration Planning

### Plan Generation Algorithm

```typescript
async function generatePlan(config: MigrationConfig): Promise<MigrationPlan> {
  // 1. Fetch schemas
  const oldSchema = await registry.getVersion(config.schema_id, config.from_version);
  const newSchema = await registry.getVersion(config.schema_id, config.to_version);

  // 2. Compute diff
  const checker = new BackwardCompatibilityChecker();
  const diff = await checker.detectChanges(oldSchema.schema, newSchema.schema);

  // 3. Generate steps
  const steps: MigrationStep[] = [];

  for (const change of diff.all_changes) {
    switch (change.type) {
      case "FIELD_ADDED":
        steps.push({
          type: "ADD_FIELD",
          field: change.field,
          default_value: change.default || null
        });
        break;

      case "FIELD_REQUIRED_ADDED":
        steps.push({
          type: "ADD_REQUIRED_FIELD",
          field: change.field,
          backfill: config.custom_transforms?.[change.field] || defaultBackfill
        });
        break;

      case "FIELD_REMOVED":
        steps.push({
          type: "REMOVE_FIELD",
          field: change.field,
          archive_table: `${config.table_name}_archive`
        });
        break;

      case "FIELD_RENAMED":
        steps.push({
          type: "RENAME_FIELD",
          from_field: change.from_field,
          to_field: change.to_field
        });
        break;

      case "FIELD_TYPE_CHANGED":
        steps.push({
          type: "CHANGE_TYPE",
          field: change.field,
          from_type: change.from_type,
          to_type: change.to_type,
          transform: config.custom_transforms?.[change.field] || defaultTransform
        });
        break;
    }
  }

  // 4. Estimate duration
  const estimated_duration_ms = estimateDuration(steps, config.total_records);

  // 5. Determine if breaking
  const breaking = diff.breaking_changes.length > 0;

  return {
    plan_id: generateId(),
    schema_id: config.schema_id,
    from_version: config.from_version,
    to_version: config.to_version,
    steps,
    estimated_duration_ms,
    requires_data_migration: steps.length > 0,
    breaking,
    created_at: new Date().toISOString()
  };
}
```

---

## Execution Strategy

### Batch Processing

```typescript
async function executeBatchProcessing(
  plan: MigrationPlan,
  options: ExecutionOptions
): Promise<MigrationResult> {
  const batch_size = options.batch_size || 10000;
  const total_batches = Math.ceil(plan.total_records / batch_size);

  let records_processed = 0;
  const start_time = Date.now();

  for (let batch_num = 0; batch_num < total_batches; batch_num++) {
    const offset = batch_num * batch_size;

    // Fetch batch
    const batch = await fetchBatch(plan.table_name, offset, batch_size);

    // Apply migration steps
    for (const step of plan.steps) {
      batch.records = await applyStep(step, batch.records);
    }

    // Update database
    await updateBatch(plan.table_name, batch.records);

    // Update progress
    records_processed += batch.records.length;
    const percent = Math.round((records_processed / plan.total_records) * 100);
    const elapsed_ms = Date.now() - start_time;
    const eta_ms = Math.round((elapsed_ms / records_processed) * (plan.total_records - records_processed));

    if (options.on_progress) {
      options.on_progress({
        migration_id: plan.plan_id,
        total_records: plan.total_records,
        records_processed,
        percent,
        eta_ms,
        current_step: batch_num + 1,
        total_steps: total_batches
      });
    }

    if (options.on_batch_complete) {
      options.on_batch_complete({
        number: batch_num + 1,
        records: batch.records.length,
        duration_ms: Date.now() - start_time
      });
    }
  }

  return {
    migration_id: plan.plan_id,
    status: 'completed',
    records_migrated: records_processed,
    records_failed: 0,
    duration_ms: Date.now() - start_time,
    started_at: new Date(start_time).toISOString(),
    completed_at: new Date().toISOString()
  };
}
```

---

## Rollback Mechanism

### Automatic Rollback

```typescript
async function rollback(plan: MigrationPlan): Promise<RollbackResult> {
  const start_time = Date.now();

  try {
    // 1. Check if backup exists
    const backup_table = `${plan.table_name}_backup_${formatDate(new Date())}`;
    const backup_exists = await tableExists(backup_table);

    if (!backup_exists) {
      throw new Error(`Backup table ${backup_table} not found`);
    }

    // 2. Restore from backup
    await db.query(`
      DELETE FROM ${plan.table_name};
      INSERT INTO ${plan.table_name}
      SELECT * FROM ${backup_table};
    `);

    // 3. Reverse DDL changes
    for (const step of plan.steps.reverse()) {
      await reverseDDLStep(step, plan.table_name);
    }

    // 4. Verify restoration
    const restored_count = await db.query(`SELECT COUNT(*) FROM ${plan.table_name}`);

    return {
      success: true,
      records_restored: restored_count.rows[0].count,
      duration_ms: Date.now() - start_time,
      backup_table,
      message: "Rollback completed successfully"
    };
  } catch (error) {
    return {
      success: false,
      records_restored: 0,
      duration_ms: Date.now() - start_time,
      error: error.message,
      message: "Rollback failed"
    };
  }
}
```

---

## Edge Cases

### Edge Case 1: Migration Interrupted Mid-Batch

**Scenario**: Server crashes during batch processing.

**Handling**: Resume from last completed batch using migration log:

```typescript
async function resumeMigration(migrationId: string): Promise<MigrationResult> {
  // Fetch migration state from log
  const state = await getMigrationState(migrationId);

  if (!state) {
    throw new Error("Migration not found");
  }

  // Resume from last completed batch
  const resumeOffset = state.last_completed_batch * state.batch_size;

  console.log(`Resuming migration from batch ${state.last_completed_batch + 1}`);

  return executeBatchProcessing(state.plan, {
    ...state.options,
    start_offset: resumeOffset
  });
}
```

---

### Edge Case 2: Very Large Tables (Billions of Records)

**Scenario**: Migrating table with 1B+ records.

**Handling**: Use parallel batch processing:

```typescript
async function executeParallelMigration(
  plan: MigrationPlan,
  options: ExecutionOptions
): Promise<MigrationResult> {
  const parallelism = options.parallelism || 4;
  const batch_size = options.batch_size || 10000;
  const total_batches = Math.ceil(plan.total_records / batch_size);

  // Split batches across workers
  const batches_per_worker = Math.ceil(total_batches / parallelism);

  const workers = [];
  for (let i = 0; i < parallelism; i++) {
    const start_batch = i * batches_per_worker;
    const end_batch = Math.min((i + 1) * batches_per_worker, total_batches);

    workers.push(
      executeBatchRange(plan, start_batch, end_batch, options)
    );
  }

  // Wait for all workers
  const results = await Promise.all(workers);

  // Aggregate results
  return aggregateResults(results);
}
```

---

### Edge Case 3: Custom Transform Fails for Some Records

**Scenario**: Backfill function throws error for invalid data.

**Handling**: Log errors and continue with default value:

```typescript
async function applyBackfill(
  step: MigrationStep,
  records: any[]
): Promise<any[]> {
  const errors = [];

  for (const record of records) {
    try {
      record[step.field] = step.backfill!(record);
    } catch (error) {
      errors.push({
        record_id: record.id,
        error: error.message
      });

      // Use default value
      record[step.field] = step.default_value || null;
    }
  }

  if (errors.length > 0) {
    console.warn(`${errors.length} backfill errors (using default value)`);
    await logBackfillErrors(step.field, errors);
  }

  return records;
}
```

---

### Edge Case 4: Rollback Fails Due to Backup Corruption

**Scenario**: Backup table is corrupted or deleted.

**Handling**: Use point-in-time recovery from database backup:

```typescript
async function rollbackWithPITR(
  plan: MigrationPlan,
  timestamp: string
): Promise<RollbackResult> {
  // Use PostgreSQL PITR
  await db.query(`
    -- Restore from WAL archive
    SELECT pg_wal_replay_resume();
    RESTORE DATABASE TO POINT IN TIME '${timestamp}';
  `);

  return {
    success: true,
    records_restored: -1, // Unknown (full database restore)
    duration_ms: -1,
    backup_table: "PITR",
    message: "Rollback completed using PITR"
  };
}
```

---

### Edge Case 5: Schema Mismatch During Migration

**Scenario**: Schema changed externally during migration.

**Handling**: Detect schema version mismatch and abort:

```typescript
async function executeSafeMigration(
  plan: MigrationPlan,
  options: ExecutionOptions
): Promise<MigrationResult> {
  // 1. Lock schema version
  await db.query(`
    SELECT pg_advisory_lock(hashtext('${plan.schema_id}'));
  `);

  // 2. Verify schema version hasn't changed
  const current_schema = await registry.getLatest(plan.schema_id);

  if (current_schema.version !== plan.from_version) {
    await db.query(`SELECT pg_advisory_unlock(hashtext('${plan.schema_id}'))`);
    throw new Error(
      `Schema version mismatch: expected ${plan.from_version}, found ${current_schema.version}`
    );
  }

  // 3. Execute migration
  try {
    const result = await executeBatchProcessing(plan, options);

    // 4. Update schema version
    await registry.publish({
      schema_id: plan.schema_id,
      version: plan.to_version,
      schema: await getSchemaDefinition(plan.to_version),
      description: `Migrated from ${plan.from_version}`,
      published_by: "migration-engine"
    });

    return result;
  } finally {
    // 5. Release lock
    await db.query(`SELECT pg_advisory_unlock(hashtext('${plan.schema_id}'))`);
  }
}
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `generatePlan()` | 2 schemas | < 500ms | Schema diff + analysis |
| `dryRun()` | 100K records | < 1s | Validation without execution |
| `estimateDuration()` | Any | < 100ms | Calculation only |
| `execute()` (10K batch) | 10K records | < 5s/batch | Actual migration |
| `execute()` (100K total) | 100K records | < 60s | Full migration (10 batches) |
| `rollback()` | 100K records | < 30s | Restore from backup |

### Throughput

- **Records/second**: 10,000-50,000 (depends on step complexity)
- **Batch size**: 10K records (configurable)

### Scalability

| Total Records | Duration (10K/batch) | Notes |
|--------------|---------------------|-------|
| < 100K | < 1 minute | Fast |
| 100K - 1M | 1-10 minutes | Acceptable |
| 1M - 10M | 10-100 minutes | Consider parallel processing |
| > 10M | > 100 minutes | Use parallel workers |

### Optimization Tips

**1. Increase Batch Size for Large Tables**

```typescript
// Bad: Small batches (many round trips)
await engine.execute(plan, {
  batch_size: 1000 // 1K records/batch
});

// Good: Larger batches (fewer round trips)
await engine.execute(plan, {
  batch_size: 50000 // 50K records/batch
});
```

**2. Use Parallel Processing for Huge Tables**

```typescript
await engine.execute(plan, {
  batch_size: 10000,
  parallelism: 8 // 8 parallel workers
});
```

**3. Disable Backup for Non-Critical Migrations**

```typescript
await engine.execute(plan, {
  create_backup: false // Skip backup (faster, but risky)
});
```

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';

export class MigrationEngine {
  constructor(private pool: Pool, private registry: SchemaRegistry) {}

  async generatePlan(config: MigrationConfig): Promise<MigrationPlan> {
    // Fetch schemas
    const oldSchema = await this.registry.getVersion(config.schema_id, config.from_version);
    const newSchema = await this.registry.getVersion(config.schema_id, config.to_version);

    if (!oldSchema || !newSchema) {
      throw new Error("Schema version not found");
    }

    // Generate steps
    const checker = new BackwardCompatibilityChecker();
    const diff = await checker.check(oldSchema.schema, newSchema.schema);

    const steps = this.generateStepsFromDiff(diff, config);

    // Estimate duration (6ms per record on average)
    const estimated_duration_ms = Math.ceil((config.total_records / 10000) * 6000);

    return {
      plan_id: this.generateId(),
      schema_id: config.schema_id,
      from_version: config.from_version,
      to_version: config.to_version,
      steps,
      estimated_duration_ms,
      requires_data_migration: steps.length > 0,
      breaking: diff.breaking_changes.length > 0,
      created_at: new Date().toISOString()
    };
  }

  async execute(
    plan: MigrationPlan,
    options: ExecutionOptions = {}
  ): Promise<MigrationResult> {
    const batch_size = options.batch_size || 10000;
    const create_backup = options.create_backup !== false; // Default: true

    const migration_id = plan.plan_id;
    const start_time = Date.now();

    try {
      // 1. Create backup
      if (create_backup) {
        const backup_table = options.backup_table || `${plan.table_name}_backup_${Date.now()}`;
        await this.createBackup(plan.table_name, backup_table);
      }

      // 2. Execute batch processing
      const total_records = await this.getRecordCount(plan.table_name);
      const total_batches = Math.ceil(total_records / batch_size);

      let records_processed = 0;

      for (let batch = 0; batch < total_batches; batch++) {
        const offset = batch * batch_size;

        // Fetch batch
        const query = `SELECT * FROM ${plan.table_name} ORDER BY id LIMIT ${batch_size} OFFSET ${offset}`;
        const result = await this.pool.query(query);

        // Apply steps
        const migrated = await this.applySteps(plan.steps, result.rows);

        // Update records
        await this.updateRecords(plan.table_name, migrated);

        // Progress
        records_processed += result.rows.length;
        const percent = Math.round((records_processed / total_records) * 100);

        if (options.on_progress) {
          options.on_progress({
            migration_id,
            total_records,
            records_processed,
            percent,
            eta_ms: Math.round(((Date.now() - start_time) / records_processed) * (total_records - records_processed)),
            current_step: batch + 1,
            total_steps: total_batches
          });
        }
      }

      return {
        migration_id,
        status: 'completed',
        records_migrated: records_processed,
        records_failed: 0,
        duration_ms: Date.now() - start_time,
        started_at: new Date(start_time).toISOString(),
        completed_at: new Date().toISOString()
      };
    } catch (error) {
      // Automatic rollback
      await this.rollback(plan);

      return {
        migration_id,
        status: 'failed',
        records_migrated: 0,
        records_failed: 0,
        duration_ms: Date.now() - start_time,
        started_at: new Date(start_time).toISOString(),
        completed_at: new Date().toISOString(),
        error: error.message
      };
    }
  }

  async dryRun(plan: MigrationPlan): Promise<DryRunResult> {
    const validation_errors = [];

    // Validate steps
    for (const step of plan.steps) {
      if (step.type === 'ADD_REQUIRED_FIELD' && !step.backfill) {
        validation_errors.push({
          step: step.type,
          message: `Required field '${step.field}' has no backfill function`
        });
      }
    }

    return {
      valid: validation_errors.length === 0,
      validation_errors,
      estimated_duration_ms: plan.estimated_duration_ms,
      estimated_batches: Math.ceil(plan.total_records / 10000),
      warnings: []
    };
  }

  private async createBackup(table: string, backup_table: string): Promise<void> {
    await this.pool.query(`CREATE TABLE ${backup_table} AS SELECT * FROM ${table}`);
  }

  private async applySteps(steps: MigrationStep[], records: any[]): Promise<any[]> {
    for (const step of steps) {
      switch (step.type) {
        case 'ADD_FIELD':
          records = records.map(r => ({ ...r, [step.field!]: step.default_value }));
          break;

        case 'ADD_REQUIRED_FIELD':
          records = records.map(r => ({ ...r, [step.field!]: step.backfill!(r) }));
          break;

        case 'RENAME_FIELD':
          records = records.map(r => {
            const value = r[step.from_field!];
            delete r[step.from_field!];
            r[step.to_field!] = step.transform ? step.transform(value) : value;
            return r;
          });
          break;

        case 'CHANGE_TYPE':
          records = records.map(r => ({
            ...r,
            [step.field!]: step.transform!(r[step.field!])
          }));
          break;
      }
    }

    return records;
  }

  private generateId(): string {
    return `mig_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

---

## Security Considerations

### 1. Backup Before Migration

Always create backup before destructive operations:

```typescript
await engine.execute(plan, {
  create_backup: true // Mandatory for production
});
```

### 2. Validate Custom Transforms

Sanitize user-provided transform functions:

```typescript
function validateTransform(transform: Function): void {
  // Check for dangerous operations
  const source = transform.toString();

  if (source.includes('eval(') || source.includes('Function(')) {
    throw new Error("Transform contains dangerous code");
  }
}
```

### 3. Transaction Isolation

Use transaction isolation to prevent race conditions:

```typescript
await db.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
await executeMigration(plan);
await db.query('COMMIT');
```

---

## Integration Patterns

### Pattern 1: CI/CD Automated Migrations

```typescript
// CI/CD pipeline: Auto-migrate on deploy
async function deployWithMigration(newSchemaVersion: string): Promise<void> {
  const engine = new MigrationEngine(pool, registry);

  // Get current version
  const current = await registry.getLatest("user_profile");

  // Generate plan
  const plan = await engine.generatePlan({
    schema_id: "user_profile",
    from_version: current.version,
    to_version: newSchemaVersion,
    table_name: "users",
    total_records: await getRecordCount("users")
  });

  // Dry run
  const dryRun = await engine.dryRun(plan);

  if (!dryRun.valid) {
    throw new Error(`Migration validation failed: ${dryRun.validation_errors}`);
  }

  // Execute
  await engine.execute(plan);

  console.log("Migration completed successfully");
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section with 7 domains: Finance, Healthcare, Legal, Research, E-commerce, SaaS, Insurance)

---

## Testing Strategy

### Unit Tests

```typescript
describe('MigrationEngine', () => {
  let engine: MigrationEngine;
  let pool: Pool;
  let registry: SchemaRegistry;

  beforeEach(async () => {
    pool = new Pool({ /* config */ });
    registry = new SchemaRegistry(pool);
    engine = new MigrationEngine(pool, registry);
    await cleanDatabase();
  });

  describe('generatePlan()', () => {
    it('should generate plan for add field', async () => {
      await registry.publish({
        schema_id: 'user',
        version: '1.0.0',
        schema: { type: 'object', properties: { id: { type: 'string' } } },
        description: 'v1'
      });

      await registry.publish({
        schema_id: 'user',
        version: '1.1.0',
        schema: {
          type: 'object',
          properties: {
            id: { type: 'string' },
            email: { type: 'string' }
          }
        },
        description: 'v1.1'
      });

      const plan = await engine.generatePlan({
        schema_id: 'user',
        from_version: '1.0.0',
        to_version: '1.1.0',
        table_name: 'users',
        total_records: 100
      });

      expect(plan.steps).toHaveLength(1);
      expect(plan.steps[0].type).toBe('ADD_FIELD');
      expect(plan.steps[0].field).toBe('email');
    });
  });

  describe('execute()', () => {
    it('should execute migration with progress tracking', async () => {
      const plan = await engine.generatePlan({
        schema_id: 'user',
        from_version: '1.0.0',
        to_version: '1.1.0',
        table_name: 'users',
        total_records: 1000
      });

      const progressUpdates = [];

      const result = await engine.execute(plan, {
        batch_size: 100,
        on_progress: (progress) => {
          progressUpdates.push(progress);
        }
      });

      expect(result.status).toBe('completed');
      expect(result.records_migrated).toBe(1000);
      expect(progressUpdates.length).toBeGreaterThan(0);
    });
  });

  describe('rollback()', () => {
    it('should rollback failed migration', async () => {
      const plan = await engine.generatePlan({
        schema_id: 'user',
        from_version: '1.0.0',
        to_version: '2.0.0',
        table_name: 'users',
        total_records: 1000
      });

      // Simulate failure
      try {
        await engine.execute(plan, {
          simulate_failure_at_percent: 50
        });
      } catch (error) {
        // Rollback
        const rollbackResult = await engine.rollback(plan);

        expect(rollbackResult.success).toBe(true);
        expect(rollbackResult.records_restored).toBeGreaterThan(0);
      }
    });
  });
});
```

---

## Migration Guide

### From Manual Migrations to MigrationEngine

```typescript
// Before: Manual SQL migration
await db.query(`
  ALTER TABLE users ADD COLUMN email VARCHAR(255);
  UPDATE users SET email = username || '@example.com';
`);

// After: Automated migration
const engine = new MigrationEngine(pool, registry);

const plan = await engine.generatePlan({
  schema_id: "user",
  from_version: "1.0.0",
  to_version: "1.1.0",
  table_name: "users",
  total_records: await getRecordCount("users"),
  custom_transforms: {
    email: (record) => `${record.username}@example.com`
  }
});

await engine.execute(plan);
```

---

## Related Primitives

- **SchemaRegistry**: Provides schema versions for migration planning
- **SchemaVersionManager**: Compares versions to determine migration type
- **BackwardCompatibilityChecker**: Detects schema changes for migration steps
- **ProvenanceLedger**: Logs migration events for audit trail

---

## References

- [Database Migration Tools](https://en.wikipedia.org/wiki/Schema_migration)
- [Flyway](https://flywaydb.org/)
- [Alembic](https://alembic.sqlalchemy.org/)
- [Objective Layer Architecture](../architecture/objective-layer.md)
- [Schema Registry Vertical 5.2](../verticals/schema-registry.md)

---

**End of MigrationEngine OL Primitive Specification**
