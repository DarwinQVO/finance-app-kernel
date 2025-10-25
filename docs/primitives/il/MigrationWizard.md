# MigrationWizard (IL Component)

**Layer:** Interaction Layer (IL)
**Domain:** Schema Registry (Vertical 5.2)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Architecture](#architecture)
4. [Wizard Steps](#wizard-steps)
5. [Visual Wireframes](#visual-wireframes)
6. [States & Lifecycle](#states--lifecycle)
7. [Multi-Domain Examples](#multi-domain-examples)
8. [User Interactions](#user-interactions)
9. [Accessibility](#accessibility)
10. [Performance Optimizations](#performance-optimizations)
11. [Integration Examples](#integration-examples)
12. [Related Components](#related-components)
13. [References](#references)

---

## Overview

### Purpose

The **MigrationWizard** is a guided, multi-step interface that walks users through the complex process of migrating data from one schema version to another. When schema changes introduce breaking modifications (field removals, type changes, stricter constraints), existing data must be transformed to comply with the new schema. This component transforms what could be a risky, manual process into a safe, auditable, and reversible operation.

### Key Capabilities

- **Multi-Step Wizard Interface**: 5-step guided flow (Select → Review → Configure → Execute → Verify)
- **Impact Analysis**: Shows which records will be affected and how
- **Migration Preview**: Dry-run mode to test migration without committing changes
- **Batch Processing**: Handles large datasets by processing in configurable batches
- **Progress Tracking**: Real-time progress bar with ETA and throughput metrics
- **Error Handling**: Graceful failure with detailed error logs and retry logic
- **Rollback Support**: One-click rollback if migration produces unexpected results
- **Backup Creation**: Automatic backup before migration begins
- **Validation**: Post-migration validation ensures all records comply with new schema
- **Audit Trail**: Complete log of migration operation for compliance

### Problem Statement

Organizations face significant challenges when evolving schemas:

1. **Data Loss Risk**: Removing fields without migration causes data loss
2. **Downtime**: Manual migrations require system downtime
3. **Inconsistency**: Partial migrations leave system in inconsistent state
4. **No Rollback**: Failed migrations are difficult to undo
5. **Lack of Visibility**: No insight into migration progress or errors
6. **Testing Difficulty**: Hard to test migrations without affecting production data
7. **Scale Issues**: Large datasets (millions of records) overwhelm simple scripts

### Solution

The MigrationWizard provides:

- **Guided Workflow**: Step-by-step interface reduces errors
- **Safety Mechanisms**: Dry-run, backup, and rollback protect against data loss
- **Transparency**: Real-time progress, error logs, and impact analysis
- **Automation**: Handles batching, retries, and validation automatically
- **Auditability**: Complete audit trail for compliance and debugging
- **Flexibility**: Configurable batch sizes, transformation logic, and validation rules

This transforms schema migrations from risky one-off scripts to repeatable, safe operations.

---

## Component Interface

### Primary Props

```typescript
interface MigrationWizardProps {
  // ============================================================
  // Schema Configuration
  // ============================================================

  /**
   * Schema identifier being migrated
   * @example 'payment_event_schema', 'patient_record_schema'
   */
  schemaId: string

  /**
   * Source schema version (migrating from)
   * @example '2.1.0'
   */
  fromVersion: string

  /**
   * Target schema version (migrating to)
   * @example '3.0.0'
   */
  toVersion: string

  /**
   * Source schema definition
   * Used to validate existing records
   */
  sourceSchema?: object

  /**
   * Target schema definition
   * Used to validate migrated records
   */
  targetSchema?: object

  /**
   * Breaking changes between versions
   * Displayed in review step
   */
  breakingChanges?: BreakingChange[]

  // ============================================================
  // Migration Configuration
  // ============================================================

  /**
   * Default batch size for processing
   * @default 1000
   */
  defaultBatchSize?: number

  /**
   * Allow user to customize batch size
   * @default true
   */
  allowBatchSizeOverride?: boolean

  /**
   * Enable dry-run mode by default
   * @default true
   */
  defaultDryRun?: boolean

  /**
   * Enable automatic backup before migration
   * @default true
   */
  autoBackup?: boolean

  /**
   * Backup location/strategy
   * @default 'database_snapshot'
   */
  backupStrategy?: 'database_snapshot' | 'export_json' | 'export_csv' | 'custom'

  /**
   * Migration timeout (milliseconds)
   * If migration exceeds this, it's cancelled
   * @default 3600000 (1 hour)
   */
  migrationTimeout?: number

  /**
   * Retry failed records
   * @default true
   */
  retryFailedRecords?: boolean

  /**
   * Maximum retry attempts per record
   * @default 3
   */
  maxRetries?: number

  // ============================================================
  // Transformation Logic
  // ============================================================

  /**
   * Custom transformation function
   * Allows domain-specific data transformations
   * @param record - Source record (old schema)
   * @param context - Migration context (fromVersion, toVersion, etc.)
   * @returns Transformed record (new schema) or null to skip
   */
  transformRecord?: (record: any, context: MigrationContext) => any | null | Promise<any | null>

  /**
   * Validation function for transformed records
   * Additional validation beyond schema compliance
   * @param record - Transformed record
   * @returns true if valid, or error message if invalid
   */
  validateTransformedRecord?: (record: any) => boolean | string

  /**
   * Pre-migration hook
   * Called before migration starts (after backup)
   * @returns true to proceed, false to abort
   */
  onBeforeMigration?: () => boolean | Promise<boolean>

  /**
   * Post-migration hook
   * Called after migration completes (before verification)
   */
  onAfterMigration?: () => void | Promise<void>

  // ============================================================
  // Data Source
  // ============================================================

  /**
   * Function to fetch records to migrate
   * @param batchSize - Number of records to fetch
   * @param offset - Offset for pagination
   * @returns Array of records and total count
   */
  fetchRecords?: (batchSize: number, offset: number) => Promise<{
    records: any[]
    totalCount: number
  }>

  /**
   * Function to apply migration to records
   * If not provided, uses default API endpoint
   * @param records - Array of transformed records
   * @param options - Migration options (dryRun, etc.)
   */
  applyMigration?: (records: any[], options: MigrationOptions) => Promise<MigrationResult>

  /**
   * Estimated record count (for progress calculation)
   * If not provided, will be fetched dynamically
   */
  estimatedRecordCount?: number

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when migration completes successfully
   * @param result - Migration result summary
   */
  onComplete?: (result: MigrationResult) => void

  /**
   * Called when user cancels migration
   * @param step - Step where cancellation occurred
   */
  onCancel?: (step: WizardStep) => void

  /**
   * Called when migration fails
   * @param error - Error details
   * @param recoverable - Whether migration can be retried
   */
  onError?: (error: MigrationError, recoverable: boolean) => void

  /**
   * Called on progress updates during execution
   * @param progress - Current progress state
   */
  onProgress?: (progress: MigrationProgress) => void

  /**
   * Called when user initiates rollback
   * @param backupId - Backup to restore from
   */
  onRollback?: (backupId: string) => Promise<void>

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Show step indicators (1 of 5)
   * @default true
   */
  showStepIndicators?: boolean

  /**
   * Show impact analysis in review step
   * @default true
   */
  showImpactAnalysis?: boolean

  /**
   * Show progress chart during execution
   * @default true
   */
  showProgressChart?: boolean

  /**
   * Show error log panel
   * @default true
   */
  showErrorLog?: boolean

  /**
   * Enable download of migration report
   * @default true
   */
  allowReportDownload?: boolean

  /**
   * Allow skipping backup step (dangerous)
   * @default false
   */
  allowSkipBackup?: boolean

  /**
   * Confirm before starting execution
   * @default true
   */
  confirmBeforeExecution?: boolean

  /**
   * Theme
   * @default 'light'
   */
  theme?: 'light' | 'dark'

  /**
   * Width of wizard modal/container
   * @default '800px'
   */
  width?: string

  /**
   * Custom footer actions
   * Allows adding domain-specific actions to wizard footer
   */
  customActions?: CustomWizardAction[]

  // ============================================================
  // Advanced Options
  // ============================================================

  /**
   * Enable parallel processing
   * Process multiple batches concurrently
   * @default false (sequential processing)
   */
  parallelProcessing?: boolean

  /**
   * Number of parallel workers
   * Only used if parallelProcessing=true
   * @default 3
   */
  parallelWorkers?: number

  /**
   * Enable streaming mode
   * Process records as they arrive (no full fetch)
   * @default false
   */
  streamingMode?: boolean

  /**
   * Migration strategy
   * @default 'in_place' - Modify existing records
   */
  migrationStrategy?: 'in_place' | 'copy' | 'dual_write'

  /**
   * Notification preferences
   * Send notifications on specific events
   */
  notifications?: {
    onStart?: boolean
    onComplete?: boolean
    onError?: boolean
    email?: string
    slack?: string
  }

  /**
   * Debug mode
   * Show additional debug information
   * @default false
   */
  debugMode?: boolean
}

// ============================================================
// Supporting Types
// ============================================================

interface BreakingChange {
  type: 'field_removed' | 'type_changed' | 'constraint_added' | 'enum_reduced'
  path: string  // JSONPath to affected field
  description: string
  impact: 'high' | 'medium' | 'low'
  affectedRecords?: number
  transformationRequired: boolean
}

interface MigrationContext {
  schemaId: string
  fromVersion: string
  toVersion: string
  dryRun: boolean
  batchNumber: number
  recordIndex: number
}

interface MigrationOptions {
  dryRun: boolean
  batchSize: number
  backup: boolean
  retryFailures: boolean
  maxRetries: number
  timeout: number
}

interface MigrationResult {
  success: boolean
  totalRecords: number
  migratedRecords: number
  failedRecords: number
  skippedRecords: number
  duration: number  // milliseconds
  throughput: number  // records/second
  backupId?: string
  errorLog?: MigrationError[]
  summary: string
}

interface MigrationProgress {
  currentStep: WizardStep
  currentBatch: number
  totalBatches: number
  processedRecords: number
  totalRecords: number
  failedRecords: number
  percentage: number
  eta: number  // milliseconds remaining
  throughput: number  // records/second
  startTime: Date
  errors: MigrationError[]
}

interface MigrationError {
  recordId: string
  recordIndex: number
  error: string
  stackTrace?: string
  timestamp: Date
  retryable: boolean
  retryCount: number
}

type WizardStep =
  | 'select'       // Select source and target versions
  | 'review'       // Review migration plan and impact
  | 'configure'    // Configure options (batch size, dry-run, backup)
  | 'execute'      // Execute migration with progress tracking
  | 'verify'       // Verify results and offer rollback

interface CustomWizardAction {
  id: string
  label: string
  icon?: React.ReactNode
  onClick: () => void
  disabled?: boolean
  showOnSteps?: WizardStep[]
}

// ============================================================
// Component State
// ============================================================

interface MigrationWizardState {
  // Wizard state
  currentStep: WizardStep
  canProceed: boolean
  canGoBack: boolean

  // Configuration state
  batchSize: number
  dryRun: boolean
  createBackup: boolean
  backupId: string | null

  // Data state
  totalRecords: number
  affectedRecords: number
  migrationPlan: MigrationPlan | null

  // Execution state
  isExecuting: boolean
  isPaused: boolean
  progress: MigrationProgress | null
  result: MigrationResult | null

  // Error state
  errors: MigrationError[]
  fatalError: string | null

  // UI state
  showCancelConfirmation: boolean
  showRollbackConfirmation: boolean
  expandedSections: Set<string>
}

interface MigrationPlan {
  breakingChanges: BreakingChange[]
  estimatedDuration: number  // milliseconds
  estimatedBatches: number
  transformationsRequired: string[]
  risks: string[]
  recommendations: string[]
}
```

---

## Architecture

### Component Hierarchy

```
MigrationWizard
├── WizardHeader
│   ├── StepIndicator (shows "Step 2 of 5")
│   ├── SchemaVersionBadge (fromVersion → toVersion)
│   └── CloseButton
├── WizardBody (dynamic based on currentStep)
│   ├── Step1_SelectVersions
│   │   ├── VersionSelector (source)
│   │   ├── VersionSelector (target)
│   │   └── CompatibilitySummary
│   ├── Step2_ReviewPlan
│   │   ├── MigrationSummary (duration, batches, records)
│   │   ├── BreakingChangesList
│   │   ├── ImpactAnalysisChart
│   │   └── TransformationPreview
│   ├── Step3_Configure
│   │   ├── BatchSizeInput
│   │   ├── DryRunToggle
│   │   ├── BackupToggle
│   │   ├── RetrySettingsPanel
│   │   └── AdvancedOptionsCollapsible
│   ├── Step4_Execute
│   │   ├── ProgressBar
│   │   ├── ProgressChart (throughput over time)
│   │   ├── MetricsPanel (processed, failed, ETA)
│   │   ├── LiveLogViewer
│   │   └── PauseResumeButtons
│   └── Step5_Verify
│       ├── ResultsSummary (success/failure counts)
│       ├── ValidationReport
│       ├── ErrorLogViewer
│       ├── RollbackButton
│       └── ExportReportButton
└── WizardFooter
    ├── BackButton
    ├── CancelButton
    └── NextButton / CompleteButton
```

### State Machine

```
┌──────────┐
│  SELECT  │ ← Step 1: Choose versions
└────┬─────┘
     │ [Next]
     ▼
┌──────────┐
│  REVIEW  │ ← Step 2: Review migration plan
└────┬─────┘
     │ [Next]
     ▼
┌───────────┐
│ CONFIGURE │ ← Step 3: Set options
└────┬──────┘
     │ [Start Migration]
     ▼
┌──────────┐
│ EXECUTE  │ ← Step 4: Run migration
└────┬─────┘   (can pause/resume)
     │
     ├─ Success ────────┐
     │                  │
     │ Failure          ▼
     │            ┌──────────┐
     └───────────▶│  VERIFY  │ ← Step 5: Review results
                  └────┬─────┘
                       │
                       ├─ [Rollback] ──┐
                       │               │
                       │ [Complete]    ▼
                       ▼         ┌──────────┐
                    [Done]       │ ROLLBACK │
                                 └──────────┘
                                       │
                                       │ [Restore Complete]
                                       ▼
                                  ┌──────────┐
                                  │  REVIEW  │ ← Back to review step
                                  └──────────┘
```

### Data Flow

```
User clicks "Start Migration"
    ↓
[Step 1: SELECT]
    ↓ User selects fromVersion and toVersion
    ↓
Fetch schema definitions
Analyze breaking changes
Calculate impact
    ↓
[Step 2: REVIEW]
    ↓ User reviews plan
    ↓ Click "Next"
    ↓
[Step 3: CONFIGURE]
    ↓ User sets batchSize, dryRun, backup options
    ↓ Click "Start Migration"
    ↓
Create backup (if enabled)
    ↓
[Step 4: EXECUTE]
    ↓
For each batch:
    ├─ Fetch records from database
    ├─ Transform each record (apply transformRecord function)
    ├─ Validate transformed records
    ├─ Apply migration (update database)
    ├─ Update progress (processedRecords, failures)
    └─ Emit onProgress event
    ↓
All batches complete
    ↓
Run post-migration validation
    ↓
[Step 5: VERIFY]
    ↓
User reviews results
    ├─ If satisfied: Click "Complete"
    ├─ If issues: Click "Rollback"
    └─ If errors: Review error log, fix issues, retry
```

---

## Wizard Steps

### Step 1: Select Versions

**Purpose:** Choose source and target schema versions for migration.

**UI Elements:**
- Version selector dropdowns (from/to)
- Compatibility summary badge
- Quick stats (breaking changes count, affected records)

**Validation:**
- Ensure `fromVersion` ≠ `toVersion`
- Ensure `toVersion` exists in registry
- Check user has permission to migrate

**Example UI:**
```
┌────────────────────────────────────────────────────┐
│ Select Schema Versions                             │
├────────────────────────────────────────────────────┤
│                                                    │
│ Migrate From:                                      │
│ [v2.1.0 ▼]                                         │
│                                                    │
│         ↓ Migration Direction                      │
│                                                    │
│ Migrate To:                                        │
│ [v3.0.0 ▼]                                         │
│                                                    │
│ ┌──────────────────────────────────────────────┐   │
│ │ Compatibility Summary                        │   │
│ │ 🔴 Breaking Changes: 3                       │   │
│ │ 🟢 Additive Changes: 2                       │   │
│ │ 📊 Affected Records: ~1,234 (estimated)      │   │
│ └──────────────────────────────────────────────┘   │
│                                                    │
│                                [Cancel] [Next >]   │
└────────────────────────────────────────────────────┘
```

### Step 2: Review Migration Plan

**Purpose:** Display detailed migration plan and impact analysis.

**UI Elements:**
- Migration summary (duration estimate, batch count)
- Breaking changes list with severity
- Impact analysis chart (pie chart of affected vs. unaffected records)
- Transformation preview (sample record transformation)

**Data Displayed:**
- Estimated duration: "~5 minutes"
- Total batches: "2 batches (1,000 records each)"
- Breaking changes: List with descriptions
- Affected records: "1,234 out of 10,000 (12%)"

**Example UI:**
```
┌────────────────────────────────────────────────────────────┐
│ Review Migration Plan                                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Migration Summary                                          │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Schema: payment_event_schema                         │   │
│ │ From: v2.1.0  →  To: v3.0.0                          │   │
│ │ Estimated Duration: ~5 minutes                       │   │
│ │ Total Records: 10,000                                │   │
│ │ Affected Records: 1,234 (12%)                        │   │
│ │ Batches: 2 (1,000 records/batch)                     │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Breaking Changes (3)                                       │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 🔴 HIGH: Field 'paymentMethod' enum reduced          │   │
│ │    Removed values: ['paypal']                        │   │
│ │    Affected: 234 records                             │   │
│ │    Transformation: Map 'paypal' → 'bank_transfer'    │   │
│ │                                                      │   │
│ │ 🟡 MEDIUM: Field 'amount' constraint added           │   │
│ │    Added: minimum: 0                                 │   │
│ │    Affected: 12 records (have negative amounts)      │   │
│ │    Transformation: Set to 0 or reject record         │   │
│ │                                                      │   │
│ │ 🟡 MEDIUM: Field 'metadata' removed                  │   │
│ │    Affected: 988 records                             │   │
│ │    Transformation: Data will be archived             │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Impact Analysis                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │     ████████████████░░░░ 12% affected                │   │
│ │                                                      │   │
│ │     10,000 total records                             │   │
│ │      ├─ 8,766 unaffected (no changes needed)         │   │
│ │      └─ 1,234 affected (require transformation)      │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Transformation Preview                                     │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Sample Record (before):                              │   │
│ │ {                                                    │   │
│ │   "transactionId": "txn_123",                        │   │
│ │   "paymentMethod": "paypal",  ← Will be transformed  │   │
│ │   "amount": 50.00,                                   │   │
│ │   "metadata": { ... }  ← Will be removed             │   │
│ │ }                                                    │   │
│ │                                                      │   │
│ │ Sample Record (after):                               │   │
│ │ {                                                    │   │
│ │   "transactionId": "txn_123",                        │   │
│ │   "paymentMethod": "bank_transfer",  ← Transformed   │   │
│ │   "amount": 50.00                                    │   │
│ │ }                                                    │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│                              [< Back] [Cancel] [Next >]   │
└────────────────────────────────────────────────────────────┘
```

### Step 3: Configure Options

**Purpose:** Set migration parameters (batch size, dry-run, backup, retries).

**UI Elements:**
- Batch size input (with recommendation)
- Dry-run toggle (enabled by default)
- Backup toggle (enabled by default)
- Retry settings (max retries, retry delay)
- Advanced options (collapsible): parallel processing, timeout, notifications

**Validation:**
- Batch size: 1 ≤ batchSize ≤ 10,000
- Timeout: ≥ 60 seconds
- Retry count: 0 ≤ maxRetries ≤ 10

**Example UI:**
```
┌────────────────────────────────────────────────────────────┐
│ Configure Migration Options                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Batch Processing                                           │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Batch Size: [1000] records/batch                     │   │
│ │ Recommended: 1,000 for datasets <100k records        │   │
│ │ Total Batches: 10                                    │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Safety Options                                             │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ ☑ Dry Run (recommended)                              │   │
│ │   Test migration without committing changes          │   │
│ │                                                      │   │
│ │ ☑ Create Backup Before Migration                    │   │
│ │   Strategy: Database Snapshot                        │   │
│ │   Estimated Size: ~2.3 GB                            │   │
│ │   Backup Duration: ~30 seconds                       │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Error Handling                                             │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ ☑ Retry Failed Records                               │   │
│ │   Max Retries: [3]                                   │   │
│ │   Retry Delay: [1000] ms                             │   │
│ │                                                      │   │
│ │ ☑ Continue on Error (don't abort entire migration)  │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ ▼ Advanced Options                                         │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ ☐ Parallel Processing (3 workers)                   │   │
│ │ ☐ Streaming Mode (process as data arrives)          │   │
│ │ Migration Timeout: [3600] seconds                    │   │
│ │ Notifications: ☑ Email on completion                 │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Summary                                                    │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ • Dry run enabled (no data will be modified)         │   │
│ │ • Backup will be created before migration            │   │
│ │ • Failed records will be retried up to 3 times       │   │
│ │ • Estimated total time: ~5 minutes (+ 30s backup)    │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│                    [< Back] [Cancel] [Start Migration]   │
└────────────────────────────────────────────────────────────┘
```

### Step 4: Execute Migration

**Purpose:** Run migration with real-time progress tracking.

**UI Elements:**
- Progress bar (overall completion)
- Metrics panel (processed, failed, success rate, ETA)
- Throughput chart (records/second over time)
- Live log viewer (scrollable, filterable)
- Pause/Resume/Cancel buttons

**States:**
- `running`: Migration in progress
- `paused`: User paused migration
- `cancelling`: Cancellation in progress (completing current batch)
- `completed`: Migration finished

**Example UI (Running):**
```
┌────────────────────────────────────────────────────────────┐
│ Executing Migration (Dry Run)                              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Progress                                                   │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ ████████████████████░░░░░░░░░░░ 68% (6,800 / 10,000) │   │
│ │ Batch 7 of 10                                        │   │
│ │ ETA: 1 minute 23 seconds                             │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Metrics                                                    │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Processed: 6,800 records                             │   │
│ │ Succeeded: 6,785 (99.8%)                             │   │
│ │ Failed: 15 (0.2%)                                    │   │
│ │ Skipped: 0                                           │   │
│ │                                                      │   │
│ │ Throughput: 142 records/sec                          │   │
│ │ Elapsed Time: 48 seconds                             │   │
│ │ Estimated Remaining: 1m 23s                          │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Throughput Chart                                           │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 200 ┤                                 ╭──╮            │   │
│ │ 150 ┤           ╭────╮          ╭────╯  ╰─╮          │   │
│ │ 100 ┤      ╭────╯    ╰─╮   ╭────╯         ╰─╮        │   │
│ │  50 ┤ ╭────╯           ╰───╯                ╰─       │   │
│ │   0 ┴──────────────────────────────────────────       │   │
│ │     0s   10s   20s   30s   40s   50s                 │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Live Log (latest events)                                   │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 12:34:56 [INFO] Processing batch 7 (6000-6999)       │   │
│ │ 12:34:57 [SUCCESS] Transformed 1000 records          │   │
│ │ 12:34:58 [WARNING] Record txn_6543: Invalid amount   │   │
│ │ 12:34:58 [INFO] Retrying txn_6543 (attempt 1/3)      │   │
│ │ 12:34:59 [SUCCESS] Batch 7 complete (995 succeeded)  │   │
│ │ 12:34:59 [INFO] Starting batch 8 (7000-7999)         │   │
│ └──────────────────────────────────────────────────────┘   │
│                                    [View Full Log]        │
│                                                            │
│                         [Pause] [Cancel] [Continue in BG] │
└────────────────────────────────────────────────────────────┘
```

**Example UI (Completed):**
```
┌────────────────────────────────────────────────────────────┐
│ Migration Complete                                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ ✅ Dry run completed successfully                          │
│                                                            │
│ Final Stats                                                │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Total Records: 10,000                                │   │
│ │ Successfully Transformed: 9,985 (99.85%)             │   │
│ │ Failed: 15 (0.15%)                                   │   │
│ │ Skipped: 0                                           │   │
│ │                                                      │   │
│ │ Duration: 1 minute 10 seconds                        │   │
│ │ Average Throughput: 143 records/sec                  │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ ⚠️  15 records failed transformation:                      │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ • txn_1234: Invalid amount (-50.00)                  │   │
│ │ • txn_5678: Missing required field 'currency'        │   │
│ │ • txn_9012: ... (13 more errors)                     │   │
│ │                              [View Error Log]        │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Next Steps                                                 │
│ • Fix errors in source data                                │
│ • Adjust transformation logic                              │
│ • Re-run migration with backup disabled                    │
│                                                            │
│                    [< Back to Config] [Cancel] [Verify >] │
└────────────────────────────────────────────────────────────┘
```

### Step 5: Verify Results

**Purpose:** Review migration results and decide next action (complete, rollback, or retry).

**UI Elements:**
- Results summary (success/failure counts)
- Validation report (schema compliance check)
- Error log viewer (detailed error messages)
- Sample transformed records viewer
- Action buttons: Complete, Rollback, Export Report

**Example UI (Success):**
```
┌────────────────────────────────────────────────────────────┐
│ Verify Migration Results                                   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ ✅ Migration Completed Successfully                         │
│                                                            │
│ Summary                                                    │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Total Records: 10,000                                │   │
│ │ Migrated: 10,000 (100%)                              │   │
│ │ Failed: 0 (0%)                                       │   │
│ │ Skipped: 0                                           │   │
│ │                                                      │   │
│ │ Duration: 4 minutes 32 seconds                       │   │
│ │ Backup ID: backup_20251025_123456                    │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Validation Report                                          │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ ✅ All migrated records comply with v3.0.0 schema    │   │
│ │ ✅ No orphaned records found                         │   │
│ │ ✅ All required fields present                       │   │
│ │ ✅ No data type mismatches                           │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Sample Transformed Records                                 │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Showing 3 of 10,000 records                          │   │
│ │                                                      │   │
│ │ txn_123: paymentMethod: paypal → bank_transfer ✓    │   │
│ │ txn_456: paymentMethod: paypal → bank_transfer ✓    │   │
│ │ txn_789: paymentMethod: card → card (unchanged) ✓    │   │
│ │                                                      │   │
│ │                              [View All Records]      │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Actions                                                    │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ [📊 Export Report] [🔄 Rollback] [✅ Complete]       │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ ⚠️  Rollback will restore all records to v2.1.0            │
│    This operation cannot be undone.                        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Example UI (With Errors):**
```
┌────────────────────────────────────────────────────────────┐
│ Verify Migration Results                                   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ ⚠️  Migration Completed with Errors                         │
│                                                            │
│ Summary                                                    │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Total Records: 10,000                                │   │
│ │ Migrated: 9,985 (99.85%)                             │   │
│ │ Failed: 15 (0.15%)                                   │   │
│ │ Skipped: 0                                           │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Error Log (15 errors)                                      │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 🔴 txn_1234 (batch 2): Invalid amount: -50.00        │   │
│ │    Error: Value fails minimum constraint (min: 0)    │   │
│ │    Retried: 3 times                                  │   │
│ │    Recommendation: Update source record or adjust    │   │
│ │    transformation logic                              │   │
│ │                                                      │   │
│ │ 🔴 txn_5678 (batch 5): Missing required field        │   │
│ │    Error: 'currency' is required but undefined       │   │
│ │    ... (13 more errors)                              │   │
│ │                                                      │   │
│ │               [Download Error Log CSV]               │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Options                                                    │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 1. Complete migration (accept 15 failed records)     │   │
│ │    Failed records remain on v2.1.0 schema            │   │
│ │                                                      │   │
│ │ 2. Rollback entire migration                         │   │
│ │    Restore all 10,000 records to v2.1.0              │   │
│ │                                                      │   │
│ │ 3. Fix errors and retry                              │   │
│ │    Go back to config, adjust settings, re-run        │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│           [Rollback] [Retry Failed Records] [Complete]   │
└────────────────────────────────────────────────────────────┘
```

---

## Visual Wireframes

### Wireframe 1: Complete Wizard Flow (All Steps)

```
┌─────────────────────────────────────────────────────────────────┐
│ Migration Wizard - payment_event_schema                         │
│ Step 1 of 5: Select Versions                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ●━━━○━━━○━━━○━━━○   Progress: Select → Review → Config ...    │
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Migrate From:                                             │   │
│ │ ┌────────────┐                                            │   │
│ │ │ v2.1.0   ▼ │                                            │   │
│ │ └────────────┘                                            │   │
│ │                                                           │   │
│ │         ↓  (Migration Direction)                          │   │
│ │                                                           │   │
│ │ Migrate To:                                               │   │
│ │ ┌────────────┐                                            │   │
│ │ │ v3.0.0   ▼ │                                            │   │
│ │ └────────────┘                                            │   │
│ │                                                           │   │
│ │ ┌─────────────────────────────────────────────────────┐   │   │
│ │ │ Compatibility Summary                               │   │   │
│ │ │ 🔴 Breaking Changes: 3                              │   │   │
│ │ │ 🟢 Additive Changes: 2                              │   │   │
│ │ │ 📊 Affected Records: ~1,234 (12%)                   │   │   │
│ │ └─────────────────────────────────────────────────────┘   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│                                        [Cancel]  [Next >]      │
└─────────────────────────────────────────────────────────────────┘

(After clicking Next...)

┌─────────────────────────────────────────────────────────────────┐
│ Migration Wizard - payment_event_schema                         │
│ Step 2 of 5: Review Migration Plan                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ●━━━●━━━○━━━○━━━○   Progress: Select → Review → Config ...    │
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Migration Summary                                         │   │
│ │ • Duration: ~5 minutes                                    │   │
│ │ • Total Records: 10,000                                   │   │
│ │ • Affected: 1,234 (12%)                                   │   │
│ │ • Batches: 10 (1,000/batch)                               │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│ Breaking Changes (3)                                            │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ 🔴 paymentMethod enum reduced (removed 'paypal')          │   │
│ │    234 records affected                                   │   │
│ │                                                           │   │
│ │ 🟡 amount constraint added (minimum: 0)                   │   │
│ │    12 records affected                                    │   │
│ │                                                           │   │
│ │ 🟡 metadata field removed                                 │   │
│ │    988 records affected                                   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│ Transformation Preview                                          │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Before: { "paymentMethod": "paypal", ... }               │   │
│ │ After:  { "paymentMethod": "bank_transfer", ... }        │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│                                  [< Back] [Cancel] [Next >]    │
└─────────────────────────────────────────────────────────────────┘

(And so on for steps 3, 4, 5...)
```

### Wireframe 2: Error Handling Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Migration Wizard - Execution Error                              │
│ Step 4 of 5: Execute Migration                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ❌ Migration Paused - Critical Error                            │
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Error Details                                             │   │
│ │ ┌─────────────────────────────────────────────────────┐   │   │
│ │ │ Connection to database lost                         │   │   │
│ │ │ Time: 12:34:56                                      │   │   │
│ │ │ Batch: 5 of 10                                      │   │   │
│ │ │ Records Processed: 4,832                            │   │   │
│ │ │                                                     │   │   │
│ │ │ Stack Trace:                                        │   │   │
│ │ │   at applyMigration (migration.ts:245)              │   │   │
│ │ │   at processBatch (migration.ts:189)                │   │   │
│ │ │   ...                                               │   │   │
│ │ └─────────────────────────────────────────────────────┘   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│ Recovery Options                                                │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ○ Retry from last successful batch                       │   │
│ │   Resume from batch 5 (4,000 records remaining)          │   │
│ │                                                           │   │
│ │ ○ Rollback to backup                                     │   │
│ │   Restore all 10,000 records to v2.1.0                   │   │
│ │                                                           │   │
│ │ ○ Abort migration                                        │   │
│ │   Keep successfully migrated records (4,832)             │   │
│ │   Leave remaining records on v2.1.0                      │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│                         [Rollback] [Retry] [Abort]             │
└─────────────────────────────────────────────────────────────────┘
```

### Wireframe 3: Rollback Confirmation Dialog

```
┌─────────────────────────────────────────────────────────────────┐
│ Confirm Rollback                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ⚠️  Are you sure you want to rollback this migration?           │
│                                                                 │
│ This will:                                                      │
│ • Restore all 10,000 records to schema v2.1.0                   │
│ • Discard all changes made during migration                     │
│ • Use backup: backup_20251025_123456                            │
│ • Take approximately 2 minutes                                  │
│                                                                 │
│ This operation cannot be undone.                                │
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Type "ROLLBACK" to confirm:                               │   │
│ │ ┌─────────────────────────────────────────────────────┐   │   │
│ │ │                                                     │   │   │
│ │ └─────────────────────────────────────────────────────┘   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│                                  [Cancel] [Confirm Rollback]   │
└─────────────────────────────────────────────────────────────────┘
```

---

## States & Lifecycle

### Component States

```typescript
type WizardState =
  | 'idle'                  // Initial state, wizard not started
  | 'selecting'             // Step 1: Selecting versions
  | 'loading_plan'          // Fetching migration plan
  | 'reviewing'             // Step 2: Reviewing plan
  | 'configuring'           // Step 3: Configuring options
  | 'creating_backup'       // Creating backup before execution
  | 'executing'             // Step 4: Migration in progress
  | 'paused'                // Execution paused by user
  | 'verifying'             // Step 5: Verifying results
  | 'rolling_back'          // Rollback in progress
  | 'completed'             // Migration completed successfully
  | 'failed'                // Migration failed
  | 'cancelled'             // User cancelled migration
```

### State Transitions

```
[idle]
  │
  ├─ User opens wizard ──▶ [selecting]
  │
[selecting]
  │
  ├─ User selects versions & clicks Next ──▶ [loading_plan]
  │
[loading_plan]
  │
  ├─ Plan loaded ──▶ [reviewing]
  ├─ Error loading ──▶ [failed]
  │
[reviewing]
  │
  ├─ User clicks Next ──▶ [configuring]
  ├─ User clicks Back ──▶ [selecting]
  │
[configuring]
  │
  ├─ User clicks Start Migration ──▶ [creating_backup] (if enabled)
  ├─ User clicks Start Migration ──▶ [executing] (if backup disabled)
  │
[creating_backup]
  │
  ├─ Backup complete ──▶ [executing]
  ├─ Backup failed ──▶ [failed]
  │
[executing]
  │
  ├─ User clicks Pause ──▶ [paused]
  ├─ Migration completes ──▶ [verifying]
  ├─ Critical error ──▶ [failed]
  ├─ User clicks Cancel ──▶ [cancelled]
  │
[paused]
  │
  ├─ User clicks Resume ──▶ [executing]
  ├─ User clicks Cancel ──▶ [cancelled]
  │
[verifying]
  │
  ├─ User clicks Complete ──▶ [completed]
  ├─ User clicks Rollback ──▶ [rolling_back]
  ├─ User clicks Retry ──▶ [configuring]
  │
[rolling_back]
  │
  ├─ Rollback complete ──▶ [reviewing]
  ├─ Rollback failed ──▶ [failed]
  │
[completed] ──▶ Close wizard
[failed] ──▶ Show error, offer retry
[cancelled] ──▶ Close wizard
```

---

## Multi-Domain Examples

### 1. Finance: Transaction Schema Migration

**Scenario:** Fintech company deprecating PayPal payment method after partnership ended.

**Migration Details:**
- **From:** v2.1.0
- **To:** v3.0.0
- **Breaking Change:** Remove 'paypal' from `paymentMethod` enum
- **Affected Records:** 234 out of 10,000 transactions
- **Transformation:** Map all PayPal transactions to 'bank_transfer'

**MigrationWizard Configuration:**
```typescript
<MigrationWizard
  schemaId="payment_event_schema"
  fromVersion="2.1.0"
  toVersion="3.0.0"
  breakingChanges={[
    {
      type: 'enum_reduced',
      path: '$.properties.paymentMethod.enum',
      description: "Removed 'paypal' from payment methods",
      impact: 'high',
      affectedRecords: 234,
      transformationRequired: true
    }
  ]}
  transformRecord={(record) => {
    if (record.paymentMethod === 'paypal') {
      return {
        ...record,
        paymentMethod: 'bank_transfer',
        migrationNotes: 'Converted from PayPal to bank transfer'
      }
    }
    return record
  }}
  onComplete={(result) => {
    notify.success(`Migrated ${result.migratedRecords} transactions`)
  }}
/>
```

**Migration Plan:**
- Estimated Duration: 2 minutes
- Batches: 10 (1,000 records each)
- Transformation: Simple field mapping
- Risk: Low (deterministic transformation)

### 2. Healthcare: Patient Record Schema Migration

**Scenario:** Hospital adding mandatory HIPAA audit fields to patient records.

**Migration Details:**
- **From:** v1.5.0
- **To:** v2.0.0
- **Breaking Change:** Add required `lastAccessedBy` and `lastAccessedAt` fields
- **Affected Records:** All 50,000 patient records
- **Transformation:** Populate with system user and current timestamp

**MigrationWizard Configuration:**
```typescript
<MigrationWizard
  schemaId="patient_record_schema"
  fromVersion="1.5.0"
  toVersion="2.0.0"
  breakingChanges={[
    {
      type: 'constraint_added',
      path: '$.required',
      description: 'Added required audit fields',
      impact: 'high',
      affectedRecords: 50000,
      transformationRequired: true
    }
  ]}
  transformRecord={(record) => {
    return {
      ...record,
      lastAccessedBy: 'system_migration',
      lastAccessedAt: new Date().toISOString(),
      migrationDate: new Date().toISOString()
    }
  }}
  defaultBatchSize={5000}  // Larger batches for simple transformations
  autoBackup={true}
  onComplete={(result) => {
    auditLog.record('patient_schema_migration', {
      version: '2.0.0',
      recordsAffected: result.migratedRecords
    })
  }}
/>
```

**Migration Plan:**
- Estimated Duration: 5 minutes
- Batches: 10 (5,000 records each)
- Transformation: Add static fields
- Risk: Low (no data loss, additive change)

### 3. Legal: Case Document Schema Migration

**Scenario:** Law firm enforcing stricter case number format after data quality issues.

**Migration Details:**
- **From:** v2.3.0
- **To:** v3.0.0
- **Breaking Change:** Add pattern validation to `caseNumber` field
- **Affected Records:** 567 out of 5,000 cases (invalid format)
- **Transformation:** Reformat or reject invalid case numbers

**MigrationWizard Configuration:**
```typescript
<MigrationWizard
  schemaId="legal_case_schema"
  fromVersion="2.3.0"
  toVersion="3.0.0"
  breakingChanges={[
    {
      type: 'constraint_added',
      path: '$.properties.caseNumber.pattern',
      description: 'Enforced case number format: XX-YYYY-NNNNNN',
      impact: 'high',
      affectedRecords: 567,
      transformationRequired: true
    }
  ]}
  transformRecord={(record) => {
    const caseNumber = record.caseNumber
    const pattern = /^[A-Z]{2}-[0-9]{4}-[0-9]{6}$/

    if (pattern.test(caseNumber)) {
      return record  // Already valid
    }

    // Attempt to reformat
    const reformatted = reformatCaseNumber(caseNumber)
    if (reformatted && pattern.test(reformatted)) {
      return {
        ...record,
        caseNumber: reformatted,
        originalCaseNumber: caseNumber
      }
    }

    // Cannot reformat - skip this record
    return null
  }}
  validateTransformedRecord={(record) => {
    const pattern = /^[A-Z]{2}-[0-9]{4}-[0-9]{6}$/
    return pattern.test(record.caseNumber) ||
      'Case number does not match required format'
  }}
  onComplete={(result) => {
    if (result.skippedRecords > 0) {
      alert(`${result.skippedRecords} cases require manual correction`)
    }
  }}
/>
```

**Migration Plan:**
- Estimated Duration: 1 minute
- Batches: 5 (1,000 records each)
- Transformation: Complex (pattern matching + reformatting)
- Risk: Medium (some records may be skipped)

### 4. Research: Experiment Result Schema Migration

**Scenario:** Research lab adding reproducibility metadata to all experiments.

**Migration Details:**
- **From:** v1.0.0
- **To:** v1.1.0
- **Breaking Change:** None (additive only)
- **Affected Records:** All 1,200 experiments
- **Transformation:** Add default reproducibility metadata

**MigrationWizard Configuration:**
```typescript
<MigrationWizard
  schemaId="experiment_result_schema"
  fromVersion="1.0.0"
  toVersion="1.1.0"
  breakingChanges={[]}  // No breaking changes
  transformRecord={(record) => {
    return {
      ...record,
      reproducibility: {
        randomSeed: null,
        environmentVersion: 'unknown',
        dependencies: [],
        migrationNote: 'Added by schema migration'
      }
    }
  }}
  defaultDryRun={false}  // No breaking changes, safe to run directly
  autoBackup={false}  // No need for backup (additive change)
  onComplete={(result) => {
    console.log('Experiment metadata migration complete')
  }}
/>
```

**Migration Plan:**
- Estimated Duration: 30 seconds
- Batches: 2 (600 records each)
- Transformation: Simple (add nested object)
- Risk: Very Low (no breaking changes)

### 5. E-commerce: Product Schema Migration

**Scenario:** E-commerce platform adding support for product variants.

**Migration Details:**
- **From:** v2.0.0
- **To:** v2.1.0
- **Breaking Change:** None (new optional field)
- **Affected Records:** None (no existing data needs transformation)
- **Transformation:** No transformation needed

**MigrationWizard Configuration:**
```typescript
<MigrationWizard
  schemaId="product_schema"
  fromVersion="2.0.0"
  toVersion="2.1.0"
  breakingChanges={[]}
  transformRecord={(record) => record}  // No transformation
  defaultDryRun={false}
  estimatedRecordCount={0}  // No records need migration
  onComplete={(result) => {
    console.log('Schema updated, no data migration needed')
  }}
/>
```

**Migration Plan:**
- Estimated Duration: Instant (schema-only update)
- No data transformation required
- Risk: None

### 6. SaaS: API Event Schema Migration

**Scenario:** SaaS platform deprecating old event types.

**Migration Details:**
- **From:** v3.5.0
- **To:** v4.0.0
- **Breaking Change:** Remove 'user.login' and 'user.logout' events
- **Affected Records:** 45,678 events
- **Transformation:** Map to new 'session.started' and 'session.ended' events

**MigrationWizard Configuration:**
```typescript
<MigrationWizard
  schemaId="api_event_schema"
  fromVersion="3.5.0"
  toVersion="4.0.0"
  breakingChanges={[
    {
      type: 'enum_reduced',
      path: '$.properties.eventType.enum',
      description: 'Replaced user.login/logout with session events',
      impact: 'high',
      affectedRecords: 45678,
      transformationRequired: true
    }
  ]}
  transformRecord={(record) => {
    const eventTypeMap = {
      'user.login': 'session.started',
      'user.logout': 'session.ended'
    }

    if (record.eventType in eventTypeMap) {
      return {
        ...record,
        eventType: eventTypeMap[record.eventType],
        legacyEventType: record.eventType
      }
    }

    return record
  }}
  defaultBatchSize={10000}  // Large batches for simple transformations
  parallelProcessing={true}  // Enable parallel processing
  parallelWorkers={5}
  onComplete={(result) => {
    analytics.track('schema_migration_complete', {
      schema: 'api_event',
      version: '4.0.0',
      records: result.migratedRecords
    })
  }}
/>
```

**Migration Plan:**
- Estimated Duration: 3 minutes
- Batches: 5 (10,000 records each)
- Transformation: Simple mapping
- Risk: Low (deterministic transformation)

### 7. Insurance: Claim Schema Migration

**Scenario:** Insurance company adding fraud detection scores to all claims.

**Migration Details:**
- **From:** v1.8.0
- **To:** v1.9.0
- **Breaking Change:** None (additive)
- **Affected Records:** All 25,000 claims
- **Transformation:** Calculate and add fraud scores retroactively

**MigrationWizard Configuration:**
```typescript
<MigrationWizard
  schemaId="insurance_claim_schema"
  fromVersion="1.8.0"
  toVersion="1.9.0"
  breakingChanges={[]}
  transformRecord={async (record) => {
    // Call ML service to calculate fraud score
    const fraudScore = await mlService.calculateFraudScore(record)

    return {
      ...record,
      fraudScore: fraudScore.score,
      flaggedReasons: fraudScore.reasons
    }
  }}
  defaultBatchSize={100}  // Smaller batches (API calls)
  parallelProcessing={true}
  parallelWorkers={3}
  migrationTimeout={7200000}  // 2 hours (API calls are slow)
  onProgress={(progress) => {
    console.log(`Processed ${progress.processedRecords} claims`)
  }}
  onComplete={(result) => {
    notify.success(`Added fraud scores to ${result.migratedRecords} claims`)
  }}
/>
```

**Migration Plan:**
- Estimated Duration: 45 minutes (slow due to ML API calls)
- Batches: 250 (100 records each)
- Transformation: Complex (async API calls)
- Risk: Medium (API availability, rate limits)

---

## User Interactions

### 1. Starting Migration Wizard

**Trigger:** User clicks "Migrate Schema" button on schema details page

**Flow:**
1. Open MigrationWizard modal/page
2. Initialize wizard with schemaId, fromVersion, toVersion
3. Fetch schema definitions from registry
4. Calculate breaking changes and impact
5. Display Step 1 (Select Versions)

### 2. Selecting Versions (Step 1)

**Interactions:**
- Dropdown select for fromVersion (shows all published versions)
- Dropdown select for toVersion (shows versions > fromVersion)
- Compatibility summary updates when versions change

**Validation:**
- fromVersion must be different from toVersion
- toVersion must exist in registry
- User must have permission to migrate

**Next Action:** Click "Next" → Move to Step 2

### 3. Reviewing Migration Plan (Step 2)

**Interactions:**
- Scroll through breaking changes list
- Click on breaking change to see details
- Hover over "Affected Records" to see record IDs
- Click "View Transformation Preview" to see sample transformations

**Information Displayed:**
- Estimated duration
- Total batches
- Breaking changes with severity
- Impact analysis (percentage affected)
- Sample record transformations

**Next Action:** Click "Next" → Move to Step 3

### 4. Configuring Options (Step 3)

**Interactions:**
- Adjust batch size slider (1-10,000)
- Toggle dry-run mode (ON by default)
- Toggle backup creation (ON by default)
- Expand "Advanced Options" to set parallel processing, timeout, etc.
- Toggle retry settings

**Real-time Feedback:**
- Estimated duration updates based on batch size
- Warning if batch size too large: "Large batches may cause timeouts"
- Info tooltip: "Dry run mode tests migration without committing changes"

**Validation:**
- Batch size must be 1-10,000
- If backup disabled, show warning: "Disabling backup is risky"

**Next Action:** Click "Start Migration" → Create backup (if enabled) → Move to Step 4

### 5. Executing Migration (Step 4)

**Real-time Updates:**
- Progress bar updates every second
- Metrics panel shows processed/failed counts
- Throughput chart updates every 5 seconds
- Live log scrolls automatically (can be paused)

**User Controls:**
- **Pause:** Pauses migration after current batch completes
- **Resume:** Resumes from where it was paused
- **Cancel:** Shows confirmation dialog, then cancels migration
- **Continue in Background:** Minimizes wizard, shows notification when complete

**Error Handling:**
- If error occurs, pause migration and show error dialog
- User can choose: Retry, Rollback, or Abort

**Completion:**
- When all batches complete, automatically move to Step 5

### 6. Verifying Results (Step 5)

**Information Displayed:**
- Success/failure counts
- Validation report (schema compliance)
- Sample transformed records
- Error log (if any failures)

**Actions:**
- **Export Report:** Download PDF/CSV with migration summary
- **Rollback:** Restore all records to previous version (requires confirmation)
- **Retry Failed Records:** Go back to Step 3, re-run for failed records only
- **Complete:** Close wizard, apply schema version to all records

**Rollback Confirmation:**
```
Are you sure you want to rollback?
This will restore all 10,000 records to v2.1.0.
Type "ROLLBACK" to confirm.

[Text Input]

[Cancel] [Confirm Rollback]
```

### 7. Handling Errors During Execution

**Scenario:** Database connection lost during batch 5

**Error Dialog:**
```
Migration Paused - Critical Error

Connection to database lost
Batch: 5 of 10
Records Processed: 4,832

Recovery Options:
○ Retry from last successful batch
○ Rollback to backup
○ Abort migration

[Rollback] [Retry] [Abort]
```

**User Actions:**
- **Retry:** Resume from batch 5
- **Rollback:** Restore from backup, return to Step 2
- **Abort:** Keep migrated records, leave remaining on old schema

### 8. Pausing and Resuming

**Pause Flow:**
1. User clicks "Pause"
2. Current batch finishes processing
3. State changes to `paused`
4. Show "Paused" badge and "Resume" button

**Resume Flow:**
1. User clicks "Resume"
2. State changes back to `executing`
3. Continue from next batch

### 9. Background Execution

**Flow:**
1. User clicks "Continue in Background"
2. Wizard minimizes to notification bar
3. Show notification: "Migration running in background (68% complete)"
4. When complete, show notification: "Migration completed successfully"
5. User can click notification to view results (Step 5)

---

## Accessibility

### WCAG 2.1 AA Compliance

#### 1. Keyboard Navigation

All wizard controls accessible via keyboard:

| Action | Keyboard Shortcut | Focus Behavior |
|--------|------------------|----------------|
| Next step | `Enter` or `Tab` to Next button | Focus moves to next step's first input |
| Previous step | `Shift+Tab` to Back button | Focus returns to previous step |
| Cancel wizard | `Esc` | Shows cancel confirmation dialog |
| Pause migration | `Space` on Pause button | Pauses execution |
| Navigate dropdown | `Arrow Up/Down` | Select version |

**Tab Order:**
```
Step 1: From Version Dropdown → To Version Dropdown → Next Button
Step 2: Breaking Changes List → Next Button
Step 3: Batch Size Input → Dry Run Toggle → Backup Toggle → Start Button
Step 4: Pause Button → Cancel Button → (non-interactive during execution)
Step 5: Export Button → Rollback Button → Complete Button
```

#### 2. Screen Reader Support

**ARIA Labels:**
```html
<div role="dialog" aria-labelledby="wizard-title" aria-describedby="wizard-description">
  <h1 id="wizard-title">Migration Wizard</h1>
  <p id="wizard-description">Migrate schema from v2.1.0 to v3.0.0</p>

  <div role="tablist" aria-label="Wizard steps">
    <button role="tab" aria-selected="true" aria-controls="step-1">Select Versions</button>
    <button role="tab" aria-selected="false" aria-controls="step-2">Review Plan</button>
    ...
  </div>

  <div id="step-1" role="tabpanel" aria-labelledby="tab-1">
    <label for="from-version">Migrate From</label>
    <select id="from-version" aria-describedby="from-version-help">
      <option value="2.1.0">v2.1.0</option>
    </select>
    <span id="from-version-help">Select source schema version</span>
  </div>
</div>
```

**Live Regions:**
- Progress updates: `aria-live="polite"` announces progress every 10%
- Error messages: `aria-live="assertive"` announces errors immediately
- Step transitions: `aria-live="polite"` announces step changes

**Announcements:**
```
"Step 2 of 5: Review Migration Plan"
"Migration 68% complete. 6,800 records processed."
"Error: Database connection lost. Migration paused."
"Migration completed successfully. 10,000 records migrated."
```

#### 3. Color Contrast

All colors meet WCAG AA standards (4.5:1 ratio):

| Element | Foreground | Background | Ratio |
|---------|-----------|------------|-------|
| Error text | #DC2626 | #FFFFFF | 5.8:1 |
| Warning text | #D97706 | #FFFFFF | 4.9:1 |
| Success text | #059669 | #FFFFFF | 4.7:1 |
| Progress bar (filled) | #3B82F6 | #E5E7EB | 4.6:1 |

**High Contrast Mode:**
- Increase border thickness (1px → 3px)
- Use pure black/white (#000000, #FFFFFF)
- Add patterns to progress bars (not just color)

#### 4. Focus Indicators

Visible focus indicators on all interactive elements:
```css
button:focus-visible,
input:focus-visible,
select:focus-visible {
  outline: 3px solid #2563EB;
  outline-offset: 2px;
}
```

#### 5. Error Identification

Errors identified in multiple ways:
- Icon: 🔴 (visual indicator)
- Text: "Error" label
- Color: Red text
- Border: Red border around input
- Announcement: Screen reader announcement

#### 6. Progress Announcements

Progress updates announced at regular intervals:
```typescript
// Announce every 10% progress
if (progress.percentage % 10 === 0) {
  announce(`Migration ${progress.percentage}% complete. ${progress.processedRecords} records processed.`)
}
```

---

## Performance Optimizations

### 1. Batch Processing

Process records in configurable batches to avoid memory issues:

```typescript
async function executeMigration(config: MigrationConfig) {
  const batchSize = config.batchSize
  const totalRecords = config.totalRecords
  const totalBatches = Math.ceil(totalRecords / batchSize)

  for (let i = 0; i < totalBatches; i++) {
    const offset = i * batchSize
    const batch = await fetchRecords(batchSize, offset)

    const transformedBatch = await Promise.all(
      batch.map(record => config.transformRecord(record))
    )

    await applyMigration(transformedBatch)

    updateProgress({
      currentBatch: i + 1,
      totalBatches,
      processedRecords: (i + 1) * batchSize
    })
  }
}
```

### 2. Parallel Processing

Process multiple batches concurrently (if enabled):

```typescript
async function executeMigrationParallel(config: MigrationConfig) {
  const workers = config.parallelWorkers || 3
  const batches = createBatches(config.totalRecords, config.batchSize)

  const pool = new WorkerPool(workers)

  await Promise.all(
    batches.map(batch => pool.run(() => processBatch(batch)))
  )
}
```

### 3. Debounced Progress Updates

Update UI at most once per second to avoid overwhelming React:

```typescript
const debouncedUpdateProgress = useMemo(
  () => debounce((progress: MigrationProgress) => {
    setProgress(progress)
  }, 1000),
  []
)
```

### 4. Virtualized Log Viewer

For large log files, virtualize rendering:

```typescript
import { FixedSizeList } from 'react-window'

<FixedSizeList
  height={400}
  itemCount={logEntries.length}
  itemSize={30}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>{logEntries[index]}</div>
  )}
</FixedSizeList>
```

### 5. Web Workers for Transformation

Offload heavy transformations to Web Workers:

```typescript
// transformation.worker.ts
self.addEventListener('message', (e) => {
  const { records, transformFn } = e.data

  const transformed = records.map(record => {
    // Execute transformation logic
    return transformFn(record)
  })

  self.postMessage({ transformed })
})

// In component
const transformationWorker = useRef<Worker>()

useEffect(() => {
  transformationWorker.current = new Worker('/transformation.worker.js')

  transformationWorker.current.onmessage = (e) => {
    setTransformedRecords(e.data.transformed)
  }

  return () => transformationWorker.current?.terminate()
}, [])
```

### 6. Incremental Backup

For large datasets, create incremental backups instead of full snapshots:

```typescript
async function createIncrementalBackup(fromVersion: string) {
  // Only backup records that will be modified
  const affectedRecordIds = await getAffectedRecordIds(fromVersion)

  await backupRecords(affectedRecordIds)
}
```

### 7. Streaming Mode

For very large datasets, use streaming instead of batching:

```typescript
async function executeMigrationStreaming(config: MigrationConfig) {
  const recordStream = createRecordStream()

  for await (const record of recordStream) {
    const transformed = await config.transformRecord(record)
    await saveRecord(transformed)

    updateProgress({ processedRecords: ++count })
  }
}
```

---

## Integration Examples

### Example 1: Basic React Integration

```typescript
import { MigrationWizard } from '@/components/il/MigrationWizard'
import { useState } from 'react'

export default function SchemaDetailsPage({ schema }: { schema: Schema }) {
  const [showWizard, setShowWizard] = useState(false)

  const handleMigrationComplete = (result: MigrationResult) => {
    console.log('Migration complete:', result)
    setShowWizard(false)

    // Refresh schema details
    mutate(`/api/schemas/${schema.id}`)
  }

  return (
    <div>
      <h1>{schema.name}</h1>
      <button onClick={() => setShowWizard(true)}>
        Migrate to v{schema.latestVersion}
      </button>

      {showWizard && (
        <MigrationWizard
          schemaId={schema.id}
          fromVersion={schema.currentVersion}
          toVersion={schema.latestVersion}
          onComplete={handleMigrationComplete}
          onCancel={() => setShowWizard(false)}
        />
      )}
    </div>
  )
}
```

### Example 2: With Custom Transformation Logic

```typescript
<MigrationWizard
  schemaId="payment_event_schema"
  fromVersion="2.1.0"
  toVersion="3.0.0"
  transformRecord={(record, context) => {
    // Custom transformation logic
    if (context.fromVersion === '2.1.0' && context.toVersion === '3.0.0') {
      // Map deprecated payment methods
      const paymentMethodMap = {
        'paypal': 'bank_transfer',
        'check': 'bank_transfer'
      }

      if (record.paymentMethod in paymentMethodMap) {
        return {
          ...record,
          paymentMethod: paymentMethodMap[record.paymentMethod],
          originalPaymentMethod: record.paymentMethod,
          migrationTimestamp: new Date().toISOString()
        }
      }
    }

    return record
  }}
  validateTransformedRecord={(record) => {
    // Custom validation
    const validMethods = ['card', 'bank_transfer', 'crypto_wallet']

    if (!validMethods.includes(record.paymentMethod)) {
      return `Invalid payment method: ${record.paymentMethod}`
    }

    return true
  }}
/>
```

### Example 3: With Progress Monitoring

```typescript
import { useToast } from '@/components/ui/use-toast'

function SchemaMigrationPage() {
  const { toast } = useToast()
  const [progress, setProgress] = useState<MigrationProgress | null>(null)

  return (
    <MigrationWizard
      schemaId="patient_record_schema"
      fromVersion="1.5.0"
      toVersion="2.0.0"
      onProgress={(prog) => {
        setProgress(prog)

        // Show toast at milestones
        if (prog.percentage === 25) {
          toast({ title: '25% complete', description: `${prog.processedRecords} records migrated` })
        }
        if (prog.percentage === 50) {
          toast({ title: 'Halfway there!', description: `${prog.processedRecords} records migrated` })
        }
        if (prog.percentage === 75) {
          toast({ title: '75% complete', description: `${prog.processedRecords} records migrated` })
        }
      }}
      onComplete={(result) => {
        toast({
          title: 'Migration Complete',
          description: `Successfully migrated ${result.migratedRecords} records in ${result.duration / 1000}s`
        })

        // Send analytics event
        analytics.track('schema_migration_complete', {
          schemaId: 'patient_record_schema',
          fromVersion: '1.5.0',
          toVersion: '2.0.0',
          recordsProcessed: result.migratedRecords,
          duration: result.duration
        })
      }}
    />
  )
}
```

### Example 4: With Error Handling

```typescript
<MigrationWizard
  schemaId="legal_case_schema"
  fromVersion="2.3.0"
  toVersion="3.0.0"
  onError={(error, recoverable) => {
    console.error('Migration error:', error)

    if (recoverable) {
      toast({
        title: 'Migration Error (Recoverable)',
        description: error.error,
        action: <Button onClick={() => retryMigration()}>Retry</Button>
      })
    } else {
      toast({
        title: 'Migration Failed',
        description: error.error,
        variant: 'destructive'
      })

      // Log to error tracking service
      Sentry.captureException(new Error(error.error), {
        tags: {
          schemaId: 'legal_case_schema',
          fromVersion: '2.3.0',
          toVersion: '3.0.0'
        }
      })
    }
  }}
  retryFailedRecords={true}
  maxRetries={3}
/>
```

### Example 5: With Background Execution

```typescript
function BackgroundMigration() {
  const [migrationId, setMigrationId] = useState<string | null>(null)

  const startBackgroundMigration = async () => {
    const id = await api.startMigration({
      schemaId: 'product_schema',
      fromVersion: '2.0.0',
      toVersion: '2.1.0'
    })

    setMigrationId(id)

    // Poll for status
    const interval = setInterval(async () => {
      const status = await api.getMigrationStatus(id)

      if (status.state === 'completed') {
        clearInterval(interval)
        toast({ title: 'Migration Complete' })
      } else if (status.state === 'failed') {
        clearInterval(interval)
        toast({ title: 'Migration Failed', variant: 'destructive' })
      }
    }, 5000)
  }

  return (
    <MigrationWizard
      schemaId="product_schema"
      fromVersion="2.0.0"
      toVersion="2.1.0"
      customActions={[
        {
          id: 'background',
          label: 'Run in Background',
          onClick: startBackgroundMigration
        }
      ]}
    />
  )
}
```

---

## Related Components

### 1. SchemaEditor

**Relationship:** SchemaEditor → MigrationWizard

When user publishes a schema with breaking changes in SchemaEditor, they can launch MigrationWizard:

```typescript
<SchemaEditor
  schemaId={schemaId}
  currentVersion="2.1.0"
  onPublish={(schema, versionBump, breakingChanges) => {
    if (breakingChanges.length > 0) {
      setShowMigrationWizard(true)
    } else {
      publishSchema(schema)
    }
  }}
/>

{showMigrationWizard && (
  <MigrationWizard
    schemaId={schemaId}
    fromVersion={currentVersion}
    toVersion={suggestedNextVersion}
    breakingChanges={breakingChanges}
  />
)}
```

### 2. CompatibilityViewer

**Relationship:** MigrationWizard uses CompatibilityViewer in Step 2

The Review Plan step displays a CompatibilityViewer showing breaking changes:

```typescript
// Inside MigrationWizard, Step 2
<CompatibilityViewer
  oldSchema={sourceSchema}
  newSchema={targetSchema}
  compatibilityReport={migrationPlan.compatibilityReport}
  showDiff={true}
/>
```

### 3. SchemaHistory

**Relationship:** SchemaHistory → MigrationWizard

Users can migrate between any two versions from SchemaHistory:

```typescript
<SchemaHistory schemaId={schemaId}>
  {versions.map((version, index) => (
    <VersionCard key={version.id} version={version}>
      {index > 0 && (
        <Button
          onClick={() => {
            setMigrationConfig({
              from: versions[index - 1].number,
              to: version.number
            })
            setShowWizard(true)
          }}
        >
          Migrate from {versions[index - 1].number}
        </Button>
      )}
    </VersionCard>
  ))}
</SchemaHistory>
```

### 4. BackupManager

**Relationship:** MigrationWizard uses BackupManager service

Before migration, create backup using BackupManager:

```typescript
import { BackupManager } from '@/services/backup-manager'

// Inside MigrationWizard
const createBackup = async () => {
  const backupManager = new BackupManager()

  const backupId = await backupManager.createSnapshot({
    schemaId,
    version: fromVersion,
    strategy: 'incremental'
  })

  setBackupId(backupId)
}
```

### 5. RollbackService

**Relationship:** MigrationWizard uses RollbackService in Step 5

When user clicks "Rollback", use RollbackService:

```typescript
import { RollbackService } from '@/services/rollback'

// Inside MigrationWizard
const handleRollback = async () => {
  const rollbackService = new RollbackService()

  await rollbackService.restoreFromBackup(backupId)

  toast({ title: 'Rollback complete' })
}
```

---

## References

### Schema Migration Strategies
- [Schema Evolution in Distributed Systems](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protobuf-thrift.html)
- [Zero-Downtime Schema Migrations](https://fly.io/blog/zero-downtime-postgres-migrations/)
- [Database Migration Best Practices](https://www.liquibase.com/blog/database-migration-best-practices)

### Batch Processing
- [Efficient Batch Processing Patterns](https://cloud.google.com/architecture/batch-processing-best-practices)
- [Backpressure and Flow Control](https://reactivemanifesto.org/glossary#Back-Pressure)

### Data Migration Tools
- [Flyway - Database Migrations](https://flywaydb.org/)
- [Liquibase - Database Change Management](https://www.liquibase.com/)
- [Alembic - SQLAlchemy Migrations](https://alembic.sqlalchemy.org/)

### Progress Tracking
- [Progressive Enhancement](https://www.smashingmagazine.com/2009/04/progressive-enhancement-what-it-is-and-how-to-use-it/)
- [Real-time Progress Updates with WebSockets](https://socket.io/docs/v4/)

### Rollback Strategies
- [Safe Database Migrations](https://www.braintreepayments.com/blog/safe-database-migrations/)
- [Blue-Green Deployments](https://martinfowler.com/bliki/BlueGreenDeployment.html)

---

**Document Version:** 1.0
**Word Count:** ~11,500 words (~1,150 lines)
**Last Updated:** 2025-10-25
**Maintained By:** Platform Team
