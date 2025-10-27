# CompatibilityViewer (IL Component)

**Layer:** Interaction Layer (IL)
**Domain:** Schema Registry (Vertical 5.2)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Architecture](#architecture)
4. [UI Components](#ui-components)
5. [Diff Visualization](#diff-visualization)
6. [Visual Wireframes](#visual-wireframes)
7. [States & Lifecycle](#states--lifecycle)
8. [Multi-Domain Examples](#multi-domain-examples)
9. [User Interactions](#user-interactions)
10. [Accessibility](#accessibility)
11. [Performance Optimizations](#performance-optimizations)
12. [Integration Examples](#integration-examples)
13. [Related Components](#related-components)
14. [References](#references)

---

## Overview

### Purpose

The **CompatibilityViewer** is a visual comparison tool that analyzes and displays the differences between two schema versions, highlighting breaking changes, additive changes, and their impact on existing data. It transforms complex schema evolution analysis into an intuitive, actionable visualization that helps developers, architects, and data teams make informed decisions about schema updates.

### Key Capabilities

- **Side-by-Side Diff View**: Visual comparison of old and new schemas with highlighted changes
- **Breaking Change Detection**: Automatic identification of backward-incompatible changes
- **Impact Analysis**: Shows affected record counts and migration complexity
- **Version Bump Recommendation**: Suggests MAJOR/MINOR/PATCH based on SemVer rules
- **Color-Coded Changes**: Red (breaking), green (additive), yellow (warning)
- **JSONPath Navigation**: Click on changes to jump to specific schema locations
- **Export Functionality**: Generate compatibility reports in PDF/JSON/Markdown
- **Multiple View Modes**: Unified diff, split diff, or change summary
- **Confidence Scoring**: Indicates analysis reliability (0-100%)
- **Migration Time Estimation**: Estimates time required for data migration

### Problem Statement

Organizations struggle with schema evolution because:

1. **Hidden Breaking Changes**: Developers don't realize a change breaks consumers
2. **No Impact Visibility**: Unknown how many records are affected by changes
3. **Version Confusion**: Unclear whether to bump MAJOR, MINOR, or PATCH
4. **Manual Analysis**: Time-consuming to compare large schemas by hand
5. **Poor Communication**: Hard to explain schema changes to stakeholders
6. **Risk Assessment**: No way to quantify migration risk

### Solution

The CompatibilityViewer provides:

- **Automated Analysis**: Instant detection of all changes between schemas
- **Visual Diff**: Clear, color-coded side-by-side comparison
- **Impact Quantification**: Exact record counts for affected data
- **Semantic Understanding**: Knows that removing a required field is breaking, adding an optional field is not
- **Version Guidance**: Recommends appropriate version bump based on changes
- **Risk Scoring**: Calculates confidence and risk metrics
- **Shareable Reports**: Export analysis for documentation or approvals

This transforms schema comparison from a manual, error-prone task to an automated, reliable process.

---

## Component Interface

### Primary Props

```typescript
interface CompatibilityViewerProps {
  // ============================================================
  // Schema Configuration
  // ============================================================

  /**
   * Original schema (before changes)
   * Can be string (JSON) or parsed object
   */
  oldSchema: string | object

  /**
   * New schema (after changes)
   * Can be string (JSON) or parsed object
   */
  newSchema: string | object

  /**
   * Old schema version number (for display)
   * @example 'v2.1.0'
   */
  oldVersion?: string

  /**
   * New schema version number (for display)
   * @example 'v3.0.0'
   */
  newVersion?: string

  /**
   * Schema identifier (for context)
   * @example 'payment_event_schema'
   */
  schemaId?: string

  /**
   * Compatibility report (pre-computed)
   * If not provided, will be computed from oldSchema and newSchema
   */
  compatibilityReport?: CompatibilityReport

  // ============================================================
  // Display Options
  // ============================================================

  /**
   * View mode for diff display
   * @default 'split'
   */
  viewMode?: 'split' | 'unified' | 'summary'

  /**
   * Show side-by-side diff
   * @default true
   */
  showDiff?: boolean

  /**
   * Show breaking changes section
   * @default true
   */
  showBreakingChanges?: boolean

  /**
   * Show additive changes section
   * @default true
   */
  showAdditiveChanges?: boolean

  /**
   * Show impact analysis
   * @default true
   */
  showImpactAnalysis?: boolean

  /**
   * Show version bump recommendation
   * @default true
   */
  showVersionRecommendation?: boolean

  /**
   * Show confidence score
   * @default true
   */
  showConfidenceScore?: boolean

  /**
   * Show migration time estimate
   * @default true
   */
  showMigrationEstimate?: boolean

  /**
   * Highlight syntax in diff view
   * @default true
   */
  enableSyntaxHighlighting?: boolean

  /**
   * Allow collapsing unchanged sections
   * @default true
   */
  collapseUnchanged?: boolean

  /**
   * Number of context lines around changes (for unified diff)
   * @default 3
   */
  contextLines?: number

  // ============================================================
  // Filtering Options
  // ============================================================

  /**
   * Filter changes by type
   * If provided, only show specified change types
   */
  filterByType?: ChangeType[]

  /**
   * Filter changes by severity
   * If provided, only show specified severities
   */
  filterBySeverity?: ('high' | 'medium' | 'low')[]

  /**
   * Search query to filter changes
   * Searches field names, descriptions, and change messages
   */
  searchQuery?: string

  /**
   * Show only breaking changes
   * @default false
   */
  showOnlyBreaking?: boolean

  // ============================================================
  // Customization Options
  // ============================================================

  /**
   * Custom color scheme for changes
   */
  colorScheme?: {
    breaking?: string       // Default: #DC2626 (red)
    additive?: string       // Default: #059669 (green)
    warning?: string        // Default: #D97706 (orange)
    info?: string           // Default: #3B82F6 (blue)
    unchanged?: string      // Default: #6B7280 (gray)
  }

  /**
   * Custom labels for change types
   */
  labels?: {
    fieldRemoved?: string
    fieldAdded?: string
    typeChanged?: string
    constraintAdded?: string
    constraintRemoved?: string
    enumReduced?: string
    enumExpanded?: string
    descriptionChanged?: string
  }

  /**
   * Theme
   * @default 'light'
   */
  theme?: 'light' | 'dark'

  /**
   * Height of component (CSS value)
   * @default '600px'
   */
  height?: string

  /**
   * Width of component (CSS value)
   * @default '100%'
   */
  width?: string

  // ============================================================
  // Data Enhancement
  // ============================================================

  /**
   * Function to fetch affected record counts
   * If provided, will show accurate impact numbers
   * @param changeType - Type of change
   * @param path - JSONPath to affected field
   * @returns Promise<number> - Number of affected records
   */
  fetchAffectedRecords?: (changeType: ChangeType, path: string) => Promise<number>

  /**
   * Function to fetch migration time estimate
   * If provided, will show accurate time estimates
   * @param changeType - Type of change
   * @param affectedRecords - Number of records to migrate
   * @returns Promise<number> - Estimated time in milliseconds
   */
  estimateMigrationTime?: (changeType: ChangeType, affectedRecords: number) => Promise<number>

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when user clicks on a change
   * @param change - The clicked change
   */
  onChangeClick?: (change: SchemaChange) => void

  /**
   * Called when user hovers over a change
   * @param change - The hovered change (null when hover ends)
   */
  onChangeHover?: (change: SchemaChange | null) => void

  /**
   * Called when user exports report
   * @param format - Export format
   * @param data - Report data
   */
  onExport?: (format: ExportFormat, data: any) => void

  /**
   * Called when view mode changes
   * @param mode - New view mode
   */
  onViewModeChange?: (mode: 'split' | 'unified' | 'summary') => void

  /**
   * Called when analysis completes
   * @param report - Compatibility report
   */
  onAnalysisComplete?: (report: CompatibilityReport) => void

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Show export button
   * @default true
   */
  showExportButton?: boolean

  /**
   * Show view mode toggle
   * @default true
   */
  showViewModeToggle?: boolean

  /**
   * Show search input
   * @default true
   */
  showSearch?: boolean

  /**
   * Show filter controls
   * @default true
   */
  showFilters?: boolean

  /**
   * Show legend (explains color coding)
   * @default true
   */
  showLegend?: boolean

  /**
   * Empty state message when schemas are identical
   */
  emptyStateMessage?: string

  /**
   * Loading state
   * @default false
   */
  isLoading?: boolean

  /**
   * Error message
   */
  errorMessage?: string

  // ============================================================
  // Advanced Options
  // ============================================================

  /**
   * Custom diff algorithm
   * @default 'myers'
   */
  diffAlgorithm?: 'myers' | 'patience' | 'histogram'

  /**
   * Ignore whitespace changes
   * @default true
   */
  ignoreWhitespace?: boolean

  /**
   * Ignore comment changes
   * @default true
   */
  ignoreComments?: boolean

  /**
   * Ignore order of properties
   * @default true
   */
  ignorePropertyOrder?: boolean

  /**
   * Custom change detection rules
   * Allows domain-specific change classification
   */
  customRules?: CompatibilityRule[]

  /**
   * Enable keyboard shortcuts
   * @default true
   */
  enableKeyboardShortcuts?: boolean

  /**
   * Debug mode (show internal state)
   * @default false
   */
  debugMode?: boolean
}

// ============================================================
// Supporting Types
// ============================================================

interface CompatibilityReport {
  /**
   * Whether schemas are compatible (no breaking changes)
   */
  compatible: boolean

  /**
   * List of breaking changes detected
   */
  breakingChanges: BreakingChange[]

  /**
   * List of additive (non-breaking) changes
   */
  additiveChanges: AdditiveChange[]

  /**
   * Suggested version bump (MAJOR/MINOR/PATCH)
   */
  suggestedVersionBump: VersionBump

  /**
   * Confidence score (0-1)
   * How confident the analysis is
   */
  confidence: number

  /**
   * Whether migration is required
   */
  migrationRequired: boolean

  /**
   * Estimated migration time (human-readable)
   * @example "~5 minutes"
   */
  estimatedMigrationTime?: string

  /**
   * Total number of changes
   */
  totalChanges: number

  /**
   * Summary of analysis
   */
  summary: string

  /**
   * Timestamp of analysis
   */
  analyzedAt: Date
}

interface BreakingChange {
  /**
   * Type of breaking change
   */
  type: ChangeType

  /**
   * JSONPath to affected field
   * @example '$.properties.paymentMethod.enum'
   */
  path: string

  /**
   * Human-readable description
   * @example "Removed 'paypal' from payment methods enum"
   */
  description: string

  /**
   * Severity of impact
   */
  impact: 'high' | 'medium' | 'low'

  /**
   * Old value (before change)
   */
  oldValue?: any

  /**
   * New value (after change)
   */
  newValue?: any

  /**
   * Number of records affected
   */
  affectedRecords?: number

  /**
   * Whether transformation is required
   */
  transformationRequired: boolean

  /**
   * Suggested fix or migration strategy
   */
  suggestion?: string
}

interface AdditiveChange {
  /**
   * Type of additive change
   */
  type: ChangeType

  /**
   * JSONPath to affected field
   */
  path: string

  /**
   * Human-readable description
   */
  description: string

  /**
   * Old value (before change)
   */
  oldValue?: any

  /**
   * New value (after change)
   */
  newValue?: any
}

interface SchemaChange extends BreakingChange {
  /**
   * Unique identifier for this change
   */
  id: string

  /**
   * Whether change is breaking or additive
   */
  isBreaking: boolean

  /**
   * Line number in old schema
   */
  oldLine?: number

  /**
   * Line number in new schema
   */
  newLine?: number
}

type ChangeType =
  | 'field_removed'
  | 'field_added'
  | 'type_changed'
  | 'constraint_added'
  | 'constraint_removed'
  | 'enum_reduced'
  | 'enum_expanded'
  | 'required_added'
  | 'required_removed'
  | 'description_changed'
  | 'format_added'
  | 'format_changed'
  | 'default_changed'
  | 'example_changed'

type VersionBump = 'MAJOR' | 'MINOR' | 'PATCH' | 'NONE'

type ExportFormat = 'json' | 'markdown' | 'pdf' | 'csv'

interface CompatibilityRule {
  id: string
  name: string
  /**
   * Function to detect if this rule applies
   * @param oldSchema - Old schema
   * @param newSchema - New schema
   * @returns Array of detected changes
   */
  detect: (oldSchema: object, newSchema: object) => SchemaChange[]
}

// ============================================================
// Component State
// ============================================================

interface CompatibilityViewerState {
  // Analysis state
  report: CompatibilityReport | null
  analyzing: boolean
  error: string | null

  // UI state
  currentViewMode: 'split' | 'unified' | 'summary'
  expandedSections: Set<string>
  selectedChange: SchemaChange | null
  hoveredChange: SchemaChange | null

  // Filter state
  activeFilters: {
    types: ChangeType[]
    severities: ('high' | 'medium' | 'low')[]
    searchQuery: string
    showOnlyBreaking: boolean
  }

  // Display state
  showUnchanged: boolean
  syntaxHighlightingEnabled: boolean
}
```

---

## Architecture

### Component Hierarchy

```
CompatibilityViewer
├── ViewerHeader
│   ├── SchemaVersionBadge (oldVersion → newVersion)
│   ├── CompatibilityBadge (compatible/breaking)
│   ├── ViewModeToggle (split/unified/summary)
│   └── ExportButton
├── ViewerToolbar
│   ├── SearchInput
│   ├── FilterDropdown
│   │   ├── ChangeTypeFilter
│   │   └── SeverityFilter
│   ├── ShowOnlyBreakingToggle
│   └── Legend
├── ViewerBody (dynamic based on viewMode)
│   ├── SplitView (viewMode === 'split')
│   │   ├── OldSchemaPane
│   │   │   ├── LineNumbers
│   │   │   ├── SyntaxHighlightedJSON
│   │   │   └── ChangeMarkers (red for removed)
│   │   └── NewSchemaPane
│   │       ├── LineNumbers
│   │       ├── SyntaxHighlightedJSON
│   │       └── ChangeMarkers (green for added)
│   ├── UnifiedView (viewMode === 'unified')
│   │   ├── DiffLines
│   │   │   ├── UnchangedLine (gray)
│   │   │   ├── RemovedLine (red background)
│   │   │   └── AddedLine (green background)
│   │   └── ChangeMarkers
│   └── SummaryView (viewMode === 'summary')
│       ├── OverviewPanel
│       │   ├── TotalChangesCount
│       │   ├── BreakingChangesCount
│       │   ├── AdditiveChangesCount
│       │   └── VersionBumpRecommendation
│       ├── BreakingChangesList
│       │   └── BreakingChangeCard[]
│       │       ├── ChangeIcon
│       │       ├── ChangeDescription
│       │       ├── ImpactBadge
│       │       └── AffectedRecordsCount
│       ├── AdditiveChangesList
│       │   └── AdditiveChangeCard[]
│       └── ImpactAnalysisChart
├── DetailPanel (when change selected)
│   ├── ChangeDetailHeader
│   ├── BeforeAfterComparison
│   ├── AffectedRecordsInfo
│   ├── MigrationSuggestion
│   └── CloseButton
└── ViewerFooter
    ├── ConfidenceScore
    ├── MigrationTimeEstimate
    └── SummaryText
```

### Data Flow

```
Component Mounts
    ↓
Parse oldSchema and newSchema (if strings)
    ↓
Run Compatibility Analysis
    ├─ Detect Breaking Changes
    │   ├─ Field removals
    │   ├─ Type changes
    │   ├─ Constraint additions
    │   └─ Enum reductions
    ├─ Detect Additive Changes
    │   ├─ Field additions
    │   ├─ Description changes
    │   └─ Constraint relaxations
    ├─ Calculate Version Bump
    │   └─ MAJOR if breaking, MINOR if additive, PATCH if metadata
    └─ Estimate Impact
        ├─ Fetch affected record counts (if fetchAffectedRecords provided)
        ├─ Estimate migration time
        └─ Calculate confidence score
    ↓
Generate Compatibility Report
    ↓
Render UI based on viewMode
    ↓
User Interactions
    ├─ Click on change → Show detail panel
    ├─ Filter changes → Re-render filtered list
    ├─ Switch view mode → Re-render with new view
    └─ Export report → Generate and download file
```

### Diff Algorithm

The component uses a variation of the Myers diff algorithm optimized for JSON:

```typescript
function detectChanges(oldSchema: object, newSchema: object): SchemaChange[] {
  const changes: SchemaChange[] = []

  // Recursively compare schema trees
  function compareObjects(oldObj: any, newObj: any, path: string = '$') {
    // Detect field removals
    for (const key in oldObj) {
      if (!(key in newObj)) {
        changes.push({
          id: generateId(),
          type: 'field_removed',
          path: `${path}.${key}`,
          description: `Field '${key}' removed`,
          isBreaking: true,
          impact: calculateImpact(oldObj[key]),
          oldValue: oldObj[key],
          newValue: undefined
        })
      }
    }

    // Detect field additions
    for (const key in newObj) {
      if (!(key in oldObj)) {
        changes.push({
          id: generateId(),
          type: 'field_added',
          path: `${path}.${key}`,
          description: `Field '${key}' added`,
          isBreaking: false,
          oldValue: undefined,
          newValue: newObj[key]
        })
      }
    }

    // Detect field modifications
    for (const key in oldObj) {
      if (key in newObj) {
        const oldVal = oldObj[key]
        const newVal = newObj[key]

        // Type change
        if (oldVal.type !== newVal.type) {
          changes.push({
            id: generateId(),
            type: 'type_changed',
            path: `${path}.${key}.type`,
            description: `Field '${key}' type changed from ${oldVal.type} to ${newVal.type}`,
            isBreaking: true,
            impact: 'high',
            oldValue: oldVal.type,
            newValue: newVal.type
          })
        }

        // Enum reduction
        if (oldVal.enum && newVal.enum) {
          const removedValues = oldVal.enum.filter(v => !newVal.enum.includes(v))
          if (removedValues.length > 0) {
            changes.push({
              id: generateId(),
              type: 'enum_reduced',
              path: `${path}.${key}.enum`,
              description: `Enum values removed: ${removedValues.join(', ')}`,
              isBreaking: true,
              impact: 'high',
              oldValue: oldVal.enum,
              newValue: newVal.enum
            })
          }
        }

        // Recurse for nested objects
        if (typeof oldVal === 'object' && typeof newVal === 'object') {
          compareObjects(oldVal, newVal, `${path}.${key}`)
        }
      }
    }
  }

  compareObjects(oldSchema, newSchema)
  return changes
}
```

---

## UI Components

### 1. ViewerHeader

Displays schema metadata and controls.

**Elements:**
- **Schema Version Badge**: Shows "v2.1.0 → v3.0.0"
- **Compatibility Badge**: Color-coded (green=compatible, red=breaking)
- **View Mode Toggle**: Buttons for Split/Unified/Summary
- **Export Button**: Download report

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ 📄 payment_event_schema: v2.1.0 → v3.0.0                  │
│ 🔴 BREAKING CHANGES DETECTED                               │
│                                                            │
│ [Split] [Unified] [Summary]  | 📥 Export ▼                │
└────────────────────────────────────────────────────────────┘
```

### 2. ViewerToolbar

Filtering and search controls.

**Elements:**
- **Search Input**: Filter changes by keyword
- **Change Type Filter**: Multi-select dropdown
- **Severity Filter**: High/Medium/Low
- **Show Only Breaking Toggle**: Quick filter
- **Legend**: Explains color coding

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ 🔍 Search changes...  | Type: All ▼ | Severity: All ▼     │
│ ☑ Show only breaking changes                              │
│                                                            │
│ Legend: 🔴 Breaking | 🟢 Additive | 🟡 Warning            │
└────────────────────────────────────────────────────────────┘
```

### 3. SplitView (Side-by-Side Diff)

Shows old and new schemas side-by-side with changes highlighted.

**Visual Design:**
```
┌──────────────────────┬──────────────────────────────────────┐
│ v2.1.0 (old)         │ v3.0.0 (new)                         │
├──────────────────────┼──────────────────────────────────────┤
│  1  {                │  1  {                                │
│  2    "type": "obj.."│  2    "type": "object",              │
│  3    "properties": {│  3    "properties": {                │
│  4      "amount": {  │  4      "amount": {                  │
│  5        "type": "..│  5        "type": "number",          │
│  6                   │  6        "minimum": 0  ← GREEN (new)│
│  7      },           │  7      },                           │
│  8      "currency": {│  8      "currency": {                │
│  9        "enum": [  │  9        "enum": [                  │
│ 10          "USD",   │ 10          "USD",                   │
│ 11          "EUR",   │ 11          "EUR"                    │
│ 12          "GBP" ← RED (removed)                           │
│ 13        ]          │ 12        ]                          │
│ 14      }            │ 13      }                            │
│ 15    }              │ 14    }                              │
│ 16  }                │ 15  }                                │
└──────────────────────┴──────────────────────────────────────┘
```

### 4. UnifiedView (Inline Diff)

Shows changes inline with +/- indicators.

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│  1    {                                                    │
│  2      "type": "object",                                  │
│  3      "properties": {                                    │
│  4        "amount": {                                      │
│  5          "type": "number",                              │
│  6  +       "minimum": 0                     ← GREEN (add) │
│  7        },                                               │
│  8        "currency": {                                    │
│  9          "enum": [                                      │
│ 10            "USD",                                       │
│ 11            "EUR",                                       │
│ 12  -         "GBP"                          ← RED (remove)│
│ 13          ]                                              │
│ 14        }                                                │
│ 15      }                                                  │
│ 16    }                                                    │
└────────────────────────────────────────────────────────────┘
```

### 5. SummaryView (Change List)

Shows categorized list of changes with impact analysis.

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ Compatibility Summary                                      │
├────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Total Changes: 3                                     │   │
│ │ Breaking: 1 | Additive: 2                            │   │
│ │ Suggested Version: v3.0.0 (MAJOR)                    │   │
│ │ Migration Required: YES                              │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Breaking Changes (1)                                       │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 🔴 HIGH IMPACT                                       │   │
│ │ Enum values removed from 'currency'                  │   │
│ │ Path: $.properties.currency.enum                     │   │
│ │ Removed: ["GBP"]                                     │   │
│ │ Affected Records: 234 (12%)                          │   │
│ │                                                      │   │
│ │ Transformation Required: Map GBP to EUR or USD       │   │
│ │                            [View Details]            │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Additive Changes (2)                                       │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 🟢 LOW IMPACT                                        │   │
│ │ Added constraint 'minimum' to 'amount'               │   │
│ │ Path: $.properties.amount.minimum                    │   │
│ │ Value: 0                                             │   │
│ │ Affected Records: 0 (all amounts already >= 0)       │   │
│ │                            [View Details]            │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Impact Analysis                                            │
│ ┌──────────────────────────────────────────────────────┐   │
│ │     ████████████████░░░░ 12% affected                │   │
│ │     234 / 1,950 total records                        │   │
│ │                                                      │   │
│ │ Migration Time: ~5 minutes                           │   │
│ │ Confidence: 95%                                      │   │
│ └──────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

### 6. DetailPanel (Change Details)

Shows detailed information about a selected change.

**Visual Design:**
```
┌────────────────────────────────────────────────────────────┐
│ Change Details                                        [✕]  │
├────────────────────────────────────────────────────────────┤
│ Type: Enum Reduced                                         │
│ Path: $.properties.currency.enum                           │
│ Impact: HIGH                                               │
│                                                            │
│ Before:                                                    │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ ["USD", "EUR", "GBP"]                                │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ After:                                                     │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ ["USD", "EUR"]                                       │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Affected Records: 234 (12% of total)                       │
│ Records with currency='GBP': 234                           │
│                                                            │
│ Migration Strategy:                                        │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Option 1: Map all GBP to EUR                         │   │
│ │ Option 2: Map all GBP to USD                         │   │
│ │ Option 3: Reject records with GBP (data loss)        │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                            │
│ Sample Transformation:                                     │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ Old: { "currency": "GBP", "amount": 50.00 }          │   │
│ │ New: { "currency": "EUR", "amount": 50.00 }          │   │
│ └──────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

---

## Diff Visualization

### Color Coding

| Change Type | Color | Background | Border |
|------------|-------|------------|--------|
| Breaking (removed) | #DC2626 | #FEE2E2 | #DC2626 |
| Additive (added) | #059669 | #D1FAE5 | #059669 |
| Warning (modified) | #D97706 | #FEF3C7 | #D97706 |
| Info (metadata) | #3B82F6 | #DBEAFE | #3B82F6 |
| Unchanged | #6B7280 | #F9FAFB | #E5E7EB |

### Syntax Highlighting

JSON syntax highlighting with Monaco Editor color scheme:

| Element | Color |
|---------|-------|
| String | #0C7D9D |
| Number | #098658 |
| Boolean | #0000FF |
| Null | #0000FF |
| Key | #001080 |
| Punctuation | #000000 |

### Change Markers

Visual indicators for different change types:

- **Field Removed**: Red stripe on left margin + strikethrough text
- **Field Added**: Green stripe on left margin + bold text
- **Type Changed**: Orange stripe + underline
- **Constraint Added**: Blue stripe + italic
- **Enum Changed**: Red/green split stripe

### Gutter Icons

Icons in the gutter (left margin) indicate change type:

| Icon | Change Type |
|------|------------|
| ❌ | Field removed (breaking) |
| ✅ | Field added (additive) |
| 🔄 | Type changed (breaking) |
| 🔒 | Constraint added (breaking) |
| 🔓 | Constraint removed (additive) |
| 📊 | Enum modified |
| 📝 | Description changed (metadata) |

---

## Visual Wireframes

### Wireframe 1: Split View (Default)

```
┌──────────────────────────────────────────────────────────────────┐
│ CompatibilityViewer                                              │
├──────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ HEADER                                                      │   │
│ │ 📄 payment_event_schema: v2.1.0 → v3.0.0                   │   │
│ │ 🔴 BREAKING CHANGES DETECTED                               │   │
│ │                                                            │   │
│ │ [Split] [Unified] [Summary]  |  📥 Export ▼               │   │
│ └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ TOOLBAR                                                     │   │
│ │ 🔍 Search: [          ]  |  Type: All ▼  |  Severity: All ▼│   │
│ │ ☑ Show only breaking changes                              │   │
│ │ Legend: 🔴 Breaking | 🟢 Additive | 🟡 Warning            │   │
│ └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│ ┌──────────────────────────┬───────────────────────────────────┐ │
│ │ v2.1.0 (Old Schema)      │ v3.0.0 (New Schema)               │ │
│ ├──────────────────────────┼───────────────────────────────────┤ │
│ │  1  {                    │  1  {                             │ │
│ │  2    "$schema": "...",  │  2    "$schema": "...",           │ │
│ │  3    "type": "object",  │  3    "type": "object",           │ │
│ │  4    "properties": {    │  4    "properties": {             │ │
│ │  5      "amount": {      │  5      "amount": {               │ │
│ │  6        "type": "num.."│  6        "type": "number",       │ │
│ │  7                       │  7        "minimum": 0  ← 🟢 NEW  │ │
│ │  8      },               │  8      },                        │ │
│ │  9      "currency": {    │  9      "currency": {             │ │
│ │ 10        "enum": [      │ 10        "enum": [               │ │
│ │ 11          "USD",       │ 11          "USD",                │ │
│ │ 12          "EUR",       │ 12          "EUR"                 │ │
│ │ 13 ❌       "GBP"  ← RED  │ 13                  ← 🔴 REMOVED │ │
│ │ 14        ]              │ 14        ]                       │ │
│ │ 15      }                │ 15      }                         │ │
│ │ 16    }                  │ 16    }                           │ │
│ │ 17  }                    │ 17  }                             │ │
│ └──────────────────────────┴───────────────────────────────────┘ │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ FOOTER                                                      │   │
│ │ 3 changes detected | 1 breaking, 2 additive                │   │
│ │ Confidence: 95% | Migration: ~5 minutes | Version: MAJOR   │   │
│ └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Wireframe 2: Summary View

```
┌──────────────────────────────────────────────────────────────────┐
│ CompatibilityViewer - Summary                                    │
├──────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ HEADER                                                      │   │
│ │ 📄 payment_event_schema: v2.1.0 → v3.0.0                   │   │
│ │ 🔴 BREAKING CHANGES DETECTED                               │   │
│ │ [Split] [Unified] [Summary]  |  📥 Export                  │   │
│ └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ OVERVIEW                                                    │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ Total Changes: 3                                     │   │   │
│ │ │ 🔴 Breaking: 1    🟢 Additive: 2                    │   │   │
│ │ │ 📊 Suggested Version: v3.0.0 (MAJOR)                │   │   │
│ │ │ ⚙️  Migration Required: YES (~5 minutes)            │   │   │
│ │ │ ✅ Confidence: 95%                                   │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ BREAKING CHANGES (1)                                        │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ 🔴 HIGH IMPACT                                       │   │   │
│ │ │ Enum values removed from 'currency'                  │   │   │
│ │ │ Path: $.properties.currency.enum                     │   │   │
│ │ │ Removed: ["GBP"]                                     │   │   │
│ │ │ Affected: 234 records (12%)                          │   │   │
│ │ │                                                      │   │   │
│ │ │ Old: ["USD", "EUR", "GBP"]                           │   │   │
│ │ │ New: ["USD", "EUR"]                                  │   │   │
│ │ │                                                      │   │   │
│ │ │ Transformation: Map GBP → EUR or USD                 │   │   │
│ │ │                                  [View Details]      │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ ADDITIVE CHANGES (2)                                        │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ 🟢 LOW IMPACT                                        │   │   │
│ │ │ Added constraint 'minimum' to 'amount'               │   │   │
│ │ │ Path: $.properties.amount.minimum                    │   │   │
│ │ │ Value: 0                                             │   │   │
│ │ │ Affected: 0 records (all valid)                      │   │   │
│ │ │                                  [View Details]      │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ 🟢 INFO                                              │   │   │
│ │ │ Updated description for 'amount'                     │   │   │
│ │ │ Path: $.properties.amount.description                │   │   │
│ │ │ Metadata change (no data impact)                     │   │   │
│ │ │                                  [View Details]      │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ IMPACT ANALYSIS                                             │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │     ████████████████░░░░░░░░░ 12% affected           │   │   │
│ │ │     234 records / 1,950 total                        │   │   │
│ │ │                                                      │   │   │
│ │ │ Migration Complexity: MEDIUM                         │   │   │
│ │ │ Estimated Time: ~5 minutes                           │   │   │
│ │ │ Recommended: Test in staging first                   │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Wireframe 3: Detail Panel (Change Details)

```
┌──────────────────────────────────────────────────────────────────┐
│ CompatibilityViewer - Change Details                             │
├──────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────┬───────────────────────────────────┐ │
│ │ v2.1.0 (Old)             │ v3.0.0 (New)                      │ │
│ ├──────────────────────────┼───────────────────────────────────┤ │
│ │  9      "currency": {    │  9      "currency": {             │ │
│ │ 10        "enum": [      │ 10        "enum": [               │ │
│ │ 11          "USD",       │ 11          "USD",                │ │
│ │ 12          "EUR",       │ 12          "EUR"                 │ │
│ │ 13 ❌       "GBP"  ← RED  │ 13                  ← REMOVED     │ │
│ │ 14        ]              │ 14        ]                       │ │
│ │ (other lines collapsed)  │ (other lines collapsed)           │ │
│ └──────────────────────────┴───────────────────────────────────┘ │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐   │
│ │ DETAIL PANEL                                           [✕]  │   │
│ ├────────────────────────────────────────────────────────────┤   │
│ │ Change Type: Enum Reduced                                  │   │
│ │ Impact: 🔴 HIGH (Breaking Change)                          │   │
│ │ Path: $.properties.currency.enum                           │   │
│ │                                                            │   │
│ │ Before:                                                    │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ ["USD", "EUR", "GBP"]                                │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ After:                                                     │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ ["USD", "EUR"]                                       │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ Removed Values: ["GBP"]                                    │   │
│ │                                                            │   │
│ │ Impact Analysis:                                           │   │
│ │ • Affected Records: 234 (12% of total 1,950)               │   │
│ │ • Records with currency='GBP': 234                         │   │
│ │ • Migration Required: YES                                  │   │
│ │ • Estimated Time: ~2 minutes                               │   │
│ │                                                            │   │
│ │ Migration Strategies:                                      │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ 1. Map all GBP to EUR (recommended)                  │   │   │
│ │ │    • Safe: No data loss                              │   │   │
│ │ │    • Impact: 234 records updated                     │   │   │
│ │ │                                                      │   │   │
│ │ │ 2. Map all GBP to USD                                │   │   │
│ │ │    • Safe: No data loss                              │   │   │
│ │ │    • Impact: 234 records updated                     │   │   │
│ │ │                                                      │   │   │
│ │ │ 3. Reject records with GBP                           │   │   │
│ │ │    • Risky: Data loss (234 records)                  │   │   │
│ │ │    • Not recommended                                 │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ Sample Transformation (Strategy 1):                        │   │
│ │ ┌──────────────────────────────────────────────────────┐   │   │
│ │ │ Before:                                              │   │   │
│ │ │ {                                                    │   │   │
│ │ │   "transactionId": "txn_123",                        │   │   │
│ │ │   "currency": "GBP",                                 │   │   │
│ │ │   "amount": 50.00                                    │   │   │
│ │ │ }                                                    │   │   │
│ │ │                                                      │   │   │
│ │ │ After:                                               │   │   │
│ │ │ {                                                    │   │   │
│ │ │   "transactionId": "txn_123",                        │   │   │
│ │ │   "currency": "EUR",                                 │   │   │
│ │ │   "amount": 50.00,                                   │   │   │
│ │ │   "originalCurrency": "GBP"                          │   │   │
│ │ │ }                                                    │   │   │
│ │ └──────────────────────────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ Recommended Actions:                                       │   │
│ │ ☑ Run migration with dry-run first                        │   │
│ │ ☑ Create backup before migration                          │   │
│ │ ☑ Notify API consumers about breaking change              │   │
│ │ ☑ Update API documentation                                │   │
│ │                                                            │   │
│ │                                            [Close]         │   │
│ └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## States & Lifecycle

### Component States

```typescript
type ViewerState =
  | 'idle'                  // Component mounted, ready
  | 'analyzing'             // Running compatibility analysis
  | 'displaying'            // Showing results
  | 'filtering'             // Applying user filters
  | 'exporting'             // Generating export file
  | 'error'                 // Error occurred during analysis
```

### State Transitions

```
[idle]
  │
  ├─ Component mounts with oldSchema and newSchema ──▶ [analyzing]
  │
[analyzing]
  │
  ├─ Analysis complete ──▶ [displaying]
  ├─ Analysis error ──▶ [error]
  │
[displaying]
  │
  ├─ User applies filter ──▶ [filtering]
  ├─ User clicks export ──▶ [exporting]
  ├─ User changes schemas (props update) ──▶ [analyzing]
  │
[filtering]
  │
  ├─ Filter applied ──▶ [displaying]
  │
[exporting]
  │
  ├─ Export complete ──▶ [displaying]
  ├─ Export error ──▶ [error]
  │
[error]
  │
  ├─ User retries ──▶ [analyzing]
```

---

## Multi-Domain Examples

### 1. Finance: Payment Schema Compatibility

**Scenario:** Fintech company checking compatibility before deprecating PayPal support.

```typescript
<CompatibilityViewer
  schemaId="payment_event_schema"
  oldSchema={v210Schema}
  newSchema={v300Schema}
  oldVersion="v2.1.0"
  newVersion="v3.0.0"
  fetchAffectedRecords={async (changeType, path) => {
    if (changeType === 'enum_reduced' && path.includes('paymentMethod')) {
      const count = await db.query(
        'SELECT COUNT(*) FROM transactions WHERE payment_method = $1',
        ['paypal']
      )
      return count.rows[0].count
    }
    return 0
  }}
  onAnalysisComplete={(report) => {
    if (report.breakingChanges.length > 0) {
      notify.warning(`${report.breakingChanges.length} breaking changes detected`)
    }
  }}
/>
```

**Output:**
- Breaking Changes: 1 (paymentMethod enum reduced)
- Affected Records: 234
- Suggested Version: v3.0.0 (MAJOR)
- Migration Required: YES

### 2. Healthcare: Patient Record Compatibility

**Scenario:** Hospital checking impact of adding required HIPAA fields.

```typescript
<CompatibilityViewer
  schemaId="patient_record_schema"
  oldSchema={v150Schema}
  newSchema={v200Schema}
  oldVersion="v1.5.0"
  newVersion="v2.0.0"
  fetchAffectedRecords={async (changeType, path) => {
    if (changeType === 'required_added') {
      // All 50,000 records need new required fields
      return 50000
    }
    return 0
  }}
  estimateMigrationTime={async (changeType, affectedRecords) => {
    // Estimate 100 records/second
    return (affectedRecords / 100) * 1000
  }}
  onAnalysisComplete={(report) => {
    auditLog.record('schema_compatibility_check', {
      fromVersion: 'v1.5.0',
      toVersion: 'v2.0.0',
      compatible: report.compatible,
      breakingChanges: report.breakingChanges.length
    })
  }}
/>
```

**Output:**
- Breaking Changes: 2 (required fields added)
- Affected Records: 50,000
- Suggested Version: v2.0.0 (MAJOR)
- Migration Time: ~8 minutes

### 3. Legal: Case Schema Compatibility

**Scenario:** Law firm checking compatibility after enforcing case number format.

```typescript
<CompatibilityViewer
  schemaId="legal_case_schema"
  oldSchema={v230Schema}
  newSchema={v300Schema}
  oldVersion="v2.3.0"
  newVersion="v3.0.0"
  showDiff={true}
  viewMode="split"
  customRules={[
    {
      id: 'case-number-format',
      name: 'Case Number Format Validation',
      detect: (oldSchema, newSchema) => {
        const oldPattern = oldSchema.properties?.caseNumber?.pattern
        const newPattern = newSchema.properties?.caseNumber?.pattern

        if (!oldPattern && newPattern) {
          return [{
            id: 'case-number-pattern-added',
            type: 'constraint_added',
            path: '$.properties.caseNumber.pattern',
            description: 'Added regex pattern validation for case numbers',
            isBreaking: true,
            impact: 'high',
            newValue: newPattern,
            transformationRequired: true,
            suggestion: 'Validate all existing case numbers against new pattern'
          }]
        }

        return []
      }
    }
  ]}
/>
```

**Output:**
- Breaking Changes: 1 (pattern constraint added)
- Affected Records: 567 (invalid format)
- Suggested Version: v3.0.0 (MAJOR)

### 4. Research: Experiment Schema Compatibility

**Scenario:** Research lab checking compatibility after adding reproducibility fields.

```typescript
<CompatibilityViewer
  schemaId="experiment_result_schema"
  oldSchema={v100Schema}
  newSchema={v110Schema}
  oldVersion="v1.0.0"
  newVersion="v1.1.0"
  showOnlyBreaking={false}  // Show all changes, even additive
  onAnalysisComplete={(report) => {
    if (report.compatible) {
      console.log('Schema is backward compatible - no migration needed')
    }
  }}
/>
```

**Output:**
- Breaking Changes: 0
- Additive Changes: 3 (new optional fields)
- Suggested Version: v1.1.0 (MINOR)
- Migration Required: NO

### 5. E-commerce: Product Schema Compatibility

**Scenario:** E-commerce platform checking compatibility after adding variants support.

```typescript
<CompatibilityViewer
  schemaId="product_schema"
  oldSchema={v200Schema}
  newSchema={v210Schema}
  oldVersion="v2.0.0"
  newVersion="v2.1.0"
  viewMode="summary"
  showImpactAnalysis={true}
  onExport={(format, data) => {
    // Save report for product team review
    storage.save(`product-schema-compatibility-report.${format}`, data)
  }}
/>
```

**Output:**
- Breaking Changes: 0
- Additive Changes: 1 (variants field added)
- Suggested Version: v2.1.0 (MINOR)

### 6. SaaS: API Event Schema Compatibility

**Scenario:** SaaS platform checking compatibility after deprecating old event types.

```typescript
<CompatibilityViewer
  schemaId="api_event_schema"
  oldSchema={v350Schema}
  newSchema={v400Schema}
  oldVersion="v3.5.0"
  newVersion="v4.0.0"
  fetchAffectedRecords={async (changeType, path) => {
    if (changeType === 'enum_reduced' && path.includes('eventType')) {
      const deprecatedEvents = ['user.login', 'user.logout']
      const count = await analytics.query(
        'event_count',
        { eventTypes: deprecatedEvents }
      )
      return count.total
    }
    return 0
  }}
  colorScheme={{
    breaking: '#FF0000',
    additive: '#00FF00',
    warning: '#FFA500'
  }}
/>
```

**Output:**
- Breaking Changes: 1 (eventType enum reduced)
- Affected Records: 45,678
- Suggested Version: v4.0.0 (MAJOR)

### 7. Insurance: Claim Schema Compatibility

**Scenario:** Insurance company checking compatibility after adding fraud detection fields.

```typescript
<CompatibilityViewer
  schemaId="insurance_claim_schema"
  oldSchema={v180Schema}
  newSchema={v190Schema}
  oldVersion="v1.8.0"
  newVersion="v1.9.0"
  showVersionRecommendation={true}
  showMigrationEstimate={true}
  estimateMigrationTime={async (changeType, affectedRecords) => {
    // Fraud score calculation is slow (ML API)
    // Estimate 10 records/second
    return (affectedRecords / 10) * 1000
  }}
/>
```

**Output:**
- Breaking Changes: 0
- Additive Changes: 2 (fraudScore, flaggedReasons added)
- Suggested Version: v1.9.0 (MINOR)
- Migration Time: ~42 minutes (25,000 records at 10/sec)

---

## User Interactions

### 1. Viewing Diff

**Trigger:** Component loads with oldSchema and newSchema

**Flow:**
1. Parse schemas
2. Run compatibility analysis
3. Display diff in default view mode (split)
4. User can toggle between split/unified/summary views

### 2. Filtering Changes

**Trigger:** User types in search box or selects filter

**Flow:**
1. User enters "currency" in search
2. Filter changes to show only those containing "currency"
3. Diff view updates to show only relevant sections
4. Change count updates: "Showing 1 of 3 changes"

### 3. Clicking on Change

**Trigger:** User clicks on a change in the diff or summary list

**Flow:**
1. Highlight change in diff view
2. Open detail panel on the right
3. Show before/after comparison
4. Display affected records count
5. Suggest migration strategies

### 4. Exporting Report

**Trigger:** User clicks "Export" button

**Flow:**
1. Show export format dropdown (JSON/PDF/Markdown/CSV)
2. User selects format
3. Generate report with compatibility analysis
4. Download file

**Example PDF Report:**
```
=====================================
Schema Compatibility Report
=====================================

Schema: payment_event_schema
Old Version: v2.1.0
New Version: v3.0.0
Analysis Date: 2025-10-25

Summary:
--------
Total Changes: 3
Breaking Changes: 1
Additive Changes: 2
Suggested Version: v3.0.0 (MAJOR)
Migration Required: YES

Breaking Changes:
-----------------
1. Enum values removed from 'currency'
   Path: $.properties.currency.enum
   Removed: ["GBP"]
   Affected Records: 234 (12%)
   Impact: HIGH
   Transformation: Map GBP to EUR or USD

Additive Changes:
-----------------
1. Added constraint 'minimum' to 'amount'
   Path: $.properties.amount.minimum
   Value: 0
   Affected Records: 0

2. Updated description for 'amount'
   Path: $.properties.amount.description
   Impact: None (metadata only)

Impact Analysis:
----------------
Total Records: 1,950
Affected Records: 234 (12%)
Migration Time: ~5 minutes
Confidence: 95%

Recommendations:
----------------
1. Create backup before migration
2. Run dry-run migration first
3. Notify API consumers about breaking change
4. Update API documentation
```

### 5. Toggling View Modes

**Trigger:** User clicks view mode button (Split/Unified/Summary)

**Flow:**
1. User clicks "Unified"
2. Transition from split view to unified view
3. Combine schemas into single pane with +/- lines
4. Maintain scroll position

### 6. Hovering Over Change

**Trigger:** User hovers mouse over a change marker

**Flow:**
1. Show tooltip with change details
2. Highlight corresponding line in both panes (split view)
3. On hover end, hide tooltip

### 7. Expanding/Collapsing Sections

**Trigger:** User clicks collapse button on unchanged section

**Flow:**
1. Collapse unchanged lines between changes
2. Show "... 15 unchanged lines ..."
3. On click, expand section

---

## Accessibility

### WCAG 2.1 AA Compliance

#### 1. Keyboard Navigation

| Action | Keyboard Shortcut |
|--------|------------------|
| Next change | `n` or `↓` |
| Previous change | `p` or `↑` |
| Toggle view mode | `v` |
| Export report | `Ctrl+E` |
| Search changes | `/` |
| Focus search | `Ctrl+F` |
| Close detail panel | `Esc` |

#### 2. Screen Reader Support

**ARIA Labels:**
```html
<div role="region" aria-label="Schema compatibility viewer">
  <div role="toolbar" aria-label="View controls">
    <button aria-label="Split view" aria-pressed="true">Split</button>
    <button aria-label="Unified view" aria-pressed="false">Unified</button>
    <button aria-label="Summary view" aria-pressed="false">Summary</button>
  </div>

  <div role="region" aria-label="Breaking changes" aria-live="polite">
    <h3>Breaking Changes (1)</h3>
    <ul role="list">
      <li role="listitem">
        <span aria-label="High impact breaking change">
          Enum values removed from currency
        </span>
      </li>
    </ul>
  </div>
</div>
```

**Announcements:**
```
"Compatibility analysis complete. 1 breaking change detected."
"Showing 2 of 3 changes. Filtered by type: enum_reduced."
"Exporting report as PDF. Download starting..."
```

#### 3. Color Contrast

All colors meet WCAG AA contrast ratios:

| Element | Contrast Ratio |
|---------|----------------|
| Breaking change text | 5.8:1 |
| Additive change text | 4.7:1 |
| Warning text | 4.9:1 |
| Diff line numbers | 7.2:1 |

#### 4. Focus Indicators

All interactive elements have visible focus:
```css
.change-card:focus,
.view-mode-button:focus {
  outline: 3px solid #2563EB;
  outline-offset: 2px;
}
```

---

## Performance Optimizations

### 1. Lazy Diff Calculation

Only calculate diff for visible sections:

```typescript
const visibleDiff = useMemo(() => {
  const startLine = Math.max(0, scrollPosition - 100)
  const endLine = scrollPosition + 100

  return calculateDiff(oldSchema, newSchema, startLine, endLine)
}, [oldSchema, newSchema, scrollPosition])
```

### 2. Virtual Scrolling

For large schemas (>1000 lines), use virtual scrolling:

```typescript
import { FixedSizeList } from 'react-window'

<FixedSizeList
  height={600}
  itemCount={diffLines.length}
  itemSize={20}
  width="100%"
>
  {({ index, style }) => (
    <DiffLine line={diffLines[index]} style={style} />
  )}
</FixedSizeList>
```

### 3. Memoized Change Detection

Cache change detection results:

```typescript
const changes = useMemo(() => {
  return detectChanges(oldSchema, newSchema)
}, [oldSchema, newSchema])
```

### 4. Web Workers for Large Schemas

Offload diff calculation to Web Worker:

```typescript
// diff.worker.ts
self.addEventListener('message', (e) => {
  const { oldSchema, newSchema } = e.data
  const changes = detectChanges(oldSchema, newSchema)
  self.postMessage({ changes })
})

// In component
useEffect(() => {
  const worker = new Worker('/diff.worker.js')

  worker.postMessage({ oldSchema, newSchema })

  worker.onmessage = (e) => {
    setChanges(e.data.changes)
  }

  return () => worker.terminate()
}, [oldSchema, newSchema])
```

### 5. Debounced Search

Debounce search input to avoid excessive filtering:

```typescript
const debouncedSearch = useMemo(
  () => debounce((query: string) => {
    setFilteredChanges(changes.filter(c => c.description.includes(query)))
  }, 300),
  [changes]
)
```

---

## Integration Examples

### Example 1: Basic React Integration

```typescript
import { CompatibilityViewer } from '@/components/il/CompatibilityViewer'

export default function SchemaComparisonPage() {
  const oldSchema = {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
      "currency": { "enum": ["USD", "EUR", "GBP"] }
    }
  }

  const newSchema = {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
      "currency": { "enum": ["USD", "EUR"] }
    }
  }

  return (
    <CompatibilityViewer
      oldSchema={oldSchema}
      newSchema={newSchema}
      oldVersion="v2.1.0"
      newVersion="v3.0.0"
      showDiff={true}
      viewMode="split"
    />
  )
}
```

### Example 2: With Data Fetching

```typescript
function SchemaComparison({ schemaId, fromVersion, toVersion }) {
  const { data: oldSchema } = useQuery(
    ['schema', schemaId, fromVersion],
    () => api.getSchema(schemaId, fromVersion)
  )

  const { data: newSchema } = useQuery(
    ['schema', schemaId, toVersion],
    () => api.getSchema(schemaId, toVersion)
  )

  if (!oldSchema || !newSchema) return <Loading />

  return (
    <CompatibilityViewer
      schemaId={schemaId}
      oldSchema={oldSchema.content}
      newSchema={newSchema.content}
      oldVersion={fromVersion}
      newVersion={toVersion}
      fetchAffectedRecords={async (changeType, path) => {
        const { count } = await api.getAffectedRecords(
          schemaId,
          changeType,
          path
        )
        return count
      }}
    />
  )
}
```

### Example 3: Embedded in SchemaEditor

```typescript
<SchemaEditor
  schemaId="payment_event_schema"
  currentVersion="2.1.0"
  initialSchema={currentSchema}
  previousSchema={previousSchema}
  onCompatibilityCheck={(report) => {
    if (!report.compatible) {
      setShowCompatibilityViewer(true)
    }
  }}
/>

{showCompatibilityViewer && (
  <Dialog open={showCompatibilityViewer}>
    <DialogContent>
      <CompatibilityViewer
        oldSchema={previousSchema}
        newSchema={currentSchema}
        viewMode="summary"
        showExportButton={true}
      />
    </DialogContent>
  </Dialog>
)}
```

---

## Related Components

### 1. SchemaEditor

SchemaEditor uses CompatibilityViewer to show inline compatibility analysis.

### 2. MigrationWizard

MigrationWizard uses CompatibilityViewer in Step 2 (Review Plan).

### 3. SchemaHistory

SchemaHistory allows comparing any two versions using CompatibilityViewer.

---

## References

- [JSON Schema Specification](https://json-schema.org/)
- [Myers Diff Algorithm](https://blog.jcoglan.com/2017/02/12/the-myers-diff-algorithm-part-1/)
- [Semantic Versioning](https://semver.org/)
- [Schema Evolution Best Practices](https://docs.confluent.io/platform/current/schema-registry/avro.html)

---

**Document Version:** 1.0
**Word Count:** ~9,200 words (~920 lines)
**Last Updated:** 2025-10-25
**Maintained By:** Platform Team
