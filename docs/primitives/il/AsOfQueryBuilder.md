# AsOfQueryBuilder

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
5. [Query Modes](#query-modes)
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

The **AsOfQueryBuilder** is an intuitive interface component that enables users to construct and execute temporal queries against bitemporal data. It answers critical questions like "What did we know on date X?" (transaction time queries), "What was actually true on date Y?" (valid time queries), and "What did we know on date X about what was true on date Y?" (bitemporal queries). This component transforms complex temporal logic into a user-friendly workflow accessible to non-technical users.

### Key Capabilities

- **Three Query Modes**: Transaction time, valid time, and bitemporal queries
- **Intuitive Date Selection**: Visual date pickers with validation and constraints
- **Quick Presets**: One-click access to common time periods (Today, Last Month, Q1 2024, etc.)
- **Query Preview**: Real-time preview of query results before execution
- **Query History**: Track and rerun previous queries
- **Comparison Mode**: Compare data states across two time points
- **Export Queries**: Save queries as shareable links or export results
- **Responsive Design**: Works seamlessly on desktop, tablet, and mobile devices
- **Accessibility**: Full keyboard navigation and screen reader support

### Design Philosophy

The AsOfQueryBuilder embodies the principle that **temporal complexity should not impede accessibility**. Unlike SQL-based temporal queries that require technical expertise, this component:

- **Abstracts complexity**: Hides bitemporal theory behind visual metaphors
- **Validates proactively**: Prevents invalid date combinations before execution
- **Educates users**: Inline help explains transaction vs. valid time
- **Optimizes for common cases**: Quick presets for 80% of use cases
- **Supports power users**: Advanced mode for complex bitemporal queries

This transforms temporal data from a specialized domain into an accessible analytical tool for all users.

---

## Component Interface

### Primary Props

```typescript
interface AsOfQueryBuilderProps {
  // ============================================================
  // Entity Configuration
  // ============================================================

  /**
   * Entity identifier to query
   * @example 'txn_abc123', 'patient_45678'
   */
  entityId: string

  /**
   * Entity type for context-specific UI
   * @example 'transaction', 'patient_record', 'legal_case'
   */
  entityType: string

  /**
   * Current entity snapshot (optional)
   * Used to show "current state" in comparison mode
   */
  currentEntity?: Entity

  // ============================================================
  // Query Configuration
  // ============================================================

  /**
   * Default query mode
   * @default 'transaction'
   */
  defaultQueryMode?: QueryMode

  /**
   * Available query modes
   * @default ['transaction', 'valid', 'bitemporal']
   */
  availableQueryModes?: QueryMode[]

  /**
   * Default date range for initial query
   * If not provided, defaults to "today"
   */
  defaultDateRange?: DateRange | 'today' | 'last_month' | 'last_quarter' | 'last_year'

  /**
   * Earliest date allowed for queries
   * Prevents querying before entity creation
   * @default Entity creation date
   */
  minDate?: Date

  /**
   * Latest date allowed for queries
   * Prevents querying future dates
   * @default Today
   */
  maxDate?: Date

  /**
   * Enable query preview before execution
   * Shows summary of what will be returned
   * @default true
   */
  enablePreview?: boolean

  /**
   * Enable query history tracking
   * @default true
   */
  enableHistory?: boolean

  /**
   * Enable comparison mode (compare two time points)
   * @default false
   */
  enableComparison?: boolean

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when user executes a query
   * @param query - Query configuration
   * @returns Promise<QueryResult> - Query results
   */
  onQuery: (query: AsOfQuery) => Promise<QueryResult>

  /**
   * Called when query configuration changes (before execution)
   * Useful for real-time preview
   * @param query - Current query configuration
   */
  onQueryChange?: (query: AsOfQuery) => void

  /**
   * Called when user exports query results
   * @param format - Export format
   * @param data - Query results to export
   */
  onExport?: (format: ExportFormat, data: QueryResult) => void

  /**
   * Called when query validation fails
   * @param errors - Validation error messages
   */
  onValidationError?: (errors: ValidationError[]) => void

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Show query mode selector
   * @default true
   */
  showQueryModeSelector?: boolean

  /**
   * Show quick preset buttons
   * @default true
   */
  showQuickPresets?: boolean

  /**
   * Custom quick presets
   * Overrides default presets
   */
  customPresets?: QueryPreset[]

  /**
   * Show field selector (query specific fields only)
   * @default false (queries all fields)
   */
  showFieldSelector?: boolean

  /**
   * Available fields for selection
   * Only used if showFieldSelector=true
   */
  availableFields?: FieldDefinition[]

  /**
   * Show inline help/tooltips
   * @default true
   */
  showHelp?: boolean

  /**
   * Component size
   * @default 'md'
   */
  size?: 'sm' | 'md' | 'lg'

  /**
   * Layout mode
   * @default 'vertical' (stacked date pickers)
   */
  layout?: 'vertical' | 'horizontal' | 'compact'

  /**
   * Loading state (external loading indicator)
   * @default false
   */
  isLoading?: boolean

  /**
   * Custom button labels
   */
  labels?: {
    executeQuery?: string
    reset?: string
    preview?: string
    export?: string
  }

  // ============================================================
  // Advanced Options
  // ============================================================

  /**
   * Enable SQL query preview for power users
   * Shows generated SQL for bitemporal query
   * @default false
   */
  showSQLPreview?: boolean

  /**
   * Custom query validator
   * Additional validation beyond built-in rules
   */
  customValidator?: (query: AsOfQuery) => ValidationError[]

  /**
   * Query timeout (milliseconds)
   * @default 30000 (30 seconds)
   */
  queryTimeout?: number

  /**
   * Enable query caching
   * Cache results for identical queries
   * @default true
   */
  enableCaching?: boolean

  /**
   * Cache duration (milliseconds)
   * @default 300000 (5 minutes)
   */
  cacheDuration?: number
}
```

### Supporting Types

```typescript
/**
 * Query mode enumeration
 */
type QueryMode =
  | 'transaction'  // "What did we know on date X?"
  | 'valid'        // "What was true on date X?"
  | 'bitemporal'   // "What did we know on TX about what was true on VX?"

/**
 * As-of query configuration
 */
interface AsOfQuery {
  /**
   * Query mode
   */
  mode: QueryMode

  /**
   * Transaction time point
   * "When was the data recorded?"
   */
  transactionTime?: Date

  /**
   * Valid time point
   * "When was the data true in real world?"
   */
  validTime?: Date

  /**
   * Fields to return (null = all fields)
   */
  fields?: string[]

  /**
   * Include metadata in results
   */
  includeMetadata?: boolean

  /**
   * Include provenance chain
   */
  includeProvenance?: boolean
}

/**
 * Query result
 */
interface QueryResult {
  /**
   * Query that was executed
   */
  query: AsOfQuery

  /**
   * Entity snapshot at specified time point
   */
  entity: Entity

  /**
   * Metadata about the query result
   */
  metadata?: {
    /**
     * Execution time (milliseconds)
     */
    executionTime: number

    /**
     * Number of events considered
     */
    eventsConsidered: number

    /**
     * Timestamp when query was executed
     */
    executedAt: string

    /**
     * Whether result was served from cache
     */
    fromCache: boolean
  }

  /**
   * Provenance chain (if includeProvenance=true)
   */
  provenance?: TimelineEvent[]

  /**
   * Confidence scores for returned values
   */
  confidence?: Record<string, number>
}

/**
 * Quick preset configuration
 */
interface QueryPreset {
  /**
   * Preset identifier
   */
  id: string

  /**
   * Display label
   */
  label: string

  /**
   * Preset description
   */
  description?: string

  /**
   * Query configuration
   */
  query: Partial<AsOfQuery>

  /**
   * Icon for preset button
   */
  icon?: string
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
 * Date range
 */
interface DateRange {
  start: Date
  end: Date
}

/**
 * Field definition
 */
interface FieldDefinition {
  name: string
  label: string
  type: 'text' | 'number' | 'date' | 'boolean' | 'enum' | 'json'
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
    version: number
  }
}

/**
 * Timeline event (from TimelineViewer)
 */
interface TimelineEvent {
  id: string
  eventType: string
  transactionTime: string
  validTime?: string
  actor: {
    id: string
    type: 'user' | 'system'
    name: string
  }
  affectedFields: Array<{
    fieldName: string
    oldValue: any
    newValue: any
  }>
  reason?: string
}

/**
 * Export format
 */
type ExportFormat = 'json' | 'csv' | 'pdf' | 'link'
```

### Default Quick Presets

```typescript
const DEFAULT_QUICK_PRESETS: QueryPreset[] = [
  {
    id: 'today',
    label: 'Today',
    description: 'Current state of the entity',
    query: {
      mode: 'transaction',
      transactionTime: new Date()
    },
    icon: '📅'
  },
  {
    id: 'yesterday',
    label: 'Yesterday',
    description: 'State as of yesterday',
    query: {
      mode: 'transaction',
      transactionTime: subDays(new Date(), 1)
    },
    icon: '📆'
  },
  {
    id: 'last_week',
    label: 'Last Week',
    description: 'State 7 days ago',
    query: {
      mode: 'transaction',
      transactionTime: subDays(new Date(), 7)
    },
    icon: '📊'
  },
  {
    id: 'last_month',
    label: 'Last Month',
    description: 'State 30 days ago',
    query: {
      mode: 'transaction',
      transactionTime: subDays(new Date(), 30)
    },
    icon: '📈'
  },
  {
    id: 'end_of_q1',
    label: 'End of Q1 2024',
    description: 'State on March 31, 2024',
    query: {
      mode: 'transaction',
      transactionTime: new Date('2024-03-31T23:59:59Z')
    },
    icon: '📉'
  },
  {
    id: 'end_of_q2',
    label: 'End of Q2 2024',
    description: 'State on June 30, 2024',
    query: {
      mode: 'transaction',
      transactionTime: new Date('2024-06-30T23:59:59Z')
    },
    icon: '📊'
  },
  {
    id: 'start_of_year',
    label: 'Start of Year',
    description: 'State on January 1, 2024',
    query: {
      mode: 'transaction',
      transactionTime: new Date('2024-01-01T00:00:00Z')
    },
    icon: '🎉'
  },
  {
    id: 'end_of_year',
    label: 'End of Last Year',
    description: 'State on December 31, 2023',
    query: {
      mode: 'transaction',
      transactionTime: new Date('2023-12-31T23:59:59Z')
    },
    icon: '🎊'
  }
]
```

---

## Architecture

### Component Structure

```
AsOfQueryBuilder/
├── AsOfQueryBuilder.tsx            # Main component
├── components/
│   ├── QueryModeSelector.tsx       # Transaction/Valid/Bitemporal selector
│   ├── DatePicker.tsx              # Date selection UI
│   ├── QuickPresets.tsx            # Quick preset buttons
│   ├── FieldSelector.tsx           # Field selection (optional)
│   ├── QueryPreview.tsx            # Preview before execution
│   ├── QueryHistory.tsx            # Previous queries list
│   ├── ComparisonView.tsx          # Compare two time points
│   ├── ValidationMessages.tsx      # Error/warning display
│   └── HelpTooltip.tsx             # Inline help
├── hooks/
│   ├── useAsOfQuery.ts             # Query state management
│   ├── useQueryValidation.ts       # Validation logic
│   ├── useQueryHistory.ts          # Query history tracking
│   └── useQueryCache.ts            # Query result caching
├── utils/
│   ├── queryValidator.ts           # Validation rules
│   ├── dateUtils.ts                # Date manipulation
│   ├── sqlGenerator.ts             # SQL query generation (for preview)
│   └── queryFormatter.ts           # Format queries for display
└── styles/
    └── AsOfQueryBuilder.module.css
```

### State Management

```typescript
interface AsOfQueryBuilderState {
  // Query configuration
  queryMode: QueryMode
  transactionTime: Date | null
  validTime: Date | null
  selectedFields: string[]

  // UI state
  showPreview: boolean
  showHistory: boolean
  showFieldSelector: boolean
  activePreset: string | null

  // Data state
  queryResult: QueryResult | null
  queryHistory: AsOfQuery[]
  validationErrors: ValidationError[]

  // Loading state
  isExecuting: boolean
  isValidating: boolean

  // Comparison mode
  comparisonEnabled: boolean
  comparisonQueryA: AsOfQuery | null
  comparisonQueryB: AsOfQuery | null
}
```

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    AsOfQueryBuilder                          │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  User Interaction                                  │    │
│  │  - Select query mode (Transaction/Valid/Both)      │    │
│  │  - Pick date(s) or click preset                    │    │
│  │  - (Optional) Select specific fields               │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Validation                                        │    │
│  │  - Date range validation                           │    │
│  │  - Transaction time >= Valid time (bitemporal)     │    │
│  │  - Date within entity lifetime                     │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Preview (Optional)                                │    │
│  │  - Show query summary                              │    │
│  │  - Estimated result                                │    │
│  │  - SQL preview (power users)                       │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Execute Query                                     │    │
│  │  - Check cache first                               │    │
│  │  - Call onQuery callback                           │    │
│  │  - Add to query history                            │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Display Result                                    │    │
│  │  - Show entity snapshot                            │    │
│  │  - Highlight changed fields (comparison mode)      │    │
│  │  - Export options                                  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## UI Components

### Transaction Time Query (Simple Mode)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ As-Of Query Builder                                              [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Query Mode:  ● Transaction Time  ○ Valid Time  ○ Bitemporal              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  What did we know on...?                                     [?]    │  │
│  │                                                                      │  │
│  │  Transaction Time: [2024-03-15 ▼] [14:30:00 ▼]                      │  │
│  │                                                                      │  │
│  │  Quick Presets:                                                      │  │
│  │  [Today] [Yesterday] [Last Week] [Last Month]                       │  │
│  │  [End of Q1 2024] [End of Q2 2024] [Start of Year]                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ ℹ Transaction Time Query                                            │  │
│  │                                                                      │  │
│  │ This query returns the entity as it was recorded in the system      │  │
│  │ on the specified date. It answers: "What did we know at that time?" │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Advanced Options: ▼                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ☐ Query specific fields only                                       │  │
│  │  ☑ Include metadata (execution time, confidence, etc.)              │  │
│  │  ☑ Include provenance chain                                         │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Show Preview]          [Reset]                    [Execute Query]        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Valid Time Query

```
┌────────────────────────────────────────────────────────────────────────────┐
│ As-Of Query Builder                                              [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Query Mode:  ○ Transaction Time  ● Valid Time  ○ Bitemporal              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  What was actually true on...?                               [?]    │  │
│  │                                                                      │  │
│  │  Valid Time: [2024-01-15 ▼] [00:00:00 ▼]                            │  │
│  │                                                                      │  │
│  │  Quick Presets:                                                      │  │
│  │  [Today] [Yesterday] [Last Week] [Last Month]                       │  │
│  │  [End of Q1 2024] [End of Q2 2024] [Start of Year]                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ ℹ Valid Time Query                                                  │  │
│  │                                                                      │  │
│  │ This query returns the entity as it was in the real world on the    │  │
│  │ specified date, regardless of when we recorded it. It answers:      │  │
│  │ "What was actually true at that time?"                              │  │
│  │                                                                      │  │
│  │ Note: If corrections were backdated, valid time may differ from     │  │
│  │ transaction time.                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Show Preview]          [Reset]                    [Execute Query]        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Bitemporal Query (Advanced)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ As-Of Query Builder (Bitemporal)                                 [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Query Mode:  ○ Transaction Time  ○ Valid Time  ● Bitemporal              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  What did we know on [Transaction Time]                      [?]    │  │
│  │  about what was true on [Valid Time]?                               │  │
│  │                                                                      │  │
│  │  Transaction Time (when recorded):                                  │  │
│  │  [2024-03-15 ▼] [14:30:00 ▼]                                        │  │
│  │                                                                      │  │
│  │  Valid Time (when true in real world):                              │  │
│  │  [2024-01-15 ▼] [00:00:00 ▼]                                        │  │
│  │                                                                      │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │ ⚠ Date Validation                                             │ │  │
│  │  │                                                                │ │  │
│  │  │ Transaction Time (Mar 15) is after Valid Time (Jan 15).       │ │  │
│  │  │ This is valid for backdated corrections.                      │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ ℹ Bitemporal Query                                                  │  │
│  │                                                                      │  │
│  │ This query returns the entity as it was recorded on the transaction │  │
│  │ time, but considering only events valid up to the valid time. It    │  │
│  │ answers: "What did we know at time X about what was true at time Y?"│  │
│  │                                                                      │  │
│  │ Use case: Auditing corrections. "On March 15, we corrected a        │  │
│  │ transaction date to January 15. What did that change look like?"    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Show Preview]          [Reset]                    [Execute Query]        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Query Preview Panel

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Query Preview                                                         [X]  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Query Summary:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Mode:             Transaction Time Query                           │  │
│  │  Entity:           Transaction #txn_abc123                          │  │
│  │  Transaction Time: March 15, 2024 at 2:30 PM UTC                    │  │
│  │  Fields:           All fields                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Expected Result:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  This query will return the transaction as it existed in the system │  │
│  │  on March 15, 2024 at 2:30 PM UTC.                                  │  │
│  │                                                                      │  │
│  │  Timeline events considered: 5                                      │  │
│  │  Last event before query time: Override on March 15, 2024 at 2:00 PM│  │
│  │                                                                      │  │
│  │  Fields that will be returned:                                      │  │
│  │  • amount: $105.50 (overridden by user_john on Mar 15)              │  │
│  │  • date: 2024-01-15 (extracted on Jan 5)                            │  │
│  │  • merchant: "Acme Corporation" (overridden by user_jane on Jan 5)  │  │
│  │  • category: "Office Supplies" (computed on Jan 5)                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  SQL Query (for reference):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  SELECT *                                                            │  │
│  │  FROM transactions                                                   │  │
│  │  FOR SYSTEM_TIME AS OF TIMESTAMP '2024-03-15 14:30:00 UTC'          │  │
│  │  WHERE id = 'txn_abc123'                                             │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Close Preview]                                   [Execute Query]         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Query History Panel

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Query History                                                         [X]  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Recent Queries:                                                            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Transaction Time: Mar 15, 2024 2:30 PM                     [Rerun]│  │
│  │    Executed: 2 minutes ago                                           │  │
│  │    Result: 4 fields returned, execution time 45ms                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 2. Valid Time: Jan 15, 2024 12:00 AM                          [Rerun]│  │
│  │    Executed: 15 minutes ago                                          │  │
│  │    Result: 4 fields returned, execution time 52ms                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 3. Bitemporal: TX=Mar 15, VX=Jan 15                           [Rerun]│  │
│  │    Executed: 1 hour ago                                              │  │
│  │    Result: 4 fields returned, execution time 78ms                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Clear History]                                                            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Comparison Mode

```
┌────────────────────────────────────────────────────────────────────────────┐
│ As-Of Query Builder - Comparison Mode                           [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Compare two time points side-by-side                                      │
│                                                                             │
│  ┌───────────────────────────────────────┬───────────────────────────────┐│
│  │ Time Point A                          │ Time Point B                  ││
│  ├───────────────────────────────────────┼───────────────────────────────┤│
│  │ Mode: Transaction Time                │ Mode: Transaction Time        ││
│  │ Date: [2024-01-15 ▼] [00:00:00 ▼]    │ Date: [2024-03-15 ▼] [14:30 ▼│││
│  │                                       │                               ││
│  │ Quick Presets:                        │ Quick Presets:                ││
│  │ [Today] [Yesterday] [Last Week]       │ [Today] [Yesterday] [Last Week││
│  │ [Last Month]                          │ [Last Month]                  ││
│  └───────────────────────────────────────┴───────────────────────────────┘│
│                                                                             │
│  [Execute Comparison]                                                       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ Comparison Result                                                    │  │
│  ├───────────────────┬─────────────────────┬─────────────────────────┬─┤  │
│  │ Field             │ Jan 15, 2024        │ Mar 15, 2024            │Δ│  │
│  ├───────────────────┼─────────────────────┼─────────────────────────┼─┤  │
│  │ amount            │ $100.00             │ $105.50                 │✓│  │
│  │ date              │ 2024-01-15          │ 2024-01-15              │ │  │
│  │ merchant          │ "Acme Corporation"  │ "Acme Corporation"      │ │  │
│  │ category          │ "Office Supplies"   │ "Office Supplies"       │ │  │
│  └───────────────────┴─────────────────────┴─────────────────────────┴─┘  │
│                                                                             │
│  Legend: ✓ Changed    (no symbol) Unchanged                                │
│                                                                             │
│  Changes Summary:                                                           │
│  • 1 field changed (amount)                                                │
│  • 3 fields unchanged                                                      │
│                                                                             │
│  [Export Comparison]                                                        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Compact Layout (Mobile)

```
┌──────────────────────────────────────┐
│ As-Of Query                     [☰] │
├──────────────────────────────────────┤
│                                      │
│  Query Mode:                         │
│  ● Transaction  ○ Valid  ○ Both      │
│                                      │
│  What did we know on...?             │
│                                      │
│  Date: [2024-03-15 ▼]                │
│  Time: [14:30:00 ▼]                  │
│                                      │
│  Quick Presets:                      │
│  ┌────────┬────────┬────────┐        │
│  │ Today  │ Yester │ Last   │        │
│  │        │ -day   │ Week   │        │
│  └────────┴────────┴────────┘        │
│  ┌────────┬────────┬────────┐        │
│  │ Last   │ End    │ End    │        │
│  │ Month  │ Q1 '24 │ Q2 '24 │        │
│  └────────┴────────┴────────┘        │
│                                      │
│  [Show Preview]                      │
│  [Execute Query]                     │
│                                      │
└──────────────────────────────────────┘
```

---

## Query Modes

### Transaction Time Queries

**Purpose:** Answer "What did we know on date X?"

**Use Cases:**
- Regulatory compliance: "What did our system report to regulators on Dec 31, 2023?"
- Audit trail: "What information did we have when we made decision Y?"
- Historical reconstruction: "What did the dashboard show on March 1?"

**SQL Equivalent:**
```sql
SELECT *
FROM entities
FOR SYSTEM_TIME AS OF TIMESTAMP '2024-03-15 14:30:00 UTC'
WHERE id = 'entity_123'
```

**Example:**
```typescript
const query: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2024-03-15T14:30:00Z')
}

const result = await onQuery(query)
// Returns entity as it was recorded on March 15, 2024 at 2:30 PM
```

**Validation Rules:**
- Transaction time cannot be in the future
- Transaction time must be after entity creation date
- Transaction time must be a valid date

### Valid Time Queries

**Purpose:** Answer "What was actually true on date X?"

**Use Cases:**
- Backdated corrections: "What was the transaction date after we corrected it?"
- Real-world timeline: "What was the patient's diagnosis on Jan 15, regardless of when it was recorded?"
- Effective date tracking: "What was the insurance policy coverage on the incident date?"

**SQL Equivalent:**
```sql
SELECT *
FROM entities
FOR BUSINESS_TIME AS OF TIMESTAMP '2024-01-15 00:00:00 UTC'
WHERE id = 'entity_123'
```

**Example:**
```typescript
const query: AsOfQuery = {
  mode: 'valid',
  validTime: new Date('2024-01-15T00:00:00Z')
}

const result = await onQuery(query)
// Returns entity as it was in the real world on January 15, 2024
```

**Validation Rules:**
- Valid time can be in the past or future (for planned events)
- Valid time must be a valid date
- If entity has no valid time events, falls back to transaction time

### Bitemporal Queries

**Purpose:** Answer "What did we know on date X about what was true on date Y?"

**Use Cases:**
- Correction auditing: "On March 15, we backdated a transaction to January 15. What changed?"
- Historical analysis: "What did we believe was true on Jan 15, based on our knowledge as of March 15?"
- Compliance verification: "What did we report to regulators about Q1, based on our Q2 data?"

**SQL Equivalent:**
```sql
SELECT *
FROM entities
FOR SYSTEM_TIME AS OF TIMESTAMP '2024-03-15 14:30:00 UTC'
FOR BUSINESS_TIME AS OF TIMESTAMP '2024-01-15 00:00:00 UTC'
WHERE id = 'entity_123'
```

**Example:**
```typescript
const query: AsOfQuery = {
  mode: 'bitemporal',
  transactionTime: new Date('2024-03-15T14:30:00Z'),
  validTime: new Date('2024-01-15T00:00:00Z')
}

const result = await onQuery(query)
// Returns entity as recorded on March 15, considering only events valid as of January 15
```

**Validation Rules:**
- Transaction time cannot be in the future
- Transaction time is typically >= valid time (for backdated corrections)
- Both dates must be valid
- Special case: If transaction time < valid time, warn user (forward-dated entry, unusual but valid)

**Visualization:**
```
Transaction Time Axis (When Recorded) →
    Jan 15          Mar 15          Today
      │               │               │
      │               ●───────────────┼──── Query Point
      │              ╱│               │     (TX = Mar 15, VX = Jan 15)
      │             ╱ │               │
      │            ╱  │               │     "What did we know on Mar 15
      │           ╱   │               │      about what was true on Jan 15?"
      ●──────────────┼───────────────┼──── Valid Time = Jan 15
                      │
```

---

## Interaction Patterns

### Quick Preset Selection

**Scenario:** User clicks "End of Q1 2024" preset

**Flow:**
1. User clicks "End of Q1 2024" button
2. Component updates state:
   - `transactionTime = new Date('2024-03-31T23:59:59Z')`
   - `activePreset = 'end_of_q1'`
3. Preset button highlights (visual feedback)
4. Date pickers update to show March 31, 2024, 11:59:59 PM
5. Validation runs automatically
6. If preview enabled, preview panel updates
7. User can execute query or refine dates

**Keyboard Shortcuts:**
- Number keys 1-8: Select preset 1-8
- Enter: Execute query with current preset

### Date Picker Interaction

**Scenario:** User selects custom date

**Flow:**
1. User clicks on date picker input
2. Calendar popup opens
3. User navigates to desired month/year
4. User clicks date
5. Calendar closes
6. Time picker appears (or updates if already visible)
7. User selects time (hours/minutes/seconds)
8. Validation runs:
   - Check date is within entity lifetime
   - Check date is not in future (for transaction time)
   - Check bitemporal consistency (if applicable)
9. If validation passes:
   - Date picker shows selected date
   - Active preset clears (custom date)
   - Preview updates (if enabled)
10. If validation fails:
    - Error message appears below date picker
    - Date picker shows red border
    - Execute button disables

**Accessibility:**
- Arrow keys: Navigate calendar dates
- Enter: Select highlighted date
- Escape: Close calendar without selecting
- Tab: Move to time picker

### Query Execution

**Scenario:** User executes query

**Flow:**
1. User clicks "Execute Query" button
2. Validation runs one final time
3. If validation fails:
   - Show validation errors
   - Highlight problematic fields
   - Disable execution
4. If validation passes:
   - Button shows loading spinner
   - Disable all form inputs
   - Check query cache:
     - If cache hit: Return cached result immediately
     - If cache miss: Execute query via `onQuery` callback
5. While query executes:
   - Show progress indicator
   - Display estimated time (if available)
   - Allow cancellation (optional)
6. On query success:
   - Display result
   - Add query to history
   - Cache result (if caching enabled)
   - Re-enable form inputs
7. On query failure:
   - Show error message
   - Offer retry option
   - Re-enable form inputs

**Timeout Handling:**
```typescript
async function executeQuery(query: AsOfQuery): Promise<QueryResult> {
  const timeoutMs = queryTimeout || 30000

  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error('Query timeout')), timeoutMs)
  })

  try {
    const result = await Promise.race([
      onQuery(query),
      timeoutPromise
    ])

    return result
  } catch (error) {
    if (error.message === 'Query timeout') {
      throw new Error(`Query exceeded timeout of ${timeoutMs}ms. Try narrowing the date range.`)
    }
    throw error
  }
}
```

### Validation Flow

**Scenario:** User enters invalid date combination

**Flow:**
1. User selects transaction time: March 15, 2024
2. User switches to bitemporal mode
3. User selects valid time: April 20, 2024 (future)
4. Validation error appears:
   - Message: "Valid time (April 20) is after transaction time (March 15). This is unusual - valid time typically precedes transaction time for backdated corrections."
   - Severity: Warning (not blocking)
   - Icon: ⚠️ (yellow warning icon)
5. User can:
   - Proceed anyway (for edge cases like planned future events)
   - Adjust dates to resolve warning
   - Click help tooltip to learn more

**Validation Rules:**
```typescript
function validateAsOfQuery(query: AsOfQuery): ValidationError[] {
  const errors: ValidationError[] = []

  // Rule 1: Transaction time not in future
  if (query.transactionTime && query.transactionTime > new Date()) {
    errors.push({
      code: 'TRANSACTION_TIME_FUTURE',
      message: 'Transaction time cannot be in the future',
      field: 'transactionTime',
      severity: 'error'
    })
  }

  // Rule 2: Transaction time after entity creation
  if (query.transactionTime && minDate && query.transactionTime < minDate) {
    errors.push({
      code: 'TRANSACTION_TIME_BEFORE_CREATION',
      message: `Transaction time cannot be before entity creation (${formatDate(minDate)})`,
      field: 'transactionTime',
      severity: 'error'
    })
  }

  // Rule 3: Valid time after transaction time (unusual, warn)
  if (query.mode === 'bitemporal' &&
      query.validTime &&
      query.transactionTime &&
      query.validTime > query.transactionTime) {
    errors.push({
      code: 'VALID_TIME_AFTER_TRANSACTION_TIME',
      message: 'Valid time is after transaction time. This is unusual but valid for planned future events.',
      field: 'validTime',
      severity: 'warning'
    })
  }

  return errors
}
```

### Comparison Mode Interaction

**Scenario:** User compares two time points

**Flow:**
1. User enables comparison mode (checkbox or toggle)
2. UI expands to show two query builders side-by-side
3. User configures Query A:
   - Mode: Transaction Time
   - Date: January 15, 2024
4. User configures Query B:
   - Mode: Transaction Time
   - Date: March 15, 2024
5. User clicks "Execute Comparison"
6. System executes both queries (can be parallelized)
7. Results displayed in comparison table:
   - Fields shown in rows
   - Time points shown in columns
   - Changed fields highlighted
   - Diff summary shown
8. User can export comparison as CSV or PDF

**Optimization:**
- Execute queries in parallel using `Promise.all()`
- Cache each query result independently
- Reuse cached results if user swaps Query A and B

---

## Multi-Domain Examples

### 1. Financial Services: Regulatory Reporting

**Use Case:** Auditor needs to verify what was reported to regulators on Dec 31, 2023

**Entity Type:** `transaction`

**Query Configuration:**
```typescript
const query: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2023-12-31T23:59:59Z'),
  includeMetadata: true,
  includeProvenance: true
}
```

**Result:**
```json
{
  "query": {
    "mode": "transaction",
    "transactionTime": "2023-12-31T23:59:59Z"
  },
  "entity": {
    "id": "txn_abc123",
    "type": "transaction",
    "fields": {
      "amount": 100.00,
      "date": "2023-12-15",
      "merchant": "ACME CORP",
      "category": "Office Supplies"
    }
  },
  "metadata": {
    "executionTime": 45,
    "eventsConsidered": 3,
    "executedAt": "2024-10-25T10:30:00Z",
    "fromCache": false
  },
  "provenance": [
    {
      "id": "evt_001",
      "eventType": "extraction",
      "transactionTime": "2023-12-16T09:00:00Z",
      "actor": { "id": "system_ocr", "type": "system", "name": "OCR System" },
      "affectedFields": [
        { "fieldName": "amount", "oldValue": null, "newValue": 100.00 }
      ]
    }
  ]
}
```

**Value Proposition:**
- **Audit compliance:** Prove exactly what was reported on year-end
- **Dispute resolution:** Resolve discrepancies with regulators
- **Historical accuracy:** Reconstruct reports from any past date

### 2. Healthcare: Backdated Diagnosis Correction

**Use Case:** Doctor corrected a diagnosis date after finding earlier medical records

**Entity Type:** `patient_record`

**Scenario:**
- On March 10, 2024, doctor records diagnosis of "Hypertension" with diagnosis_date = March 10
- On April 5, 2024, doctor finds earlier records showing diagnosis was actually on January 15, 2024
- Doctor makes retroactive correction, setting diagnosis_date = January 15 (valid time)

**Bitemporal Query:**
```typescript
// Query 1: What did we know on March 10?
const queryMarch: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2024-03-10T23:59:59Z')
}
// Result: diagnosis_date = "2024-03-10"

// Query 2: What was actually true on March 10?
const queryValid: AsOfQuery = {
  mode: 'valid',
  validTime: new Date('2024-03-10T00:00:00Z')
}
// Result: diagnosis_date = "2024-01-15" (corrected value)

// Query 3: What did we know on April 5 about what was true on January 15?
const queryBitemporal: AsOfQuery = {
  mode: 'bitemporal',
  transactionTime: new Date('2024-04-05T23:59:59Z'),
  validTime: new Date('2024-01-15T00:00:00Z')
}
// Result: diagnosis_date = "2024-01-15" (backdated correction visible)
```

**UI Interaction:**
```
User: "I need to see when this diagnosis was actually made, not when we recorded it."

1. User selects "Valid Time" query mode
2. User picks January 15, 2024
3. User executes query
4. Result shows: diagnosis_date = "2024-01-15"
5. User clicks "Show Provenance"
6. Provenance shows: "Corrected on April 5, 2024 by Dr. Jones. Reason: Found earlier records."
```

**Value Proposition:**
- **Clinical accuracy:** Ensure medical timeline reflects true events
- **Legal compliance:** Meet HIPAA audit requirements for corrections
- **Research integrity:** Accurate retrospective studies with corrected dates

### 3. Legal: Statute of Limitations Verification

**Use Case:** Attorney needs to verify filing date for statute of limitations

**Entity Type:** `legal_case_document`

**Scenario:**
- Document filed on January 12, 2024 (actual court stamp)
- System recorded filing on January 15, 2024 (data entry delay)
- On February 1, 2024, paralegal corrected filing_date to January 12 (backdated)
- Statute of limitations: Must be filed by January 15, 2024

**Queries:**
```typescript
// Query 1: What did we know on January 20? (before correction)
const queryBeforeCorrection: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2024-01-20T00:00:00Z')
}
// Result: filing_date = "2024-01-15" (incorrect, but within statute)

// Query 2: What was actually true? (after correction)
const queryActualTruth: AsOfQuery = {
  mode: 'valid',
  validTime: new Date('2024-01-20T00:00:00Z')
}
// Result: filing_date = "2024-01-12" (corrected, well within statute)

// Query 3: Comparison
const comparison = {
  queryA: queryBeforeCorrection,
  queryB: queryActualTruth
}
// Shows: filing_date changed from Jan 15 → Jan 12
```

**Value Proposition:**
- **Legal accuracy:** Prove actual filing date for court proceedings
- **Risk mitigation:** Avoid missed statute of limitations due to data entry errors
- **Audit trail:** Document all corrections with reasons

### 4. Research (RSRCH - Utilitario): Founder Fact Version Control

**Use Case:** RSRCH analyst needs to reproduce fact analysis from investor report date

**Entity Type:** `founder_fact_record`

**Scenario:**
- Investment report generated on June 1, 2024
- Fact corrected on July 15, 2024 (found Sam Altman investment amount updated)
- VC partners request reproducibility on August 1, 2024

**Query:**
```typescript
const queryReportDate: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2024-06-01T23:59:59Z')
}
// Returns founder facts exactly as known on report date
// Ensures analysis can be perfectly reproduced
```

**Workflow:**
1. RSRCH analyst opens AsOfQueryBuilder
2. Selects "Transaction Time" mode
3. Enters June 1, 2024
4. Executes query
5. Exports results as CSV
6. Reruns investment analysis on exported data
7. Confirms results match original report

**Value Proposition:**
- **Reproducibility:** Perfect replication of historical fact analysis
- **Audit trail:** Transparent fact versioning for VC partners
- **Integrity:** Document all fact corrections with timestamps

### 5. E-commerce: Order History Investigation

**Use Case:** Customer service investigates "My order address was wrong!" complaint

**Entity Type:** `order`

**Scenario:**
- Order placed on March 1 with address A
- Customer changed address to B on March 2
- Customer now claims address was always B

**Queries:**
```typescript
// Query 1: What was the address when order was placed?
const queryOrderPlacement: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2024-03-01T14:30:00Z')
}
// Result: shipping_address = "123 Main St" (Address A)

// Query 2: What is the address now?
const queryNow: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date()
}
// Result: shipping_address = "456 Oak Ave" (Address B)

// Query 3: Comparison
const comparison = {
  queryA: queryOrderPlacement,
  queryB: queryNow
}
// Shows: shipping_address changed from "123 Main St" → "456 Oak Ave" on March 2
```

**Customer Service Workflow:**
1. Agent opens order page
2. Clicks "View History"
3. AsOfQueryBuilder opens with order timeline
4. Agent clicks "Order Placement Date" preset
5. Sees original address was A
6. Agent explains to customer: "Your order was placed with address A, then you changed it to B the next day."

**Value Proposition:**
- **Dispute resolution:** Objective timeline of address changes
- **Customer trust:** Transparent history builds confidence
- **Fraud prevention:** Detect suspicious order modifications

### 6. SaaS: Billing Dispute Resolution

**Use Case:** Customer disputes prorated charge for mid-month upgrade

**Entity Type:** `subscription`

**Scenario:**
- Customer upgraded from Starter to Pro on February 15
- Customer now claims they upgraded on February 1 (wants smaller proration)

**Queries:**
```typescript
// Query 1: When did upgrade actually happen?
const queryUpgrade: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2024-02-16T00:00:00Z')
}
// Result: plan changed from "starter" to "professional" on Feb 15 at 4:20 PM

// Query 2: What was plan on February 1?
const queryFeb1: AsOfQuery = {
  mode: 'transaction',
  transactionTime: new Date('2024-02-01T00:00:00Z')
}
// Result: plan = "starter" (no upgrade yet)
```

**Support Workflow:**
1. Support agent opens subscription page
2. Clicks "Query History"
3. AsOfQueryBuilder opens
4. Agent enters February 1 in date picker
5. Executes query: Shows plan was "starter"
6. Agent enters February 16
7. Executes query: Shows plan changed to "professional" on Feb 15
8. Agent shows customer: "Our records show upgrade on Feb 15, here's the timeline"

**Value Proposition:**
- **Billing accuracy:** Prove exact upgrade timestamp
- **Customer trust:** Transparent billing history
- **Reduced churn:** Quick resolution prevents cancellations

### 7. Insurance: Claim Effective Date Verification

**Use Case:** Adjuster verifies coverage on incident date

**Entity Type:** `insurance_policy`

**Scenario:**
- Policy purchased on January 1, 2024
- Incident occurred on March 15, 2024
- Policy was cancelled on April 1, 2024
- Claim filed on April 10, 2024
- Question: Was policy active on incident date?

**Query:**
```typescript
const queryIncidentDate: AsOfQuery = {
  mode: 'valid',
  validTime: new Date('2024-03-15T00:00:00Z')
}
// Result: policy_status = "active", coverage = "comprehensive"
```

**Adjuster Workflow:**
1. Adjuster opens policy page
2. Clicks "Check Coverage As Of Date"
3. AsOfQueryBuilder opens
4. Adjuster selects "Valid Time" mode
5. Enters incident date: March 15, 2024
6. Executes query
7. Result: Policy was active on incident date
8. Adjuster approves claim

**Value Proposition:**
- **Claim accuracy:** Verify coverage on exact incident date
- **Fraud prevention:** Detect backdated policy purchases
- **Compliance:** Meet regulatory requirements for claim verification

---

## Edge Cases & Error Handling

### Query Before Entity Creation

**Scenario:** User queries transaction time before entity was created

**Handling:**
- **Validation:** Block query execution
- **Error message:** "Cannot query before entity creation. Entity was created on {creation_date}."
- **UI:** Date picker disables dates before creation date
- **Auto-correct:** If user clicks preset that's before creation, auto-adjust to creation date

**Implementation:**
```typescript
if (query.transactionTime < entity.metadata.createdAt) {
  return {
    code: 'QUERY_BEFORE_CREATION',
    message: `Cannot query before entity creation (${formatDate(entity.metadata.createdAt)})`,
    field: 'transactionTime',
    severity: 'error'
  }
}
```

### Query in Future

**Scenario:** User attempts to query transaction time in the future

**Handling:**
- **Validation:** Block for transaction time, allow for valid time
- **Error message (transaction):** "Transaction time cannot be in the future. Please select a past date."
- **Warning message (valid):** "Valid time is in the future. This is valid for planned events."

**Implementation:**
```typescript
if (query.mode === 'transaction' && query.transactionTime > new Date()) {
  return {
    code: 'TRANSACTION_TIME_FUTURE',
    message: 'Transaction time cannot be in the future',
    field: 'transactionTime',
    severity: 'error'
  }
}

if (query.mode === 'valid' && query.validTime > new Date()) {
  return {
    code: 'VALID_TIME_FUTURE',
    message: 'Valid time is in the future. This is valid for planned future events.',
    field: 'validTime',
    severity: 'warning'
  }
}
```

### No Events Found

**Scenario:** Query returns no events (entity unchanged during period)

**Handling:**
- **Result:** Return entity with earliest known state
- **Message:** "No changes recorded during this period. Showing earliest known state."
- **UI:** Display result normally with info banner

**Implementation:**
```typescript
async function executeQuery(query: AsOfQuery): Promise<QueryResult> {
  const events = await fetchTimelineEvents(entityId)
  const relevantEvents = events.filter(e =>
    new Date(e.transactionTime) <= query.transactionTime
  )

  if (relevantEvents.length === 0) {
    return {
      query,
      entity: getInitialEntityState(),
      metadata: {
        executionTime: 10,
        eventsConsidered: 0,
        executedAt: new Date().toISOString(),
        fromCache: false
      },
      warning: 'No changes recorded before this date. Showing initial state.'
    }
  }

  // Reconstruct entity from events...
}
```

### Bitemporal Date Reversal

**Scenario:** Valid time is after transaction time (forward-dated entry)

**Example:**
- Transaction time: March 15, 2024
- Valid time: April 20, 2024
- Meaning: "On March 15, we recorded something that will be true on April 20"

**Handling:**
- **Validation:** Allow but warn
- **Warning message:** "Valid time (April 20) is after transaction time (March 15). This represents a forward-dated entry."
- **Use case:** Planned future events (scheduled price changes, future appointments)

**Implementation:**
```typescript
if (query.mode === 'bitemporal' &&
    query.validTime > query.transactionTime) {
  return {
    code: 'FORWARD_DATED_ENTRY',
    message: 'Valid time is after transaction time. This represents a planned future event.',
    field: 'validTime',
    severity: 'warning'
  }
}
```

### Query Timeout

**Scenario:** Query takes longer than timeout limit (30 seconds default)

**Handling:**
- **Timeout detection:** Use Promise.race with timeout promise
- **Error message:** "Query timed out after 30 seconds. Try narrowing the date range or contact support."
- **Recovery:** Offer retry with suggested optimizations
- **Logging:** Log slow queries for performance analysis

**Implementation:**
```typescript
async function executeQueryWithTimeout(
  query: AsOfQuery,
  timeoutMs: number = 30000
): Promise<QueryResult> {
  const controller = new AbortController()

  const timeoutId = setTimeout(() => {
    controller.abort()
  }, timeoutMs)

  try {
    const result = await onQuery(query, { signal: controller.signal })
    clearTimeout(timeoutId)
    return result
  } catch (error) {
    clearTimeout(timeoutId)

    if (error.name === 'AbortError') {
      throw new Error(
        `Query timed out after ${timeoutMs / 1000} seconds. ` +
        `Try narrowing the date range or contact support.`
      )
    }

    throw error
  }
}
```

### Network Failure

**Scenario:** Query execution fails due to network error

**Handling:**
- **Retry logic:** Automatic retry with exponential backoff (3 attempts)
- **Error message:** "Network error. Retrying... (Attempt 2 of 3)"
- **Final failure:** "Failed to execute query after 3 attempts. Please check your connection and try again."
- **Offline detection:** Special message if offline

**Implementation:**
```typescript
async function executeQueryWithRetry(
  query: AsOfQuery,
  maxRetries: number = 3
): Promise<QueryResult> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      setRetryStatus(`Executing query (Attempt ${attempt} of ${maxRetries})...`)
      const result = await onQuery(query)
      return result
    } catch (error) {
      if (attempt === maxRetries) {
        throw new Error(
          `Failed to execute query after ${maxRetries} attempts. ` +
          `Please check your connection and try again.`
        )
      }

      // Exponential backoff: 1s, 2s, 4s
      const delay = 1000 * Math.pow(2, attempt - 1)
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }

  throw new Error('Query failed')
}
```

### Invalid Date Input

**Scenario:** User enters malformed date (e.g., "2024-13-45")

**Handling:**
- **Validation:** Reject invalid dates immediately
- **Error message:** "Invalid date. Please enter a valid date in format YYYY-MM-DD."
- **UI:** Date picker only allows valid dates (calendar prevents invalid selection)
- **Manual input:** If user types manually, validate on blur

**Implementation:**
```typescript
function validateDate(dateString: string): boolean {
  const date = new Date(dateString)
  return date instanceof Date && !isNaN(date.getTime())
}

function handleDateInput(input: string) {
  if (!validateDate(input)) {
    setError({
      field: 'transactionTime',
      message: 'Invalid date. Please enter a valid date in format YYYY-MM-DD.',
      severity: 'error'
    })
    return
  }

  setTransactionTime(new Date(input))
  setError(null)
}
```

### Cache Invalidation

**Scenario:** Entity updated while user has query builder open

**Handling:**
- **Cache TTL:** Expire cached results after 5 minutes
- **Invalidation events:** Listen for entity update events, invalidate cache
- **Stale indicator:** Show "⚠️ Results may be outdated" if cache is stale
- **Refresh button:** Allow user to force refresh

**Implementation:**
```typescript
function useQueryCache(entityId: string) {
  const cache = useRef<Map<string, CachedResult>>(new Map())

  useEffect(() => {
    // Listen for entity updates
    const unsubscribe = subscribeToEntityUpdates(entityId, () => {
      // Invalidate all cached queries for this entity
      cache.current.clear()
      setStaleWarning(true)
    })

    return unsubscribe
  }, [entityId])

  function getCachedResult(query: AsOfQuery): QueryResult | null {
    const cacheKey = JSON.stringify(query)
    const cached = cache.current.get(cacheKey)

    if (!cached) return null

    // Check TTL
    const age = Date.now() - cached.timestamp
    if (age > cacheDuration) {
      cache.current.delete(cacheKey)
      return null
    }

    return cached.result
  }

  function setCachedResult(query: AsOfQuery, result: QueryResult) {
    const cacheKey = JSON.stringify(query)
    cache.current.set(cacheKey, {
      result,
      timestamp: Date.now()
    })
  }

  return { getCachedResult, setCachedResult }
}
```

---

## Implementation Guide

### Installation

```bash
npm install date-fns react-datepicker
npm install --save-dev @types/react-datepicker
```

### Basic Usage

```typescript
import { AsOfQueryBuilder } from '@/components/AsOfQueryBuilder'

function MyComponent() {
  async function handleQuery(query: AsOfQuery): Promise<QueryResult> {
    const response = await fetch(`/api/entities/${entityId}/query`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(query)
    })

    if (!response.ok) {
      throw new Error('Query failed')
    }

    return response.json()
  }

  return (
    <AsOfQueryBuilder
      entityId="txn_abc123"
      entityType="transaction"
      defaultQueryMode="transaction"
      onQuery={handleQuery}
    />
  )
}
```

### Advanced Usage with Custom Presets

```typescript
function MyComponent() {
  const customPresets: QueryPreset[] = [
    {
      id: 'fiscal_year_end',
      label: 'Fiscal Year End',
      description: 'March 31, 2024',
      query: {
        mode: 'transaction',
        transactionTime: new Date('2024-03-31T23:59:59Z')
      },
      icon: '📊'
    },
    {
      id: 'audit_date',
      label: 'Last Audit',
      description: 'June 15, 2024',
      query: {
        mode: 'transaction',
        transactionTime: new Date('2024-06-15T00:00:00Z')
      },
      icon: '🔍'
    }
  ]

  return (
    <AsOfQueryBuilder
      entityId="txn_abc123"
      entityType="transaction"
      customPresets={customPresets}
      enableComparison={true}
      enableHistory={true}
      showSQLPreview={true}
      onQuery={handleQuery}
      onQueryChange={(query) => console.log('Query changed:', query)}
    />
  )
}
```

### Bitemporal Query Example

```typescript
function BitemporalAnalysis() {
  const [queryResult, setQueryResult] = useState<QueryResult | null>(null)

  async function handleBitemporalQuery(query: AsOfQuery): Promise<QueryResult> {
    // Fetch entity state as of transaction time
    const events = await fetchTimelineEvents(entityId)

    // Filter events by transaction time
    const eventsAsOfTX = events.filter(e =>
      new Date(e.transactionTime) <= query.transactionTime!
    )

    // Filter events by valid time
    const eventsAsOfVX = eventsAsOfTX.filter(e => {
      const validTime = e.validTime ? new Date(e.validTime) : new Date(e.transactionTime)
      return validTime <= query.validTime!
    })

    // Reconstruct entity from filtered events
    const entity = reconstructEntity(eventsAsOfVX)

    return {
      query,
      entity,
      metadata: {
        executionTime: 100,
        eventsConsidered: eventsAsOfVX.length,
        executedAt: new Date().toISOString(),
        fromCache: false
      },
      provenance: eventsAsOfVX
    }
  }

  return (
    <div>
      <AsOfQueryBuilder
        entityId="patient_001"
        entityType="patient_record"
        defaultQueryMode="bitemporal"
        onQuery={handleBitemporalQuery}
        showHelp={true}
        showSQLPreview={true}
      />

      {queryResult && (
        <div className="query-result">
          <h3>Query Result</h3>
          <pre>{JSON.stringify(queryResult.entity, null, 2)}</pre>

          <h4>Provenance Chain</h4>
          <ul>
            {queryResult.provenance?.map(event => (
              <li key={event.id}>
                {event.eventType} on {formatDate(event.transactionTime)}
                {event.validTime && ` (valid: ${formatDate(event.validTime)})`}
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  )
}
```

### Custom Validation

```typescript
function MyComponent() {
  function customValidator(query: AsOfQuery): ValidationError[] {
    const errors: ValidationError[] = []

    // Custom rule: Prevent queries more than 5 years in past
    if (query.transactionTime) {
      const fiveYearsAgo = subYears(new Date(), 5)
      if (query.transactionTime < fiveYearsAgo) {
        errors.push({
          code: 'QUERY_TOO_OLD',
          message: 'Queries older than 5 years are not supported. Please contact support for historical data.',
          field: 'transactionTime',
          severity: 'error'
        })
      }
    }

    return errors
  }

  return (
    <AsOfQueryBuilder
      entityId="txn_abc123"
      entityType="transaction"
      customValidator={customValidator}
      onQuery={handleQuery}
    />
  )
}
```

---

## Testing Strategy

### Unit Tests

```typescript
describe('AsOfQueryBuilder', () => {
  it('renders with default transaction mode', () => {
    const { getByRole } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={jest.fn()}
      />
    )

    expect(getByRole('radio', { name: /transaction time/i })).toBeChecked()
  })

  it('validates transaction time not in future', async () => {
    const onQuery = jest.fn()
    const { getByLabelText, getByRole } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={onQuery}
      />
    )

    // Enter future date
    const datePicker = getByLabelText(/transaction time/i)
    await userEvent.type(datePicker, '2099-12-31')

    // Try to execute
    const executeButton = getByRole('button', { name: /execute query/i })
    await userEvent.click(executeButton)

    // Verify error shown and query not executed
    expect(screen.getByText(/cannot be in the future/i)).toBeInTheDocument()
    expect(onQuery).not.toHaveBeenCalled()
  })

  it('applies quick preset correctly', async () => {
    const onQuery = jest.fn().mockResolvedValue({ entity: {} })
    const { getByRole } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={onQuery}
      />
    )

    // Click "Yesterday" preset
    const yesterdayButton = getByRole('button', { name: /yesterday/i })
    await userEvent.click(yesterdayButton)

    // Execute query
    const executeButton = getByRole('button', { name: /execute query/i })
    await userEvent.click(executeButton)

    // Verify query executed with yesterday's date
    expect(onQuery).toHaveBeenCalledWith(
      expect.objectContaining({
        mode: 'transaction',
        transactionTime: expect.any(Date)
      })
    )

    const calledDate = onQuery.mock.calls[0][0].transactionTime
    expect(isSameDay(calledDate, subDays(new Date(), 1))).toBe(true)
  })

  it('shows bitemporal date pickers when mode is bitemporal', async () => {
    const { getByRole, getByLabelText } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={jest.fn()}
      />
    )

    // Switch to bitemporal mode
    const bitemporalRadio = getByRole('radio', { name: /bitemporal/i })
    await userEvent.click(bitemporalRadio)

    // Verify both date pickers visible
    expect(getByLabelText(/transaction time/i)).toBeInTheDocument()
    expect(getByLabelText(/valid time/i)).toBeInTheDocument()
  })

  it('caches query results', async () => {
    const onQuery = jest.fn().mockResolvedValue({
      entity: { id: 'txn_001', fields: {} },
      metadata: { fromCache: false }
    })

    const { getByRole } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={onQuery}
        enableCaching={true}
      />
    )

    // Execute query first time
    const executeButton = getByRole('button', { name: /execute query/i })
    await userEvent.click(executeButton)

    expect(onQuery).toHaveBeenCalledTimes(1)

    // Execute same query again
    await userEvent.click(executeButton)

    // Verify query not executed again (served from cache)
    expect(onQuery).toHaveBeenCalledTimes(1)
  })
})
```

### Integration Tests

```typescript
describe('AsOfQueryBuilder Integration', () => {
  it('executes query and displays result', async () => {
    const mockResult: QueryResult = {
      query: {
        mode: 'transaction',
        transactionTime: new Date('2024-03-15')
      },
      entity: {
        id: 'txn_001',
        type: 'transaction',
        fields: {
          amount: 105.50,
          date: '2024-01-15',
          merchant: 'Acme Corp'
        }
      },
      metadata: {
        executionTime: 45,
        eventsConsidered: 5,
        executedAt: new Date().toISOString(),
        fromCache: false
      }
    }

    const onQuery = jest.fn().mockResolvedValue(mockResult)

    const { getByRole, getByText } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={onQuery}
      />
    )

    // Execute query
    const executeButton = getByRole('button', { name: /execute query/i })
    await userEvent.click(executeButton)

    // Wait for result
    await waitFor(() => {
      expect(getByText(/amount.*105\.50/i)).toBeInTheDocument()
    })
  })

  it('handles query timeout gracefully', async () => {
    const onQuery = jest.fn(() => new Promise(() => {})) // Never resolves

    const { getByRole, getByText } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={onQuery}
        queryTimeout={1000} // 1 second timeout
      />
    )

    // Execute query
    const executeButton = getByRole('button', { name: /execute query/i })
    await userEvent.click(executeButton)

    // Wait for timeout
    await waitFor(() => {
      expect(getByText(/query timed out/i)).toBeInTheDocument()
    }, { timeout: 2000 })
  })
})
```

### Accessibility Tests

```typescript
describe('AsOfQueryBuilder Accessibility', () => {
  it('passes axe accessibility tests', async () => {
    const { container } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={jest.fn()}
      />
    )

    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })

  it('supports keyboard navigation', async () => {
    const onQuery = jest.fn().mockResolvedValue({ entity: {} })
    const { getByRole } = render(
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={onQuery}
      />
    )

    // Tab to execute button
    const executeButton = getByRole('button', { name: /execute query/i })
    executeButton.focus()

    // Press Enter to execute
    await userEvent.keyboard('{Enter}')

    expect(onQuery).toHaveBeenCalled()
  })
})
```

---

## Accessibility

### WCAG 2.1 AA Compliance

#### Keyboard Navigation

**Requirements:**
- All controls keyboard accessible (Tab, Shift+Tab)
- Date pickers navigable with arrow keys
- Presets selectable with number keys (1-8)
- Execute with Enter, Reset with Escape

**Implementation:**
```typescript
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    // Number keys (1-8) select presets
    const presetIndex = parseInt(e.key) - 1
    if (presetIndex >= 0 && presetIndex < quickPresets.length) {
      applyPreset(quickPresets[presetIndex])
      e.preventDefault()
    }

    // Enter: Execute query
    if (e.key === 'Enter' && !isExecuting && isValid) {
      executeQuery()
      e.preventDefault()
    }

    // Escape: Reset form
    if (e.key === 'Escape') {
      resetForm()
      e.preventDefault()
    }
  }

  document.addEventListener('keydown', handleKeyDown)
  return () => document.removeEventListener('keydown', handleKeyDown)
}, [isExecuting, isValid])
```

#### Screen Reader Support

**Requirements:**
- All form controls have labels
- Error messages announced
- Query results announced
- Loading states announced

**Implementation:**
```typescript
<div role="form" aria-label="As-Of Query Builder">
  {/* Query mode selector */}
  <fieldset>
    <legend>Query Mode</legend>
    <input
      type="radio"
      id="mode-transaction"
      name="queryMode"
      value="transaction"
      aria-describedby="mode-transaction-help"
    />
    <label htmlFor="mode-transaction">Transaction Time</label>
    <p id="mode-transaction-help" className="sr-only">
      Query data as it was recorded in the system on a specific date
    </p>
  </fieldset>

  {/* Date picker */}
  <label htmlFor="transaction-time">
    Transaction Time
    <HelpTooltip id="transaction-time-help">
      Select when the data was recorded in the system
    </HelpTooltip>
  </label>
  <input
    type="datetime-local"
    id="transaction-time"
    aria-describedby="transaction-time-help"
    aria-invalid={hasError('transactionTime')}
    aria-errormessage={hasError('transactionTime') ? 'transaction-time-error' : undefined}
  />

  {/* Error message */}
  {hasError('transactionTime') && (
    <div
      id="transaction-time-error"
      role="alert"
      aria-live="polite"
    >
      {getError('transactionTime')}
    </div>
  )}

  {/* Loading state */}
  {isExecuting && (
    <div role="status" aria-live="polite" className="sr-only">
      Executing query, please wait...
    </div>
  )}

  {/* Result announcement */}
  {queryResult && (
    <div role="status" aria-live="polite" className="sr-only">
      Query complete. {Object.keys(queryResult.entity.fields).length} fields returned.
    </div>
  )}
</div>
```

#### Color Contrast

**Requirements:**
- 4.5:1 minimum for normal text
- 3:1 minimum for large text and UI components
- Don't rely on color alone

**Implementation:**
```css
/* Error state */
.date-picker--error {
  border: 2px solid #DC2626; /* Red, 4.5:1 contrast */
  background-color: #FEF2F2; /* Light red background */
}

.error-message {
  color: #991B1B; /* Dark red, 7:1 contrast on white */
  font-weight: 500; /* Bold for additional distinction */
}

/* Success state */
.date-picker--valid {
  border: 2px solid #059669; /* Green, 4.5:1 contrast */
}

/* Warning state */
.warning-message {
  color: #92400E; /* Dark orange, 7:1 contrast */
  background-color: #FEF3C7; /* Light yellow background */
  border-left: 4px solid #F59E0B; /* Orange border for non-color distinction */
}

/* High contrast mode */
@media (prefers-contrast: high) {
  .date-picker {
    border-width: 3px;
  }

  .error-message, .warning-message {
    font-weight: 700;
  }
}
```

#### Focus Management

```typescript
// Focus trap for preview panel
useFocusTrap(previewPanelRef, showPreview)

// Restore focus when closing preview
function closePreview() {
  setShowPreview(false)

  // Return focus to "Show Preview" button
  if (previewButtonRef.current) {
    previewButtonRef.current.focus()
  }
}

// CSS for focus indicators
```

```css
.date-picker:focus,
.preset-button:focus,
.execute-button:focus {
  outline: 3px solid #2563EB;
  outline-offset: 2px;
}

.preset-button:focus-visible {
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.3);
}
```

---

## Performance Considerations

### Query Caching

```typescript
import { useQuery, QueryClient } from '@tanstack/react-query'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 30 * 60 * 1000 // 30 minutes
    }
  }
})

function useAsOfQuery(entityId: string, query: AsOfQuery) {
  return useQuery({
    queryKey: ['asof', entityId, query],
    queryFn: () => executeAsOfQuery(entityId, query),
    enabled: !!query.transactionTime, // Only run if date selected
    staleTime: 5 * 60 * 1000,
    retry: 3,
    retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 30000)
  })
}
```

### Debouncing Date Input

```typescript
import { useDebouncedCallback } from 'use-debounce'

function AsOfQueryBuilder({ onQueryChange }: Props) {
  const [transactionTime, setTransactionTime] = useState<Date | null>(null)

  const debouncedQueryChange = useDebouncedCallback(
    (query: AsOfQuery) => {
      onQueryChange?.(query)
    },
    500 // 500ms delay
  )

  useEffect(() => {
    if (transactionTime) {
      debouncedQueryChange({
        mode: 'transaction',
        transactionTime
      })
    }
  }, [transactionTime])

  return (
    <DatePicker
      selected={transactionTime}
      onChange={setTransactionTime}
    />
  )
}
```

### Lazy Loading Preview

```typescript
function AsOfQueryBuilder({ onQuery, enablePreview }: Props) {
  const [showPreview, setShowPreview] = useState(false)
  const [previewResult, setPreviewResult] = useState<QueryResult | null>(null)

  // Only fetch preview when user opens panel
  useEffect(() => {
    if (showPreview && !previewResult) {
      fetchPreview()
    }
  }, [showPreview])

  async function fetchPreview() {
    setIsLoadingPreview(true)
    try {
      // Execute query with limit for preview
      const result = await onQuery({
        ...currentQuery,
        limit: 1 // Only need one result for preview
      })
      setPreviewResult(result)
    } finally {
      setIsLoadingPreview(false)
    }
  }

  return (
    <>
      <button onClick={() => setShowPreview(true)}>
        Show Preview
      </button>

      {showPreview && (
        <PreviewPanel result={previewResult} isLoading={isLoadingPreview} />
      )}
    </>
  )
}
```

### Memoization

```typescript
// Memoize quick presets (only recompute if custom presets change)
const allPresets = useMemo(() => {
  return customPresets || DEFAULT_QUICK_PRESETS
}, [customPresets])

// Memoize validation errors (only recompute when query changes)
const validationErrors = useMemo(() => {
  return validateAsOfQuery(currentQuery, { minDate, maxDate })
}, [currentQuery, minDate, maxDate])

// Memoize SQL preview (expensive string formatting)
const sqlPreview = useMemo(() => {
  if (!showSQLPreview) return null
  return generateSQLQuery(currentQuery)
}, [currentQuery, showSQLPreview])
```

---

## Related Components

### TimelineViewer

**Relationship:** AsOfQueryBuilder generates queries that filter TimelineViewer

**Integration:**
```typescript
function EntityHistory() {
  const [asOfDate, setAsOfDate] = useState<Date | null>(null)
  const [timelineEvents, setTimelineEvents] = useState<TimelineEvent[]>([])

  async function handleQuery(query: AsOfQuery): Promise<QueryResult> {
    const result = await executeQuery(query)
    setAsOfDate(query.transactionTime)
    return result
  }

  return (
    <>
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={handleQuery}
      />

      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={timelineEvents}
        highlightedEvents={
          asOfDate ? [{
            eventId: findEventAtDate(timelineEvents, asOfDate),
            color: '#EF4444',
            label: 'As-of query point'
          }] : []
        }
      />
    </>
  )
}
```

### RetroactiveCorrectionDialog

**Relationship:** Corrections create timeline events that affect query results

**Integration:**
```typescript
function EntityWorkflow() {
  const [showCorrectionDialog, setShowCorrectionDialog] = useState(false)
  const [queryResult, setQueryResult] = useState<QueryResult | null>(null)

  async function handleCorrection(correction: Correction) {
    await saveCorrection(correction)

    // Re-run query to see effect of correction
    if (currentQuery) {
      const updatedResult = await executeQuery(currentQuery)
      setQueryResult(updatedResult)
    }

    setShowCorrectionDialog(false)
  }

  return (
    <>
      <AsOfQueryBuilder
        entityId="txn_001"
        entityType="transaction"
        onQuery={async (query) => {
          const result = await executeQuery(query)
          setQueryResult(result)
          return result
        }}
      />

      {queryResult && (
        <div>
          <h3>Query Result</h3>
          <p>Amount: {queryResult.entity.fields.amount}</p>

          <button onClick={() => setShowCorrectionDialog(true)}>
            Make Correction
          </button>
        </div>
      )}

      {showCorrectionDialog && (
        <RetroactiveCorrectionDialog
          entity={queryResult.entity}
          editableFields={fields}
          onSave={handleCorrection}
          onCancel={() => setShowCorrectionDialog(false)}
        />
      )}
    </>
  )
}
```

---

## References

### Technical Documentation

- **Date-fns Documentation**: https://date-fns.org/
  - Date manipulation: https://date-fns.org/docs/Getting-Started
  - Formatting: https://date-fns.org/docs/format

- **React DatePicker**: https://reactdatepicker.com/
  - Configuration: https://reactdatepicker.com/#example-custom-date-format

- **TanStack Query**: https://tanstack.com/query/latest
  - Caching: https://tanstack.com/query/latest/docs/react/guides/caching

### Bitemporal Theory

- **Snodgrass, Richard T.** (1999). *Developing Time-Oriented Database Applications in SQL*. Morgan Kaufmann.
  - Chapter 3: Temporal Queries

- **Johnston, Tom** (2014). *Bitemporal Data: Theory and Practice*. Morgan Kaufmann.
  - Part II: Bitemporal Queries
  - Chapter 9: As-Of Queries

### SQL Standards

- **SQL:2011 Temporal Features**: https://standards.iso.org/ittf/PubliclyAvailableStandards/
  - FOR SYSTEM_TIME AS OF
  - FOR BUSINESS_TIME AS OF

### Accessibility Standards

- **WCAG 2.1 AA**: https://www.w3.org/WAI/WCAG21/quickref/
  - 1.3.1 Info and Relationships
  - 2.1.1 Keyboard
  - 3.3.2 Labels or Instructions

- **ARIA Authoring Practices**: https://www.w3.org/WAI/ARIA/apg/
  - Date Picker Pattern: https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/examples/datepicker-dialog/

---

**Component Version:** 1.0.0
**Last Updated:** 2025-10-25
**Author:** Darwin Borges
**Review Status:** Draft - Pending Technical Review
