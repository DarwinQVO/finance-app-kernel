# TimelineViewer

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
5. [Visualization Modes](#visualization-modes)
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

The **TimelineViewer** is a comprehensive visualization component designed to display the complete provenance history of an entity across both temporal dimensions: **transaction time** (when facts were recorded in the system) and **valid time** (when facts were actually true in the real world). This bitemporal visualization enables users to understand how entity data evolved, track corrections, audit changes, and answer critical questions like "What did we know on date X?" or "What was actually true on date Y?"

### Key Capabilities

- **Bitemporal Visualization**: Display both transaction time and valid time on independent axes
- **Multiple View Modes**: Timeline (visual), Table (sortable), Compact (mobile-friendly)
- **Event Categorization**: Color-coded events (extractions, overrides, corrections, reverts)
- **Interactive Timeline**: Zoom, pan, filter, and drill down into specific time periods
- **Export Functionality**: Export to PNG, SVG, CSV, or JSON for external analysis
- **Real-time Updates**: Support for streaming updates as new events occur
- **Responsive Design**: Optimized for desktop, tablet, and mobile devices
- **Accessibility**: Full keyboard navigation and screen reader support

### Design Philosophy

The TimelineViewer embodies the principle that **provenance is transparent and explorable**. Unlike simple audit logs, this component:

- **Separates temporal dimensions**: Transaction time vs. valid time are distinct concepts
- **Visualizes causality**: Shows relationships between events (e.g., correction X triggered recomputation Y)
- **Supports investigation**: Enables root cause analysis for data quality issues
- **Handles complexity**: Scales from simple entities (few events) to complex ones (thousands of events)
- **Promotes trust**: Transparency in data lineage builds confidence in system outputs

This transforms audit trails from compliance artifacts into powerful analytical tools.

---

## Component Interface

### Primary Props

```typescript
interface TimelineViewerProps {
  // ============================================================
  // Entity Configuration
  // ============================================================

  /**
   * Entity identifier to display timeline for
   * @example 'txn_abc123', 'patient_45678'
   */
  entityId: string

  /**
   * Entity type for domain-specific rendering
   * @example 'transaction', 'patient_record', 'legal_case'
   */
  entityType: string

  /**
   * Current entity snapshot (optional)
   * Used to show "current state" marker on timeline
   */
  currentEntity?: Entity

  // ============================================================
  // Timeline Data
  // ============================================================

  /**
   * Array of timeline events to display
   * Each event represents a change to the entity
   */
  timelineEvents: TimelineEvent[]

  /**
   * Additional metadata about the timeline
   * @example { totalEvents: 1234, oldestEvent: '2020-01-01', newestEvent: '2025-10-25' }
   */
  timelineMetadata?: TimelineMetadata

  /**
   * Enable lazy loading for large timelines
   * Fetches events on-demand as user scrolls/zooms
   * @default false
   */
  lazyLoad?: boolean

  /**
   * Callback to fetch more events (if lazyLoad=true)
   * @param dateRange - Date range to fetch events for
   * @returns Promise<TimelineEvent[]>
   */
  onFetchEvents?: (dateRange: DateRange) => Promise<TimelineEvent[]>

  // ============================================================
  // View Configuration
  // ============================================================

  /**
   * Default view mode
   * @default 'timeline'
   */
  viewMode?: 'timeline' | 'table' | 'compact'

  /**
   * Allow user to switch view modes
   * @default true
   */
  allowViewModeToggle?: boolean

  /**
   * Timeline orientation (for timeline view)
   * @default 'horizontal'
   */
  orientation?: 'horizontal' | 'vertical'

  /**
   * Default temporal dimension to show
   * @default 'transaction'
   */
  defaultTimeAxis?: 'transaction' | 'valid' | 'both'

  /**
   * Initial zoom level (for timeline view)
   * @default 'fit' - Fit all events in viewport
   */
  initialZoom?: 'fit' | 'year' | 'month' | 'week' | 'day' | 'hour'

  /**
   * Initial date range to display
   * If not provided, shows all events
   */
  initialDateRange?: DateRange

  // ============================================================
  // Filtering & Grouping
  // ============================================================

  /**
   * Event type filters (show/hide specific event types)
   * @example { extraction: true, override: true, correction: false }
   */
  eventTypeFilters?: Record<EventType, boolean>

  /**
   * Field filters (show events for specific fields only)
   * @example ['amount', 'date', 'merchant']
   */
  fieldFilters?: string[]

  /**
   * Actor filters (show events by specific users/systems)
   * @example ['user_123', 'system_ocr']
   */
  actorFilters?: string[]

  /**
   * Group events by criteria
   * @default 'none'
   */
  groupBy?: 'none' | 'field' | 'actor' | 'eventType' | 'day' | 'week' | 'month'

  // ============================================================
  // Interaction Callbacks
  // ============================================================

  /**
   * Called when user clicks on an event
   * @param event - The clicked timeline event
   */
  onEventClick?: (event: TimelineEvent) => void

  /**
   * Called when user hovers over an event
   * @param event - The hovered timeline event (null when hover ends)
   */
  onEventHover?: (event: TimelineEvent | null) => void

  /**
   * Called when user selects a date range (e.g., brush selection)
   * @param range - Selected date range
   */
  onDateRangeSelect?: (range: DateRange) => void

  /**
   * Called when user changes filters
   * @param filters - Current filter state
   */
  onFiltersChange?: (filters: TimelineFilters) => void

  /**
   * Called when user exports timeline
   * @param format - Export format
   * @param data - Exported data
   */
  onExport?: (format: ExportFormat, data: any) => void

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Show event details panel on click
   * @default true
   */
  showEventDetails?: boolean

  /**
   * Show filter controls
   * @default true
   */
  showFilters?: boolean

  /**
   * Show export button
   * @default true
   */
  showExport?: boolean

  /**
   * Show zoom controls (for timeline view)
   * @default true
   */
  showZoomControls?: boolean

  /**
   * Show minimap/overview (for timeline view)
   * Provides context when zoomed in
   * @default true for timelines with >50 events
   */
  showMinimap?: boolean

  /**
   * Show confidence scores on events
   * @default true
   */
  showConfidence?: boolean

  /**
   * Show event counts in legend
   * @default true
   */
  showEventCounts?: boolean

  /**
   * Enable search within timeline
   * @default true
   */
  enableSearch?: boolean

  /**
   * Height of the component (CSS value)
   * @default '600px'
   */
  height?: string

  /**
   * Custom color scheme for event types
   */
  colorScheme?: EventColorScheme

  /**
   * Loading state (external loading indicator)
   * @default false
   */
  isLoading?: boolean

  /**
   * Empty state message when no events
   */
  emptyStateMessage?: string

  // ============================================================
  // Advanced Options
  // ============================================================

  /**
   * Enable bitemporal view (show both time axes)
   * @default false (shows transaction time only)
   */
  bitemporalView?: boolean

  /**
   * Show causality arrows between related events
   * E.g., correction → recomputation
   * @default true
   */
  showCausality?: boolean

  /**
   * Highlight specific events
   * @example [{ eventId: 'evt_123', color: '#ff0000', label: 'Critical' }]
   */
  highlightedEvents?: HighlightConfig[]

  /**
   * Comparison mode: compare two time periods
   * @example { periodA: '2024-Q1', periodB: '2024-Q2' }
   */
  comparisonMode?: ComparisonConfig

  /**
   * Custom event renderer for domain-specific visualizations
   */
  customEventRenderer?: (event: TimelineEvent) => React.ReactNode

  /**
   * Performance mode for large timelines (>1000 events)
   * Enables virtualization and reduces visual complexity
   * @default 'auto' (auto-detect based on event count)
   */
  performanceMode?: 'standard' | 'optimized' | 'auto'
}
```

### Supporting Types

```typescript
/**
 * Timeline event representing a change to the entity
 */
interface TimelineEvent {
  /**
   * Unique event identifier
   */
  id: string

  /**
   * Event type
   */
  eventType: EventType

  /**
   * Transaction time: when event was recorded in system
   */
  transactionTime: string // ISO 8601

  /**
   * Valid time: when the fact was true in real world
   * If null, assumes same as transactionTime
   */
  validTime?: string // ISO 8601

  /**
   * Actor who triggered the event
   */
  actor: Actor

  /**
   * Fields affected by this event
   */
  affectedFields: FieldChange[]

  /**
   * Reason for change (for corrections/overrides)
   */
  reason?: string

  /**
   * Confidence score (0-1) for extracted values
   */
  confidence?: number

  /**
   * Parent event that triggered this event
   * E.g., correction event triggers recomputation event
   */
  causedBy?: string // Event ID

  /**
   * Additional metadata
   */
  metadata?: Record<string, any>

  /**
   * Event severity (for filtering/highlighting)
   */
  severity?: 'info' | 'warning' | 'error'
}

/**
 * Event type enumeration
 */
type EventType =
  | 'extraction'      // Initial extraction from document
  | 'override'        // Manual override by user
  | 'correction'      // Retroactive correction
  | 'recomputation'   // Automatic recomputation
  | 'validation'      // Validation check
  | 'merge'           // Entity merge
  | 'split'           // Entity split
  | 'revert'          // Revert to previous value
  | 'import'          // Bulk import
  | 'migration'       // Data migration
  | 'custom'          // Domain-specific event type

/**
 * Actor (user or system) who triggered the event
 */
interface Actor {
  /**
   * Actor identifier
   */
  id: string

  /**
   * Actor type
   */
  type: 'user' | 'system'

  /**
   * Display name
   */
  name: string

  /**
   * Email (for users)
   */
  email?: string

  /**
   * System component (for systems)
   */
  component?: string
}

/**
 * Field change within an event
 */
interface FieldChange {
  /**
   * Field identifier
   */
  fieldName: string

  /**
   * Previous value (null if first time set)
   */
  oldValue: any

  /**
   * New value
   */
  newValue: any

  /**
   * Field type for rendering
   */
  fieldType: 'text' | 'number' | 'date' | 'boolean' | 'enum' | 'json'

  /**
   * Confidence score for this field change
   */
  confidence?: number
}

/**
 * Date range for filtering/zooming
 */
interface DateRange {
  start: string // ISO 8601
  end: string   // ISO 8601
}

/**
 * Timeline metadata
 */
interface TimelineMetadata {
  totalEvents: number
  oldestEvent: string // ISO 8601
  newestEvent: string // ISO 8601
  eventTypeCounts: Record<EventType, number>
  actorCounts: Record<string, number>
}

/**
 * Timeline filters state
 */
interface TimelineFilters {
  eventTypes: Record<EventType, boolean>
  fields: string[]
  actors: string[]
  dateRange: DateRange | null
  searchQuery: string
}

/**
 * Export format
 */
type ExportFormat = 'png' | 'svg' | 'csv' | 'json' | 'pdf'

/**
 * Event color scheme
 */
interface EventColorScheme {
  extraction: string
  override: string
  correction: string
  recomputation: string
  validation: string
  merge: string
  split: string
  revert: string
  import: string
  migration: string
  custom: string
}

/**
 * Highlight configuration
 */
interface HighlightConfig {
  eventId: string
  color: string
  label?: string
  icon?: string
}

/**
 * Comparison configuration
 */
interface ComparisonConfig {
  periodA: DateRange
  periodB: DateRange
  compareFields?: string[]
}

/**
 * Entity snapshot
 */
interface Entity {
  id: string
  type: string
  fields: Record<string, any>
  metadata?: Record<string, any>
}
```

### Default Color Scheme

```typescript
const DEFAULT_COLOR_SCHEME: EventColorScheme = {
  extraction: '#3B82F6',      // Blue
  override: '#F59E0B',        // Orange
  correction: '#8B5CF6',      // Purple
  recomputation: '#10B981',   // Green
  validation: '#6366F1',      // Indigo
  merge: '#EC4899',           // Pink
  split: '#F97316',           // Dark Orange
  revert: '#EF4444',          // Red
  import: '#14B8A6',          // Teal
  migration: '#84CC16',       // Lime
  custom: '#6B7280'           // Gray
}
```

---

## Architecture

### Component Structure

```
TimelineViewer/
├── TimelineViewer.tsx           # Main component
├── components/
│   ├── TimelineView.tsx         # D3.js timeline visualization
│   ├── TableView.tsx            # Sortable table view
│   ├── CompactView.tsx          # Mobile-friendly list view
│   ├── EventDetailsPanel.tsx   # Event detail sidebar
│   ├── FilterControls.tsx       # Filter UI
│   ├── ZoomControls.tsx         # Zoom/pan controls
│   ├── Minimap.tsx              # Overview minimap
│   ├── EventTooltip.tsx         # Hover tooltip
│   ├── ExportMenu.tsx           # Export options
│   └── SearchBar.tsx            # Search within timeline
├── hooks/
│   ├── useTimelineData.ts       # Data management
│   ├── useTimelineZoom.ts       # Zoom/pan logic
│   ├── useTimelineFilters.ts    # Filter state
│   └── useTimelineExport.ts     # Export logic
├── utils/
│   ├── timelineRenderer.ts      # D3.js rendering logic
│   ├── eventGrouping.ts         # Event grouping algorithms
│   ├── dateUtils.ts             # Date manipulation
│   └── exportFormats.ts         # Export formatters
└── styles/
    └── TimelineViewer.module.css
```

### State Management

```typescript
interface TimelineViewerState {
  // View state
  currentViewMode: 'timeline' | 'table' | 'compact'
  timeAxis: 'transaction' | 'valid' | 'both'

  // Data state
  events: TimelineEvent[]
  filteredEvents: TimelineEvent[]
  selectedEvent: TimelineEvent | null
  hoveredEvent: TimelineEvent | null

  // Zoom/pan state
  zoomLevel: number
  panOffset: number
  visibleDateRange: DateRange

  // Filter state
  filters: TimelineFilters

  // UI state
  isLoading: boolean
  error: Error | null
  showEventDetails: boolean
  showFilters: boolean

  // Export state
  exportInProgress: boolean
  exportFormat: ExportFormat | null
}
```

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      TimelineViewer                         │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Props (timelineEvents, filters, callbacks)        │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  useTimelineData Hook                              │    │
│  │  - Process raw events                              │    │
│  │  - Apply filters                                   │    │
│  │  - Group events (if enabled)                       │    │
│  │  - Calculate layout coordinates                    │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  View Mode Router                                  │    │
│  │  - Timeline View (D3.js)                           │    │
│  │  - Table View (React Table)                        │    │
│  │  - Compact View (List)                             │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  User Interactions                                 │    │
│  │  - Click event → onEventClick                      │    │
│  │  - Zoom/pan → update visibleDateRange              │    │
│  │  - Filter change → update filteredEvents           │    │
│  │  - Export → generate file                          │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## UI Components

### Timeline View (Horizontal)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ TimelineViewer: Transaction #txn_abc123                            [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  View: ● Timeline  ○ Table  ○ Compact    [🔍 Search]  [⚙ Filters]  [↓ Export]│
│                                                                             │
│  Legend: ● Extraction  ● Override  ● Correction  ● Recomputation           │
│          ● Validation  ● Revert                                            │
│                                                                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Transaction Time Axis                                                      │
│  ├────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┤ │
│  Jan 2024     Feb       Mar       Apr       May       Jun       Jul    Aug │
│                                                                             │
│  Amount Field:                                                              │
│  ──●─────────────●───────────────────────────●──────────────────────────  │
│     │             │                           │                             │
│   $100.00      $105.50                    $105.50                          │
│   (Extract)    (Override)                 (Revert)                         │
│                                                                             │
│  Date Field:                                                                │
│  ──────●────────────────────────────────────────────────────────────────  │
│         │                                                                   │
│    2024-01-15                                                               │
│    (Extract)                                                                │
│                                                                             │
│  Merchant Field:                                                            │
│  ──●─────────────────────────────────────────────────────────────────────  │
│     │                                                                       │
│  "Acme Corp"                                                                │
│  (Extract)                                                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │ Event Details                                          [X]  │          │
│  ├─────────────────────────────────────────────────────────────┤          │
│  │ Type:        Override                                        │          │
│  │ Timestamp:   2024-02-15 14:32:11 UTC                        │          │
│  │ Actor:       John Doe (john@example.com)                    │          │
│  │                                                              │          │
│  │ Field Changes:                                               │          │
│  │  • amount: $100.00 → $105.50                                │          │
│  │                                                              │          │
│  │ Reason:                                                      │          │
│  │  "Correcting OCR error - receipt shows $105.50"             │          │
│  │                                                              │          │
│  │ Confidence: 95%                                              │          │
│  └─────────────────────────────────────────────────────────────┘          │
│                                                                             │
│  Minimap:                                                                   │
│  [■■■■■■■■░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]                │
│   └──────── Current View ────────┘                                         │
│                                                                             │
│  Zoom: [−]  ═══●══════  [+]    Pan: [◄] [►]    Reset: [⊙]                │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Bitemporal View (Advanced)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ TimelineViewer: Patient Record #patient_45678 (Bitemporal)       [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Time Axis: ● Transaction Time  ○ Valid Time  ● Both (Bitemporal)         │
│                                                                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         Transaction Time (When Recorded) →                 │
│                    Jan 2024      Feb      Mar      Apr      May            │
│                    ├────────┼────────┼────────┼────────┼────────┤          │
│                    │                                                        │
│  Valid Time        │     ●──────────────────────────────────────           │
│  (When True)       │    ╱│                                                 │
│       ↓            │   ╱ │  Event 1: Diagnosis recorded on Feb 1           │
│                    │  ╱  │  Valid from Jan 15 (backdated)                  │
│  Jan 2024 ─────────┼─────┼────────────────────────────────────            │
│                    │     │                                                 │
│  Feb      ─────────┼─────●────────────────────────────────────            │
│                    │     │ Event 2: Diagnosis corrected on Feb 20          │
│                    │     │ Valid from Feb 20 (same day)                    │
│  Mar      ─────────┼─────┼────────●───────────────────────────            │
│                    │     │        │                                        │
│                    │     │        │ Event 3: Treatment date corrected      │
│                    │     │        │ on Mar 10, valid from Feb 28           │
│  Apr      ─────────┼─────┼────────┼───────────────────────────            │
│                    │     │        │                                        │
│                                                                             │
│  Legend:                                                                    │
│    ● Event point                                                            │
│    ── Horizontal line: Same transaction and valid time                     │
│    ╱  Diagonal line: Backdated (valid time before transaction time)        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Table View

```
┌────────────────────────────────────────────────────────────────────────────┐
│ TimelineViewer: Legal Case #case_789                             [-][□][X]│
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  View: ○ Timeline  ● Table  ○ Compact    [🔍 Search]  [⚙ Filters]  [↓ Export]│
│                                                                             │
│  Showing 47 events (Filtered from 152 total)                               │
│                                                                             │
├────┬──────────────┬─────────────────┬────────────────┬─────────────┬──────┤
│ #  │ Event Type   │ Transaction Time│ Actor          │ Fields      │ ···  │
├────┼──────────────┼─────────────────┼────────────────┼─────────────┼──────┤
│ 1  │ ● Extraction │ 2024-01-05      │ system_ocr     │ case_number │ [>]  │
│    │              │ 09:23:15        │                │ plaintiff   │      │
│    │              │                 │                │ defendant   │      │
├────┼──────────────┼─────────────────┼────────────────┼─────────────┼──────┤
│ 2  │ ● Override   │ 2024-01-05      │ Jane Smith     │ case_number │ [>]  │
│    │              │ 14:45:22        │ (legal_clerk)  │             │      │
├────┼──────────────┼─────────────────┼────────────────┼─────────────┼──────┤
│ 3  │ ● Correction │ 2024-02-10      │ John Doe       │ filing_date │ [>]  │
│    │              │ 11:12:33        │ (attorney)     │             │      │
├────┼──────────────┼─────────────────┼────────────────┼─────────────┼──────┤
│ 4  │ ● Validation │ 2024-02-10      │ system_        │ filing_date │ [>]  │
│    │              │ 11:12:35        │ validator      │ court_name  │      │
├────┼──────────────┼─────────────────┼────────────────┼─────────────┼──────┤
│... │              │                 │                │             │      │
├────┴──────────────┴─────────────────┴────────────────┴─────────────┴──────┤
│                                                                             │
│  Sort by: Transaction Time ▼   Group by: None ▼   Items per page: 25 ▼    │
│                                                                             │
│  [◄ Previous]  Page 1 of 2  [Next ►]                                       │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Compact View (Mobile)

```
┌──────────────────────────────────────┐
│ TimelineViewer                  [☰] │
├──────────────────────────────────────┤
│                                      │
│  Transaction #txn_abc123             │
│                                      │
│  [🔍 Search events...]               │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ ● Extraction                   │ │
│  │ Jan 5, 2024 9:23 AM            │ │
│  │ by system_ocr                  │ │
│  │                                │ │
│  │ Fields: amount, date, merchant │ │
│  │ Confidence: 87%                │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ ● Override                     │ │
│  │ Feb 15, 2024 2:32 PM           │ │
│  │ by John Doe                    │ │
│  │                                │ │
│  │ amount: $100.00 → $105.50      │ │
│  │                                │ │
│  │ Reason: "OCR error correction" │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ ● Correction                   │ │
│  │ Mar 20, 2024 10:15 AM          │ │
│  │ by system_recompute            │ │
│  │                                │ │
│  │ Fields: tax_amount             │ │
│  │ Caused by: Event #2            │ │
│  └────────────────────────────────┘ │
│                                      │
│  [Load More Events...]               │
│                                      │
│  Showing 3 of 47 events              │
│                                      │
└──────────────────────────────────────┘
```

### Filter Panel

```
┌────────────────────────────────────┐
│ Filters                       [X]  │
├────────────────────────────────────┤
│                                    │
│  Date Range:                       │
│  From: [2024-01-01 ▼]              │
│  To:   [2024-12-31 ▼]              │
│                                    │
│  Quick Presets:                    │
│  [Today] [Last 7 Days] [This Month]│
│  [This Quarter] [This Year] [All]  │
│                                    │
│  Event Types:                      │
│  ☑ Extraction (23)                 │
│  ☑ Override (12)                   │
│  ☑ Correction (8)                  │
│  ☑ Recomputation (15)              │
│  ☑ Validation (34)                 │
│  ☐ Merge (0)                       │
│  ☐ Split (0)                       │
│  ☑ Revert (2)                      │
│                                    │
│  Fields:                           │
│  ☑ amount (18)                     │
│  ☑ date (9)                        │
│  ☑ merchant (14)                   │
│  ☑ category (11)                   │
│  ☐ tax_amount (7)                  │
│                                    │
│  Actors:                           │
│  ☑ system_ocr (23)                 │
│  ☑ John Doe (8)                    │
│  ☑ Jane Smith (4)                  │
│  ☑ system_recompute (15)           │
│                                    │
│  Confidence:                       │
│  ├─────●─────┤                     │
│  50%      100%                     │
│                                    │
│  [Reset All]    [Apply Filters]    │
│                                    │
└────────────────────────────────────┘
```

### Export Menu

```
┌────────────────────────────────────┐
│ Export Timeline                    │
├────────────────────────────────────┤
│                                    │
│  Format:                           │
│  ○ PNG (Image)                     │
│  ○ SVG (Vector)                    │
│  ● CSV (Data)                      │
│  ○ JSON (Raw Data)                 │
│  ○ PDF (Report)                    │
│                                    │
│  Include:                          │
│  ☑ Filtered events only            │
│  ☑ Event details                   │
│  ☑ Actor information               │
│  ☑ Confidence scores               │
│  ☐ Metadata                        │
│                                    │
│  CSV Options:                      │
│  Date format: [YYYY-MM-DD ▼]       │
│  Delimiter:   [Comma ▼]            │
│                                    │
│  [Cancel]           [Export (47)]  │
│                                    │
└────────────────────────────────────┘
```

---

## Visualization Modes

### 1. Timeline View (D3.js)

**Purpose:** Visual exploration of temporal patterns

**Implementation:**
- Use D3.js for rendering SVG timeline
- Support horizontal and vertical orientations
- Implement zoom and pan with d3-zoom
- Use d3-scale for time axis scaling
- Implement brush selection for date range filtering

**Key Features:**
- **Swim lanes:** One lane per field or group by criteria
- **Event markers:** Circles, diamonds, or custom shapes
- **Connecting lines:** Show value continuity
- **Causality arrows:** Link related events (e.g., correction → recomputation)
- **Zoom levels:** Year/quarter/month/week/day/hour
- **Minimap:** Overview when zoomed in

**Rendering Strategy:**
```typescript
// Pseudo-code for D3.js timeline rendering
function renderTimeline(events: TimelineEvent[], config: TimelineConfig) {
  // 1. Create scales
  const xScale = d3.scaleTime()
    .domain([minDate, maxDate])
    .range([0, width])

  const yScale = d3.scalePoint()
    .domain(uniqueFields)
    .range([0, height])

  // 2. Create axes
  const xAxis = d3.axisBottom(xScale)
  const yAxis = d3.axisLeft(yScale)

  // 3. Render swim lanes
  svg.selectAll('.lane')
    .data(uniqueFields)
    .enter()
    .append('rect')
    .attr('class', 'lane')
    .attr('y', d => yScale(d))
    .attr('height', laneHeight)

  // 4. Render events
  svg.selectAll('.event')
    .data(events)
    .enter()
    .append('circle')
    .attr('cx', d => xScale(d.transactionTime))
    .attr('cy', d => yScale(d.field))
    .attr('r', 6)
    .attr('fill', d => getEventColor(d.eventType))
    .on('click', handleEventClick)
    .on('mouseenter', handleEventHover)

  // 5. Add zoom behavior
  const zoom = d3.zoom()
    .scaleExtent([0.5, 10])
    .on('zoom', (event) => {
      svg.attr('transform', event.transform)
    })

  svg.call(zoom)
}
```

### 2. Table View (React Table)

**Purpose:** Detailed data inspection and sorting

**Implementation:**
- Use TanStack Table (React Table v8) for sorting, filtering, pagination
- Support column resizing and reordering
- Implement virtual scrolling for large datasets (>1000 events)

**Columns:**
- **Event #:** Sequential number
- **Event Type:** Icon + label with color coding
- **Transaction Time:** Sortable timestamp
- **Valid Time:** Sortable timestamp (if bitemporal)
- **Actor:** User/system name with avatar
- **Fields:** Affected field names (truncated with tooltip)
- **Confidence:** Progress bar (0-100%)
- **Reason:** Truncated text with "Read more"
- **Actions:** Expand details, copy ID, navigate to related events

**Sorting:**
- Default: Transaction time descending (newest first)
- Support multi-column sorting (shift + click)
- Persist sort state in URL query params

**Pagination:**
- Default: 25 events per page
- Options: 10, 25, 50, 100, All
- Show total count and current range

### 3. Compact View (Mobile List)

**Purpose:** Mobile-friendly linear timeline

**Implementation:**
- Infinite scroll with lazy loading
- Swipe gestures for quick actions (e.g., swipe right to expand)
- Condensed cards with expandable details

**Card Structure:**
```
┌─────────────────────────────────┐
│ ● Event Type                    │
│ Timestamp                       │
│ by Actor                        │
│                                 │
│ Field changes (summary)         │
│ Confidence: XX%                 │
│                                 │
│ [Tap to expand]                 │
└─────────────────────────────────┘
```

**Optimization:**
- Render only visible cards (virtual scrolling)
- Lazy load images and avatars
- Use CSS transforms for smooth animations
- Debounce search input

---

## Interaction Patterns

### Event Click Interaction

**Scenario:** User clicks on a timeline event

**Flow:**
1. Event marker highlights (border glow)
2. Event details panel slides in from right
3. Panel shows:
   - Event type with icon
   - Timestamp (transaction + valid time)
   - Actor with avatar
   - Field changes (before/after comparison)
   - Reason (if provided)
   - Confidence score
   - Related events (caused by, caused)
4. User can:
   - Navigate to related events (click "Caused by: Event #X")
   - Copy event ID
   - See full metadata (expandable section)
   - Close panel (click X or ESC key)

**Keyboard Shortcuts:**
- `Enter`: Open details for selected event
- `ESC`: Close details panel
- `←/→`: Navigate to previous/next event
- `/`: Focus search bar

### Zoom & Pan Interaction

**Scenario:** User wants to explore a specific time period

**Zoom Methods:**
1. **Mouse wheel:** Zoom in/out at cursor position
2. **Pinch gesture:** Zoom on touch devices
3. **Zoom controls:** Click +/- buttons
4. **Brush selection:** Drag on minimap to zoom to range

**Pan Methods:**
1. **Click & drag:** Pan timeline left/right
2. **Keyboard:** Arrow keys for fine-grained panning
3. **Pan controls:** Click ◄/► buttons
4. **Minimap:** Drag viewport rectangle

**Constraints:**
- Min zoom: Fit all events in viewport
- Max zoom: Show 1 hour of time
- Pan bounds: Cannot pan beyond first/last event

### Filter Interaction

**Scenario:** User wants to see only specific event types

**Flow:**
1. Click "Filters" button
2. Filter panel slides in from left
3. User toggles checkboxes for event types, fields, actors
4. Filters apply in real-time (debounced 300ms)
5. Event counts update in legend and filter labels
6. Timeline re-renders with filtered events
7. "Reset All" button clears all filters

**Filter Persistence:**
- Filters saved to URL query params
- Shareable URLs with pre-applied filters
- Local storage for user preferences

### Search Interaction

**Scenario:** User searches for "invoice"

**Flow:**
1. User types in search bar
2. Search applies to:
   - Event reasons
   - Field names
   - Actor names
   - Field values (if indexed)
3. Matching events highlight
4. Non-matching events dim (opacity 0.3)
5. Search results count shows: "5 results"
6. User can navigate results with Enter/Shift+Enter

**Search Syntax:**
- Plain text: Case-insensitive substring match
- `field:value`: Search specific field
- `actor:john`: Search by actor
- `type:correction`: Search by event type
- `"exact phrase"`: Exact match

### Export Interaction

**Scenario:** User exports timeline as CSV

**Flow:**
1. Click "Export" button
2. Export menu opens
3. User selects format (CSV)
4. User configures options (date format, delimiter)
5. Click "Export"
6. Browser downloads file: `timeline_txn_abc123_2024-10-25.csv`

**CSV Format:**
```csv
Event ID,Event Type,Transaction Time,Valid Time,Actor,Actor Type,Fields,Confidence,Reason
evt_001,extraction,2024-01-05T09:23:15Z,,system_ocr,system,"amount,date,merchant",0.87,
evt_002,override,2024-02-15T14:32:11Z,,john@example.com,user,amount,0.95,"Correcting OCR error"
```

---

## Multi-Domain Examples

### 1. Financial Services: Transaction Provenance

**Use Case:** Track the complete history of a financial transaction

**Entity Type:** `transaction`

**Timeline Events:**
1. **Extraction** (2024-01-05 09:23:15) - system_ocr extracts transaction from bank statement PDF
   - Fields: amount=$100.00, date=2024-01-05, merchant="ACME CORP", category=null
   - Confidence: 87%

2. **Override** (2024-01-05 14:30:22) - user_jane corrects merchant name
   - Fields: merchant="ACME CORP" → "Acme Corporation"
   - Reason: "Standardizing merchant names"
   - Confidence: 100%

3. **Recomputation** (2024-01-05 14:30:23) - system_rules recalculates category
   - Fields: category=null → "Office Supplies"
   - Caused by: evt_002 (merchant override)
   - Confidence: 92%

4. **Correction** (2024-02-15 10:15:00) - user_john backdates the transaction
   - Fields: date=2024-01-05 → 2024-01-03
   - Valid time: 2024-01-03 (backdated)
   - Reason: "Receipt shows purchase date as Jan 3"
   - Confidence: 100%

5. **Validation** (2024-02-15 10:15:01) - system_validator checks date consistency
   - Fields: date (validated against receipt timestamp)
   - Confidence: 98%

**Timeline Visualization:**
```
Amount:    ●────────────────────────────────────────────
           $100.00 (Extract)

Merchant:  ●─────●──────────────────────────────────────
           ACME   Acme Corporation (Override)

Category:  ──────●──────────────────────────────────────
                 Office Supplies (Recomputation)

Date:      ●─────────────────●────────────────────────
           Jan 5              Jan 3 (Corrected, backdated)
```

**Value Proposition:**
- **Audit compliance:** Complete audit trail for regulatory reporting
- **Dispute resolution:** Trace back to original extraction if customer disputes
- **Quality improvement:** Identify systematic OCR errors for merchant names

### 2. Healthcare: Patient Record Corrections

**Use Case:** Track corrections to patient medical records

**Entity Type:** `patient_record`

**Timeline Events:**
1. **Import** (2024-01-10 08:00:00) - system_migration imports legacy record
   - Fields: diagnosis="Hypertension", diagnosis_date=2023-05-15, provider="Dr. Smith"
   - Confidence: 75% (legacy data quality uncertain)

2. **Validation** (2024-01-10 08:00:01) - system_validator flags date inconsistency
   - Fields: diagnosis_date (flagged: before patient admission date)
   - Severity: warning

3. **Correction** (2024-01-11 14:20:00) - user_dr_jones corrects diagnosis date
   - Fields: diagnosis_date=2023-05-15 → 2023-06-20
   - Valid time: 2023-06-20 (backdated to actual diagnosis date)
   - Reason: "Correcting data entry error from legacy system"
   - Confidence: 100%

4. **Override** (2024-02-05 09:15:00) - user_dr_jones updates diagnosis
   - Fields: diagnosis="Hypertension" → "Stage 2 Hypertension"
   - Reason: "Adding specificity based on recent labs"
   - Confidence: 100%

5. **Merge** (2024-03-01 16:30:00) - system_dedupe merges duplicate records
   - Fields: merged_from=[record_456, record_789]
   - Reason: "Duplicate records from multiple clinics identified"

**Timeline Visualization (Bitemporal):**
```
Transaction Time (When Recorded) →
      Jan 10      Jan 11      Feb 5       Mar 1
         │           │          │          │
Valid    │   Import  │          │          │
Time     │      ●────●──────────┼──────────┼──── Diagnosis Date
         │     ╱     │Correction│          │
         │    ╱      │          │          │
May 2023 ┼───────────┼──────────┼──────────┼────
         │           │          │          │
Jun 2023 ┼───────────●──────────┼──────────┼──── (Corrected to June 20)
         │           │          │          │
         │           │          ●          │
         │           │     Override        │     (Stage 2 Hypertension)
```

**Value Proposition:**
- **Clinical accuracy:** Ensure medical history reflects true diagnosis dates
- **Legal compliance:** Meet HIPAA audit requirements for record changes
- **Research integrity:** Enable accurate retrospective studies with corrected dates

### 3. Legal: Case Document Timeline

**Use Case:** Track document submissions and amendments in legal case

**Entity Type:** `legal_case_document`

**Timeline Events:**
1. **Extraction** (2024-01-15 09:00:00) - system_ocr extracts filing from court document
   - Fields: filing_date=2024-01-15, case_number="CV-2024-0123", document_type="Complaint"
   - Confidence: 91%

2. **Override** (2024-01-15 11:30:00) - user_paralegal corrects case number
   - Fields: case_number="CV-2024-0123" → "CV-2024-0132"
   - Reason: "OCR misread last digit"
   - Confidence: 100%

3. **Correction** (2024-02-01 14:00:00) - user_attorney backdates filing
   - Fields: filing_date=2024-01-15 → 2024-01-12
   - Valid time: 2024-01-12 (backdated to actual court stamped date)
   - Reason: "Court stamp shows Jan 12, not Jan 15"
   - Confidence: 100%

4. **Validation** (2024-02-01 14:00:01) - system_validator checks date against statute of limitations
   - Fields: filing_date (validated)
   - Confidence: 100%

5. **Split** (2024-03-10 10:00:00) - user_attorney splits document into multiple filings
   - Fields: split_into=[doc_890, doc_891]
   - Reason: "Separating Complaint and Motion to Dismiss"

**Timeline Visualization:**
```
Filing Date:  ●────────────────●──────────────────────
              Jan 15            Jan 12 (Corrected)
                               (Backdated to court stamp)

Case Number:  ●───●────────────────────────────────────
              0123  0132 (Corrected)

Document:     ●──────────────────────────────────●─────
              Complaint                          Split
                                                 (2 docs)
```

**Value Proposition:**
- **Court compliance:** Accurate filing dates critical for statute of limitations
- **Case management:** Track document evolution for case strategy
- **Billing accuracy:** Audit trail for attorney time tracking

### 4. Research: Dataset Provenance

**Use Case:** Track corrections and versions in research dataset

**Entity Type:** `research_dataset_record`

**Timeline Events:**
1. **Extraction** (2024-01-20 10:00:00) - system_etl ingests data from external API
   - Fields: participant_id="P001", measurement=120.5, measurement_date=2024-01-15
   - Confidence: 95%

2. **Validation** (2024-01-20 10:00:01) - system_validator checks measurement range
   - Fields: measurement (out of expected range 60-100)
   - Severity: warning

3. **Override** (2024-01-22 09:30:00) - user_researcher corrects measurement
   - Fields: measurement=120.5 → 12.05
   - Reason: "Decimal point error in source data"
   - Confidence: 100%

4. **Recomputation** (2024-01-22 09:30:01) - system_stats recalculates derived metrics
   - Fields: z_score=-0.5 → 1.2
   - Caused by: evt_003 (measurement correction)
   - Confidence: 100%

5. **Correction** (2024-02-10 11:00:00) - user_researcher backdates measurement
   - Fields: measurement_date=2024-01-15 → 2024-01-10
   - Valid time: 2024-01-10 (backdated to actual lab date)
   - Reason: "Lab report shows specimen collected on Jan 10"
   - Confidence: 100%

**Timeline Visualization:**
```
Measurement:  ●────●──────────────────────────────────
              120.5  12.05 (Corrected)

Z-Score:      ────●───●───────────────────────────────
              -0.5  1.2 (Recomputed)

Lab Date:     ●─────────────●─────────────────────────
              Jan 15        Jan 10 (Corrected)
```

**Value Proposition:**
- **Research integrity:** Document all data corrections for publication
- **Reproducibility:** Enable exact replication of analysis with version history
- **Peer review:** Transparent provenance for reviewer scrutiny

### 5. E-commerce: Order History

**Use Case:** Track order modifications and fulfillment

**Entity Type:** `order`

**Timeline Events:**
1. **Extraction** (2024-03-01 14:30:00) - system_checkout creates order from cart
   - Fields: total=$99.99, shipping_address="123 Main St", status="pending"
   - Confidence: 100%

2. **Override** (2024-03-01 14:35:00) - user_customer modifies shipping address
   - Fields: shipping_address="123 Main St" → "456 Oak Ave"
   - Reason: "Customer requested address change"
   - Confidence: 100%

3. **Recomputation** (2024-03-01 14:35:01) - system_shipping recalculates shipping cost
   - Fields: shipping_cost=$5.99 → $7.99, total=$99.99 → $101.99
   - Caused by: evt_002 (address change)
   - Confidence: 100%

4. **Override** (2024-03-02 09:00:00) - user_support applies discount code
   - Fields: discount=$0 → $10.00, total=$101.99 → $91.99
   - Reason: "Customer service goodwill gesture"
   - Confidence: 100%

5. **Validation** (2024-03-02 10:00:00) - system_fraud checks order for fraud signals
   - Fields: fraud_score=0.02 (low risk)
   - Confidence: 97%

6. **Override** (2024-03-03 08:00:00) - system_fulfillment updates status
   - Fields: status="pending" → "shipped", tracking_number="1Z999AA10123456784"
   - Confidence: 100%

**Timeline Visualization:**
```
Shipping Address:  ●───●───────────────────────────────────
                   Main  Oak (Modified by customer)

Total:             ●───────●───────────●─────────────────
                   $99.99  $101.99     $91.99
                           (Shipping)  (Discount)

Status:            ●────────────────────────────●────────
                   Pending                      Shipped
```

**Value Proposition:**
- **Customer service:** Quick resolution of "What happened to my order?" questions
- **Fraud investigation:** Audit trail for suspicious order modifications
- **Business intelligence:** Analyze patterns in order changes (e.g., address changes)

### 6. SaaS: Subscription Changes

**Use Case:** Track subscription plan changes and billing

**Entity Type:** `subscription`

**Timeline Events:**
1. **Extraction** (2024-01-01 00:00:00) - system_billing creates subscription from signup
   - Fields: plan="starter", price=$29/mo, seats=1, status="active"
   - Confidence: 100%

2. **Override** (2024-02-15 16:20:00) - user_customer upgrades plan
   - Fields: plan="starter" → "professional", price=$29/mo → $99/mo, seats=1 → 5
   - Reason: "Customer upgraded via self-service portal"
   - Confidence: 100%

3. **Recomputation** (2024-02-15 16:20:01) - system_billing calculates prorated charge
   - Fields: prorated_charge=$52.50 (14 days remaining in month)
   - Caused by: evt_002 (plan upgrade)
   - Confidence: 100%

4. **Override** (2024-03-10 10:00:00) - user_support adds seats
   - Fields: seats=5 → 8, price=$99/mo → $129/mo
   - Reason: "Customer added 3 team members"
   - Confidence: 100%

5. **Correction** (2024-03-12 09:00:00) - user_support backdates seat addition
   - Fields: seat_addition_date=2024-03-10 → 2024-03-01
   - Valid time: 2024-03-01 (backdated to when employees actually started)
   - Reason: "Customer contacted support late, but employees started March 1"
   - Confidence: 100%

6. **Recomputation** (2024-03-12 09:00:01) - system_billing recalculates prorated charge
   - Fields: prorated_charge=$18 → $58 (more days charged)
   - Caused by: evt_005 (backdated seat addition)
   - Confidence: 100%

**Timeline Visualization (Bitemporal):**
```
Transaction Time (When Recorded) →
      Jan 1      Feb 15     Mar 10      Mar 12
         │          │          │           │
Valid    │  Signup  │          │           │
Time     │     ●────●──────────┼───────────┼──── Plan: Starter → Pro
         │          │          │           │
         │          │          ●───────────┼──── Seats: 5 → 8
         │          │         ╱│           │
         │          │        ╱ │           │
Mar 1    ┼──────────┼───────●──┼───────────┼──── (Backdated seat addition)
         │          │          │           │
         │          │          │           ●──── (Recomputed prorated charge)
```

**Value Proposition:**
- **Billing accuracy:** Correct prorated charges when backdating seat additions
- **Customer trust:** Transparent billing history resolves disputes
- **Revenue operations:** Analyze upgrade/downgrade patterns

### 7. Insurance: Claim Processing

**Use Case:** Track claim status changes and adjudication

**Entity Type:** `insurance_claim`

**Timeline Events:**
1. **Extraction** (2024-01-15 09:00:00) - system_ocr extracts claim from submitted form
   - Fields: claim_amount=$5,000, incident_date=2024-01-10, diagnosis="Broken arm"
   - Confidence: 89%

2. **Validation** (2024-01-15 09:00:01) - system_rules validates claim against policy
   - Fields: policy_coverage=true, deductible_met=false
   - Confidence: 100%

3. **Override** (2024-01-16 10:30:00) - user_adjuster corrects claim amount
   - Fields: claim_amount=$5,000 → $4,200
   - Reason: "Adjusted to in-network rates"
   - Confidence: 100%

4. **Recomputation** (2024-01-16 10:30:01) - system_billing recalculates patient responsibility
   - Fields: patient_responsibility=$1,000 → $840
   - Caused by: evt_003 (claim amount adjustment)
   - Confidence: 100%

5. **Correction** (2024-01-20 14:00:00) - user_adjuster backdates incident
   - Fields: incident_date=2024-01-10 → 2024-01-08
   - Valid time: 2024-01-08 (backdated to actual ER visit)
   - Reason: "Hospital records show ER visit on Jan 8, not Jan 10"
   - Confidence: 100%

6. **Validation** (2024-01-20 14:00:01) - system_rules re-validates policy coverage
   - Fields: policy_coverage=true (still covered)
   - Caused by: evt_005 (incident date change)
   - Confidence: 100%

7. **Override** (2024-01-25 16:00:00) - user_manager approves claim
   - Fields: status="pending" → "approved", approved_amount=$4,200
   - Reason: "All documentation verified"
   - Confidence: 100%

**Timeline Visualization:**
```
Claim Amount:    ●─────●─────────────────────────────────
                 $5K   $4.2K (Adjusted to in-network)

Incident Date:   ●───────────────●───────────────────────
                 Jan 10          Jan 8 (Corrected)

Status:          ●───────────────────────────────●───────
                 Pending                         Approved
```

**Value Proposition:**
- **Regulatory compliance:** Complete audit trail for claims processing
- **Fraud detection:** Identify patterns of suspicious backdated incidents
- **Customer service:** Quickly explain claim adjustments to policyholders

---

## Edge Cases & Error Handling

### Empty Timeline

**Scenario:** No events exist for the entity

**Handling:**
- Display empty state message: "No timeline events found"
- Show illustration (empty timeline graphic)
- Provide guidance: "Events will appear as data is extracted and edited"
- Hide filters, zoom controls, export button

**Implementation:**
```typescript
if (timelineEvents.length === 0) {
  return (
    <EmptyState
      icon={<TimelineIcon />}
      title="No timeline events"
      description="Events will appear as data is extracted and edited"
    />
  )
}
```

### Very Large Timeline (>10,000 Events)

**Scenario:** Entity has thousands of events (e.g., high-frequency trading transaction)

**Handling:**
- **Performance mode:** Auto-enable optimized rendering
  - Use virtualization (only render visible events)
  - Aggregate events by time period (e.g., show daily summaries instead of individual events)
  - Lazy load event details on demand
- **UI adaptations:**
  - Show aggregated event counts on timeline (e.g., "127 events on Jan 15")
  - Click to expand and see individual events
  - Default to table view (more efficient for large datasets)
- **Warnings:**
  - Show banner: "Large timeline detected (10,234 events). Some features optimized for performance."

**Implementation:**
```typescript
const performanceMode = useMemo(() => {
  if (props.performanceMode === 'auto') {
    return timelineEvents.length > 1000 ? 'optimized' : 'standard'
  }
  return props.performanceMode
}, [timelineEvents.length, props.performanceMode])

if (performanceMode === 'optimized') {
  // Use event aggregation
  const aggregatedEvents = aggregateEventsByDay(timelineEvents)
  return <OptimizedTimelineView events={aggregatedEvents} />
}
```

### Bitemporal Complexity

**Scenario:** Event has very different transaction and valid times (e.g., backdated 5 years)

**Handling:**
- **Visualization:**
  - Use diagonal line to connect transaction time and valid time
  - Add warning icon if gap > 30 days
  - Tooltip explains: "This event was backdated 5 years 3 months"
- **Filtering:**
  - Allow filter by both time dimensions independently
  - "Show events recorded in Q1 2024" vs "Show events valid in Q1 2024"

**Implementation:**
```typescript
function renderBitemporalEvent(event: TimelineEvent) {
  const txTime = new Date(event.transactionTime)
  const validTime = event.validTime ? new Date(event.validTime) : txTime
  const daysDiff = Math.abs((txTime.getTime() - validTime.getTime()) / (1000 * 60 * 60 * 24))

  const showWarning = daysDiff > 30

  return (
    <>
      {/* Draw diagonal line if backdated */}
      {validTime < txTime && (
        <line
          x1={xScale(validTime)}
          y1={yScale(event.field)}
          x2={xScale(txTime)}
          y2={yScale(event.field) - 20}
          stroke="#F59E0B"
          strokeDasharray="4"
        />
      )}

      {/* Event marker */}
      <circle
        cx={xScale(txTime)}
        cy={yScale(event.field)}
        r={6}
        fill={getEventColor(event.eventType)}
      />

      {/* Warning icon */}
      {showWarning && (
        <WarningIcon
          x={xScale(txTime) + 10}
          y={yScale(event.field) - 10}
          tooltip={`Backdated ${daysDiff} days`}
        />
      )}
    </>
  )
}
```

### Causality Cycles

**Scenario:** Event A caused Event B, which caused Event C, which somehow references Event A (cycle)

**Handling:**
- **Detection:** Detect cycles in `causedBy` graph
- **Visualization:** Break cycle visually, show warning icon
- **Error message:** "Causality cycle detected: Event A → B → C → A"
- **Prevention:** Validation in backend to prevent cycle creation

**Implementation:**
```typescript
function detectCausalityCycles(events: TimelineEvent[]): string[][] {
  const graph: Map<string, string[]> = new Map()

  events.forEach(event => {
    if (event.causedBy) {
      if (!graph.has(event.causedBy)) {
        graph.set(event.causedBy, [])
      }
      graph.get(event.causedBy)!.push(event.id)
    }
  })

  const cycles: string[][] = []
  const visited = new Set<string>()
  const recStack = new Set<string>()

  function dfs(nodeId: string, path: string[]): void {
    visited.add(nodeId)
    recStack.add(nodeId)
    path.push(nodeId)

    const neighbors = graph.get(nodeId) || []
    for (const neighbor of neighbors) {
      if (!visited.has(neighbor)) {
        dfs(neighbor, path)
      } else if (recStack.has(neighbor)) {
        // Cycle detected
        const cycleStartIndex = path.indexOf(neighbor)
        cycles.push(path.slice(cycleStartIndex))
      }
    }

    recStack.delete(nodeId)
    path.pop()
  }

  events.forEach(event => {
    if (!visited.has(event.id)) {
      dfs(event.id, [])
    }
  })

  return cycles
}
```

### Network Failure During Lazy Load

**Scenario:** User zooms in, triggering lazy load, but network request fails

**Handling:**
- **Retry logic:** Automatic retry with exponential backoff (3 attempts)
- **Error UI:** Show inline error in timeline: "Failed to load events. [Retry]"
- **Graceful degradation:** Continue showing already-loaded events
- **Offline mode:** If offline, show message: "Offline. Some events may not be visible."

**Implementation:**
```typescript
async function fetchEventsWithRetry(
  dateRange: DateRange,
  maxRetries = 3
): Promise<TimelineEvent[]> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const events = await onFetchEvents(dateRange)
      return events
    } catch (error) {
      if (attempt === maxRetries) {
        setError(new Error('Failed to load events after 3 attempts'))
        throw error
      }

      // Exponential backoff: 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, attempt - 1)))
    }
  }

  return []
}
```

### Overlapping Events (Same Timestamp)

**Scenario:** Multiple events recorded at exact same timestamp

**Handling:**
- **Visualization:** Stack events vertically with slight offset
- **Grouping:** Show event count badge (e.g., "3 events")
- **Interaction:** Click to expand and see individual events in list
- **Sorting:** Secondary sort by event ID for deterministic ordering

**Implementation:**
```typescript
function layoutOverlappingEvents(events: TimelineEvent[]): EventLayout[] {
  const layouts: EventLayout[] = []
  const eventsByTime = groupBy(events, e => e.transactionTime)

  Object.entries(eventsByTime).forEach(([timestamp, eventsAtTime]) => {
    if (eventsAtTime.length === 1) {
      layouts.push({ event: eventsAtTime[0], offset: 0 })
    } else {
      // Stack vertically with 15px offset
      eventsAtTime.forEach((event, index) => {
        layouts.push({ event, offset: index * 15 })
      })
    }
  })

  return layouts
}
```

### Missing Actor Information

**Scenario:** Event has no actor (orphaned event from data migration)

**Handling:**
- **Display:** Show placeholder: "Unknown Actor"
- **Icon:** Use generic system icon
- **Tooltip:** "Actor information not available (legacy data)"
- **Filtering:** Group under "Unknown" in actor filter

**Implementation:**
```typescript
function getActorDisplay(actor: Actor | undefined): ActorDisplay {
  if (!actor) {
    return {
      name: 'Unknown Actor',
      icon: <SystemIcon />,
      tooltip: 'Actor information not available (legacy data)'
    }
  }

  return {
    name: actor.name,
    icon: actor.type === 'user' ? <UserIcon /> : <SystemIcon />,
    tooltip: actor.email || actor.component
  }
}
```

### Export Failure (File Too Large)

**Scenario:** User tries to export 50,000 events as PNG

**Handling:**
- **Size limit:** Warn if export > 10MB estimated size
- **Recommendation:** Suggest filtering or CSV format
- **Chunking:** For CSV/JSON, offer to split into multiple files
- **Error message:** "Export too large (estimated 50MB). Try filtering events or use CSV format."

**Implementation:**
```typescript
function estimateExportSize(
  events: TimelineEvent[],
  format: ExportFormat
): number {
  const avgEventSize = {
    png: 500,    // 500 bytes per event (approximate)
    svg: 1000,   // 1KB per event
    csv: 200,    // 200 bytes per event
    json: 500,   // 500 bytes per event
    pdf: 800     // 800 bytes per event
  }

  return events.length * avgEventSize[format]
}

async function exportTimeline(
  events: TimelineEvent[],
  format: ExportFormat
): Promise<void> {
  const estimatedSize = estimateExportSize(events, format)
  const maxSize = 10 * 1024 * 1024 // 10MB

  if (estimatedSize > maxSize) {
    const shouldContinue = await confirm(
      `Export is estimated to be ${formatBytes(estimatedSize)}. ` +
      `This may take a while and could fail. Continue?`
    )

    if (!shouldContinue) return
  }

  // Proceed with export...
}
```

---

## Implementation Guide

### Installation

```bash
npm install d3 d3-zoom d3-scale react-table date-fns
npm install --save-dev @types/d3
```

### Basic Usage

```typescript
import { TimelineViewer } from '@/components/TimelineViewer'

function MyComponent() {
  const timelineEvents: TimelineEvent[] = [
    {
      id: 'evt_001',
      eventType: 'extraction',
      transactionTime: '2024-01-05T09:23:15Z',
      actor: { id: 'sys_ocr', type: 'system', name: 'OCR System' },
      affectedFields: [
        {
          fieldName: 'amount',
          oldValue: null,
          newValue: 100.00,
          fieldType: 'number',
          confidence: 0.87
        }
      ]
    },
    // ... more events
  ]

  return (
    <TimelineViewer
      entityId="txn_abc123"
      entityType="transaction"
      timelineEvents={timelineEvents}
      viewMode="timeline"
      onEventClick={(event) => console.log('Clicked:', event)}
    />
  )
}
```

### Advanced Usage with Lazy Loading

```typescript
function MyComponent() {
  const [events, setEvents] = useState<TimelineEvent[]>([])
  const [isLoading, setIsLoading] = useState(false)

  async function fetchEvents(dateRange: DateRange): Promise<TimelineEvent[]> {
    setIsLoading(true)
    try {
      const response = await fetch(
        `/api/entities/txn_abc123/timeline?start=${dateRange.start}&end=${dateRange.end}`
      )
      const newEvents = await response.json()
      return newEvents
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <TimelineViewer
      entityId="txn_abc123"
      entityType="transaction"
      timelineEvents={events}
      lazyLoad={true}
      onFetchEvents={fetchEvents}
      isLoading={isLoading}
      bitemporalView={true}
      showCausality={true}
      performanceMode="auto"
    />
  )
}
```

### Custom Event Renderer

```typescript
function MyComponent() {
  const customRenderer = (event: TimelineEvent) => {
    if (event.eventType === 'custom') {
      return (
        <div className="custom-event">
          <CustomIcon />
          <span>{event.metadata?.customLabel}</span>
        </div>
      )
    }
    return null // Use default renderer
  }

  return (
    <TimelineViewer
      entityId="txn_abc123"
      entityType="transaction"
      timelineEvents={events}
      customEventRenderer={customRenderer}
    />
  )
}
```

### D3.js Timeline Implementation

```typescript
// timelineRenderer.ts
import * as d3 from 'd3'

export function renderD3Timeline(
  svgRef: SVGSVGElement,
  events: TimelineEvent[],
  config: TimelineConfig
) {
  const svg = d3.select(svgRef)
  const width = config.width
  const height = config.height
  const margin = { top: 40, right: 40, bottom: 60, left: 100 }

  // Clear previous render
  svg.selectAll('*').remove()

  // Create scales
  const xScale = d3.scaleTime()
    .domain([
      d3.min(events, d => new Date(d.transactionTime))!,
      d3.max(events, d => new Date(d.transactionTime))!
    ])
    .range([margin.left, width - margin.right])

  const fields = Array.from(new Set(events.flatMap(e => e.affectedFields.map(f => f.fieldName))))
  const yScale = d3.scalePoint()
    .domain(fields)
    .range([margin.top, height - margin.bottom])
    .padding(0.5)

  // Create axes
  const xAxis = d3.axisBottom(xScale)
    .ticks(10)
    .tickFormat(d3.timeFormat('%b %d'))

  const yAxis = d3.axisLeft(yScale)

  svg.append('g')
    .attr('class', 'x-axis')
    .attr('transform', `translate(0,${height - margin.bottom})`)
    .call(xAxis)

  svg.append('g')
    .attr('class', 'y-axis')
    .attr('transform', `translate(${margin.left},0)`)
    .call(yAxis)

  // Render swim lanes
  svg.append('g')
    .attr('class', 'lanes')
    .selectAll('rect')
    .data(fields)
    .enter()
    .append('rect')
    .attr('x', margin.left)
    .attr('y', d => yScale(d)! - 15)
    .attr('width', width - margin.left - margin.right)
    .attr('height', 30)
    .attr('fill', '#f9fafb')
    .attr('stroke', '#e5e7eb')

  // Render events
  const eventGroups = svg.append('g')
    .attr('class', 'events')
    .selectAll('g')
    .data(events)
    .enter()
    .append('g')
    .attr('class', 'event-group')

  // Event circles
  eventGroups.append('circle')
    .attr('cx', d => xScale(new Date(d.transactionTime)))
    .attr('cy', d => {
      const field = d.affectedFields[0]?.fieldName
      return yScale(field) || 0
    })
    .attr('r', 8)
    .attr('fill', d => config.colorScheme[d.eventType])
    .attr('stroke', '#fff')
    .attr('stroke-width', 2)
    .style('cursor', 'pointer')
    .on('click', (event, d) => config.onEventClick?.(d))
    .on('mouseenter', (event, d) => {
      d3.select(event.currentTarget)
        .attr('r', 12)
        .attr('stroke-width', 3)
      config.onEventHover?.(d)
    })
    .on('mouseleave', (event, d) => {
      d3.select(event.currentTarget)
        .attr('r', 8)
        .attr('stroke-width', 2)
      config.onEventHover?.(null)
    })

  // Add zoom behavior
  const zoom = d3.zoom<SVGSVGElement, unknown>()
    .scaleExtent([0.5, 10])
    .on('zoom', (event) => {
      const transform = event.transform

      // Update x scale
      const newXScale = transform.rescaleX(xScale)

      // Re-render axes
      svg.select<SVGGElement>('.x-axis')
        .call(d3.axisBottom(newXScale).tickFormat(d3.timeFormat('%b %d')))

      // Re-position events
      svg.selectAll<SVGCircleElement, TimelineEvent>('.event-group circle')
        .attr('cx', d => newXScale(new Date(d.transactionTime)))
    })

  svg.call(zoom)
}
```

---

## Testing Strategy

### Unit Tests

```typescript
describe('TimelineViewer', () => {
  it('renders timeline view by default', () => {
    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
      />
    )

    expect(container.querySelector('.timeline-view')).toBeInTheDocument()
  })

  it('switches to table view when mode changed', async () => {
    const { getByRole } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
      />
    )

    const tableViewButton = getByRole('radio', { name: /table/i })
    await userEvent.click(tableViewButton)

    expect(document.querySelector('.table-view')).toBeInTheDocument()
  })

  it('calls onEventClick when event clicked', async () => {
    const onEventClick = jest.fn()
    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
        onEventClick={onEventClick}
      />
    )

    const firstEvent = container.querySelector('.event-marker')
    await userEvent.click(firstEvent!)

    expect(onEventClick).toHaveBeenCalledWith(mockEvents[0])
  })

  it('filters events by event type', async () => {
    const { getByRole, getAllByTestId } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
      />
    )

    // Open filters
    const filtersButton = getByRole('button', { name: /filters/i })
    await userEvent.click(filtersButton)

    // Uncheck 'extraction' events
    const extractionCheckbox = getByRole('checkbox', { name: /extraction/i })
    await userEvent.click(extractionCheckbox)

    // Verify only non-extraction events shown
    const visibleEvents = getAllByTestId('timeline-event')
    expect(visibleEvents).toHaveLength(3) // Assuming 3 non-extraction events
  })

  it('handles empty timeline gracefully', () => {
    const { getByText } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={[]}
      />
    )

    expect(getByText(/no timeline events/i)).toBeInTheDocument()
  })

  it('enables performance mode for large timelines', () => {
    const largeTimeline = Array.from({ length: 2000 }, (_, i) => ({
      id: `evt_${i}`,
      eventType: 'extraction' as const,
      transactionTime: new Date(2024, 0, 1 + i).toISOString(),
      actor: { id: 'sys', type: 'system' as const, name: 'System' },
      affectedFields: []
    }))

    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={largeTimeline}
        performanceMode="auto"
      />
    )

    expect(container.querySelector('.optimized-view')).toBeInTheDocument()
  })
})
```

### Integration Tests

```typescript
describe('TimelineViewer Integration', () => {
  it('fetches and displays lazy-loaded events on zoom', async () => {
    const mockFetchEvents = jest.fn().mockResolvedValue([
      { id: 'evt_100', /* ... */ }
    ])

    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
        lazyLoad={true}
        onFetchEvents={mockFetchEvents}
      />
    )

    // Simulate zoom in
    const zoomInButton = container.querySelector('.zoom-in')
    await userEvent.click(zoomInButton!)

    // Wait for fetch
    await waitFor(() => {
      expect(mockFetchEvents).toHaveBeenCalled()
    })

    // Verify new events rendered
    expect(container.querySelector('[data-event-id="evt_100"]')).toBeInTheDocument()
  })

  it('exports timeline as CSV', async () => {
    global.URL.createObjectURL = jest.fn(() => 'blob:mock-url')
    const downloadSpy = jest.spyOn(document, 'createElement')

    const { getByRole } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
      />
    )

    // Open export menu
    const exportButton = getByRole('button', { name: /export/i })
    await userEvent.click(exportButton)

    // Select CSV format
    const csvOption = getByRole('radio', { name: /csv/i })
    await userEvent.click(csvOption)

    // Trigger export
    const exportConfirm = getByRole('button', { name: /export/i })
    await userEvent.click(exportConfirm)

    // Verify download triggered
    expect(downloadSpy).toHaveBeenCalledWith('a')
  })
})
```

### Visual Regression Tests

```typescript
describe('TimelineViewer Visual Regression', () => {
  it('matches snapshot for timeline view', () => {
    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
        viewMode="timeline"
      />
    )

    expect(container).toMatchSnapshot()
  })

  it('matches snapshot for bitemporal view', () => {
    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEventsWithBackdating}
        bitemporalView={true}
      />
    )

    expect(container).toMatchSnapshot()
  })
})
```

### Accessibility Tests

```typescript
describe('TimelineViewer Accessibility', () => {
  it('passes axe accessibility tests', async () => {
    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
      />
    )

    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })

  it('supports keyboard navigation', async () => {
    const onEventClick = jest.fn()
    const { container } = render(
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={mockEvents}
        onEventClick={onEventClick}
      />
    )

    // Tab to first event
    const firstEvent = container.querySelector('.event-marker')
    firstEvent!.focus()

    // Press Enter to open details
    await userEvent.keyboard('{Enter}')

    expect(onEventClick).toHaveBeenCalled()
  })
})
```

---

## Accessibility

### WCAG 2.1 AA Compliance

#### Keyboard Navigation

**Requirements:**
- All interactive elements must be keyboard accessible
- Logical tab order
- Visible focus indicators
- Keyboard shortcuts documented

**Implementation:**
```typescript
// Event markers
<circle
  tabIndex={0}
  role="button"
  aria-label={`${event.eventType} event on ${formatDate(event.transactionTime)}`}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      onEventClick(event)
    }
  }}
/>

// Keyboard shortcuts
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    // ESC: Close details panel
    if (e.key === 'Escape' && showEventDetails) {
      setShowEventDetails(false)
    }

    // Arrow keys: Navigate events
    if (e.key === 'ArrowRight') {
      navigateToNextEvent()
    }
    if (e.key === 'ArrowLeft') {
      navigateToPreviousEvent()
    }

    // /: Focus search
    if (e.key === '/' && !isInputFocused()) {
      e.preventDefault()
      searchInputRef.current?.focus()
    }
  }

  document.addEventListener('keydown', handleKeyDown)
  return () => document.removeEventListener('keydown', handleKeyDown)
}, [showEventDetails])
```

#### Screen Reader Support

**Requirements:**
- Semantic HTML elements
- ARIA labels and roles
- Live regions for dynamic updates
- Alternative text for visualizations

**Implementation:**
```typescript
<div role="region" aria-label="Timeline visualization">
  {/* Timeline SVG */}
  <svg role="img" aria-label={`Timeline for ${entityType} ${entityId} with ${eventCount} events`}>
    {/* Events */}
    {events.map(event => (
      <g key={event.id}>
        <circle
          role="button"
          aria-label={getEventAriaLabel(event)}
          aria-describedby={`event-${event.id}-description`}
        />
        <desc id={`event-${event.id}-description`}>
          {event.eventType} event on {formatDate(event.transactionTime)}.
          {event.affectedFields.length} field(s) changed.
          {event.actor.name} made this change.
        </desc>
      </g>
    ))}
  </svg>

  {/* Live region for filter updates */}
  <div role="status" aria-live="polite" className="sr-only">
    {filteredEvents.length} events visible after filtering
  </div>
</div>
```

#### Color Contrast

**Requirements:**
- Minimum 4.5:1 contrast ratio for normal text
- Minimum 3:1 contrast ratio for large text and UI components
- Don't rely on color alone to convey information

**Implementation:**
```css
/* Event type colors with sufficient contrast */
.event-extraction {
  background-color: #2563EB; /* Blue - WCAG AA compliant */
  border: 2px solid #1E3A8A; /* Dark blue border for additional distinction */
}

.event-override {
  background-color: #D97706; /* Orange - WCAG AA compliant */
  border: 2px solid #92400E;
}

.event-correction {
  background-color: #7C3AED; /* Purple - WCAG AA compliant */
  border: 2px solid #5B21B6;
}

/* High contrast mode support */
@media (prefers-contrast: high) {
  .event-marker {
    border-width: 3px;
    filter: contrast(1.5);
  }
}
```

**Additional Indicators:**
- Use shapes in addition to colors (circles, diamonds, squares)
- Add text labels where space permits
- Use patterns/textures in exported images

#### Focus Management

**Requirements:**
- Visible focus indicators
- Logical focus order
- Focus trap in modals
- Restore focus when closing dialogs

**Implementation:**
```typescript
// Focus trap for event details panel
useFocusTrap(eventDetailsPanelRef, showEventDetails)

// Restore focus when closing panel
function closeEventDetails() {
  const previouslyFocusedElement = document.activeElement
  setShowEventDetails(false)

  // Return focus to event marker that opened the panel
  if (selectedEventRef.current) {
    selectedEventRef.current.focus()
  }
}

// CSS for focus indicators
```

```css
.event-marker:focus {
  outline: 3px solid #2563EB;
  outline-offset: 2px;
}

.event-marker:focus-visible {
  outline: 3px solid #2563EB;
  outline-offset: 2px;
  box-shadow: 0 0 0 5px rgba(37, 99, 235, 0.2);
}
```

#### Screen Reader Announcements

```typescript
// Announce filter changes
function announceFilterChange(filteredCount: number, totalCount: number) {
  const message = `Showing ${filteredCount} of ${totalCount} events`
  announceToScreenReader(message)
}

// Utility function
function announceToScreenReader(message: string) {
  const announcement = document.createElement('div')
  announcement.setAttribute('role', 'status')
  announcement.setAttribute('aria-live', 'polite')
  announcement.className = 'sr-only'
  announcement.textContent = message
  document.body.appendChild(announcement)

  setTimeout(() => {
    document.body.removeChild(announcement)
  }, 1000)
}
```

---

## Performance Considerations

### Rendering Optimization

#### Virtualization for Large Timelines

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

function OptimizedTableView({ events }: { events: TimelineEvent[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: events.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60, // Row height
    overscan: 10 // Render 10 extra rows above/below viewport
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => {
          const event = events[virtualRow.index]
          return (
            <div
              key={event.id}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`
              }}
            >
              <EventRow event={event} />
            </div>
          )
        })}
      </div>
    </div>
  )
}
```

#### Memoization

```typescript
// Memoize expensive computations
const filteredEvents = useMemo(() => {
  return timelineEvents.filter(event => {
    // Apply filters
    if (!filters.eventTypes[event.eventType]) return false
    if (filters.fields.length > 0 && !filters.fields.some(f =>
      event.affectedFields.some(af => af.fieldName === f)
    )) return false
    // ... more filters
    return true
  })
}, [timelineEvents, filters])

const eventsByField = useMemo(() => {
  return groupBy(filteredEvents, event =>
    event.affectedFields.map(f => f.fieldName)
  )
}, [filteredEvents])

// Memoize event components
const EventMarker = React.memo(({ event, onClick }: EventMarkerProps) => {
  return (
    <circle
      cx={event.x}
      cy={event.y}
      r={8}
      fill={getEventColor(event.eventType)}
      onClick={() => onClick(event)}
    />
  )
}, (prevProps, nextProps) => {
  return prevProps.event.id === nextProps.event.id
})
```

#### Debouncing & Throttling

```typescript
import { useDebouncedCallback } from 'use-debounce'

// Debounce search input
const debouncedSearch = useDebouncedCallback(
  (query: string) => {
    setSearchQuery(query)
  },
  300 // 300ms delay
)

// Throttle zoom events
const throttledZoom = useThrottledCallback(
  (zoomLevel: number) => {
    setZoomLevel(zoomLevel)
    refetchVisibleEvents()
  },
  100, // Max 10 updates per second
  { trailing: true }
)
```

### Data Loading Optimization

#### Lazy Loading Strategy

```typescript
function TimelineViewer({ entityId, lazyLoad, onFetchEvents }: Props) {
  const [loadedDateRanges, setLoadedDateRanges] = useState<DateRange[]>([])

  // Fetch events when visible date range changes
  useEffect(() => {
    if (!lazyLoad || !onFetchEvents) return

    // Check if current visible range is already loaded
    const needsLoading = !loadedDateRanges.some(range =>
      isDateRangeContained(visibleDateRange, range)
    )

    if (needsLoading) {
      fetchVisibleEvents()
    }
  }, [visibleDateRange, lazyLoad])

  async function fetchVisibleEvents() {
    setIsLoading(true)
    try {
      const newEvents = await onFetchEvents(visibleDateRange)
      setEvents(prev => [...prev, ...newEvents])
      setLoadedDateRanges(prev => [...prev, visibleDateRange])
    } catch (error) {
      setError(error)
    } finally {
      setIsLoading(false)
    }
  }
}
```

#### Caching Strategy

```typescript
import { QueryClient, useQuery } from '@tanstack/react-query'

const queryClient = new QueryClient()

function useTimelineEvents(
  entityId: string,
  dateRange: DateRange
) {
  return useQuery({
    queryKey: ['timeline', entityId, dateRange],
    queryFn: () => fetchTimelineEvents(entityId, dateRange),
    staleTime: 5 * 60 * 1000, // Cache for 5 minutes
    cacheTime: 30 * 60 * 1000, // Keep in memory for 30 minutes
    retry: 3,
    retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 30000)
  })
}
```

### Memory Management

#### Cleanup on Unmount

```typescript
useEffect(() => {
  // D3 zoom behavior
  const zoom = d3.zoom().on('zoom', handleZoom)
  svg.call(zoom)

  // Cleanup
  return () => {
    zoom.on('zoom', null)
    svg.on('.zoom', null)
  }
}, [])
```

#### Event Aggregation for Large Timelines

```typescript
function aggregateEventsByDay(events: TimelineEvent[]): AggregatedEvent[] {
  const eventsByDay = new Map<string, TimelineEvent[]>()

  events.forEach(event => {
    const day = format(new Date(event.transactionTime), 'yyyy-MM-dd')
    if (!eventsByDay.has(day)) {
      eventsByDay.set(day, [])
    }
    eventsByDay.get(day)!.push(event)
  })

  return Array.from(eventsByDay.entries()).map(([day, eventsOnDay]) => ({
    date: day,
    count: eventsOnDay.length,
    events: eventsOnDay,
    eventTypes: countBy(eventsOnDay, e => e.eventType)
  }))
}
```

### Bundle Size Optimization

```typescript
// Lazy load D3 modules only when needed
const loadD3Timeline = async () => {
  const d3 = await import('d3')
  return d3
}

// Code splitting for view modes
const TimelineView = lazy(() => import('./components/TimelineView'))
const TableView = lazy(() => import('./components/TableView'))
const CompactView = lazy(() => import('./components/CompactView'))

function TimelineViewer({ viewMode }: Props) {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      {viewMode === 'timeline' && <TimelineView />}
      {viewMode === 'table' && <TableView />}
      {viewMode === 'compact' && <CompactView />}
    </Suspense>
  )
}
```

---

## Related Components

### AsOfQueryBuilder

**Relationship:** Generates queries that filter TimelineViewer events

**Integration:**
```typescript
function EntityHistory() {
  const [asOfDate, setAsOfDate] = useState<Date | null>(null)
  const [queryType, setQueryType] = useState<'transaction' | 'valid'>('transaction')

  return (
    <>
      <AsOfQueryBuilder
        entityId="txn_001"
        onQuery={(date, type) => {
          setAsOfDate(date)
          setQueryType(type)
        }}
      />

      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={events}
        highlightedEvents={
          asOfDate ? [
            {
              eventId: findEventAtDate(events, asOfDate, queryType),
              color: '#EF4444',
              label: 'As-of query result'
            }
          ] : []
        }
      />
    </>
  )
}
```

### RetroactiveCorrectionDialog

**Relationship:** Creates timeline events that appear in TimelineViewer

**Integration:**
```typescript
function EntityEditor() {
  const [showCorrectionDialog, setShowCorrectionDialog] = useState(false)
  const [timelineEvents, setTimelineEvents] = useState<TimelineEvent[]>([])

  async function handleCorrection(correction: Correction) {
    // Save correction
    await saveCorrection(correction)

    // Create new timeline event
    const newEvent: TimelineEvent = {
      id: generateId(),
      eventType: 'correction',
      transactionTime: new Date().toISOString(),
      validTime: correction.effectiveDate,
      actor: currentUser,
      affectedFields: correction.fieldChanges,
      reason: correction.reason
    }

    // Add to timeline
    setTimelineEvents(prev => [...prev, newEvent])
    setShowCorrectionDialog(false)
  }

  return (
    <>
      <TimelineViewer
        entityId="txn_001"
        entityType="transaction"
        timelineEvents={timelineEvents}
      />

      <RetroactiveCorrectionDialog
        entity={entity}
        editableFields={fields}
        onSave={handleCorrection}
        onCancel={() => setShowCorrectionDialog(false)}
      />
    </>
  )
}
```

### EventDetailsPanel

**Relationship:** Child component that shows detailed event information

**Integration:**
```typescript
function TimelineViewer({ timelineEvents }: Props) {
  const [selectedEvent, setSelectedEvent] = useState<TimelineEvent | null>(null)

  return (
    <div className="timeline-container">
      <TimelineView
        events={timelineEvents}
        onEventClick={setSelectedEvent}
      />

      {selectedEvent && (
        <EventDetailsPanel
          event={selectedEvent}
          onClose={() => setSelectedEvent(null)}
          onNavigateToRelated={(relatedEventId) => {
            const relatedEvent = timelineEvents.find(e => e.id === relatedEventId)
            setSelectedEvent(relatedEvent || null)
          }}
        />
      )}
    </div>
  )
}
```

### CorrectionDialog (from Vertical 4.3)

**Relationship:** Creates override/correction events

**Integration:**
```typescript
function EntityWorkflow() {
  return (
    <Tabs>
      <TabPanel label="Current Data">
        <EntityForm />
      </TabPanel>

      <TabPanel label="History">
        <TimelineViewer
          entityId="txn_001"
          entityType="transaction"
          timelineEvents={events}
        />
      </TabPanel>

      <TabPanel label="Make Correction">
        <CorrectionDialog
          entity={entity}
          editableFields={fields}
          onSave={async (overrides) => {
            await saveOverrides(overrides)
            // Refresh timeline
            refetchTimelineEvents()
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

- **D3.js Documentation**: https://d3js.org/
  - Time scales: https://github.com/d3/d3-scale#time-scales
  - Zoom behavior: https://github.com/d3/d3-zoom
  - Axis generation: https://github.com/d3/d3-axis

- **TanStack Table**: https://tanstack.com/table/latest
  - Sorting: https://tanstack.com/table/latest/docs/guide/sorting
  - Filtering: https://tanstack.com/table/latest/docs/guide/filters
  - Virtualization: https://tanstack.com/virtual/latest

- **React Query**: https://tanstack.com/query/latest
  - Caching: https://tanstack.com/query/latest/docs/react/guides/caching
  - Retry logic: https://tanstack.com/query/latest/docs/react/guides/query-retries

### Bitemporal Database Theory

- **Snodgrass, Richard T.** (1999). *Developing Time-Oriented Database Applications in SQL*. Morgan Kaufmann.
  - Chapter 2: Bitemporal Databases
  - Chapter 6: Temporal Queries

- **Johnston, Tom** (2014). *Bitemporal Data: Theory and Practice*. Morgan Kaufmann.
  - Part I: Bitemporal Fundamentals
  - Part II: Bitemporal Queries

### Accessibility Standards

- **WCAG 2.1 AA**: https://www.w3.org/WAI/WCAG21/quickref/
  - 1.4.3 Contrast (Minimum)
  - 2.1.1 Keyboard
  - 4.1.3 Status Messages

- **ARIA Authoring Practices**: https://www.w3.org/WAI/ARIA/apg/
  - Timeline pattern: https://www.w3.org/WAI/ARIA/apg/patterns/timeline/

### Data Provenance Standards

- **PROV-DM**: W3C Provenance Data Model
  - https://www.w3.org/TR/prov-dm/
  - Entity, Activity, Agent model

- **ISO 21964:2019**: Provenance Information Model for Biological Specimen
  - Applicable to general provenance tracking

### Related Patterns

- **Event Sourcing**: https://martinfowler.com/eaaDev/EventSourcing.html
- **Audit Log**: https://martinfowler.com/eaaCatalog/auditLog.html
- **Temporal Patterns**: https://martinfowler.com/eaaDev/TemporalPattern.html

---

**Component Version:** 1.0.0
**Last Updated:** 2025-10-25
**Author:** Darwin Borges
**Review Status:** Draft - Pending Technical Review
