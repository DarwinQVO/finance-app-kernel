# RetroactiveCorrectionDialog

**Layer:** Interaction Layer (IL)
**Domain:** Provenance Ledger (Vertical 5.1)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Architecture](#architecture)
4. [UI Components](#ui-components)
5. [Correction Workflows](#correction-workflows)
6. [Interaction Patterns](#interaction-patterns)
7. [Multi-Domain Examples](#multi-domain-examples)
8. [Edge Cases & Error Handling](#edge-cases--error-handling)
9. [Implementation Guide](#implementation-guide)
10. [Testing Strategy](#testing-strategy)
11. [Accessibility](#accessibility)
12. [Performance Considerations](#performance-considerations)
13. [Related Components](#related-components)

---

## Overview

### Purpose

The **RetroactiveCorrectionDialog** is a specialized modal/drawer component designed for making corrections to historical data with explicit effective date control. Unlike simple field edits, retroactive corrections update the **valid time** dimension while preserving the **transaction time** dimension, enabling bitemporal tracking. This component ensures that corrections are auditable, justified, and accurately reflect when facts were true in the real world versus when they were recorded in the system.

### Key Capabilities

- **Effective Date Selection**: Choose when the correction was actually true (Today, Original Date, Custom)
- **Field-Specific Editing**: Select and edit individual fields with type-aware controls
- **Impact Preview**: See downstream effects before committing (e.g., "3 calculations will be rerun")
- **Mandatory Justification**: Require reason for all corrections (audit compliance)
- **Real-time Validation**: Inline validation with immediate feedback
- **Batch Corrections**: Apply same correction to multiple entities
- **Undo/Revert**: Ability to revert corrections with full audit trail
- **Responsive Design**: Works on desktop, tablet, and mobile devices
- **Accessibility**: Full keyboard navigation and screen reader support

### Design Philosophy

The RetroactiveCorrectionDialog embodies the principle that **corrections are accountability events**. Unlike ad-hoc edits, retroactive corrections:

- **Preserve history**: Original values remain in transaction time dimension
- **Document intent**: Mandatory reason field explains why correction was needed
- **Show impact**: Preview downstream effects before committing
- **Enable rollback**: Corrections can be reverted with full audit trail
- **Support compliance**: Meet regulatory requirements for data correction

This transforms corrections from technical operations into auditable business processes.

---

## Component Interface

### Primary Props

```typescript
interface RetroactiveCorrectionDialogProps {
  // ============================================================
  // Entity Configuration
  // ============================================================

  /**
   * Entity to correct
   * Must include current field values
   */
  entity: Entity

  /**
   * Entity type for context-specific UI
   * @example 'transaction', 'patient_record', 'legal_case'
   */
  entityType: string

  /**
   * Fields that can be corrected
   * Defines UI controls, validation, and labels
   */
  editableFields: FieldDefinition[]

  /**
   * Read-only fields to display for context
   * @example ['id', 'created_at', 'created_by']
   */
  contextFields?: FieldDefinition[]

  // ============================================================
  // Correction Configuration
  // ============================================================

  /**
   * Default effective date mode
   * @default 'today'
   */
  defaultEffectiveDate?: 'today' | 'original' | 'custom'

  /**
   * Allow backdating corrections
   * If false, only 'today' is available
   * @default true
   */
  allowBackdating?: boolean

  /**
   * Allow future-dating corrections
   * If false, effective date cannot be in future
   * @default false
   */
  allowFutureDating?: boolean

  /**
   * Earliest allowed effective date
   * @default Entity creation date
   */
  minEffectiveDate?: Date

  /**
   * Latest allowed effective date
   * @default Today (or no limit if allowFutureDating=true)
   */
  maxEffectiveDate?: Date

  /**
   * Pre-populate with suggested corrections
   * Useful for batch corrections or templates
   */
  suggestedCorrections?: Record<string, any>

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Dialog mode (modal or drawer)
   * @default 'modal'
   */
  mode?: 'modal' | 'drawer'

  /**
   * Dialog size
   * @default 'lg'
   */
  size?: 'sm' | 'md' | 'lg' | 'xl' | 'full'

  /**
   * Show impact preview before saving
   * Displays downstream effects of correction
   * @default true
   */
  showImpactPreview?: boolean

  /**
   * Show field history in dialog
   * Displays previous values and changes
   * @default true
   */
  showFieldHistory?: boolean

  /**
   * Show confidence scores from Objective Layer
   * Helps identify low-confidence fields needing correction
   * @default true
   */
  showConfidence?: boolean

  /**
   * Require reason for all corrections
   * @default true (strongly recommended for audit compliance)
   */
  requireReason?: boolean

  /**
   * Minimum reason length (characters)
   * @default 10
   */
  minReasonLength?: number

  /**
   * Enable preview step before saving
   * Shows summary of changes for confirmation
   * @default true
   */
  enablePreview?: boolean

  /**
   * Custom dialog title
   * @default 'Make Retroactive Correction'
   */
  title?: string

  /**
   * Custom submit button label
   * @default 'Save Correction'
   */
  submitLabel?: string

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when user saves correction
   * Must return promise to support async validation
   *
   * @param correction - Correction configuration
   * @returns Promise<void> - Resolves on success, rejects with error
   */
  onSave: (correction: RetroactiveCorrection) => Promise<void>

  /**
   * Called when user cancels without saving
   */
  onCancel: () => void

  /**
   * Called when validation state changes
   * Useful for enabling/disabling save button externally
   */
  onValidationChange?: (isValid: boolean, errors: ValidationError[]) => void

  /**
   * Called when user changes field values (before save)
   * Useful for real-time impact calculation
   */
  onChange?: (changes: FieldChange[]) => void

  /**
   * Called to calculate impact preview
   * Returns downstream effects of correction
   *
   * @param correction - Correction configuration
   * @returns Promise<ImpactAnalysis> - Impact analysis result
   */
  onCalculateImpact?: (correction: RetroactiveCorrection) => Promise<ImpactAnalysis>

  // ============================================================
  // Advanced Options
  // ============================================================

  /**
   * Custom validation rules beyond field-level validation
   * @example Cross-field validation (start_date < end_date)
   */
  customValidationRules?: ValidationRule[]

  /**
   * Enable batch correction mode
   * Allows applying same correction to multiple entities
   * @default false
   */
  enableBatchMode?: boolean

  /**
   * Entities for batch correction (if enableBatchMode=true)
   */
  batchEntities?: Entity[]

  /**
   * Loading state (external loading indicator)
   * @default false
   */
  isLoading?: boolean

  /**
   * Auto-focus first editable field on open
   * @default true
   */
  autoFocus?: boolean

  /**
   * Show warnings for high-impact corrections
   * @example "This will affect 50+ downstream records"
   * @default true
   */
  showWarnings?: boolean

  /**
   * Warning threshold for impact count
   * @default 10
   */
  warningThreshold?: number
}
```

### Supporting Types

```typescript
/**
 * Retroactive correction configuration
 */
interface RetroactiveCorrection {
  /**
   * Entity being corrected
   */
  entityId: string

  /**
   * Entity type
   */
  entityType: string

  /**
   * Field changes
   */
  fieldChanges: FieldChange[]

  /**
   * Effective date (valid time)
   * When the corrected value was actually true
   */
  effectiveDate: Date

  /**
   * Reason for correction (required for audit)
   */
  reason: string

  /**
   * Additional metadata
   */
  metadata?: {
    /**
     * Correction source (manual, import, migration, etc.)
     */
    source?: string

    /**
     * Related document/evidence
     */
    documentId?: string

    /**
     * User-defined tags
     */
    tags?: string[]
  }
}

/**
 * Field change within a correction
 */
interface FieldChange {
  /**
   * Field identifier
   */
  fieldName: string

  /**
   * Current value (before correction)
   */
  oldValue: any

  /**
   * New value (after correction)
   */
  newValue: any

  /**
   * Field type for validation
   */
  fieldType: 'text' | 'number' | 'date' | 'boolean' | 'enum' | 'json'

  /**
   * Previous confidence score (if available)
   */
  oldConfidence?: number

  /**
   * New confidence score (always 1.0 for manual corrections)
   */
  newConfidence?: number
}

/**
 * Field definition with UI and validation metadata
 */
interface FieldDefinition {
  /**
   * Field identifier (must match entity schema)
   */
  name: string

  /**
   * Display label
   */
  label: string

  /**
   * Field type (determines UI control)
   */
  type: 'text' | 'number' | 'date' | 'boolean' | 'enum' | 'json'

  /**
   * Field description (help text)
   */
  description?: string

  /**
   * Required field
   * @default false
   */
  required?: boolean

  /**
   * Read-only field (cannot be edited)
   * @default false
   */
  readOnly?: boolean

  /**
   * Validation rules
   */
  validation?: {
    min?: number
    max?: number
    pattern?: RegExp
    customValidator?: (value: any) => string | null
  }

  /**
   * Enum options (for enum type)
   */
  options?: Array<{ value: any; label: string }>

  /**
   * Placeholder text
   */
  placeholder?: string

  /**
   * Current confidence score (0-1)
   */
  confidence?: number
}

/**
 * Impact analysis result
 */
interface ImpactAnalysis {
  /**
   * Number of downstream entities affected
   */
  affectedEntities: number

  /**
   * Types of affected entities
   */
  affectedEntityTypes: string[]

  /**
   * Specific downstream effects
   */
  effects: ImpactEffect[]

  /**
   * Estimated processing time (milliseconds)
   */
  estimatedProcessingTime?: number

  /**
   * Warning messages
   */
  warnings?: string[]
}

/**
 * Individual impact effect
 */
interface ImpactEffect {
  /**
   * Effect type
   */
  type: 'recomputation' | 'validation' | 'cascade' | 'notification'

  /**
   * Description of effect
   */
  description: string

  /**
   * Number of records affected
   */
  count: number

  /**
   * Severity level
   */
  severity: 'low' | 'medium' | 'high'
}

/**
 * Validation error
 */
interface ValidationError {
  /**
   * Error code
   */
  code: string

  /**
   * Human-readable message
   */
  message: string

  /**
   * Field that triggered error
   */
  field?: string

  /**
   * Error severity
   */
  severity: 'error' | 'warning'
}

/**
 * Validation rule
 */
interface ValidationRule {
  /**
   * Rule name
   */
  name: string

  /**
   * Validator function
   */
  validator: (entity: Entity, changes: FieldChange[]) => ValidationError | null

  /**
   * Rule description
   */
  description?: string
}

/**
 * Entity snapshot
 */
interface Entity {
  id: string
  type: string
  fields: Record<string, any>
  metadata?: {
    createdAt: string
    updatedAt: string
    createdBy?: string
    version?: number
  }
}
```

---

## Architecture

### Component Structure

```
RetroactiveCorrectionDialog/
├── RetroactiveCorrectionDialog.tsx     # Main component
├── components/
│   ├── FieldEditor.tsx                 # Field editing controls
│   ├── EffectiveDateSelector.tsx       # Effective date selection
│   ├── ReasonInput.tsx                 # Reason textarea
│   ├── ImpactPreview.tsx               # Impact analysis display
│   ├── FieldHistory.tsx                # Field history timeline
│   ├── ConfidenceIndicator.tsx         # Confidence score display
│   ├── PreviewStep.tsx                 # Preview before save
│   ├── ValidationMessages.tsx          # Error/warning display
│   └── BatchCorrectionList.tsx         # Batch entity list
├── hooks/
│   ├── useRetroactiveCorrection.ts     # Correction state management
│   ├── useCorrectionValidation.ts      # Validation logic
│   ├── useImpactAnalysis.ts            # Impact calculation
│   └── useFieldHistory.ts              # Field history fetching
├── utils/
│   ├── correctionValidator.ts          # Validation rules
│   ├── impactCalculator.ts             # Impact analysis
│   ├── dateUtils.ts                    # Date manipulation
│   └── fieldFormatters.ts              # Field value formatting
└── styles/
    └── RetroactiveCorrectionDialog.module.css
```

### State Management

```typescript
interface RetroactiveCorrectionDialogState {
  // Correction configuration
  fieldChanges: Map<string, FieldChange>
  effectiveDate: Date | null
  effectiveDateMode: 'today' | 'original' | 'custom'
  reason: string

  // UI state
  currentStep: 'edit' | 'preview' | 'saving'
  showImpactPreview: boolean
  showFieldHistory: Map<string, boolean>
  focusedField: string | null

  // Validation state
  validationErrors: ValidationError[]
  isDirty: boolean
  isValid: boolean

  // Impact analysis
  impactAnalysis: ImpactAnalysis | null
  isCalculatingImpact: boolean

  // Loading state
  isSaving: boolean
  error: Error | null

  // Batch mode
  selectedBatchEntities: Set<string>
}
```

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│              RetroactiveCorrectionDialog                     │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  User Interaction                                  │    │
│  │  1. Select field to correct                        │    │
│  │  2. Enter new value                                │    │
│  │  3. Select effective date                          │    │
│  │  4. Enter reason                                   │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Real-time Validation                              │    │
│  │  - Field type validation                           │    │
│  │  - Required field check                            │    │
│  │  - Cross-field validation                          │    │
│  │  - Effective date validation                       │    │
│  │  - Reason length check                             │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Impact Analysis (Optional)                        │    │
│  │  - Calculate downstream effects                    │    │
│  │  - Count affected entities                         │    │
│  │  - Estimate processing time                        │    │
│  │  - Generate warnings if high impact                │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Preview Step (Optional)                           │    │
│  │  - Show summary of changes                         │    │
│  │  - Display impact analysis                         │    │
│  │  - Confirm correction details                      │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Save Correction                                   │    │
│  │  - Call onSave callback                            │    │
│  │  - Handle success/error                            │    │
│  │  - Close dialog on success                         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## UI Components

### Main Dialog (Modal Mode)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Make Retroactive Correction                                       [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Entity: Transaction #txn_abc123                                           │
│  Current State: Extracted on Jan 5, 2024                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Context Fields (Read-only)                                          │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │  Transaction ID:  txn_abc123                                        │  │
│  │  Created:         Jan 5, 2024 9:23 AM                               │  │
│  │  Created By:      system_ocr                                        │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Editable Fields                                                      │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │                                                                      │  │
│  │  Amount *                                                 [History] │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │ Current:  $ 100.00                                             │ │  │
│  │  │ Confidence: ████████░░ 87%                                     │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  New Value:                                                          │  │
│  │  $ [105.50                                                    ]      │  │
│  │                                                                      │  │
│  │  ─────────────────────────────────────────────────────────────────  │  │
│  │                                                                      │  │
│  │  Date                                                      [History] │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │ Current:  2024-01-05                                           │ │  │
│  │  │ Confidence: ████████████ 95%                                  │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  New Value:                                                          │  │
│  │  [2024-01-03                                               ▼]        │  │
│  │                                                                      │  │
│  │  ─────────────────────────────────────────────────────────────────  │  │
│  │                                                                      │  │
│  │  Merchant                                                  [History] │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │ Current:  "ACME CORP"                                          │ │  │
│  │  │ Confidence: ████████░░ 82%                                     │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  New Value:                                                          │  │
│  │  [Acme Corporation                                          ]        │  │
│  │                                                                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Effective Date                                               [?]    │  │
│  │                                                                      │  │
│  │ When was this correction actually true?                             │  │
│  │                                                                      │  │
│  │  ● Today (Oct 25, 2024)                                             │  │
│  │    Correction is effective as of today                              │  │
│  │                                                                      │  │
│  │  ○ Original Date (Jan 5, 2024)                                      │  │
│  │    Correction was true on the original transaction date             │  │
│  │                                                                      │  │
│  │  ○ Custom Date                                                      │  │
│  │    [Select custom date ▼]                                           │  │
│  │                                                                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Reason for Correction *                                             │  │
│  │                                                                      │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │ Correcting OCR errors based on original receipt. Amount should │ │  │
│  │  │ be $105.50, not $100.00. Date should be Jan 3, not Jan 5.     │ │  │
│  │  │                                                                 │ │  │
│  │  │                                                                 │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  85 characters (minimum 10)                                         │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Show Impact Preview]                                                      │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ ⚠ Impact Analysis                                                   │  │
│  │                                                                      │  │
│  │ This correction will affect:                                        │  │
│  │  • 3 calculated fields will be recomputed                           │  │
│  │  • 1 downstream report will be updated                              │  │
│  │  • Estimated processing time: 2 seconds                             │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Cancel]                                              [Save Correction]   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Drawer Mode (Alternative Layout)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                        [X] │
│  Make Retroactive Correction                                              │
│  ═══════════════════════════════════════════════════════════════════════  │
│                                                                             │
│  Transaction #txn_abc123                                                   │
│  Extracted on Jan 5, 2024 by system_ocr                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Amount *                                              Confidence: 87%│  │
│  │ Current: $100.00                                                     │  │
│  │ New:     $ [105.50                                              ]    │  │
│  │                                                                      │  │
│  │ [View History]                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Date                                                  Confidence: 95%│  │
│  │ Current: 2024-01-05                                                  │  │
│  │ New:     [2024-01-03                                        ▼]       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Effective Date                                                       │  │
│  │                                                                      │  │
│  │ ● Today       ○ Original Date (Jan 5)     ○ Custom                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Reason *                                                             │  │
│  │ ┌──────────────────────────────────────────────────────────────────┐ │  │
│  │ │ Correcting OCR errors from receipt...                           │ │  │
│  │ │                                                                  │ │  │
│  │ └──────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Impact: 3 fields will be recomputed                                 │  │
│  │ [View Details]                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ [Cancel]                                      [Save Correction]      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Field History Popup

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Field History: Amount                                                 [X] │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Timeline:                                                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Jan 5, 2024 9:23 AM                                                 │  │
│  │ ● Extraction                                                         │  │
│  │   Value: $100.00                                                    │  │
│  │   By: system_ocr                                                    │  │
│  │   Confidence: 87%                                                   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Feb 15, 2024 2:32 PM                                                │  │
│  │ ● Override                                                           │  │
│  │   Value: $100.00 → $105.50                                          │  │
│  │   By: John Doe (john@example.com)                                   │  │
│  │   Reason: "Correcting OCR error - receipt shows $105.50"            │  │
│  │   Confidence: 100%                                                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Mar 20, 2024 10:15 AM                                               │  │
│  │ ● Revert                                                             │  │
│  │   Value: $105.50 → $100.00                                          │  │
│  │   By: Jane Smith (jane@example.com)                                 │  │
│  │   Reason: "Reverting - original OCR was correct"                    │  │
│  │   Confidence: 100%                                                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Current Value: $100.00                                                    │
│                                                                             │
│  [Close]                                                                    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Impact Preview Panel

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Impact Analysis                                                       [X] │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Analyzing impact of correction...                                         │
│                                                                             │
│  Changes:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  • amount: $100.00 → $105.50                                        │  │
│  │  • date: 2024-01-05 → 2024-01-03                                    │  │
│  │  • merchant: "ACME CORP" → "Acme Corporation"                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Effective Date: Original Date (Jan 5, 2024)                               │
│  Note: This correction will be backdated to the original transaction date  │
│                                                                             │
│  Downstream Effects:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ✓ Recomputation (Low Impact)                                       │  │
│  │    3 calculated fields will be recomputed:                          │  │
│  │    • tax_amount (based on new amount)                               │  │
│  │    • total (based on new amount)                                    │  │
│  │    • category (based on new merchant name)                          │  │
│  │                                                                      │  │
│  │  ✓ Validation (Low Impact)                                          │  │
│  │    1 validation check will be re-run:                               │  │
│  │    • Date consistency check                                         │  │
│  │                                                                      │  │
│  │  ⚠ Cascade Update (Medium Impact)                                   │  │
│  │    1 downstream report will be updated:                             │  │
│  │    • Monthly expense report (Jan 2024)                              │  │
│  │                                                                      │  │
│  │  ℹ Notification (Low Impact)                                        │  │
│  │    1 user will be notified:                                         │  │
│  │    • Report owner (john@example.com)                                │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Summary:                                                                   │
│  • Total affected entities: 5                                              │
│  • Estimated processing time: 2.3 seconds                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ ℹ This is a low-to-medium impact correction. You can proceed safely.│  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Close]                                         [Proceed with Correction] │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Preview Step (Before Save)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Preview Correction                                                [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Please review your correction before saving.                              │
│                                                                             │
│  Entity: Transaction #txn_abc123                                           │
│                                                                             │
│  Changes:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Field      │ Before        │ After                  │ Change        │  │
│  ├─────────────┼───────────────┼────────────────────────┼───────────────┤  │
│  │  amount     │ $100.00       │ $105.50                │ +$5.50        │  │
│  │  date       │ 2024-01-05    │ 2024-01-03             │ -2 days       │  │
│  │  merchant   │ "ACME CORP"   │ "Acme Corporation"     │ (normalized)  │  │
│  └─────────────┴───────────────┴────────────────────────┴───────────────┘  │
│                                                                             │
│  Effective Date:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Original Date (Jan 5, 2024)                                        │  │
│  │  This correction will be backdated to the original transaction date.│  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Reason:                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  "Correcting OCR errors based on original receipt. Amount should be │  │
│  │   $105.50, not $100.00. Date should be Jan 3, not Jan 5."           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Impact:                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  • 3 calculated fields will be recomputed                           │  │
│  │  • 1 validation check will be re-run                                │  │
│  │  • 1 downstream report will be updated                              │  │
│  │  • Estimated processing time: 2.3 seconds                           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Metadata:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Correction will be saved as:                                       │  │
│  │  • Transaction Time: Oct 25, 2024 10:30 AM (now)                    │  │
│  │  • Valid Time: Jan 5, 2024 (original date)                          │  │
│  │  • Actor: John Doe (john@example.com)                               │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [← Back to Edit]                                  [Confirm & Save]        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Batch Correction Mode

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Batch Retroactive Correction                                     [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Apply the same correction to multiple entities                            │
│                                                                             │
│  Selected Entities: 5 of 10                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ☑ Transaction #txn_001                                             │  │
│  │  ☑ Transaction #txn_002                                             │  │
│  │  ☐ Transaction #txn_003                                             │  │
│  │  ☑ Transaction #txn_004                                             │  │
│  │  ☐ Transaction #txn_005                                             │  │
│  │  ☑ Transaction #txn_006                                             │  │
│  │  ☐ Transaction #txn_007                                             │  │
│  │  ☑ Transaction #txn_008                                             │  │
│  │  ☐ Transaction #txn_009                                             │  │
│  │  ☐ Transaction #txn_010                                             │  │
│  │                                                                      │  │
│  │  [Select All]  [Deselect All]                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Correction Configuration:                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Field to Correct: [category                                    ▼]  │  │
│  │                                                                      │  │
│  │  New Value for All: [Office Supplies                             ]  │  │
│  │                                                                      │  │
│  │  Effective Date: ● Today  ○ Original Date  ○ Custom                │  │
│  │                                                                      │  │
│  │  Reason *:                                                           │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │ Batch correction to standardize category names                │ │  │
│  │  │                                                                │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ ⚠ Batch Impact Warning                                              │  │
│  │                                                                      │  │
│  │ This batch correction will affect:                                  │  │
│  │  • 5 entities                                                       │  │
│  │  • 15 calculated fields (3 per entity)                              │  │
│  │  • 5 validation checks                                              │  │
│  │  • Estimated processing time: 8 seconds                             │  │
│  │                                                                      │  │
│  │ This is a high-impact operation. Please review carefully.           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Cancel]                                         [Apply to 5 Entities]    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Validation Error Display

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Make Retroactive Correction                                       [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ ✕ Validation Errors                                                 │  │
│  │                                                                      │  │
│  │  Please fix the following errors before saving:                     │  │
│  │                                                                      │  │
│  │  • Amount: Value must be greater than 0                             │  │
│  │  • Reason: Reason must be at least 10 characters (currently 5)      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Amount *                                                                   │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ $ [-10.00                                                     ] ✕   │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│  ✕ Value must be greater than 0                                           │
│                                                                             │
│  Reason *                                                                   │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │ Fix error                                                   ✕       │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│  ✕ Reason must be at least 10 characters (currently 9)                    │
│                                                                             │
│  [Cancel]                                      [Save Correction] (disabled)│
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Correction Workflows

### Workflow 1: Simple Field Correction

**Scenario:** User corrects OCR error in transaction amount

**Steps:**
1. User opens entity detail page
2. User clicks "Make Correction" button
3. RetroactiveCorrectionDialog opens
4. User selects "Amount" field
5. User enters new value: $105.50 (was $100.00)
6. User selects "Today" as effective date
7. User enters reason: "Correcting OCR error - receipt shows $105.50"
8. User clicks "Save Correction"
9. Validation passes
10. Impact analysis runs (3 fields will be recomputed)
11. Correction saves successfully
12. Dialog closes, page refreshes with updated values

**Timeline Event Created:**
```json
{
  "id": "evt_123",
  "eventType": "correction",
  "transactionTime": "2024-10-25T10:30:00Z",
  "validTime": "2024-10-25T10:30:00Z",
  "actor": {
    "id": "user_john",
    "type": "user",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "affectedFields": [
    {
      "fieldName": "amount",
      "oldValue": 100.00,
      "newValue": 105.50,
      "fieldType": "number"
    }
  ],
  "reason": "Correcting OCR error - receipt shows $105.50"
}
```

### Workflow 2: Backdated Correction

**Scenario:** User corrects diagnosis date after finding earlier medical records

**Steps:**
1. User opens patient record
2. User clicks "Make Correction"
3. RetroactiveCorrectionDialog opens
4. User selects "Diagnosis Date" field
5. User enters new value: 2024-01-15 (was 2024-03-10)
6. User selects "Custom Date" as effective date
7. User picks: 2024-01-15 (same as diagnosis date)
8. User enters reason: "Found earlier medical records showing diagnosis on Jan 15"
9. User clicks "Save Correction"
10. Impact preview shows: "This will affect 2 downstream care plans"
11. User reviews impact and confirms
12. Correction saves successfully
13. Timeline shows bitemporal event (recorded today, valid on Jan 15)

**Timeline Event Created (Bitemporal):**
```json
{
  "id": "evt_456",
  "eventType": "correction",
  "transactionTime": "2024-10-25T14:20:00Z",
  "validTime": "2024-01-15T00:00:00Z",
  "actor": {
    "id": "user_dr_jones",
    "type": "user",
    "name": "Dr. Jones",
    "email": "dr.jones@hospital.com"
  },
  "affectedFields": [
    {
      "fieldName": "diagnosis_date",
      "oldValue": "2024-03-10",
      "newValue": "2024-01-15",
      "fieldType": "date"
    }
  ],
  "reason": "Found earlier medical records showing diagnosis on Jan 15"
}
```

### Workflow 3: Multi-Field Correction

**Scenario:** User corrects multiple fields based on original receipt

**Steps:**
1. User opens transaction
2. User clicks "Make Correction"
3. User corrects:
   - Amount: $100.00 → $105.50
   - Date: 2024-01-05 → 2024-01-03
   - Merchant: "ACME CORP" → "Acme Corporation"
4. User selects "Original Date" (Jan 5) as effective date
5. User enters reason: "Correcting multiple OCR errors from receipt"
6. User clicks "Save Correction"
7. Impact preview shows: "5 entities affected, 3 recomputations"
8. User proceeds
9. Correction saves all three field changes atomically
10. Timeline shows single correction event with multiple field changes

### Workflow 4: Batch Correction

**Scenario:** User applies same correction to 10 transactions

**Steps:**
1. User selects 10 transactions on list page
2. User clicks "Batch Correct"
3. RetroactiveCorrectionDialog opens in batch mode
4. User sees list of 10 selected entities
5. User selects field to correct: "Category"
6. User enters new value: "Office Supplies"
7. User selects "Today" as effective date
8. User enters reason: "Batch correction to standardize category names"
9. User clicks "Apply to 10 Entities"
10. Impact preview shows: "30 fields affected (3 per entity)"
11. High-impact warning appears
12. User confirms
13. Batch correction processes (with progress bar)
14. Success: "10 entities corrected successfully"
15. Timeline shows 10 separate correction events (one per entity)

### Workflow 5: Correction with Validation Error

**Scenario:** User attempts to save invalid correction

**Steps:**
1. User opens correction dialog
2. User enters amount: -$10.00 (negative)
3. Validation error appears: "Amount must be greater than 0"
4. Save button disables
5. User corrects to: $105.50
6. Validation passes, error clears
7. User forgets to enter reason
8. User clicks "Save Correction"
9. Validation error appears: "Reason is required"
10. User enters reason
11. Validation passes
12. User saves successfully

---

## Interaction Patterns

### Field Selection Interaction

**Scenario:** User wants to correct the "Amount" field

**Flow:**
1. Dialog opens with all editable fields visible
2. Each field shows:
   - Label ("Amount")
   - Current value ("$100.00")
   - Confidence score (if available)
   - Input field for new value
   - "History" button
3. User clicks into "Amount" input field
4. Field highlights (border glow)
5. Placeholder clears
6. User types new value
7. Real-time validation runs:
   - Check type (must be number)
   - Check range (must be > 0)
   - Format as currency ($105.50)
8. If valid:
   - Show checkmark icon
   - Update change summary
9. If invalid:
   - Show error icon
   - Display error message
   - Disable save button

**Keyboard Navigation:**
- Tab: Move to next field
- Shift+Tab: Move to previous field
- Enter: Save (if valid)
- Escape: Cancel

### Effective Date Selection Interaction

**Scenario:** User backdates correction to original transaction date

**Flow:**
1. User sees three radio options:
   - Today
   - Original Date (Jan 5, 2024)
   - Custom Date
2. "Today" is selected by default
3. User clicks "Original Date" radio button
4. Radio button highlights
5. Effective date updates to: Jan 5, 2024
6. Info message appears: "This correction will be backdated to the original transaction date"
7. If user clicks "Custom Date":
   - Date picker appears
   - User selects date: Jan 3, 2024
   - Calendar closes
   - Effective date updates to: Jan 3, 2024
   - If date is in past: Info message shows "This correction will be backdated"
   - If date is in future (and allowed): Warning shows "This correction will be forward-dated"

**Validation:**
- Cannot select date before entity creation
- Cannot select future date (unless `allowFutureDating=true`)
- If bitemporal: Warn if effective date > today (unusual)

### Reason Input Interaction

**Scenario:** User enters justification for correction

**Flow:**
1. Reason textarea appears (required field marked with *)
2. Placeholder text: "Explain why this correction is needed (minimum 10 characters)"
3. User clicks into textarea
4. User types reason
5. Character counter updates in real-time: "25 characters (minimum 10)"
6. If below minimum:
   - Counter shows in red
   - Error message: "Reason must be at least 10 characters"
   - Save button disabled
7. If above minimum:
   - Counter shows in green
   - Checkmark icon appears
   - Save button enabled (if all other validation passes)

**Auto-save (Optional):**
- Save reason to local storage every 5 seconds
- Restore on dialog reopen (prevent data loss)

### Impact Preview Interaction

**Scenario:** User wants to see downstream effects before saving

**Flow:**
1. User makes changes to fields
2. User clicks "Show Impact Preview" button
3. Loading spinner appears: "Analyzing impact..."
4. System calls `onCalculateImpact` callback
5. Impact analysis returns:
   - 3 calculated fields will be recomputed
   - 1 validation check will be re-run
   - 1 downstream report will be updated
6. Impact preview panel displays results
7. If high impact (>10 affected entities):
   - Warning icon appears
   - Message: "This is a high-impact correction. Please review carefully."
8. User reviews impact
9. User can:
   - Close preview and continue editing
   - Proceed with correction (if confident)

**Impact Calculation Example:**
```typescript
async function calculateImpact(
  correction: RetroactiveCorrection
): Promise<ImpactAnalysis> {
  const affectedFields = await findDependentFields(
    correction.entityId,
    correction.fieldChanges.map(fc => fc.fieldName)
  )

  const affectedReports = await findReportsUsingEntity(
    correction.entityId
  )

  return {
    affectedEntities: affectedFields.length + affectedReports.length,
    affectedEntityTypes: ['calculated_field', 'report'],
    effects: [
      {
        type: 'recomputation',
        description: `${affectedFields.length} calculated fields will be recomputed`,
        count: affectedFields.length,
        severity: affectedFields.length > 10 ? 'high' : 'low'
      },
      {
        type: 'cascade',
        description: `${affectedReports.length} downstream reports will be updated`,
        count: affectedReports.length,
        severity: affectedReports.length > 5 ? 'high' : 'medium'
      }
    ],
    estimatedProcessingTime: (affectedFields.length * 100) + (affectedReports.length * 500)
  }
}
```

### Field History Interaction

**Scenario:** User views previous changes to a field

**Flow:**
1. User clicks "History" button next to field label
2. Field history popup opens
3. Loading spinner appears: "Loading field history..."
4. System fetches timeline events for this field
5. History displays in reverse chronological order:
   - Most recent event at top
   - Event type (Extraction, Override, Correction, Revert)
   - Timestamp
   - Actor (user or system)
   - Value change (before → after)
   - Reason (if provided)
   - Confidence score
6. User scrolls through history
7. User can click on any event to see full details
8. User closes popup

**History Display Example:**
```
Mar 20, 2024 10:15 AM
● Revert
  $105.50 → $100.00
  By: Jane Smith
  Reason: "Reverting - original OCR was correct"

Feb 15, 2024 2:32 PM
● Override
  $100.00 → $105.50
  By: John Doe
  Reason: "Correcting OCR error"

Jan 5, 2024 9:23 AM
● Extraction
  null → $100.00
  By: system_ocr
  Confidence: 87%
```

### Save Interaction

**Scenario:** User saves correction

**Flow:**
1. User clicks "Save Correction" button
2. Final validation runs:
   - Check all required fields filled
   - Check all values valid
   - Check reason meets minimum length
3. If validation fails:
   - Show validation errors
   - Scroll to first error
   - Keep dialog open
4. If validation passes:
   - Button shows loading spinner
   - Disable all form inputs
   - Call `onSave` callback
5. While saving:
   - Show progress message: "Saving correction..."
   - Disable cancel button (prevent accidental close)
6. On save success:
   - Show success toast: "Correction saved successfully"
   - Close dialog automatically (after 1 second)
   - Refresh parent page
   - Add event to timeline
7. On save error:
   - Show error message in dialog
   - Re-enable form inputs
   - Keep dialog open
   - Offer retry button

**Error Handling:**
```typescript
async function handleSave() {
  setIsSaving(true)

  try {
    await onSave(correction)

    // Success
    showToast('Correction saved successfully', 'success')
    setTimeout(() => {
      onCancel() // Close dialog
    }, 1000)
  } catch (error) {
    // Error
    setError(error.message)
    showToast('Failed to save correction', 'error')
  } finally {
    setIsSaving(false)
  }
}
```

---

## Multi-Domain Examples

### 1. Financial Services: Backdated Transaction Correction

**Use Case:** Correct transaction date based on actual receipt date

**Entity Type:** `transaction`

**Scenario:**
- Transaction extracted on Jan 15 with date=Jan 15 (OCR read bank statement date)
- User finds original receipt showing date=Jan 12
- User makes retroactive correction

**Correction Configuration:**
```typescript
const correction: RetroactiveCorrection = {
  entityId: 'txn_abc123',
  entityType: 'transaction',
  fieldChanges: [
    {
      fieldName: 'date',
      oldValue: '2024-01-15',
      newValue: '2024-01-12',
      fieldType: 'date'
    }
  ],
  effectiveDate: new Date('2024-01-12'), // Backdated to actual receipt date
  reason: 'Receipt shows purchase on Jan 12, not Jan 15 (bank processing date)'
}
```

**Impact:**
- Monthly expense report for January recalculated
- Tax calculations updated
- Budget tracking updated for correct week

**Value Proposition:**
- Accurate financial records for tax purposes
- Correct expense timing for budgets
- Audit trail shows when error was discovered vs when it occurred

### 2. Healthcare: Diagnosis Date Correction

**Use Case:** Doctor backdates diagnosis after finding earlier records

**Entity Type:** `patient_record`

**Scenario:**
- Diagnosis of "Hypertension" recorded on March 10, 2024
- Doctor finds earlier records from different clinic showing diagnosis on Jan 15, 2024
- Doctor makes retroactive correction

**Correction Configuration:**
```typescript
const correction: RetroactiveCorrection = {
  entityId: 'patient_45678',
  entityType: 'patient_record',
  fieldChanges: [
    {
      fieldName: 'diagnosis_date',
      oldValue: '2024-03-10',
      newValue: '2024-01-15',
      fieldType: 'date'
    },
    {
      fieldName: 'diagnosis_source',
      oldValue: 'Initial consultation',
      newValue: 'Transfer from other clinic',
      fieldType: 'text'
    }
  ],
  effectiveDate: new Date('2024-01-15'), // Backdated to actual diagnosis
  reason: 'Found earlier diagnosis records from transferred patient file',
  metadata: {
    documentId: 'med_record_789',
    tags: ['transfer', 'backdate']
  }
}
```

**Impact:**
- Care plan timeline adjusted
- Medication start dates validated against new diagnosis date
- Insurance claims may need review

**Value Proposition:**
- Accurate medical history for treatment planning
- Compliance with clinical documentation standards
- Legal protection with complete audit trail

### 3. Legal: Filing Date Correction

**Use Case:** Attorney corrects filing date based on court stamp

**Entity Type:** `legal_case_document`

**Scenario:**
- Document recorded in system on Jan 20 (data entry date)
- Court stamp shows actual filing on Jan 12
- Attorney makes retroactive correction (critical for statute of limitations)

**Correction Configuration:**
```typescript
const correction: RetroactiveCorrection = {
  entityId: 'doc_cv2024_0132',
  entityType: 'legal_case_document',
  fieldChanges: [
    {
      fieldName: 'filing_date',
      oldValue: '2024-01-20',
      newValue: '2024-01-12',
      fieldType: 'date'
    }
  ],
  effectiveDate: new Date('2024-01-12'), // Court-stamped date
  reason: 'Court stamp shows filing on Jan 12. Data entry delayed due to court closure.',
  metadata: {
    documentId: 'court_stamp_scan_456',
    tags: ['statute_of_limitations', 'critical']
  }
}
```

**Impact:**
- Case timeline updated
- Statute of limitations calculation corrected
- Docket updated with correct filing sequence

**Value Proposition:**
- Legal compliance (accurate filing dates critical)
- Risk mitigation (avoid missed statute of limitations)
- Court record accuracy

### 4. Research: Dataset Correction

**Use Case:** Researcher corrects measurement error in published dataset

**Entity Type:** `research_dataset_record`

**Scenario:**
- Measurement recorded as 120.5 (decimal point error)
- Researcher discovers error during peer review
- Researcher makes retroactive correction

**Correction Configuration:**
```typescript
const correction: RetroactiveCorrection = {
  entityId: 'participant_001',
  entityType: 'research_dataset_record',
  fieldChanges: [
    {
      fieldName: 'measurement',
      oldValue: 120.5,
      newValue: 12.05,
      fieldType: 'number',
      oldConfidence: 0.95,
      newConfidence: 1.0
    }
  ],
  effectiveDate: new Date('2024-01-15'), // Original measurement date
  reason: 'Decimal point error in data entry. Confirmed with original lab report.',
  metadata: {
    documentId: 'lab_report_123',
    tags: ['correction', 'published_data']
  }
}
```

**Impact:**
- Statistical analyses recomputed
- Z-scores recalculated
- Published figures updated with erratum

**Value Proposition:**
- Research integrity (transparent correction process)
- Reproducibility (correction tracked in provenance)
- Peer review trust (complete audit trail)

### 5. E-commerce: Order Modification

**Use Case:** Customer service backdates discount application

**Entity Type:** `order`

**Scenario:**
- Customer placed order on March 1 without discount code
- Customer contacted support on March 2, claiming they tried to use code but it didn't work
- Support verifies code was valid on March 1, applies retroactive correction

**Correction Configuration:**
```typescript
const correction: RetroactiveCorrection = {
  entityId: 'order_ORD12345',
  entityType: 'order',
  fieldChanges: [
    {
      fieldName: 'discount_code',
      oldValue: null,
      newValue: 'SAVE10',
      fieldType: 'text'
    },
    {
      fieldName: 'discount_amount',
      oldValue: 0,
      newValue: 10.00,
      fieldType: 'number'
    },
    {
      fieldName: 'total',
      oldValue: 99.99,
      newValue: 89.99,
      fieldType: 'number'
    }
  ],
  effectiveDate: new Date('2024-03-01T14:30:00Z'), // Original order date
  reason: 'Customer service goodwill gesture. Customer attempted to use code but encountered technical issue.',
  metadata: {
    source: 'customer_support',
    tags: ['refund', 'discount_issue']
  }
}
```

**Impact:**
- Refund of $10 issued to customer
- Order history updated
- Discount code usage statistics updated

**Value Proposition:**
- Customer satisfaction (retroactive correction feels fair)
- Transparent process (customer can see correction in order history)
- Fraud prevention (all corrections logged with reason)

### 6. SaaS: Subscription Backdating

**Use Case:** Account manager backdates seat addition for existing employees

**Entity Type:** `subscription`

**Scenario:**
- Customer added 3 seats on March 10
- Customer contacted support on March 12: "These employees started March 1, can you backdate?"
- Support verifies employees were indeed active March 1, applies correction

**Correction Configuration:**
```typescript
const correction: RetroactiveCorrection = {
  entityId: 'sub_PREMIUM_789',
  entityType: 'subscription',
  fieldChanges: [
    {
      fieldName: 'seats',
      oldValue: 5,
      newValue: 8,
      fieldType: 'number'
    },
    {
      fieldName: 'seat_addition_date',
      oldValue: '2024-03-10',
      newValue: '2024-03-01',
      fieldType: 'date'
    }
  ],
  effectiveDate: new Date('2024-03-01'), // When employees actually started
  reason: 'Customer requested backdate. Employees were active since March 1 but seats not added until March 10.',
  metadata: {
    source: 'account_manager',
    tags: ['backdate', 'prorated_charge']
  }
}
```

**Impact:**
- Prorated charge recalculated (9 additional days @ $30/day = $270)
- Invoice adjusted
- Subscription timeline updated

**Value Proposition:**
- Fair billing (customer pays for actual usage period)
- Customer trust (flexible support for honest mistakes)
- Revenue accuracy (correct revenue recognition timing)

### 7. Insurance: Incident Date Correction

**Use Case:** Adjuster corrects incident date based on ER records

**Entity Type:** `insurance_claim`

**Scenario:**
- Claim filed on Jan 20 with incident_date=Jan 18 (patient's recollection)
- Adjuster receives ER records showing actual visit on Jan 15
- Adjuster makes retroactive correction

**Correction Configuration:**
```typescript
const correction: RetroactiveCorrection = {
  entityId: 'claim_INS987654',
  entityType: 'insurance_claim',
  fieldChanges: [
    {
      fieldName: 'incident_date',
      oldValue: '2024-01-18',
      newValue: '2024-01-15',
      fieldType: 'date'
    }
  ],
  effectiveDate: new Date('2024-01-15'), // Actual ER visit date
  reason: 'Hospital ER records confirm visit on Jan 15, not Jan 18. Patient initially misremembered date.',
  metadata: {
    documentId: 'er_record_456',
    tags: ['medical_record', 'date_verification']
  }
}
```

**Impact:**
- Policy coverage verified for correct date
- Deductible calculation updated
- Claim approval timeline adjusted

**Value Proposition:**
- Claim accuracy (based on medical records, not memory)
- Fraud prevention (verifiable correction with evidence)
- Regulatory compliance (complete audit trail)

---

## Edge Cases & Error Handling

### Invalid Field Value

**Scenario:** User enters text in a number field

**Handling:**
- Real-time validation catches error
- Error message: "Amount must be a number"
- Input border turns red
- Save button disabled
- User corrects value
- Validation passes, error clears

**Implementation:**
```typescript
function validateFieldValue(
  field: FieldDefinition,
  value: any
): string | null {
  if (field.type === 'number' && isNaN(Number(value))) {
    return `${field.label} must be a number`
  }

  if (field.type === 'date' && !isValidDate(value)) {
    return `${field.label} must be a valid date`
  }

  if (field.required && (value === null || value === undefined || value === '')) {
    return `${field.label} is required`
  }

  return null
}
```

### Effective Date Before Entity Creation

**Scenario:** User selects effective date before entity was created

**Handling:**
- Validation error: "Effective date cannot be before entity creation ({creation_date})"
- Date picker disables dates before creation
- If user somehow bypasses and submits, backend validation catches
- Error returned to frontend, displayed in dialog

**Implementation:**
```typescript
function validateEffectiveDate(
  effectiveDate: Date,
  entity: Entity
): ValidationError | null {
  const createdAt = new Date(entity.metadata.createdAt)

  if (effectiveDate < createdAt) {
    return {
      code: 'EFFECTIVE_DATE_BEFORE_CREATION',
      message: `Effective date cannot be before entity creation (${formatDate(createdAt)})`,
      field: 'effectiveDate',
      severity: 'error'
    }
  }

  return null
}
```

### Reason Too Short

**Scenario:** User enters 5-character reason (minimum is 10)

**Handling:**
- Character counter shows: "5 characters (minimum 10)" in red
- Error message: "Reason must be at least 10 characters"
- Save button disabled
- User adds more text
- Counter turns green when minimum met
- Error clears, save button enables

**Implementation:**
```typescript
function validateReason(
  reason: string,
  minLength: number = 10
): ValidationError | null {
  if (reason.trim().length < minLength) {
    return {
      code: 'REASON_TOO_SHORT',
      message: `Reason must be at least ${minLength} characters (currently ${reason.trim().length})`,
      field: 'reason',
      severity: 'error'
    }
  }

  return null
}
```

### No Changes Made

**Scenario:** User opens dialog, doesn't change any values, tries to save

**Handling:**
- Save button disabled (no changes detected)
- Info message: "No changes to save"
- Or: Allow save but show warning: "No changes detected. Do you want to close?"

**Implementation:**
```typescript
const hasChanges = useMemo(() => {
  return fieldChanges.size > 0
}, [fieldChanges])

// Disable save button if no changes
<button
  onClick={handleSave}
  disabled={!hasChanges || !isValid || isSaving}
>
  Save Correction
</button>
```

### High Impact Warning

**Scenario:** Correction affects >50 downstream entities

**Handling:**
- Impact analysis shows high severity
- Warning banner: "⚠️ High Impact: This will affect 52 entities. Processing may take 30+ seconds."
- Require explicit confirmation: "I understand this is a high-impact operation"
- Show detailed impact breakdown
- Optional: Require manager approval for very high impact (>100 entities)

**Implementation:**
```typescript
if (impactAnalysis.affectedEntities > 50) {
  return (
    <WarningBanner severity="high">
      High Impact: This will affect {impactAnalysis.affectedEntities} entities.
      Processing may take {Math.ceil(impactAnalysis.estimatedProcessingTime / 1000)} seconds.

      <Checkbox
        checked={confirmedHighImpact}
        onChange={setConfirmedHighImpact}
      >
        I understand this is a high-impact operation
      </Checkbox>
    </WarningBanner>
  )
}

// Disable save unless confirmed
<button
  onClick={handleSave}
  disabled={!confirmedHighImpact}
>
  Save Correction
</button>
```

### Save Failure (Network Error)

**Scenario:** Save request fails due to network error

**Handling:**
- Catch error in `onSave` callback
- Show error message: "Failed to save correction: {error_message}"
- Keep dialog open (preserve user's work)
- Re-enable save button
- Offer retry button
- Auto-save to local storage (prevent data loss)

**Implementation:**
```typescript
async function handleSave() {
  setIsSaving(true)

  try {
    await onSave(correction)
    onCancel() // Close on success
  } catch (error) {
    setError(error.message)
    showRetryButton(true)

    // Auto-save to local storage
    localStorage.setItem(
      `correction_draft_${entity.id}`,
      JSON.stringify(correction)
    )
  } finally {
    setIsSaving(false)
  }
}

// Restore from local storage on open
useEffect(() => {
  const draft = localStorage.getItem(`correction_draft_${entity.id}`)
  if (draft) {
    const correction = JSON.parse(draft)
    // Restore field values
    setFieldChanges(correction.fieldChanges)
    setEffectiveDate(correction.effectiveDate)
    setReason(correction.reason)

    // Ask user if they want to restore
    if (confirm('Restore unsaved correction?')) {
      // Already restored above
    } else {
      localStorage.removeItem(`correction_draft_${entity.id}`)
    }
  }
}, [entity.id])
```

### Concurrent Modification Conflict

**Scenario:** Another user modifies entity while correction dialog is open

**Handling:**
- Detect version conflict on save (optimistic locking)
- Show error: "Entity was modified by another user. Please refresh and try again."
- Offer to save correction as draft
- Offer to reload entity and reapply correction
- Prevent data loss

**Implementation:**
```typescript
async function handleSave() {
  try {
    await onSave({
      ...correction,
      expectedVersion: entity.metadata.version
    })
  } catch (error) {
    if (error.code === 'VERSION_CONFLICT') {
      const shouldReload = await confirm(
        'Entity was modified by another user. Reload and reapply your correction?'
      )

      if (shouldReload) {
        const latestEntity = await fetchEntity(entity.id)
        // Reapply correction to latest version
        await onSave({
          ...correction,
          expectedVersion: latestEntity.metadata.version
        })
      }
    }
  }
}
```

### Batch Correction Partial Failure

**Scenario:** Batch correction succeeds for 8/10 entities, fails for 2

**Handling:**
- Process batch with progress bar
- Track successes and failures
- On completion, show summary:
  - "8 of 10 entities corrected successfully"
  - "2 entities failed: [list entity IDs]"
  - "View failed entities" button
- Allow retry for failed entities only
- Rollback option if needed

**Implementation:**
```typescript
async function handleBatchSave() {
  const results = {
    succeeded: [],
    failed: []
  }

  for (const entityId of selectedBatchEntities) {
    setProgress(results.succeeded.length + results.failed.length, selectedBatchEntities.size)

    try {
      await onSave({
        ...correction,
        entityId
      })
      results.succeeded.push(entityId)
    } catch (error) {
      results.failed.push({ entityId, error: error.message })
    }
  }

  if (results.failed.length === 0) {
    showToast(`All ${results.succeeded.length} entities corrected successfully`, 'success')
    onCancel()
  } else {
    showBatchResultSummary(results)
  }
}
```

---

## Implementation Guide

### Installation

```bash
npm install date-fns react-hook-form zod
npm install --save-dev @types/react
```

### Basic Usage

```typescript
import { RetroactiveCorrectionDialog } from '@/components/RetroactiveCorrectionDialog'

function MyComponent() {
  const [showDialog, setShowDialog] = useState(false)

  const editableFields: FieldDefinition[] = [
    {
      name: 'amount',
      label: 'Amount',
      type: 'number',
      required: true,
      validation: {
        min: 0,
        max: 1000000
      }
    },
    {
      name: 'date',
      label: 'Transaction Date',
      type: 'date',
      required: true
    },
    {
      name: 'merchant',
      label: 'Merchant Name',
      type: 'text',
      required: true
    }
  ]

  async function handleSave(correction: RetroactiveCorrection): Promise<void> {
    const response = await fetch(`/api/entities/${correction.entityId}/correct`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(correction)
    })

    if (!response.ok) {
      throw new Error('Failed to save correction')
    }
  }

  return (
    <>
      <button onClick={() => setShowDialog(true)}>
        Make Correction
      </button>

      {showDialog && (
        <RetroactiveCorrectionDialog
          entity={entity}
          entityType="transaction"
          editableFields={editableFields}
          onSave={handleSave}
          onCancel={() => setShowDialog(false)}
        />
      )}
    </>
  )
}
```

### Advanced Usage with Impact Analysis

```typescript
function MyComponent() {
  async function calculateImpact(
    correction: RetroactiveCorrection
  ): Promise<ImpactAnalysis> {
    const response = await fetch(`/api/entities/${correction.entityId}/impact`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(correction)
    })

    return response.json()
  }

  return (
    <RetroactiveCorrectionDialog
      entity={entity}
      entityType="transaction"
      editableFields={editableFields}
      onSave={handleSave}
      onCancel={() => setShowDialog(false)}
      showImpactPreview={true}
      onCalculateImpact={calculateImpact}
      showFieldHistory={true}
      requireReason={true}
      minReasonLength={20}
    />
  )
}
```

### Batch Correction Example

```typescript
function BatchCorrectionExample() {
  const [selectedEntities, setSelectedEntities] = useState<Entity[]>([])

  return (
    <RetroactiveCorrectionDialog
      entity={selectedEntities[0]} // Primary entity for preview
      entityType="transaction"
      editableFields={editableFields}
      enableBatchMode={true}
      batchEntities={selectedEntities}
      onSave={async (correction) => {
        // Apply to all entities
        for (const entity of selectedEntities) {
          await saveCorrectionForEntity(entity.id, correction)
        }
      }}
      onCancel={() => setShowDialog(false)}
      warningThreshold={5} // Warn if >5 entities
    />
  )
}
```

### Custom Validation Example

```typescript
function MyComponent() {
  const customValidationRules: ValidationRule[] = [
    {
      name: 'date_before_today',
      validator: (entity, changes) => {
        const dateChange = changes.find(c => c.fieldName === 'date')
        if (dateChange && new Date(dateChange.newValue) > new Date()) {
          return {
            code: 'DATE_IN_FUTURE',
            message: 'Transaction date cannot be in the future',
            field: 'date',
            severity: 'error'
          }
        }
        return null
      },
      description: 'Ensure transaction date is not in future'
    },
    {
      name: 'amount_matches_receipt',
      validator: (entity, changes) => {
        const amountChange = changes.find(c => c.fieldName === 'amount')
        if (amountChange && amountChange.newValue < 0) {
          return {
            code: 'NEGATIVE_AMOUNT',
            message: 'Amount cannot be negative',
            field: 'amount',
            severity: 'error'
          }
        }
        return null
      }
    }
  ]

  return (
    <RetroactiveCorrectionDialog
      entity={entity}
      entityType="transaction"
      editableFields={editableFields}
      customValidationRules={customValidationRules}
      onSave={handleSave}
      onCancel={() => setShowDialog(false)}
    />
  )
}
```

---

## Testing Strategy

### Unit Tests

```typescript
describe('RetroactiveCorrectionDialog', () => {
  it('renders with entity data', () => {
    const { getByText } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={jest.fn()}
        onCancel={jest.fn()}
      />
    )

    expect(getByText(/transaction #txn_abc123/i)).toBeInTheDocument()
  })

  it('validates required fields', async () => {
    const onSave = jest.fn()
    const { getByRole, getByText } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={onSave}
        onCancel={jest.fn()}
      />
    )

    // Try to save without changes
    const saveButton = getByRole('button', { name: /save correction/i })
    await userEvent.click(saveButton)

    // Verify validation error shown
    expect(getByText(/reason is required/i)).toBeInTheDocument()
    expect(onSave).not.toHaveBeenCalled()
  })

  it('allows field editing', async () => {
    const { getByLabelText } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={jest.fn()}
        onCancel={jest.fn()}
      />
    )

    const amountInput = getByLabelText(/amount/i)
    await userEvent.clear(amountInput)
    await userEvent.type(amountInput, '105.50')

    expect(amountInput).toHaveValue('105.50')
  })

  it('validates effective date', async () => {
    const { getByLabelText, getByText } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={jest.fn()}
        onCancel={jest.fn()}
        minEffectiveDate={new Date('2024-01-01')}
      />
    )

    // Select custom date
    const customDateRadio = getByLabelText(/custom date/i)
    await userEvent.click(customDateRadio)

    // Enter date before minEffectiveDate
    const datePicker = getByLabelText(/effective date/i)
    await userEvent.type(datePicker, '2023-12-31')

    // Verify error shown
    expect(getByText(/cannot be before/i)).toBeInTheDocument()
  })

  it('calls onSave with correction data', async () => {
    const onSave = jest.fn().mockResolvedValue(undefined)
    const { getByLabelText, getByRole } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={onSave}
        onCancel={jest.fn()}
      />
    )

    // Edit amount
    const amountInput = getByLabelText(/amount/i)
    await userEvent.clear(amountInput)
    await userEvent.type(amountInput, '105.50')

    // Enter reason
    const reasonInput = getByLabelText(/reason/i)
    await userEvent.type(reasonInput, 'Correcting OCR error from receipt')

    // Save
    const saveButton = getByRole('button', { name: /save correction/i })
    await userEvent.click(saveButton)

    // Verify onSave called with correct data
    expect(onSave).toHaveBeenCalledWith(
      expect.objectContaining({
        entityId: mockEntity.id,
        fieldChanges: expect.arrayContaining([
          expect.objectContaining({
            fieldName: 'amount',
            newValue: 105.50
          })
        ]),
        reason: 'Correcting OCR error from receipt'
      })
    )
  })
})
```

### Integration Tests

```typescript
describe('RetroactiveCorrectionDialog Integration', () => {
  it('saves correction and closes dialog', async () => {
    const onSave = jest.fn().mockResolvedValue(undefined)
    const onCancel = jest.fn()

    const { getByLabelText, getByRole } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={onSave}
        onCancel={onCancel}
      />
    )

    // Make changes
    await userEvent.type(getByLabelText(/amount/i), '105.50')
    await userEvent.type(getByLabelText(/reason/i), 'Test correction')

    // Save
    await userEvent.click(getByRole('button', { name: /save/i }))

    // Wait for success
    await waitFor(() => {
      expect(onSave).toHaveBeenCalled()
      expect(onCancel).toHaveBeenCalled() // Dialog closed
    })
  })

  it('shows impact preview when requested', async () => {
    const onCalculateImpact = jest.fn().mockResolvedValue({
      affectedEntities: 3,
      effects: [
        { type: 'recomputation', description: '3 fields recomputed', count: 3 }
      ]
    })

    const { getByRole, getByText } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={jest.fn()}
        onCancel={jest.fn()}
        onCalculateImpact={onCalculateImpact}
        showImpactPreview={true}
      />
    )

    // Click impact preview button
    const previewButton = getByRole('button', { name: /show impact/i })
    await userEvent.click(previewButton)

    // Wait for impact calculation
    await waitFor(() => {
      expect(getByText(/3 fields recomputed/i)).toBeInTheDocument()
    })
  })
})
```

### Accessibility Tests

```typescript
describe('RetroactiveCorrectionDialog Accessibility', () => {
  it('passes axe accessibility tests', async () => {
    const { container } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={jest.fn()}
        onCancel={jest.fn()}
      />
    )

    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })

  it('supports keyboard navigation', async () => {
    const { getByLabelText, getByRole } = render(
      <RetroactiveCorrectionDialog
        entity={mockEntity}
        entityType="transaction"
        editableFields={mockFields}
        onSave={jest.fn()}
        onCancel={jest.fn()}
      />
    )

    // Tab through fields
    const amountInput = getByLabelText(/amount/i)
    amountInput.focus()

    // Tab to reason
    await userEvent.tab()
    expect(getByLabelText(/reason/i)).toHaveFocus()

    // Shift+Tab back
    await userEvent.tab({ shift: true })
    expect(amountInput).toHaveFocus()
  })
})
```

---

## Accessibility

### WCAG 2.1 AA Compliance

#### Keyboard Navigation

**Requirements:**
- All form controls keyboard accessible
- Tab order logical (top to bottom)
- Modal traps focus (cannot Tab outside)
- Escape key closes dialog

**Implementation:**
```typescript
// Focus trap
useFocusTrap(dialogRef, isOpen)

// Keyboard shortcuts
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    // Escape: Close dialog
    if (e.key === 'Escape') {
      onCancel()
    }

    // Ctrl+Enter: Save (power user shortcut)
    if (e.key === 'Enter' && e.ctrlKey && isValid) {
      handleSave()
    }
  }

  if (isOpen) {
    document.addEventListener('keydown', handleKeyDown)
    return () => document.removeEventListener('keydown', handleKeyDown)
  }
}, [isOpen, isValid])
```

#### Screen Reader Support

**Requirements:**
- All fields have labels
- Error messages announced
- Progress announced
- Success/failure announced

**Implementation:**
```typescript
<div role="dialog" aria-labelledby="dialog-title" aria-describedby="dialog-description">
  <h2 id="dialog-title">Make Retroactive Correction</h2>
  <p id="dialog-description" className="sr-only">
    Correct field values with effective date control
  </p>

  {/* Field with error */}
  <label htmlFor="amount">Amount *</label>
  <input
    id="amount"
    type="number"
    aria-invalid={hasError('amount')}
    aria-errormessage={hasError('amount') ? 'amount-error' : undefined}
  />
  {hasError('amount') && (
    <div id="amount-error" role="alert" aria-live="polite">
      {getError('amount')}
    </div>
  )}

  {/* Save in progress */}
  {isSaving && (
    <div role="status" aria-live="polite" className="sr-only">
      Saving correction, please wait...
    </div>
  )}

  {/* Save success */}
  {saveSuccess && (
    <div role="alert" aria-live="polite" className="sr-only">
      Correction saved successfully
    </div>
  )}
</div>
```

#### Color Contrast & Visual Indicators

```css
/* Error state */
.field--error {
  border: 2px solid #DC2626; /* Red */
  background-color: #FEF2F2; /* Light red */
}

.error-message {
  color: #991B1B; /* Dark red, 7:1 contrast */
  font-weight: 500;
}

/* Success state */
.field--valid {
  border: 2px solid #059669; /* Green */
}

/* High impact warning */
.warning-banner--high {
  background-color: #FEF3C7; /* Light yellow */
  border-left: 4px solid #F59E0B; /* Orange */
  color: #92400E; /* Dark orange, 7:1 contrast */
}

/* High contrast mode */
@media (prefers-contrast: high) {
  .field {
    border-width: 3px;
  }

  .error-message {
    font-weight: 700;
    text-decoration: underline;
  }
}
```

---

## Performance Considerations

### Form State Management

```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import * as z from 'zod'

// Define validation schema
const correctionSchema = z.object({
  amount: z.number().min(0, 'Amount must be positive'),
  date: z.string().refine(isValidDate, 'Invalid date'),
  merchant: z.string().min(1, 'Merchant is required'),
  effectiveDate: z.date(),
  reason: z.string().min(10, 'Reason must be at least 10 characters')
})

// Use in component
function RetroactiveCorrectionDialog() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(correctionSchema)
  })

  const onSubmit = (data) => {
    onSave(data)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('amount')} />
      {errors.amount && <span>{errors.amount.message}</span>}
    </form>
  )
}
```

### Debounced Impact Calculation

```typescript
import { useDebouncedCallback } from 'use-debounce'

const debouncedCalculateImpact = useDebouncedCallback(
  async (correction: RetroactiveCorrection) => {
    setIsCalculatingImpact(true)
    try {
      const impact = await onCalculateImpact(correction)
      setImpactAnalysis(impact)
    } finally {
      setIsCalculatingImpact(false)
    }
  },
  500 // 500ms delay
)

// Trigger on field change
useEffect(() => {
  if (fieldChanges.size > 0) {
    debouncedCalculateImpact(currentCorrection)
  }
}, [fieldChanges])
```

### Memoization

```typescript
// Memoize validation
const validationErrors = useMemo(() => {
  return validateCorrection(currentCorrection, customValidationRules)
}, [currentCorrection, customValidationRules])

// Memoize field history (expensive)
const fieldHistory = useMemo(() => {
  return groupEventsByField(timelineEvents)
}, [timelineEvents])
```

---

## Related Components

### AsOfQueryBuilder

**Relationship:** Query correction results by effective date

**Integration:**
```typescript
function EntityWorkflow() {
  const [lastCorrectionDate, setLastCorrectionDate] = useState<Date | null>(null)

  async function handleCorrectionSave(correction: RetroactiveCorrection) {
    await saveCorrection(correction)
    setLastCorrectionDate(correction.effectiveDate)
  }

  return (
    <>
      <RetroactiveCorrectionDialog
        entity={entity}
        onSave={handleCorrectionSave}
        onCancel={() => setShowDialog(false)}
      />

      {lastCorrectionDate && (
        <AsOfQueryBuilder
          entityId={entity.id}
          defaultQueryMode="valid"
          initialDate={lastCorrectionDate}
          onQuery={executeQuery}
        />
      )}
    </>
  )
}
```

### TimelineViewer

**Relationship:** View correction in entity timeline

**Integration:**
```typescript
function EntityPage() {
  return (
    <Tabs>
      <TabPanel label="Current Data">
        <EntityDetails entity={entity} />
        <button onClick={() => setShowCorrection(true)}>
          Make Correction
        </button>
      </TabPanel>

      <TabPanel label="Timeline">
        <TimelineViewer
          entityId={entity.id}
          timelineEvents={events}
          onEventClick={(event) => {
            if (event.eventType === 'correction') {
              // Show correction details
            }
          }}
        />
      </TabPanel>
    </Tabs>
  )
}
```

---

## References

### Technical Documentation

- **React Hook Form**: https://react-hook-form.com/
- **Zod Validation**: https://zod.dev/
- **Date-fns**: https://date-fns.org/

### Bitemporal Theory

- **Snodgrass, Richard T.** *Developing Time-Oriented Database Applications in SQL*
- **Johnston, Tom** *Bitemporal Data: Theory and Practice*

### Accessibility Standards

- **WCAG 2.1 AA**: https://www.w3.org/WAI/WCAG21/quickref/
- **ARIA Authoring Practices**: https://www.w3.org/WAI/ARIA/apg/

---

**Component Version:** 1.0.0
**Last Updated:** 2025-10-25
**Author:** Darwin Borges
**Review Status:** Draft - Pending Technical Review
