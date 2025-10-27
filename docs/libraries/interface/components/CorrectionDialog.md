# CorrectionDialog

**Layer:** Interaction Layer (IL)
**Domain:** Corrections Flow (Vertical 4.3)
**Status:** Draft
**Last Updated:** 2025-10-24

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Architecture](#architecture)
4. [UI Components](#ui-components)
5. [Interaction Patterns](#interaction-patterns)
6. [Multi-Domain Examples](#multi-domain-examples)
7. [Edge Cases & Error Handling](#edge-cases--error-handling)
8. [Implementation Guide](#implementation-guide)
9. [Testing Strategy](#testing-strategy)
10. [Accessibility](#accessibility)
11. [Performance Considerations](#performance-considerations)
12. [Related Components](#related-components)

---

## Overview

### Purpose

The **CorrectionDialog** is a modal/drawer UI component designed for editing field values with comprehensive validation feedback, audit trail integration, and support for both single-entity and bulk-edit workflows. It serves as the primary interface for users to override extracted or computed values in the Corrections Flow.

### Key Capabilities

- **Single & Bulk Edit Modes**: Edit one entity or apply changes across multiple entities
- **Real-time Validation**: Inline validation with immediate feedback
- **Audit Trail Integration**: Display field history and require correction reasons
- **Preview Before Save**: Review changes before committing (especially in bulk mode)
- **Keyboard-First Design**: Optimized for power users with keyboard shortcuts
- **Responsive & Accessible**: Works on mobile, tablet, and desktop with full accessibility support

### Design Philosophy

The CorrectionDialog embodies the principle that **corrections are first-class operations** in the system. Unlike simple form edits, corrections:

- Require explicit justification (reason field)
- Create an audit trail (who, when, why)
- Support validation against business rules
- Enable rollback and version comparison
- Surface confidence signals from the Objective Layer

This elevates data quality management from ad-hoc fixes to a structured, accountable workflow.

---

## Component Interface

### Primary Props

```typescript
interface CorrectionDialogProps {
  // ============================================================
  // Mode Configuration
  // ============================================================

  /**
   * Edit mode: single entity or bulk edit across multiple entities
   * @default 'single'
   */
  mode: 'single' | 'bulk'

  // ============================================================
  // Data
  // ============================================================

  /**
   * Entity to edit (single mode)
   * Must include all fields from editableFields
   */
  entity?: Entity

  /**
   * Entities to edit (bulk mode)
   * All must be of the same entityType
   */
  entities?: Entity[]

  /**
   * Entity type identifier
   * Used for domain-specific validation and rendering
   * @example 'transaction', 'patient_record', 'legal_case'
   */
  entityType: string

  // ============================================================
  // Field Configuration
  // ============================================================

  /**
   * Fields that can be edited in this dialog
   * Defines UI controls, validation, and labels
   */
  editableFields: FieldDefinition[]

  /**
   * Pre-populate form with these values
   * Useful for templates or default corrections
   */
  initialValues?: Record<string, any>

  /**
   * Custom validation rules beyond field-level validation
   * @example Cross-field validation (start_date < end_date)
   */
  validationRules?: ValidationRule[]

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when user saves corrections
   * Must return promise to support async validation
   *
   * @param overrides - Array of field overrides with metadata
   * @returns Promise<void> - Resolves on success, rejects with error message
   */
  onSave: (overrides: FieldOverride[]) => Promise<void>

  /**
   * Called when user cancels without saving
   */
  onCancel: () => void

  /**
   * Called when validation state changes
   * Useful for enabling/disabling save button externally
   */
  onValidationChange?: (isValid: boolean) => void

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Show field history in dialog (audit trail)
   * Displays previous values and who changed them
   * @default true
   */
  showHistory?: boolean

  /**
   * Show confidence scores from Objective Layer
   * Helps users understand which fields may need correction
   * @default true
   */
  showConfidence?: boolean

  /**
   * Require user to provide reason for correction
   * @default true (strongly recommended for audit compliance)
   */
  requireReason?: boolean

  /**
   * Enable preview step before saving (bulk mode)
   * Shows summary of changes across all entities
   * @default true for bulk mode
   */
  enablePreview?: boolean

  /**
   * Dialog size
   * @default 'lg' (large)
   */
  size?: 'sm' | 'md' | 'lg' | 'xl' | 'full'

  /**
   * Custom title for dialog
   * @default 'Edit {entityType}' or 'Bulk Edit {count} {entityType}s'
   */
  title?: string

  /**
   * Loading state (external loading indicator)
   * @default false
   */
  isLoading?: boolean
}
```

### Supporting Types

```typescript
/**
 * Field definition with UI and validation metadata
 */
interface FieldDefinition {
  /**
   * Field identifier (must match entity schema)
   */
  field_name: string

  /**
   * Human-readable label
   */
  label: string

  /**
   * Field type determines UI control
   */
  type: 'text' | 'number' | 'date' | 'datetime' | 'select' | 'multiselect' | 'textarea' | 'boolean' | 'currency' | 'email' | 'url'

  /**
   * Options for select/multiselect fields
   */
  options?: SelectOption[]

  /**
   * Placeholder text
   */
  placeholder?: string

  /**
   * Is field required?
   */
  required?: boolean

  /**
   * Help text or description
   */
  helpText?: string

  /**
   * Validation regex (for text fields)
   */
  pattern?: string

  /**
   * Custom validation function
   * @returns Error message if invalid, undefined if valid
   */
  validate?: (value: any, allValues: Record<string, any>) => string | undefined

  /**
   * Min/max for number fields
   */
  min?: number
  max?: number

  /**
   * Format for display (e.g., currency formatting)
   */
  format?: (value: any) => string

  /**
   * Parse user input before validation
   */
  parse?: (value: string) => any

  /**
   * Should this field be read-only?
   */
  readOnly?: boolean

  /**
   * Conditional visibility
   * @example { field: 'type', value: 'expense' } - only show when type=expense
   */
  visibleWhen?: FieldCondition

  /**
   * Grid column span (1-12 for responsive layout)
   */
  span?: number
}

interface SelectOption {
  label: string
  value: string | number
  description?: string  // Additional context shown in dropdown
  disabled?: boolean
}

interface FieldCondition {
  field: string
  operator: 'eq' | 'neq' | 'in' | 'notIn' | 'gt' | 'lt'
  value: any
}

/**
 * Validation rule for cross-field validation
 */
interface ValidationRule {
  /**
   * Rule identifier
   */
  id: string

  /**
   * Fields involved in this rule
   */
  fields: string[]

  /**
   * Validation function
   * @returns Error message if invalid, undefined if valid
   */
  validate: (values: Record<string, any>) => string | undefined

  /**
   * Error message to display
   */
  message?: string

  /**
   * Severity: error blocks save, warning allows save with confirmation
   */
  severity: 'error' | 'warning'
}

/**
 * Field override created by user correction
 */
interface FieldOverride {
  /**
   * Entity ID being corrected
   */
  entity_id: string

  /**
   * Field being corrected
   */
  field_name: string

  /**
   * Original value (before correction)
   */
  original_value: any

  /**
   * New value (after correction)
   */
  override_value: any

  /**
   * User-provided reason for correction
   */
  reason: string

  /**
   * Timestamp of correction
   */
  timestamp: string

  /**
   * User who made the correction
   */
  user_id: string

  /**
   * Additional metadata
   */
  metadata?: {
    confidence_before?: number
    confidence_after?: number
    source?: string  // e.g., 'manual_correction', 'bulk_edit'
    [key: string]: any
  }
}

/**
 * Entity being edited
 */
interface Entity {
  /**
   * Unique identifier
   */
  entity_id: string

  /**
   * Entity type
   */
  type: string

  /**
   * Field values (must include all fields from editableFields)
   */
  [field: string]: any

  /**
   * Optional: Field-level confidence scores
   */
  _confidence?: Record<string, number>

  /**
   * Optional: Field-level metadata
   */
  _metadata?: Record<string, any>
}
```

---

## Architecture

### Component Structure

```
CorrectionDialog/
├── CorrectionDialog.tsx           # Main component
├── components/
│   ├── SingleEditForm.tsx         # Single entity edit form
│   ├── BulkEditForm.tsx          # Bulk edit form
│   ├── FieldInput.tsx            # Dynamic field input renderer
│   ├── FieldHistory.tsx          # Audit trail display
│   ├── ConfidenceIndicator.tsx   # Confidence score badge
│   ├── PreviewStep.tsx           # Preview before save (bulk)
│   ├── ValidationSummary.tsx     # Validation errors summary
│   └── ReasonTextarea.tsx        # Correction reason input
├── hooks/
│   ├── useFormState.ts           # Form state management
│   ├── useValidation.ts          # Validation logic
│   ├── useFieldHistory.ts        # Fetch field history
│   └── useBulkPreview.ts         # Generate bulk preview
├── utils/
│   ├── validation.ts             # Validation helpers
│   ├── formatting.ts             # Field formatting
│   └── diffing.ts                # Compute field diffs
└── types.ts                       # TypeScript definitions
```

### State Management

The CorrectionDialog manages complex state through a combination of React hooks and local state:

```typescript
interface CorrectionDialogState {
  // Form data
  formValues: Record<string, any>

  // Validation
  errors: Record<string, string>
  touched: Record<string, boolean>
  isValid: boolean

  // UI state
  step: 'edit' | 'preview' | 'saving'
  expandedHistory: Set<string>  // Which fields show history

  // Bulk mode
  previewData?: BulkPreviewData

  // API state
  isSaving: boolean
  saveError?: string
}
```

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     CorrectionDialog                         │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Form State (useFormState)              │    │
│  │  • formValues: Record<string, any>                 │    │
│  │  • initialValues: Record<string, any>              │    │
│  │  • isDirty: boolean                                │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                          │
│                   ▼                                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │          Validation (useValidation)                 │    │
│  │  • Field-level validation                          │    │
│  │  • Cross-field validation                          │    │
│  │  • Real-time error computation                     │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                          │
│                   ▼                                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │            UI Rendering                             │    │
│  │  • FieldInput (dynamic control)                    │    │
│  │  • ValidationSummary                               │    │
│  │  • FieldHistory (audit trail)                      │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                          │
│                   ▼                                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Save Handler (onSave)                       │    │
│  │  • Build FieldOverride[] from formValues           │    │
│  │  • Call parent onSave callback                     │    │
│  │  • Handle success/error                            │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Integration Points

1. **Objective Layer**: Fetch confidence scores for fields
2. **Canonical Ledger**: Save field overrides as corrections
3. **Audit Service**: Log correction events
4. **Validation Service**: Apply domain-specific validation rules
5. **User Service**: Get current user for audit metadata

---

## UI Components

### Overall Dialog Structure

```
┌────────────────────────────────────────────────────────────────┐
│  Edit Transaction                                    [X] Close  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Form Fields                                              │ │
│  │  ┌─────────────────────┐  ┌──────────────────────────┐  │ │
│  │  │ Merchant Name *     │  │ Category *               │  │ │
│  │  │ [Starbucks Corp   ] │  │ [Food & Dining ▼]        │  │ │
│  │  │ ⓘ 87% confidence    │  │ ⓘ 92% confidence         │  │ │
│  │  │ 📜 View history     │  │                          │  │ │
│  │  └─────────────────────┘  └──────────────────────────┘  │ │
│  │                                                           │ │
│  │  ┌─────────────────────┐  ┌──────────────────────────┐  │ │
│  │  │ Amount *            │  │ Date *                   │  │ │
│  │  │ [$24.50          ]  │  │ [2025-10-24]  📅         │  │ │
│  │  │                     │  │                          │  │ │
│  │  └─────────────────────┘  └──────────────────────────┘  │ │
│  │                                                           │ │
│  │  ┌──────────────────────────────────────────────────────┐│ │
│  │  │ Reason for Correction *                              ││ │
│  │  │ ┌─────────────────────────────────────────────────┐ ││ │
│  │  │ │ Receipt shows correct merchant is "Starbucks    │ ││ │
│  │  │ │ Reserve" not generic "Starbucks"                │ ││ │
│  │  │ └─────────────────────────────────────────────────┘ ││ │
│  │  └──────────────────────────────────────────────────────┘│ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ⚠️ 1 validation warning:                                      │
│     Amount exceeds typical range for this category             │
│                                                                 │
├────────────────────────────────────────────────────────────────┤
│  [Cancel]                              [Preview]  [Save] (⌘↵)  │
└────────────────────────────────────────────────────────────────┘
```

### Field Input Variations

#### Text Input
```
┌─────────────────────────────────────────┐
│ Merchant Name *                          │
│ ┌─────────────────────────────────────┐ │
│ │ Starbucks Reserve                   │ │
│ └─────────────────────────────────────┘ │
│ ⓘ 87% confidence    📜 View history     │
└─────────────────────────────────────────┘
```

#### Select Input
```
┌─────────────────────────────────────────┐
│ Category *                               │
│ ┌─────────────────────────────────────┐ │
│ │ Food & Dining              ▼        │ │
│ └─────────────────────────────────────┘ │
│ ⓘ 92% confidence                        │
└─────────────────────────────────────────┘
```

#### Number Input with Currency
```
┌─────────────────────────────────────────┐
│ Amount *                                 │
│ ┌─────────────────────────────────────┐ │
│ │ $ 24.50                             │ │
│ └─────────────────────────────────────┘ │
│ Range: $0 - $1,000                      │
└─────────────────────────────────────────┘
```

#### Date Input
```
┌─────────────────────────────────────────┐
│ Transaction Date *                       │
│ ┌─────────────────────────────────────┐ │
│ │ 2025-10-24                    📅    │ │
│ └─────────────────────────────────────┘ │
│ Format: YYYY-MM-DD                      │
└─────────────────────────────────────────┘
```

#### Textarea
```
┌─────────────────────────────────────────┐
│ Reason for Correction *                  │
│ ┌─────────────────────────────────────┐ │
│ │ Receipt shows correct merchant is   │ │
│ │ "Starbucks Reserve" not generic     │ │
│ │ "Starbucks"                         │ │
│ │                                     │ │
│ └─────────────────────────────────────┘ │
│ 0 / 500 characters                      │
└─────────────────────────────────────────┘
```

### Field History Panel

When user clicks "View history" on a field:

```
┌────────────────────────────────────────────────────────────┐
│ Merchant Name History                          [X] Collapse │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Current Value (87% confidence)                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Starbucks                                             │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Previous Corrections:                                      │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ ✏️ Manual Correction                                  │ │
│  │ Changed from: "STARBUCKS 123"                         │ │
│  │ Changed to:   "Starbucks"                             │ │
│  │ By: john.doe@example.com                              │ │
│  │ On: 2025-10-20 14:32:15                               │ │
│  │ Reason: "Normalize merchant name format"              │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 🤖 Auto-Extracted (PDF)                               │ │
│  │ Value: "STARBUCKS 123"                                │ │
│  │ Confidence: 91%                                        │ │
│  │ On: 2025-10-18 09:15:00                               │ │
│  │ Source: bank_statement_oct2025.pdf (page 3)           │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Bulk Edit Mode

When editing multiple entities:

```
┌────────────────────────────────────────────────────────────────┐
│  Bulk Edit 12 Transactions                          [X] Close  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ℹ️ Changes will apply to all 12 selected transactions         │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Fields to Edit (leave blank to keep original value)     │ │
│  │                                                           │ │
│  │  ┌─────────────────────┐  ┌──────────────────────────┐  │ │
│  │  │ Category            │  │ [Food & Dining ▼]        │  │ │
│  │  │                     │  │                          │  │ │
│  │  └─────────────────────┘  └──────────────────────────┘  │ │
│  │                                                           │ │
│  │  ┌──────────────────────────────────────────────────────┐│ │
│  │  │ Reason for Correction *                              ││ │
│  │  │ ┌─────────────────────────────────────────────────┐ ││ │
│  │  │ │ Reclassify all Starbucks to Food & Dining       │ ││ │
│  │  │ └─────────────────────────────────────────────────┘ ││ │
│  │  └──────────────────────────────────────────────────────┘│ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
├────────────────────────────────────────────────────────────────┤
│  [Cancel]                              [Preview Changes]       │
└────────────────────────────────────────────────────────────────┘
```

### Bulk Edit Preview Step

After clicking "Preview Changes":

```
┌────────────────────────────────────────────────────────────────┐
│  Preview Changes (12 transactions)                  [X] Close  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Summary of Changes:                                           │
│  • Category: 12 changes (all → "Food & Dining")                │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  txn_001 - Starbucks - $4.50 - 2025-10-01                │ │
│  │  Category: Shopping → Food & Dining                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  txn_002 - Starbucks - $5.25 - 2025-10-03                │ │
│  │  Category: Shopping → Food & Dining                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  txn_003 - Starbucks - $6.00 - 2025-10-05                │ │
│  │  Category: Shopping → Food & Dining                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ... and 9 more                                                │
│                                                                 │
├────────────────────────────────────────────────────────────────┤
│  [← Back to Edit]                                  [Save All]  │
└────────────────────────────────────────────────────────────────┘
```

### Validation States

#### No Errors (Valid State)
```
┌────────────────────────────────────────┐
│ Merchant Name *                         │
│ ┌────────────────────────────────────┐ │
│ │ Starbucks Reserve               ✓  │ │
│ └────────────────────────────────────┘ │
└────────────────────────────────────────┘
```

#### Field Error
```
┌────────────────────────────────────────┐
│ Amount *                                │
│ ┌────────────────────────────────────┐ │
│ │ -50.00                          ✗  │ │
│ └────────────────────────────────────┘ │
│ ⚠️ Amount must be positive             │
└────────────────────────────────────────┘
```

#### Field Warning (Non-blocking)
```
┌────────────────────────────────────────┐
│ Amount *                                │
│ ┌────────────────────────────────────┐ │
│ │ 250.00                          ⚠️ │ │
│ └────────────────────────────────────┘ │
│ ⚠️ Unusually high for this category    │
└────────────────────────────────────────┘
```

#### Cross-Field Validation Error
```
┌────────────────────────────────────────────────────────┐
│  ⚠️ Validation Errors (1)                              │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Start date must be before end date               │ │
│  │ Related fields: start_date, end_date             │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

### Loading States

#### Saving
```
┌────────────────────────────────────────────────────────────────┐
│  Edit Transaction                                   [X] Close  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│             ⏳ Saving corrections...                            │
│                                                                 │
│             [████████░░░░░░░░] 60%                              │
│                                                                 │
├────────────────────────────────────────────────────────────────┤
│  [Cancel]                              [Saving...] (disabled)  │
└────────────────────────────────────────────────────────────────┘
```

#### Fetching Field History
```
┌────────────────────────────────────────────────────────────┐
│ Merchant Name History                                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│             ⏳ Loading history...                           │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Mobile Responsive Layout

On mobile (< 640px), the dialog becomes fullscreen:

```
┌─────────────────────────────┐
│ ← Edit Transaction          │
├─────────────────────────────┤
│                             │
│ Merchant Name *             │
│ ┌─────────────────────────┐│
│ │ Starbucks Reserve       ││
│ └─────────────────────────┘│
│ ⓘ 87%  📜 History          │
│                             │
│ Category *                  │
│ ┌─────────────────────────┐│
│ │ Food & Dining    ▼      ││
│ └─────────────────────────┘│
│                             │
│ Amount *                    │
│ ┌─────────────────────────┐│
│ │ $ 24.50                 ││
│ └─────────────────────────┘│
│                             │
│ Date *                      │
│ ┌─────────────────────────┐│
│ │ 2025-10-24        📅    ││
│ └─────────────────────────┘│
│                             │
│ Reason *                    │
│ ┌─────────────────────────┐│
│ │ Receipt shows...        ││
│ │                         ││
│ └─────────────────────────┘│
│                             │
├─────────────────────────────┤
│ [Cancel]          [Save]    │
└─────────────────────────────┘
```

---

## Interaction Patterns

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd/Ctrl + Enter` | Save corrections (if valid) |
| `Esc` | Cancel and close dialog |
| `Tab` | Navigate to next field |
| `Shift + Tab` | Navigate to previous field |
| `Cmd/Ctrl + Z` | Undo last field change |
| `Cmd/Ctrl + K` | Focus reason textarea |
| `↑` / `↓` | Navigate options in select field |
| `Enter` | Select option in open select field |

### Form Interaction Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    User Opens Dialog                        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│         Load Initial State                                   │
│  • Pre-fill form with entity values                         │
│  • Fetch confidence scores (if showConfidence=true)         │
│  • Fetch field history (if showHistory=true)                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│         User Edits Fields                                    │
│  • onChange → Update formValues                             │
│  • onBlur → Mark field as touched                           │
│  • Validate field (real-time)                               │
│  • Display inline errors                                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│         User Clicks "Preview" (bulk mode)                    │
│  • Generate BulkPreviewData                                 │
│  • Show PreviewStep component                               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│         User Clicks "Save"                                   │
│  • Validate all fields                                      │
│  • Check cross-field validation rules                       │
│  • If invalid → Show errors, prevent save                   │
│  • If valid → Build FieldOverride[] → Call onSave()         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│         Handle Save Response                                 │
│  • Success → Close dialog, show toast notification          │
│  • Error → Show error message, keep dialog open             │
│  • Partial failure (bulk) → Show which items failed         │
└─────────────────────────────────────────────────────────────┘
```

### Validation Flow

```
Field Change
     │
     ▼
┌─────────────────────┐
│ Field-Level         │
│ Validation          │
│ • Required check    │
│ • Type check        │
│ • Pattern match     │
│ • Min/max           │
│ • Custom validate() │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Update Error State  │
│ errors[field] = msg │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Cross-Field         │
│ Validation          │
│ (on form submit)    │
│ • Date ranges       │
│ • Dependent fields  │
│ • Business rules    │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Compute isValid     │
│ (enable/disable     │
│  Save button)       │
└─────────────────────┘
```

### Field History Interaction

```
User Clicks "View History"
         │
         ▼
┌─────────────────────────────┐
│ Fetch Field History         │
│ GET /api/entities/{id}/     │
│     field_history/{field}   │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Display History Panel       │
│ • Current value             │
│ • Previous corrections      │
│ • Auto-extracted values     │
│ • Timestamps & users        │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ User Clicks "Collapse"      │
│ → Hide history panel        │
└─────────────────────────────┘
```

---

## Multi-Domain Examples

### 1. Finance: Edit Transaction

**Use Case**: User notices a transaction was miscategorized by the auto-extractor.

**Configuration**:
```typescript
<CorrectionDialog
  mode="single"
  entity={{
    entity_id: "txn_12345",
    type: "transaction",
    merchant: "STARBUCKS 123",
    category: "Shopping",
    amount: 24.50,
    date: "2025-10-24",
    _confidence: {
      merchant: 0.87,
      category: 0.65,  // Low confidence
      amount: 0.99,
      date: 0.98
    }
  }}
  entityType="transaction"
  editableFields={[
    {
      field_name: "merchant",
      label: "Merchant Name",
      type: "text",
      required: true,
      helpText: "Normalized merchant name"
    },
    {
      field_name: "category",
      label: "Category",
      type: "select",
      required: true,
      options: [
        { label: "Food & Dining", value: "food_dining" },
        { label: "Shopping", value: "shopping" },
        { label: "Transportation", value: "transportation" },
        { label: "Entertainment", value: "entertainment" },
        { label: "Bills & Utilities", value: "bills_utilities" },
        { label: "Healthcare", value: "healthcare" }
      ]
    },
    {
      field_name: "amount",
      label: "Amount",
      type: "currency",
      required: true,
      min: 0,
      validate: (value) => {
        if (value > 10000) {
          return "Large transaction - please verify amount";
        }
      }
    },
    {
      field_name: "date",
      label: "Transaction Date",
      type: "date",
      required: true,
      validate: (value) => {
        const date = new Date(value);
        const now = new Date();
        if (date > now) {
          return "Date cannot be in the future";
        }
      }
    }
  ]}
  validationRules={[
    {
      id: "amount_category_check",
      fields: ["amount", "category"],
      severity: "warning",
      validate: (values) => {
        if (values.category === "food_dining" && values.amount > 100) {
          return "Large amount for Food & Dining category";
        }
      }
    }
  ]}
  onSave={async (overrides) => {
    await api.saveCorrections(overrides);
    toast.success("Transaction updated successfully");
  }}
  onCancel={() => setDialogOpen(false)}
  showHistory={true}
  showConfidence={true}
  requireReason={true}
/>
```

**Expected Workflow**:
1. User opens dialog, sees "Category" has low confidence (65%)
2. User clicks "View history" to see extraction source
3. User changes category from "Shopping" to "Food & Dining"
4. User adds reason: "Receipt confirms this is a coffee purchase"
5. System shows warning: "Large amount for Food & Dining category"
6. User confirms and saves
7. FieldOverride created with full audit trail

---

### 2. Healthcare: Edit Patient Record

**Use Case**: Nurse corrects diagnosis code extracted from handwritten notes.

**Configuration**:
```typescript
<CorrectionDialog
  mode="single"
  entity={{
    entity_id: "patient_98765",
    type: "patient_record",
    diagnosis_code: "J20.9",  // Acute bronchitis
    diagnosis_description: "Acute bronchitis, unspecified",
    provider: "Dr. Smith",
    procedure_date: "2025-10-20",
    procedure_code: "99213",
    _confidence: {
      diagnosis_code: 0.72,  // OCR from handwritten notes
      provider: 0.95,
      procedure_date: 0.99
    }
  }}
  entityType="patient_record"
  editableFields={[
    {
      field_name: "diagnosis_code",
      label: "ICD-10 Diagnosis Code",
      type: "text",
      required: true,
      pattern: "^[A-Z][0-9]{2}(\\.[0-9]{1,2})?$",
      helpText: "Format: A00.0",
      validate: (value) => {
        if (!ICD10_CODES.includes(value)) {
          return "Invalid ICD-10 code. Please verify.";
        }
      }
    },
    {
      field_name: "diagnosis_description",
      label: "Diagnosis Description",
      type: "textarea",
      required: true
    },
    {
      field_name: "provider",
      label: "Provider Name",
      type: "select",
      required: true,
      options: [
        { label: "Dr. John Smith", value: "dr_smith" },
        { label: "Dr. Jane Doe", value: "dr_doe" },
        { label: "Dr. Bob Johnson", value: "dr_johnson" }
      ]
    },
    {
      field_name: "procedure_date",
      label: "Procedure Date",
      type: "date",
      required: true
    },
    {
      field_name: "procedure_code",
      label: "CPT Procedure Code",
      type: "text",
      required: true,
      pattern: "^[0-9]{5}$",
      helpText: "5-digit CPT code"
    }
  ]}
  validationRules={[
    {
      id: "diagnosis_procedure_match",
      fields: ["diagnosis_code", "procedure_code"],
      severity: "warning",
      validate: (values) => {
        if (!isProcedureValidForDiagnosis(values.procedure_code, values.diagnosis_code)) {
          return "This procedure code is unusual for this diagnosis";
        }
      }
    }
  ]}
  onSave={async (overrides) => {
    await api.savePatientRecordCorrections(overrides);
    await api.auditLog({
      action: "patient_record_correction",
      entity_id: "patient_98765",
      user: currentUser,
      timestamp: new Date().toISOString()
    });
    toast.success("Patient record updated");
  }}
  showHistory={true}
  requireReason={true}
  size="lg"
/>
```

**Compliance Notes**:
- All corrections logged for HIPAA audit trail
- Reason required for medical record corrections
- Validation against ICD-10 and CPT code databases
- Cross-field validation for diagnosis-procedure consistency

---

### 3. Legal: Edit Case Information

**Use Case**: Paralegal corrects case filing date after reviewing court documents.

**Configuration**:
```typescript
<CorrectionDialog
  mode="single"
  entity={{
    entity_id: "case_54321",
    type: "legal_case",
    case_number: "2025-CV-12345",
    attorney: "Sarah Johnson, Esq.",
    filing_date: "2025-10-15",
    case_type: "Civil Litigation",
    court: "Superior Court of California",
    status: "Active",
    _confidence: {
      case_number: 0.98,
      filing_date: 0.68,  // OCR from scanned document
      attorney: 0.92
    }
  }}
  entityType="legal_case"
  editableFields={[
    {
      field_name: "case_number",
      label: "Case Number",
      type: "text",
      required: true,
      pattern: "^[0-9]{4}-[A-Z]{2}-[0-9]{5}$",
      helpText: "Format: YYYY-XX-#####"
    },
    {
      field_name: "attorney",
      label: "Assigned Attorney",
      type: "select",
      required: true,
      options: [
        { label: "Sarah Johnson, Esq.", value: "sjohnson" },
        { label: "Michael Chen, Esq.", value: "mchen" },
        { label: "Emily Rodriguez, Esq.", value: "erodriguez" }
      ]
    },
    {
      field_name: "filing_date",
      label: "Filing Date",
      type: "date",
      required: true,
      validate: (value) => {
        const filingDate = new Date(value);
        const today = new Date();
        if (filingDate > today) {
          return "Filing date cannot be in the future";
        }
      }
    },
    {
      field_name: "case_type",
      label: "Case Type",
      type: "select",
      required: true,
      options: [
        { label: "Civil Litigation", value: "civil_litigation" },
        { label: "Criminal Defense", value: "criminal_defense" },
        { label: "Family Law", value: "family_law" },
        { label: "Corporate Law", value: "corporate_law" },
        { label: "Intellectual Property", value: "ip" }
      ]
    },
    {
      field_name: "court",
      label: "Court",
      type: "text",
      required: true
    },
    {
      field_name: "status",
      label: "Case Status",
      type: "select",
      required: true,
      options: [
        { label: "Active", value: "active" },
        { label: "Pending", value: "pending" },
        { label: "Closed", value: "closed" },
        { label: "Settled", value: "settled" }
      ]
    }
  ]}
  validationRules={[
    {
      id: "status_filing_date",
      fields: ["status", "filing_date"],
      severity: "error",
      validate: (values) => {
        if (values.status === "closed") {
          const filingDate = new Date(values.filing_date);
          const today = new Date();
          const daysSinceFiling = Math.floor((today - filingDate) / (1000 * 60 * 60 * 24));
          if (daysSinceFiling < 30) {
            return "Cases cannot be closed within 30 days of filing";
          }
        }
      }
    }
  ]}
  onSave={async (overrides) => {
    await api.saveLegalCaseCorrections(overrides);
    toast.success("Case information updated");
  }}
  showHistory={true}
  requireReason={true}
/>
```

---

### 4. Research (RSRCH - Utilitario): Edit Founder Fact Metadata

**Use Case**: RSRCH analyst corrects entity names and investment amounts extracted from web sources.

**Configuration**:
```typescript
<CorrectionDialog
  mode="single"
  entity={{
    entity_id: "fact_sama_helion_001",
    type: "founder_fact",
    claim: "Sam Altman invested $375M in Helion Energy",
    subject_entity: "@sama",
    object_entity: "Helion",
    investment_amount: 375000000,
    source_type: "web_article",
    source_url: "https://techcrunch.com/sama-helion-investment",
    discovered_at: "2024-03-15T10:00:00Z",
    _confidence: {
      subject_entity: 0.78,  // Twitter handle needs normalization
      investment_amount: 0.95,
      object_entity: 0.92
    }
  }}
  entityType="founder_fact"
  editableFields={[
    {
      field_name: "claim",
      label: "Fact Claim",
      type: "textarea",
      required: true,
      helpText: "Full fact statement"
    },
    {
      field_name: "subject_entity",
      label: "Subject Entity",
      type: "text",
      required: true,
      helpText: "Canonical entity name (e.g., 'Sam Altman')",
      validate: (value) => {
        if (value.startsWith("@")) {
          return "Use canonical name, not Twitter handle";
        }
      }
    },
    {
      field_name: "investment_amount",
      label: "Investment Amount",
      type: "number",
      required: true,
      min: 0,
      helpText: "Amount in USD",
      validate: (value) => {
        if (value < 0) {
          return "Amount cannot be negative";
        }
      }
    },
    {
      field_name: "object_entity",
      label: "Object Entity",
      type: "text",
      required: true,
      helpText: "Company or entity receiving investment"
    },
    {
      field_name: "source_type",
      label: "Source Type",
      type: "select",
      options: ["web_article", "podcast", "tweet", "interview"],
      required: true
    },
    {
      field_name: "source_url",
      label: "Source URL",
      type: "text",
      pattern: "^https?://.*$",
      helpText: "Format: https://..."
    },
    {
      field_name: "discovered_at",
      label: "Discovery Date",
      type: "date",
      helpText: "When this fact was first discovered"
    }
  ]}
  onSave={async (overrides) => {
    await api.saveFounderFactCorrections(overrides);
    toast.success("Founder fact metadata updated");
  }}
  showHistory={true}
  showConfidence={true}
/>
```

---

### 5. E-commerce: Bulk Edit Products

**Use Case**: Store manager reclassifies 50 products from "Electronics" to "Smart Home".

**Configuration**:
```typescript
<CorrectionDialog
  mode="bulk"
  entities={selectedProducts}  // 50 products
  entityType="product"
  editableFields={[
    {
      field_name: "category",
      label: "Category",
      type: "select",
      required: true,
      options: [
        { label: "Electronics", value: "electronics" },
        { label: "Smart Home", value: "smart_home" },
        { label: "Appliances", value: "appliances" },
        { label: "Accessories", value: "accessories" }
      ]
    },
    {
      field_name: "price",
      label: "Price",
      type: "currency",
      helpText: "Leave blank to keep original price"
    },
    {
      field_name: "sku",
      label: "SKU Prefix",
      type: "text",
      helpText: "Optional: Update SKU prefix for all products"
    }
  ]}
  onSave={async (overrides) => {
    // overrides will have 50 FieldOverride objects (one per product)
    const results = await api.bulkUpdateProducts(overrides);

    const successCount = results.filter(r => r.success).length;
    const failCount = results.filter(r => !r.success).length;

    if (failCount === 0) {
      toast.success(`All ${successCount} products updated successfully`);
    } else {
      toast.warning(`${successCount} products updated, ${failCount} failed`);
    }
  }}
  enablePreview={true}
  requireReason={true}
  size="xl"
/>
```

**Bulk Preview**:
```
┌────────────────────────────────────────────────────────────────┐
│  Preview Changes (50 products)                                 │
├────────────────────────────────────────────────────────────────┤
│  Summary:                                                       │
│  • Category: 50 changes (all → "Smart Home")                   │
│                                                                 │
│  Products to Update:                                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ SKU-001 - Echo Dot                                        │ │
│  │ Category: Electronics → Smart Home                        │ │
│  └──────────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ SKU-002 - Google Nest Hub                                │ │
│  │ Category: Electronics → Smart Home                        │ │
│  └──────────────────────────────────────────────────────────┘ │
│  ... and 48 more                                               │
│                                                                 │
│  [← Back to Edit]                              [Save All (50)] │
└────────────────────────────────────────────────────────────────┘
```

---

### 6. SaaS: Edit Subscription

**Use Case**: Customer success manager corrects subscription plan and MRR.

**Configuration**:
```typescript
<CorrectionDialog
  mode="single"
  entity={{
    entity_id: "sub_99887",
    type: "subscription",
    customer_name: "Acme Corp",
    plan: "professional",
    mrr: 299,
    billing_cycle: "monthly",
    start_date: "2025-01-15",
    status: "active",
    _confidence: {
      plan: 0.82,  // Inferred from usage patterns
      mrr: 0.95
    }
  }}
  entityType="subscription"
  editableFields={[
    {
      field_name: "plan",
      label: "Plan",
      type: "select",
      required: true,
      options: [
        { label: "Starter ($99/mo)", value: "starter" },
        { label: "Professional ($299/mo)", value: "professional" },
        { label: "Enterprise ($999/mo)", value: "enterprise" }
      ]
    },
    {
      field_name: "mrr",
      label: "Monthly Recurring Revenue",
      type: "currency",
      required: true,
      min: 0,
      validate: (value, allValues) => {
        const expectedMRR = {
          starter: 99,
          professional: 299,
          enterprise: 999
        };

        if (value !== expectedMRR[allValues.plan]) {
          return `MRR should be $${expectedMRR[allValues.plan]} for ${allValues.plan} plan`;
        }
      }
    },
    {
      field_name: "billing_cycle",
      label: "Billing Cycle",
      type: "select",
      required: true,
      options: [
        { label: "Monthly", value: "monthly" },
        { label: "Quarterly", value: "quarterly" },
        { label: "Annual", value: "annual" }
      ]
    },
    {
      field_name: "status",
      label: "Status",
      type: "select",
      required: true,
      options: [
        { label: "Active", value: "active" },
        { label: "Paused", value: "paused" },
        { label: "Cancelled", value: "cancelled" },
        { label: "Trial", value: "trial" }
      ]
    }
  ]}
  validationRules={[
    {
      id: "plan_mrr_consistency",
      fields: ["plan", "mrr"],
      severity: "error",
      validate: (values) => {
        const expectedMRR = {
          starter: 99,
          professional: 299,
          enterprise: 999
        };

        if (values.mrr !== expectedMRR[values.plan]) {
          return `MRR ($${values.mrr}) does not match ${values.plan} plan ($${expectedMRR[values.plan]})`;
        }
      }
    }
  ]}
  onSave={async (overrides) => {
    await api.saveSubscriptionCorrections(overrides);

    // Trigger MRR recalculation
    await api.recalculateMRR();

    toast.success("Subscription updated");
  }}
  showHistory={true}
  requireReason={true}
/>
```

---

### 7. Insurance: Edit Claim

**Use Case**: Claims adjuster corrects claim amount and approval status.

**Configuration**:
```typescript
<CorrectionDialog
  mode="single"
  entity={{
    entity_id: "claim_77665",
    type: "insurance_claim",
    claim_number: "CLM-2025-10-24-001",
    policy_holder: "Jane Smith",
    claim_type: "Medical",
    claim_amount: 1250.00,
    approved_amount: 0,
    status: "Under Review",
    submission_date: "2025-10-20",
    _confidence: {
      claim_amount: 0.89,  // OCR from submitted forms
      status: 0.95
    }
  }}
  entityType="insurance_claim"
  editableFields={[
    {
      field_name: "claim_amount",
      label: "Claim Amount",
      type: "currency",
      required: true,
      min: 0,
      helpText: "Total amount claimed"
    },
    {
      field_name: "approved_amount",
      label: "Approved Amount",
      type: "currency",
      required: true,
      min: 0,
      validate: (value, allValues) => {
        if (value > allValues.claim_amount) {
          return "Approved amount cannot exceed claim amount";
        }
      }
    },
    {
      field_name: "status",
      label: "Claim Status",
      type: "select",
      required: true,
      options: [
        { label: "Under Review", value: "under_review" },
        { label: "Approved", value: "approved" },
        { label: "Partially Approved", value: "partially_approved" },
        { label: "Denied", value: "denied" },
        { label: "Requires Additional Info", value: "pending_info" }
      ]
    },
    {
      field_name: "denial_reason",
      label: "Denial Reason",
      type: "textarea",
      visibleWhen: {
        field: "status",
        operator: "eq",
        value: "denied"
      },
      required: true
    }
  ]}
  validationRules={[
    {
      id: "approved_amount_status",
      fields: ["approved_amount", "status"],
      severity: "error",
      validate: (values) => {
        if (values.status === "approved" && values.approved_amount === 0) {
          return "Approved claims must have an approved amount > 0";
        }

        if (values.status === "denied" && values.approved_amount > 0) {
          return "Denied claims cannot have an approved amount";
        }
      }
    }
  ]}
  onSave={async (overrides) => {
    await api.saveClaimCorrections(overrides);

    // Send notification to policy holder
    if (overrides.some(o => o.field_name === "status")) {
      await api.sendClaimStatusNotification(entity.entity_id);
    }

    toast.success("Claim updated");
  }}
  showHistory={true}
  requireReason={true}
/>
```

---

## Edge Cases & Error Handling

### 1. Validation Errors on Save

**Scenario**: User submits form with invalid data.

**Handling**:
```typescript
const handleSave = async () => {
  // Client-side validation
  const errors = validateAllFields(formValues, editableFields, validationRules);

  if (Object.keys(errors).length > 0) {
    setErrors(errors);
    toast.error("Please fix validation errors before saving");

    // Focus first field with error
    const firstErrorField = Object.keys(errors)[0];
    document.getElementById(`field-${firstErrorField}`)?.focus();

    return;
  }

  // Attempt save
  try {
    setIsSaving(true);
    await onSave(buildOverrides(formValues));
    toast.success("Changes saved successfully");
    onCancel();  // Close dialog
  } catch (error) {
    // Server-side validation error
    if (error.code === "VALIDATION_ERROR") {
      setErrors(error.fieldErrors);
      toast.error(error.message || "Validation failed");
    } else {
      toast.error("Failed to save changes. Please try again.");
    }
  } finally {
    setIsSaving(false);
  }
};
```

**UI Display**:
```
┌────────────────────────────────────────────────────────────┐
│  ⚠️ Validation Errors (2)                                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ • Amount must be positive                            │ │
│  │ • Date cannot be in the future                       │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

---

### 2. Concurrent Edit Warning

**Scenario**: Field was updated by another user while dialog was open.

**Detection**:
```typescript
const handleSave = async () => {
  try {
    await onSave(buildOverrides(formValues));
  } catch (error) {
    if (error.code === "CONCURRENT_MODIFICATION") {
      // Show conflict resolution UI
      setConflictData({
        field: error.field,
        yourValue: formValues[error.field],
        theirValue: error.currentValue,
        theirUser: error.modifiedBy,
        theirTimestamp: error.modifiedAt
      });
      setShowConflictDialog(true);
    }
  }
};
```

**Conflict Resolution UI**:
```
┌────────────────────────────────────────────────────────────┐
│  ⚠️ Concurrent Edit Detected                               │
├────────────────────────────────────────────────────────────┤
│  The "category" field was updated by another user while    │
│  you were editing.                                         │
│                                                             │
│  Your value:        Food & Dining                          │
│  Their value:       Shopping                               │
│  Modified by:       jane.doe@example.com                   │
│  Modified at:       2025-10-24 10:32:15                    │
│                                                             │
│  What would you like to do?                                │
│                                                             │
│  [ Keep My Value ]  [ Use Their Value ]  [ Cancel Edit ]   │
└────────────────────────────────────────────────────────────┘
```

---

### 3. Bulk Edit with Partial Failures

**Scenario**: In bulk mode, some entities succeed and others fail.

**Handling**:
```typescript
const handleBulkSave = async () => {
  const overrides = entities.map(entity =>
    buildOverridesForEntity(entity, formValues)
  ).flat();

  try {
    const results = await onSave(overrides);

    // Results: [{ entity_id, success, error? }]
    const succeeded = results.filter(r => r.success);
    const failed = results.filter(r => !r.success);

    if (failed.length === 0) {
      toast.success(`All ${succeeded.length} items updated`);
      onCancel();
    } else {
      // Show partial failure UI
      setPartialFailureData({
        succeeded: succeeded.length,
        failed: failed.length,
        failures: failed
      });
      setShowPartialFailureDialog(true);
    }
  } catch (error) {
    toast.error("Bulk update failed. No changes were saved.");
  }
};
```

**Partial Failure UI**:
```
┌────────────────────────────────────────────────────────────┐
│  ⚠️ Partial Success (47/50 items updated)                  │
├────────────────────────────────────────────────────────────┤
│  47 items were updated successfully.                       │
│  3 items failed with errors:                               │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ txn_123 - Amount validation failed                   │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ txn_456 - Concurrent modification detected           │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ txn_789 - Insufficient permissions                   │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  [ Download Error Report ]               [ Close ]         │
└────────────────────────────────────────────────────────────┘
```

---

### 4. Long Field Lists (Scrollable)

**Scenario**: Entity has 20+ editable fields.

**Handling**:
- Group fields into sections
- Use accordion/collapsible sections
- Implement virtual scrolling for performance
- Add "Jump to field" quick navigation

**UI with Sections**:
```
┌────────────────────────────────────────────────────────────┐
│  Edit Transaction                              [X] Close   │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ▼ Basic Information                                       │
│    • Merchant Name                                         │
│    • Category                                              │
│    • Amount                                                │
│    • Date                                                  │
│                                                             │
│  ▶ Advanced Details (12 fields)                            │
│                                                             │
│  ▶ Metadata (5 fields)                                     │
│                                                             │
│  ▼ Correction Details                                      │
│    • Reason for Correction                                 │
│                                                             │
├────────────────────────────────────────────────────────────┤
│  [Cancel]                                         [Save]    │
└────────────────────────────────────────────────────────────┘
```

---

### 5. Mobile Responsiveness

**Challenges**:
- Limited screen space
- Touch interactions
- Keyboard visibility

**Solutions**:
- Fullscreen dialog on mobile
- Single-column layout
- Larger touch targets (min 44px)
- Fixed header/footer with scrollable content
- Auto-scroll to focused field when keyboard opens

**Mobile Layout**:
```
┌─────────────────────────────┐
│ ← Edit Transaction          │  ← Fixed header
├─────────────────────────────┤
│                             │
│ Merchant Name *             │  ↓
│ ┌─────────────────────────┐│  |
│ │ Starbucks               ││  |
│ └─────────────────────────┘│  |
│                             │  |
│ Category *                  │  | Scrollable
│ ┌─────────────────────────┐│  | content
│ │ Food & Dining    ▼      ││  |
│ └─────────────────────────┘│  |
│                             │  |
│ Amount *                    │  |
│ ┌─────────────────────────┐│  |
│ │ $ 24.50                 ││  |
│ └─────────────────────────┘│  ↑
│                             │
├─────────────────────────────┤
│ [Cancel]          [Save]    │  ← Fixed footer
└─────────────────────────────┘
```

---

### 6. Network Errors

**Scenario**: Save request fails due to network issue.

**Handling**:
```typescript
const handleSave = async () => {
  try {
    setIsSaving(true);
    await onSave(buildOverrides(formValues));
  } catch (error) {
    if (error.code === "NETWORK_ERROR" || !navigator.onLine) {
      toast.error("No network connection. Changes not saved.");

      // Optionally: Save to local storage for retry
      localStorage.setItem(
        `pending_correction_${entity.entity_id}`,
        JSON.stringify(formValues)
      );

      setShowRetryButton(true);
    } else {
      toast.error("Failed to save. Please try again.");
    }
  } finally {
    setIsSaving(false);
  }
};
```

---

### 7. Unsaved Changes Warning

**Scenario**: User tries to close dialog with unsaved changes.

**Handling**:
```typescript
const handleCancel = () => {
  if (isDirty) {
    if (confirm("You have unsaved changes. Are you sure you want to close?")) {
      onCancel();
    }
  } else {
    onCancel();
  }
};

// Browser navigation guard
useEffect(() => {
  const handleBeforeUnload = (e: BeforeUnloadEvent) => {
    if (isDirty) {
      e.preventDefault();
      e.returnValue = "";
    }
  };

  window.addEventListener("beforeunload", handleBeforeUnload);
  return () => window.removeEventListener("beforeunload", handleBeforeUnload);
}, [isDirty]);
```

---

## Implementation Guide

### Basic Implementation

```typescript
// CorrectionDialog.tsx
import React, { useState, useEffect } from 'react';
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogFooter
} from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { FieldInput } from './components/FieldInput';
import { ValidationSummary } from './components/ValidationSummary';
import { ReasonTextarea } from './components/ReasonTextarea';
import { useFormState } from './hooks/useFormState';
import { useValidation } from './hooks/useValidation';

export function CorrectionDialog({
  mode = 'single',
  entity,
  entities,
  entityType,
  editableFields,
  validationRules = [],
  onSave,
  onCancel,
  showHistory = true,
  showConfidence = true,
  requireReason = true,
  enablePreview = true,
  size = 'lg'
}: CorrectionDialogProps) {

  // ============================================================
  // State Management
  // ============================================================

  const initialValues = mode === 'single'
    ? extractFieldValues(entity, editableFields)
    : {};

  const {
    formValues,
    updateField,
    resetForm,
    isDirty
  } = useFormState(initialValues);

  const {
    errors,
    touched,
    isValid,
    validateField,
    validateAll,
    markTouched
  } = useValidation(formValues, editableFields, validationRules);

  const [step, setStep] = useState<'edit' | 'preview'>('edit');
  const [isSaving, setIsSaving] = useState(false);
  const [saveError, setSaveError] = useState<string | null>(null);

  // ============================================================
  // Handlers
  // ============================================================

  const handleFieldChange = (fieldName: string, value: any) => {
    updateField(fieldName, value);
    validateField(fieldName, value);
  };

  const handleFieldBlur = (fieldName: string) => {
    markTouched(fieldName);
  };

  const handleSave = async () => {
    // Validate all fields
    const validationErrors = validateAll();
    if (Object.keys(validationErrors).length > 0) {
      toast.error("Please fix validation errors");
      return;
    }

    // Build overrides
    const overrides = buildFieldOverrides(
      mode === 'single' ? [entity!] : entities!,
      formValues,
      editableFields,
      getCurrentUser()
    );

    // Save
    try {
      setIsSaving(true);
      setSaveError(null);
      await onSave(overrides);
      toast.success(
        mode === 'single'
          ? "Changes saved successfully"
          : `${overrides.length} items updated`
      );
      onCancel();
    } catch (error) {
      setSaveError(error.message);
      toast.error("Failed to save changes");
    } finally {
      setIsSaving(false);
    }
  };

  const handleCancel = () => {
    if (isDirty) {
      if (confirm("You have unsaved changes. Discard?")) {
        onCancel();
      }
    } else {
      onCancel();
    }
  };

  const handlePreview = () => {
    if (mode === 'bulk' && enablePreview) {
      setStep('preview');
    } else {
      handleSave();
    }
  };

  // ============================================================
  // Keyboard Shortcuts
  // ============================================================

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      // Cmd/Ctrl + Enter to save
      if ((e.metaKey || e.ctrlKey) && e.key === 'Enter') {
        e.preventDefault();
        if (isValid && !isSaving) {
          handleSave();
        }
      }

      // Escape to cancel
      if (e.key === 'Escape' && !isSaving) {
        e.preventDefault();
        handleCancel();
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [isValid, isSaving, isDirty]);

  // ============================================================
  // Render
  // ============================================================

  const dialogTitle = mode === 'single'
    ? `Edit ${entityType}`
    : `Bulk Edit ${entities?.length} ${entityType}s`;

  return (
    <Dialog open onOpenChange={handleCancel}>
      <DialogContent className={`max-w-${size}`}>
        <DialogHeader>
          <DialogTitle>{dialogTitle}</DialogTitle>
        </DialogHeader>

        {step === 'edit' && (
          <div className="space-y-4 max-h-[60vh] overflow-y-auto">
            {/* Validation Summary */}
            {Object.keys(errors).length > 0 && (
              <ValidationSummary errors={errors} />
            )}

            {/* Form Fields */}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              {editableFields.map((field) => (
                <FieldInput
                  key={field.field_name}
                  field={field}
                  value={formValues[field.field_name]}
                  error={touched[field.field_name] ? errors[field.field_name] : undefined}
                  onChange={(value) => handleFieldChange(field.field_name, value)}
                  onBlur={() => handleFieldBlur(field.field_name)}
                  showConfidence={showConfidence}
                  confidence={entity?._confidence?.[field.field_name]}
                  showHistory={showHistory}
                  entityId={entity?.entity_id}
                />
              ))}
            </div>

            {/* Reason Textarea */}
            {requireReason && (
              <ReasonTextarea
                value={formValues._reason}
                onChange={(value) => handleFieldChange('_reason', value)}
                required
                error={touched._reason ? errors._reason : undefined}
              />
            )}

            {/* Save Error */}
            {saveError && (
              <div className="bg-red-50 border border-red-200 rounded p-3 text-red-700">
                {saveError}
              </div>
            )}
          </div>
        )}

        {step === 'preview' && (
          <PreviewStep
            entities={entities!}
            formValues={formValues}
            editableFields={editableFields}
            onBack={() => setStep('edit')}
          />
        )}

        <DialogFooter>
          <Button
            variant="outline"
            onClick={handleCancel}
            disabled={isSaving}
          >
            Cancel
          </Button>

          {step === 'edit' && (
            <Button
              onClick={handlePreview}
              disabled={!isValid || isSaving}
            >
              {mode === 'bulk' && enablePreview ? 'Preview' : 'Save'}
              {!isSaving && <kbd className="ml-2">⌘↵</kbd>}
            </Button>
          )}

          {step === 'preview' && (
            <Button
              onClick={handleSave}
              disabled={isSaving}
            >
              {isSaving ? 'Saving...' : `Save All (${entities?.length})`}
            </Button>
          )}
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

---

### FieldInput Component

```typescript
// components/FieldInput.tsx
import React, { useState } from 'react';
import { Input } from '@/components/ui/input';
import { Select } from '@/components/ui/select';
import { Textarea } from '@/components/ui/textarea';
import { Label } from '@/components/ui/label';
import { ConfidenceIndicator } from './ConfidenceIndicator';
import { FieldHistory } from './FieldHistory';

export function FieldInput({
  field,
  value,
  error,
  onChange,
  onBlur,
  showConfidence,
  confidence,
  showHistory,
  entityId
}: FieldInputProps) {

  const [historyExpanded, setHistoryExpanded] = useState(false);

  // Render appropriate input based on field type
  const renderInput = () => {
    switch (field.type) {
      case 'text':
      case 'email':
      case 'url':
        return (
          <Input
            id={`field-${field.field_name}`}
            type={field.type}
            value={value || ''}
            onChange={(e) => onChange(e.target.value)}
            onBlur={onBlur}
            placeholder={field.placeholder}
            disabled={field.readOnly}
            className={error ? 'border-red-500' : ''}
          />
        );

      case 'number':
      case 'currency':
        return (
          <Input
            id={`field-${field.field_name}`}
            type="number"
            value={value || ''}
            onChange={(e) => onChange(parseFloat(e.target.value))}
            onBlur={onBlur}
            placeholder={field.placeholder}
            min={field.min}
            max={field.max}
            step={field.type === 'currency' ? '0.01' : '1'}
            disabled={field.readOnly}
            className={error ? 'border-red-500' : ''}
          />
        );

      case 'date':
      case 'datetime':
        return (
          <Input
            id={`field-${field.field_name}`}
            type={field.type === 'datetime' ? 'datetime-local' : 'date'}
            value={value || ''}
            onChange={(e) => onChange(e.target.value)}
            onBlur={onBlur}
            disabled={field.readOnly}
            className={error ? 'border-red-500' : ''}
          />
        );

      case 'select':
        return (
          <Select
            value={value}
            onValueChange={onChange}
            onBlur={onBlur}
            disabled={field.readOnly}
          >
            {field.options?.map((option) => (
              <option key={option.value} value={option.value}>
                {option.label}
              </option>
            ))}
          </Select>
        );

      case 'textarea':
        return (
          <Textarea
            id={`field-${field.field_name}`}
            value={value || ''}
            onChange={(e) => onChange(e.target.value)}
            onBlur={onBlur}
            placeholder={field.placeholder}
            disabled={field.readOnly}
            rows={4}
            className={error ? 'border-red-500' : ''}
          />
        );

      case 'boolean':
        return (
          <input
            id={`field-${field.field_name}`}
            type="checkbox"
            checked={value || false}
            onChange={(e) => onChange(e.target.checked)}
            onBlur={onBlur}
            disabled={field.readOnly}
          />
        );

      default:
        return null;
    }
  };

  return (
    <div className={`space-y-2 ${field.span ? `col-span-${field.span}` : ''}`}>
      <Label htmlFor={`field-${field.field_name}`}>
        {field.label}
        {field.required && <span className="text-red-500 ml-1">*</span>}
      </Label>

      {renderInput()}

      {/* Help Text */}
      {field.helpText && !error && (
        <p className="text-xs text-gray-500">{field.helpText}</p>
      )}

      {/* Error Message */}
      {error && (
        <p className="text-xs text-red-500">⚠️ {error}</p>
      )}

      {/* Confidence & History */}
      <div className="flex items-center gap-3 text-sm">
        {showConfidence && confidence !== undefined && (
          <ConfidenceIndicator confidence={confidence} />
        )}

        {showHistory && entityId && (
          <button
            type="button"
            className="text-blue-600 hover:underline"
            onClick={() => setHistoryExpanded(!historyExpanded)}
          >
            📜 {historyExpanded ? 'Hide' : 'View'} history
          </button>
        )}
      </div>

      {/* Field History Panel */}
      {historyExpanded && (
        <FieldHistory
          entityId={entityId!}
          fieldName={field.field_name}
          onClose={() => setHistoryExpanded(false)}
        />
      )}
    </div>
  );
}
```

---

### useFormState Hook

```typescript
// hooks/useFormState.ts
import { useState, useCallback } from 'react';

export function useFormState(initialValues: Record<string, any>) {
  const [formValues, setFormValues] = useState(initialValues);
  const [initialState] = useState(initialValues);

  const updateField = useCallback((fieldName: string, value: any) => {
    setFormValues(prev => ({
      ...prev,
      [fieldName]: value
    }));
  }, []);

  const updateFields = useCallback((updates: Record<string, any>) => {
    setFormValues(prev => ({
      ...prev,
      ...updates
    }));
  }, []);

  const resetForm = useCallback(() => {
    setFormValues(initialState);
  }, [initialState]);

  const isDirty = JSON.stringify(formValues) !== JSON.stringify(initialState);

  return {
    formValues,
    updateField,
    updateFields,
    resetForm,
    isDirty
  };
}
```

---

### useValidation Hook

```typescript
// hooks/useValidation.ts
import { useState, useCallback } from 'react';

export function useValidation(
  formValues: Record<string, any>,
  editableFields: FieldDefinition[],
  validationRules: ValidationRule[]
) {

  const [errors, setErrors] = useState<Record<string, string>>({});
  const [touched, setTouched] = useState<Record<string, boolean>>({});

  const validateField = useCallback((fieldName: string, value: any) => {
    const field = editableFields.find(f => f.field_name === fieldName);
    if (!field) return;

    let error: string | undefined;

    // Required check
    if (field.required && !value) {
      error = `${field.label} is required`;
    }

    // Pattern check
    if (value && field.pattern) {
      const regex = new RegExp(field.pattern);
      if (!regex.test(value)) {
        error = `${field.label} format is invalid`;
      }
    }

    // Min/max check
    if (value !== undefined && field.type === 'number') {
      if (field.min !== undefined && value < field.min) {
        error = `${field.label} must be at least ${field.min}`;
      }
      if (field.max !== undefined && value > field.max) {
        error = `${field.label} must be at most ${field.max}`;
      }
    }

    // Custom validation
    if (!error && field.validate) {
      error = field.validate(value, formValues);
    }

    setErrors(prev => {
      const next = { ...prev };
      if (error) {
        next[fieldName] = error;
      } else {
        delete next[fieldName];
      }
      return next;
    });

    return error;
  }, [editableFields, formValues]);

  const validateAll = useCallback(() => {
    const newErrors: Record<string, string> = {};

    // Field-level validation
    editableFields.forEach(field => {
      const error = validateField(field.field_name, formValues[field.field_name]);
      if (error) {
        newErrors[field.field_name] = error;
      }
    });

    // Cross-field validation
    validationRules.forEach(rule => {
      const error = rule.validate(formValues);
      if (error) {
        // Associate error with first field in rule
        newErrors[rule.fields[0]] = error;
      }
    });

    setErrors(newErrors);

    // Mark all fields as touched
    const allTouched = editableFields.reduce((acc, field) => {
      acc[field.field_name] = true;
      return acc;
    }, {} as Record<string, boolean>);
    setTouched(allTouched);

    return newErrors;
  }, [editableFields, validationRules, formValues, validateField]);

  const markTouched = useCallback((fieldName: string) => {
    setTouched(prev => ({ ...prev, [fieldName]: true }));
  }, []);

  const isValid = Object.keys(errors).length === 0;

  return {
    errors,
    touched,
    isValid,
    validateField,
    validateAll,
    markTouched
  };
}
```

---

### Build Field Overrides

```typescript
// utils/overrides.ts
export function buildFieldOverrides(
  entities: Entity[],
  formValues: Record<string, any>,
  editableFields: FieldDefinition[],
  currentUser: User
): FieldOverride[] {

  const overrides: FieldOverride[] = [];
  const timestamp = new Date().toISOString();
  const reason = formValues._reason || '';

  entities.forEach(entity => {
    editableFields.forEach(field => {
      const newValue = formValues[field.field_name];
      const originalValue = entity[field.field_name];

      // Only create override if value changed
      if (newValue !== undefined && newValue !== originalValue) {
        overrides.push({
          entity_id: entity.entity_id,
          field_name: field.field_name,
          original_value: originalValue,
          override_value: newValue,
          reason,
          timestamp,
          user_id: currentUser.id,
          metadata: {
            confidence_before: entity._confidence?.[field.field_name],
            source: 'manual_correction'
          }
        });
      }
    });
  });

  return overrides;
}
```

---

## Testing Strategy

### Unit Tests

```typescript
// CorrectionDialog.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { CorrectionDialog } from './CorrectionDialog';

describe('CorrectionDialog', () => {

  it('renders with initial values', () => {
    const entity = {
      entity_id: 'test_1',
      type: 'transaction',
      merchant: 'Starbucks',
      amount: 24.50
    };

    render(
      <CorrectionDialog
        mode="single"
        entity={entity}
        entityType="transaction"
        editableFields={[
          { field_name: 'merchant', label: 'Merchant', type: 'text', required: true },
          { field_name: 'amount', label: 'Amount', type: 'currency', required: true }
        ]}
        onSave={jest.fn()}
        onCancel={jest.fn()}
      />
    );

    expect(screen.getByLabelText('Merchant')).toHaveValue('Starbucks');
    expect(screen.getByLabelText('Amount')).toHaveValue('24.50');
  });

  it('validates required fields', async () => {
    const onSave = jest.fn();

    render(
      <CorrectionDialog
        mode="single"
        entity={{ entity_id: 'test_1', type: 'transaction' }}
        entityType="transaction"
        editableFields={[
          { field_name: 'merchant', label: 'Merchant', type: 'text', required: true }
        ]}
        onSave={onSave}
        onCancel={jest.fn()}
      />
    );

    // Try to save without filling required field
    fireEvent.click(screen.getByText('Save'));

    await waitFor(() => {
      expect(screen.getByText('Merchant is required')).toBeInTheDocument();
    });

    expect(onSave).not.toHaveBeenCalled();
  });

  it('calls onSave with correct overrides', async () => {
    const onSave = jest.fn().mockResolvedValue(undefined);

    render(
      <CorrectionDialog
        mode="single"
        entity={{
          entity_id: 'test_1',
          type: 'transaction',
          merchant: 'Starbucks'
        }}
        entityType="transaction"
        editableFields={[
          { field_name: 'merchant', label: 'Merchant', type: 'text', required: true }
        ]}
        onSave={onSave}
        onCancel={jest.fn()}
        requireReason={false}
      />
    );

    // Change merchant
    const merchantInput = screen.getByLabelText('Merchant');
    fireEvent.change(merchantInput, { target: { value: 'Starbucks Reserve' } });

    // Save
    fireEvent.click(screen.getByText('Save'));

    await waitFor(() => {
      expect(onSave).toHaveBeenCalledWith([
        expect.objectContaining({
          entity_id: 'test_1',
          field_name: 'merchant',
          original_value: 'Starbucks',
          override_value: 'Starbucks Reserve'
        })
      ]);
    });
  });

  it('supports keyboard shortcuts', async () => {
    const onSave = jest.fn().mockResolvedValue(undefined);
    const onCancel = jest.fn();

    render(
      <CorrectionDialog
        mode="single"
        entity={{
          entity_id: 'test_1',
          type: 'transaction',
          merchant: 'Starbucks'
        }}
        entityType="transaction"
        editableFields={[
          { field_name: 'merchant', label: 'Merchant', type: 'text', required: true }
        ]}
        onSave={onSave}
        onCancel={onCancel}
        requireReason={false}
      />
    );

    // Cmd+Enter to save
    fireEvent.keyDown(window, { key: 'Enter', metaKey: true });

    await waitFor(() => {
      expect(onSave).toHaveBeenCalled();
    });

    // Escape to cancel
    fireEvent.keyDown(window, { key: 'Escape' });

    expect(onCancel).toHaveBeenCalled();
  });

});
```

---

### Integration Tests

```typescript
// CorrectionDialog.integration.test.tsx
describe('CorrectionDialog Integration', () => {

  it('full correction flow with API', async () => {
    const mockAPI = {
      saveCorrections: jest.fn().mockResolvedValue({ success: true })
    };

    render(
      <CorrectionDialog
        mode="single"
        entity={{
          entity_id: 'txn_123',
          type: 'transaction',
          merchant: 'STARBUCKS 123',
          category: 'Shopping',
          amount: 24.50,
          _confidence: {
            merchant: 0.87,
            category: 0.65
          }
        }}
        entityType="transaction"
        editableFields={[...]}
        onSave={mockAPI.saveCorrections}
        onCancel={jest.fn()}
        showHistory={true}
        showConfidence={true}
        requireReason={true}
      />
    );

    // 1. User sees confidence scores
    expect(screen.getByText('87% confidence')).toBeInTheDocument();
    expect(screen.getByText('65% confidence')).toBeInTheDocument();

    // 2. User clicks "View history" on merchant field
    fireEvent.click(screen.getByText('View history'));

    await waitFor(() => {
      expect(screen.getByText('Merchant Name History')).toBeInTheDocument();
    });

    // 3. User changes category
    const categorySelect = screen.getByLabelText('Category');
    fireEvent.change(categorySelect, { target: { value: 'food_dining' } });

    // 4. User adds reason
    const reasonTextarea = screen.getByLabelText('Reason for Correction');
    fireEvent.change(reasonTextarea, {
      target: { value: 'Receipt confirms coffee purchase' }
    });

    // 5. User saves
    fireEvent.click(screen.getByText('Save'));

    // 6. Verify API called with correct data
    await waitFor(() => {
      expect(mockAPI.saveCorrections).toHaveBeenCalledWith([
        expect.objectContaining({
          entity_id: 'txn_123',
          field_name: 'category',
          original_value: 'Shopping',
          override_value: 'food_dining',
          reason: 'Receipt confirms coffee purchase'
        })
      ]);
    });
  });

});
```

---

## Accessibility

### ARIA Labels

```tsx
<Dialog
  open
  onOpenChange={handleCancel}
  aria-labelledby="correction-dialog-title"
  aria-describedby="correction-dialog-description"
>
  <DialogContent>
    <DialogHeader>
      <DialogTitle id="correction-dialog-title">
        Edit Transaction
      </DialogTitle>
      <p id="correction-dialog-description" className="sr-only">
        Edit transaction fields and provide a reason for corrections
      </p>
    </DialogHeader>
    {/* ... */}
  </DialogContent>
</Dialog>
```

### Keyboard Navigation

- **Tab Order**: Logical flow through fields
- **Focus Management**: Focus first field on mount, focus first error on validation failure
- **Escape to Close**: Standard dialog behavior
- **Enter to Submit**: In single-line inputs (not textareas)

### Screen Reader Announcements

```tsx
// Announce validation errors
<div role="alert" aria-live="assertive">
  {Object.keys(errors).length > 0 && (
    <p>
      {Object.keys(errors).length} validation error(s). Please review the form.
    </p>
  )}
</div>

// Announce save success
{saveSuccess && (
  <div role="status" aria-live="polite">
    Changes saved successfully
  </div>
)}
```

### Focus Trapping

```tsx
import { FocusTrap } from '@radix-ui/react-focus-trap';

<FocusTrap>
  <DialogContent>
    {/* Dialog content */}
  </DialogContent>
</FocusTrap>
```

---

## Performance Considerations

### 1. Virtual Scrolling for Long Field Lists

For entities with 100+ fields:

```tsx
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={editableFields.length}
  itemSize={120}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      <FieldInput field={editableFields[index]} {...props} />
    </div>
  )}
</FixedSizeList>
```

### 2. Debounced Validation

Avoid validating on every keystroke:

```tsx
import { useDebouncedCallback } from 'use-debounce';

const debouncedValidate = useDebouncedCallback(
  (fieldName, value) => validateField(fieldName, value),
  300
);

const handleFieldChange = (fieldName: string, value: any) => {
  updateField(fieldName, value);
  debouncedValidate(fieldName, value);
};
```

### 3. Lazy Load Field History

Only fetch history when user expands it:

```tsx
const { data: history, isLoading } = useQuery(
  ['field-history', entityId, fieldName],
  () => fetchFieldHistory(entityId, fieldName),
  {
    enabled: historyExpanded  // Only fetch when expanded
  }
);
```

### 4. Memoize Expensive Computations

```tsx
const sortedFields = useMemo(
  () => editableFields.sort((a, b) => a.label.localeCompare(b.label)),
  [editableFields]
);

const previewData = useMemo(
  () => generateBulkPreview(entities, formValues),
  [entities, formValues]
);
```

---

## Related Components

### 1. CorrectionsPanel
- **Purpose**: Displays list of all corrections for an entity
- **Relationship**: CorrectionDialog is opened from CorrectionsPanel
- **Shared Data**: Both consume FieldOverride[] from Canonical Ledger

### 2. FieldHistoryViewer
- **Purpose**: Standalone viewer for field audit trail
- **Relationship**: Embedded within CorrectionDialog
- **Shared Data**: Field history from Objective Layer

### 3. ValidationService
- **Purpose**: Domain-specific validation rules
- **Relationship**: CorrectionDialog calls ValidationService for cross-field validation
- **Shared Data**: ValidationRule[]

### 4. ConfidenceIndicator
- **Purpose**: Visual indicator of field confidence score
- **Relationship**: Embedded within FieldInput component
- **Shared Data**: Confidence scores from Objective Layer

### 5. BulkOperationsManager
- **Purpose**: Manages bulk operations (beyond just corrections)
- **Relationship**: Uses CorrectionDialog for bulk edits
- **Shared Data**: Selected entities, operation type

---

## Appendix: Complete Example

### Finance Dashboard with CorrectionDialog

```typescript
// pages/FinanceDashboard.tsx
import React, { useState } from 'react';
import { CorrectionDialog } from '@/components/CorrectionDialog';
import { TransactionTable } from '@/components/TransactionTable';

export function FinanceDashboard() {

  const [transactions, setTransactions] = useState<Transaction[]>([]);
  const [selectedTransaction, setSelectedTransaction] = useState<Transaction | null>(null);
  const [dialogOpen, setDialogOpen] = useState(false);

  const handleEditTransaction = (transaction: Transaction) => {
    setSelectedTransaction(transaction);
    setDialogOpen(true);
  };

  const handleSaveCorrection = async (overrides: FieldOverride[]) => {
    // Save to API
    await api.post('/corrections', { overrides });

    // Update local state
    setTransactions(prev =>
      prev.map(txn =>
        txn.entity_id === selectedTransaction?.entity_id
          ? applyOverrides(txn, overrides)
          : txn
      )
    );

    // Close dialog
    setDialogOpen(false);
    setSelectedTransaction(null);
  };

  return (
    <div>
      <h1>Transactions</h1>

      <TransactionTable
        transactions={transactions}
        onEdit={handleEditTransaction}
      />

      {dialogOpen && selectedTransaction && (
        <CorrectionDialog
          mode="single"
          entity={selectedTransaction}
          entityType="transaction"
          editableFields={[
            {
              field_name: 'merchant',
              label: 'Merchant Name',
              type: 'text',
              required: true
            },
            {
              field_name: 'category',
              label: 'Category',
              type: 'select',
              required: true,
              options: [
                { label: 'Food & Dining', value: 'food_dining' },
                { label: 'Shopping', value: 'shopping' },
                { label: 'Transportation', value: 'transportation' }
              ]
            },
            {
              field_name: 'amount',
              label: 'Amount',
              type: 'currency',
              required: true,
              min: 0
            },
            {
              field_name: 'date',
              label: 'Transaction Date',
              type: 'date',
              required: true
            }
          ]}
          validationRules={[
            {
              id: 'future_date',
              fields: ['date'],
              severity: 'error',
              validate: (values) => {
                if (new Date(values.date) > new Date()) {
                  return 'Date cannot be in the future';
                }
              }
            }
          ]}
          onSave={handleSaveCorrection}
          onCancel={() => {
            setDialogOpen(false);
            setSelectedTransaction(null);
          }}
          showHistory={true}
          showConfidence={true}
          requireReason={true}
        />
      )}
    </div>
  );
}
```

---

## Summary

The **CorrectionDialog** is a comprehensive, production-ready component for managing data corrections across all domains. Key takeaways:

1. **Flexible**: Supports single and bulk edit modes
2. **Validated**: Real-time field and cross-field validation
3. **Auditable**: Integrates with field history and requires correction reasons
4. **Accessible**: Full keyboard navigation and screen reader support
5. **Performant**: Optimized for large field lists and bulk operations
6. **Domain-Agnostic**: Works for finance, healthcare, legal, research, e-commerce, SaaS, and more

**Next Steps**:
- Implement in your domain (see multi-domain examples)
- Customize field types as needed
- Integrate with your Canonical Ledger for persistence
- Add domain-specific validation rules
- Configure audit trail requirements

**File:** `/Users/darwinborges/personal control system /personal-os-v1/docs/primitives/il/CorrectionDialog.md`
**Line Count:** 1,200+
**Status:** Complete ✅
