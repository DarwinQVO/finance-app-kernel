# QueueMonitorPanel (IL Component)

**Layer:** Interaction Layer (IL)
**Domain:** Rule Performance & Logs (Vertical 5.3)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Architecture](#architecture)
4. [Panel Components](#panel-components)
5. [Queue Health Indicators](#queue-health-indicators)
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

The **QueueMonitorPanel** is a real-time monitoring component designed to provide comprehensive visibility into document processing queue health, depth, throughput, and stuck document detection. It transforms queue monitoring from manual log analysis into a visual, interactive dashboard with proactive alerts, drill-down capabilities, and manual intervention controls. This component serves as the operational command center for queue management, enabling teams to identify bottlenecks, resolve stuck documents, and maintain processing SLAs.

### Key Capabilities

- **Real-Time Queue Metrics**: Live updates via WebSocket or polling (configurable interval)
- **Health Status Badges**: Color-coded indicators (Healthy/Warning/Critical) based on thresholds
- **Queue Depth Visualization**: Stacked area chart showing pending/in-progress/stuck documents
- **Processing Rate Tracking**: Line chart of documents processed per minute
- **Stuck Document Detection**: Automatic flagging of documents exceeding time threshold
- **Manual Interventions**: Pause/resume queue, retry stuck documents, skip failed items
- **Drill-Down Navigation**: Click queue card to view detailed queue status page
- **Export & Alerts**: CSV export, threshold breach notifications
- **Historical Comparison**: Compare current metrics vs. 1h/24h/7d averages
- **Multi-Queue Support**: Monitor multiple queues simultaneously in card grid layout

### Problem Statement

Operations teams struggle with queue monitoring because:

1. **No Real-Time Visibility**: Can't see queue depth or processing rate without checking logs
2. **Reactive Response**: Only discover backlogs after users complain about delays
3. **Stuck Documents**: No automated detection of documents stuck in processing
4. **Manual Investigation**: Requires SSH access to servers to diagnose queue issues
5. **Unknown Bottlenecks**: Can't identify which step in pipeline is slow
6. **No Intervention Controls**: Can't manually retry or skip failed documents
7. **Alert Fatigue**: Generic alerts without context or recommended actions

### Solution

The QueueMonitorPanel provides:

- **Proactive Monitoring**: Real-time alerts when queue depth exceeds thresholds
- **Visual Health Indicators**: At-a-glance status with color-coded badges
- **Automatic Detection**: Flag stuck documents (>30min) with drill-down to details
- **One-Click Actions**: Retry, skip, pause, resume queue from UI
- **Root Cause Analysis**: Identify slow processing steps via drill-down
- **Actionable Alerts**: Alerts include recommended actions (scale workers, investigate stuck docs)
- **Historical Context**: Compare current vs. baseline to detect anomalies

This transforms queue operations from reactive troubleshooting to proactive management.

---

## Component Interface

### Primary Props

```typescript
interface QueueMonitorPanelProps {
  // ============================================================
  // Queue Configuration
  // ============================================================

  /**
   * List of queues to monitor
   * If not provided, fetches all queues from API
   */
  queues?: string[]

  /**
   * Queue display configuration
   * Maps queue name to display name and metadata
   */
  queueConfig?: Record<string, QueueConfig>

  /**
   * Auto-refresh interval in seconds
   * Set to 0 to disable auto-refresh
   * @default 30
   */
  refreshInterval?: 10 | 30 | 60 | 120 | 0

  /**
   * Enable real-time updates via WebSocket
   * Falls back to polling if WebSocket unavailable
   * @default false
   */
  enableRealTime?: boolean

  /**
   * Time range for historical data (charts)
   * @default '24h'
   */
  timeRange?: '1h' | '6h' | '24h' | '7d'

  // ============================================================
  // Alert Threshold Configuration
  // ============================================================

  /**
   * Alert thresholds for queue health
   */
  alertThresholds?: {
    /**
     * Pending document count thresholds
     */
    pending?: {
      warning: number              // Default: 100
      critical: number             // Default: 500
    }

    /**
     * Stuck document time threshold (minutes)
     * Documents in-progress longer than this are flagged
     * @default 30
     */
    stuckDocumentMinutes?: number

    /**
     * Processing rate thresholds (docs/minute)
     * Alert if rate drops below threshold
     */
    processingRate?: {
      warning: number              // Default: 10
      critical: number             // Default: 5
    }

    /**
     * Queue age threshold (minutes)
     * Alert if oldest pending document exceeds this
     * @default 60
     */
    queueAgeMinutes?: number
  }

  /**
   * Show visual alerts when thresholds breached
   * @default true
   */
  showAlerts?: boolean

  /**
   * Play sound on critical alerts
   * @default false
   */
  playSoundAlerts?: boolean

  /**
   * Browser notification on critical alerts
   * Requires notification permission
   * @default false
   */
  enableBrowserNotifications?: boolean

  // ============================================================
  // Display Configuration
  // ============================================================

  /**
   * Layout mode
   * @default 'grid' (cards in grid)
   */
  layoutMode?: 'grid' | 'list' | 'compact'

  /**
   * Cards per row (in grid mode)
   * @default 2
   */
  cardsPerRow?: 1 | 2 | 3 | 4

  /**
   * Show queue depth chart
   * @default true
   */
  showDepthChart?: boolean

  /**
   * Show processing rate chart
   * @default true
   */
  showRateChart?: boolean

  /**
   * Chart height
   * @default '300px'
   */
  chartHeight?: string

  /**
   * Show queue cards
   * @default true
   */
  showQueueCards?: boolean

  /**
   * Sort queues by
   * @default 'status' (critical first)
   */
  sortBy?: 'status' | 'pending' | 'name' | 'rate'

  /**
   * Filter queues by status
   * @default 'all'
   */
  filterByStatus?: 'all' | 'healthy' | 'warning' | 'critical'

  // ============================================================
  // Feature Configuration
  // ============================================================

  /**
   * Enable manual queue controls (pause/resume)
   * @default true
   */
  enableQueueControls?: boolean

  /**
   * Enable stuck document actions (retry/skip)
   * @default true
   */
  enableStuckDocActions?: boolean

  /**
   * Show stuck document count in queue cards
   * @default true
   */
  showStuckCount?: boolean

  /**
   * Show oldest document age in queue cards
   * @default true
   */
  showOldestDocAge?: boolean

  /**
   * Enable queue card click to drill-down
   * @default true
   */
  enableDrillDown?: boolean

  /**
   * Enable export to CSV
   * @default true
   */
  enableExport?: boolean

  /**
   * Show comparison to historical baseline
   * @default true
   */
  showHistoricalComparison?: boolean

  // ============================================================
  // Data Source Configuration
  // ============================================================

  /**
   * API endpoint for fetching queue metrics
   * @example '/api/queues/metrics'
   */
  metricsApiEndpoint?: string

  /**
   * WebSocket URL for real-time updates
   * @example 'wss://api.example.com/queues/stream'
   */
  websocketUrl?: string

  /**
   * Custom data fetcher function
   */
  dataFetcher?: () => Promise<QueueMetrics>

  /**
   * Enable data caching
   * @default true
   */
  enableCaching?: boolean

  /**
   * Cache TTL in seconds
   * @default 30
   */
  cacheTtl?: number

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when user clicks a queue card
   * @param queueName - Name of clicked queue
   */
  onQueueClick?: (queueName: string) => void

  /**
   * Called when user pauses a queue
   * @param queueName - Name of queue
   */
  onQueuePause?: (queueName: string) => Promise<void>

  /**
   * Called when user resumes a queue
   * @param queueName - Name of queue
   */
  onQueueResume?: (queueName: string) => Promise<void>

  /**
   * Called when user retries stuck documents
   * @param queueName - Name of queue
   * @param documentIds - Array of document IDs to retry
   */
  onRetryStuckDocuments?: (queueName: string, documentIds: string[]) => Promise<void>

  /**
   * Called when user skips a document
   * @param queueName - Name of queue
   * @param documentId - Document ID to skip
   */
  onSkipDocument?: (queueName: string, documentId: string) => Promise<void>

  /**
   * Called when alert threshold is breached
   * @param alert - Alert details
   */
  onAlertTriggered?: (alert: QueueAlert) => void

  /**
   * Called when user exports queue data
   * @param format - Export format
   * @param data - Queue metrics
   */
  onExport?: (format: 'csv' | 'json', data: QueueMetrics) => void

  /**
   * Called when metrics are refreshed
   * @param metrics - Updated metrics
   */
  onRefresh?: (metrics: QueueMetrics) => void

  /**
   * Called on API errors
   * @param error - Error object
   */
  onError?: (error: Error) => void

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Panel title
   * @default 'Queue Monitor'
   */
  title?: string

  /**
   * Show panel header
   * @default true
   */
  showHeader?: boolean

  /**
   * Show refresh controls
   * @default true
   */
  showRefreshControls?: boolean

  /**
   * Show filter controls
   * @default true
   */
  showFilters?: boolean

  /**
   * Theme
   * @default 'auto'
   */
  theme?: 'light' | 'dark' | 'auto'

  /**
   * Compact mode (smaller cards, less padding)
   * @default false
   */
  compactMode?: boolean

  /**
   * Loading skeleton style
   * @default 'shimmer'
   */
  loadingStyle?: 'shimmer' | 'spinner' | 'pulse'

  /**
   * Empty state message
   */
  emptyStateMessage?: string

  /**
   * Custom CSS class name
   */
  className?: string

  /**
   * Custom inline styles
   */
  style?: React.CSSProperties

  // ============================================================
  // Advanced Options
  // ============================================================

  /**
   * Enable debug mode
   * @default false
   */
  debugMode?: boolean

  /**
   * Locale for formatting
   * @default 'en-US'
   */
  locale?: string

  /**
   * Timezone for timestamp display
   * @default 'UTC'
   */
  timezone?: string

  /**
   * Custom formatters
   */
  formatters?: {
    queueDepth?: (count: number) => string
    processingRate?: (rate: number) => string
    age?: (minutes: number) => string
    timestamp?: (date: Date) => string
  }
}

// ============================================================
// Supporting Types
// ============================================================

interface QueueConfig {
  displayName: string
  description?: string
  priority?: 'high' | 'medium' | 'low'
  expectedRate?: number       // Expected docs/minute
  maxDepth?: number           // Max pending before alert
}

interface QueueMetrics {
  summary: {
    totalQueues: number
    healthyQueues: number
    warningQueues: number
    criticalQueues: number
    totalPending: number
    totalInProgress: number
    totalStuck: number
  }
  queues: QueueStatus[]
  depthTimeSeries: DepthTimeSeriesPoint[]
  rateTimeSeries: RateTimeSeriesPoint[]
  generatedAt: Date
}

interface QueueStatus {
  name: string
  displayName: string
  status: 'healthy' | 'warning' | 'critical' | 'paused'
  metrics: {
    pending: number
    inProgress: number
    completed: number         // Last hour
    failed: number            // Last hour
    stuck: number
    processingRate: number    // Docs/minute
    avgProcessingTime: number // Milliseconds
    oldestDocumentAge: number // Minutes
  }
  workers: {
    active: number
    total: number
    utilization: number       // Percentage
  }
  alerts: QueueAlert[]
  paused: boolean
  lastProcessed?: Date
}

interface DepthTimeSeriesPoint {
  timestamp: Date
  queues: Record<string, {
    pending: number
    inProgress: number
    stuck: number
  }>
}

interface RateTimeSeriesPoint {
  timestamp: Date
  queues: Record<string, number>  // Queue name → processing rate
}

interface QueueAlert {
  id: string
  queueName: string
  type: 'high_pending' | 'stuck_documents' | 'low_rate' | 'queue_age' | 'worker_down'
  severity: 'warning' | 'critical'
  message: string
  value: number
  threshold: number
  timestamp: Date
  recommendedAction?: string
}

interface StuckDocument {
  documentId: string
  queueName: string
  enqueuedAt: Date
  startedAt: Date
  age: number                 // Minutes
  retryCount: number
  lastError?: string
  metadata?: Record<string, any>
}

// ============================================================
// Component State
// ============================================================

interface QueueMonitorPanelState {
  // Data state
  metrics: QueueMetrics | null
  stuckDocuments: StuckDocument[]
  loading: boolean
  error: Error | null
  lastUpdated: Date | null

  // UI state
  selectedQueue: string | null
  filterStatus: 'all' | 'healthy' | 'warning' | 'critical'
  sortBy: 'status' | 'pending' | 'name' | 'rate'

  // Real-time state
  websocketConnected: boolean
  refreshCountdown: number    // Seconds until next refresh

  // Alert state
  activeAlerts: QueueAlert[]
  mutedAlerts: Set<string>    // Alert IDs that are muted

  // Action state
  pausingQueue: string | null
  resumingQueue: string | null
  retryingDocuments: string[] // Document IDs being retried

  // Modal state
  showStuckDocsModal: boolean
  showPauseConfirmDialog: boolean
}
```

---

## Architecture

### Component Hierarchy

```
QueueMonitorPanel
├── PanelHeader
│   ├── TitleSection
│   ├── FilterControls
│   │   ├── StatusFilter (all/healthy/warning/critical)
│   │   └── SortDropdown
│   └── ActionButtons
│       ├── RefreshButton
│       ├── ExportButton
│       └── RefreshIntervalSelector
├── AlertBanner (if alerts active)
│   └── AlertList
│       └── AlertCard
│           ├── AlertIcon
│           ├── AlertMessage
│           ├── RecommendedAction
│           └── MuteButton
├── SummaryMetrics
│   ├── MetricCard (Total Queues)
│   ├── MetricCard (Healthy Queues)
│   ├── MetricCard (Warning Queues)
│   ├── MetricCard (Critical Queues)
│   └── MetricCard (Total Stuck Documents)
├── ChartsSection
│   ├── QueueDepthChart (stacked area chart)
│   │   ├── ChartHeader
│   │   ├── AreaChart
│   │   │   ├── XAxis (time)
│   │   │   ├── YAxis (document count)
│   │   │   ├── Tooltip
│   │   │   ├── Legend
│   │   │   └── Areas (pending, in-progress, stuck)
│   │   └── ThresholdLines (warning, critical)
│   └── ProcessingRateChart (line chart)
│       ├── ChartHeader
│       └── LineChart
│           ├── XAxis (time)
│           ├── YAxis (docs/minute)
│           ├── Tooltip
│           ├── Legend
│           └── Lines (one per queue)
├── QueueCardsGrid
│   └── QueueCard[] (one per queue)
│       ├── CardHeader
│       │   ├── QueueName
│       │   ├── StatusBadge (🟢/🟡/🔴)
│       │   └── PausedIndicator (if paused)
│       ├── MetricsSection
│       │   ├── PendingCount
│       │   ├── InProgressCount
│       │   ├── StuckCount (highlighted if >0)
│       │   ├── ProcessingRate
│       │   └── OldestDocAge
│       ├── WorkerUtilization (progress bar)
│       └── ActionButtons
│           ├── ViewDetailsButton
│           ├── PauseResumeButton
│           └── ViewStuckDocsButton (if stuck >0)
└── Modals
    ├── StuckDocumentsModal
    │   ├── ModalHeader
    │   ├── StuckDocumentsTable
    │   │   └── StuckDocumentRow
    │   │       ├── DocumentId
    │   │       ├── Age
    │   │       ├── RetryCount
    │   │       ├── LastError
    │   │       └── Actions (Retry/Skip)
    │   └── ModalFooter (Retry All/Close)
    └── PauseConfirmDialog
        ├── ConfirmMessage
        └── Actions (Confirm/Cancel)
```

### Data Flow

```
Component Mount
    ↓
Fetch Initial Metrics
    ↓
GET /api/queues/metrics?timeRange=24h
    ↓
    ├── Returns queue status for all queues
    ├── Returns depth time series (last 24h)
    └── Returns rate time series (last 24h)
    ↓
Calculate Health Status
    ↓
    ├── For each queue:
    │   ├── If pending > critical threshold → status = 'critical'
    │   ├── Else if pending > warning threshold → status = 'warning'
    │   ├── Else if stuck > 0 → status = 'warning'
    │   └── Else → status = 'healthy'
    ↓
Check Alert Thresholds
    ↓
    ├── If any queue has status = 'critical' → trigger alert
    └── If stuck documents detected → trigger alert
    ↓
Render Component
    ↓
    ├── Render summary metrics
    ├── Render depth chart (stacked area)
    ├── Render rate chart (line)
    └── Render queue cards (sorted by status)
    ↓
Start Auto-Refresh Loop (30s)
    ↓
    ├── Fetch updated metrics
    ├── Merge with cached data
    ├── Update charts with new data points
    └── Check for new alerts
    ↓
User Interaction: Click "View Stuck Docs"
    ↓
Fetch Stuck Documents
    ↓
GET /api/queues/{queueName}/stuck
    ↓
Show Modal with Stuck Documents Table
    ↓
User Clicks "Retry All"
    ↓
POST /api/queues/{queueName}/retry
    ↓
Success → Refresh metrics → Close modal
Error → Show error toast → Keep modal open
```

---

## Panel Components

### 1. Summary Metrics

Five metric cards at the top:

```
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ 10       │ 🟢 8     │ 🟡 2     │ 🔴 0     │ 23       │
│ Total    │ Healthy  │ Warning  │ Critical │ Stuck    │
│ Queues   │ Queues   │ Queues   │ Queues   │ Documents│
└──────────┴──────────┴──────────┴──────────┴──────────┘
```

### 2. Queue Depth Chart (Stacked Area)

Shows pending, in-progress, and stuck documents over time:

```
┌────────────────────────────────────────────────────────┐
│ Queue Depth (Last 24h)                                 │
├────────────────────────────────────────────────────────┤
│ Count                                                  │
│   │                                                    │
│500├                                    ▓▓▓▓▓▓▓▓       │
│   │                               ▓▓▓▓▓▒▒▒▒▒▒▒▒▓▓▓▓  │
│400├                          ▓▓▓▓▓▒▒▒▒▒░░░░░░░░▒▒▒▒▓▓│
│   │                     ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░▒▒▒▒│
│300├ - - - - - - - ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░│
│   │          ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
│200├     ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
│   │▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
│100├▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
│   ├────────────────────────────────────────────────┤
│  0h        6h        12h        18h        24h      │
└────────────────────────────────────────────────────────┘
▓ Stuck  ▒ In Progress  ░ Pending
```

**Threshold Lines:**
- Yellow dashed line at 100 (warning threshold)
- Red dashed line at 500 (critical threshold)

### 3. Processing Rate Chart (Line)

Shows documents processed per minute:

```
┌────────────────────────────────────────────────────────┐
│ Processing Rate (Last 24h)                             │
├────────────────────────────────────────────────────────┤
│ Docs/min                                               │
│   │                                                    │
│100├                        ●●●●●●                     │
│   │                    ●●●●      ●●●●                 │
│ 80├                ●●●●              ●●●●             │
│   │            ●●●●                      ●●●●         │
│ 60├        ●●●●                              ●●●●     │
│   │    ●●●●                                      ●●●●│
│ 40├●●●●                                              │
│   ├────────────────────────────────────────────────┤
│  0h        6h        12h        18h        24h      │
└────────────────────────────────────────────────────────┘
— invoice-queue  — receipt-queue  — transaction-queue
```

### 4. Queue Card

Individual card for each queue:

```
┌──────────────────────────────────────────────────┐
│ Invoice Processing Queue     🟡 WARNING   ⏸ PAUSED│
├──────────────────────────────────────────────────┤
│ Pending: 156    In Progress: 12    Stuck: 3     │
│                                                  │
│ Processing Rate: 23 docs/min                     │
│ Oldest Document: 45 minutes                      │
│                                                  │
│ Worker Utilization: ████████████░░░░ 75%         │
│                                                  │
│ Alerts:                                          │
│  🟡 Pending count (156) exceeds warning (100)   │
│  🔴 3 documents stuck for >30 minutes            │
│                                                  │
│ [View Details] [Resume] [View Stuck Docs (3)]   │
└──────────────────────────────────────────────────┘
```

**Status Badge Colors:**
- 🟢 Green: Healthy (pending < warning threshold, stuck = 0)
- 🟡 Yellow: Warning (pending ≥ warning threshold OR stuck > 0)
- 🔴 Red: Critical (pending ≥ critical threshold)
- ⏸ Gray: Paused

---

## Queue Health Indicators

### Health Status Logic

```typescript
function calculateQueueHealth(queue: QueueStatus, thresholds: AlertThresholds): QueueHealth {
  const { pending, stuck } = queue.metrics
  const { processingRate, oldestDocumentAge } = queue.metrics

  if (queue.paused) {
    return { status: 'paused', alerts: [] }
  }

  const alerts: QueueAlert[] = []

  // Check pending threshold
  if (pending >= thresholds.pending.critical) {
    alerts.push({
      type: 'high_pending',
      severity: 'critical',
      message: `Pending count (${pending}) exceeds critical threshold (${thresholds.pending.critical})`,
      recommendedAction: 'Scale workers or investigate processing delays'
    })
  } else if (pending >= thresholds.pending.warning) {
    alerts.push({
      type: 'high_pending',
      severity: 'warning',
      message: `Pending count (${pending}) exceeds warning threshold (${thresholds.pending.warning})`
    })
  }

  // Check stuck documents
  if (stuck > 0) {
    alerts.push({
      type: 'stuck_documents',
      severity: 'critical',
      message: `${stuck} documents stuck for >${thresholds.stuckDocumentMinutes} minutes`,
      recommendedAction: 'Review stuck documents and retry or skip'
    })
  }

  // Check processing rate
  if (processingRate < thresholds.processingRate.critical) {
    alerts.push({
      type: 'low_rate',
      severity: 'critical',
      message: `Processing rate (${processingRate} docs/min) below critical threshold`
    })
  }

  // Determine overall status
  const hasCriticalAlert = alerts.some(a => a.severity === 'critical')
  const hasWarningAlert = alerts.some(a => a.severity === 'warning')

  const status = hasCriticalAlert ? 'critical'
    : hasWarningAlert ? 'warning'
    : 'healthy'

  return { status, alerts }
}
```

---

## Visual Wireframes

### Wireframe 1: Queue Monitor Panel (Desktop)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Queue Monitor                              [Filter: All ▼] [⟳] [Export]   │
├────────────────────────────────────────────────────────────────────────────┤
│ Auto-refresh: ●30s | Last updated: Just now                                │
├────────────────────────────────────────────────────────────────────────────┤
│ 🔴 CRITICAL: 3 documents stuck in invoice-processing queue                 │
├────────────────────────────────────────────────────────────────────────────┤
│ SUMMARY                                                                    │
│ ┌──────────┬──────────┬──────────┬──────────┬──────────────────┐         │
│ │ 10       │ 🟢 8     │ 🟡 2     │ 🔴 0     │ 23               │         │
│ │ Total    │ Healthy  │ Warning  │ Critical │ Stuck Documents  │         │
│ └──────────┴──────────┴──────────┴──────────┴──────────────────┘         │
├────────────────────────────────────────────────────────────────────────────┤
│ QUEUE DEPTH (Last 24h)                                                     │
│ ┌──────────────────────────────────────────────────────────────────────┐   │
│ │500─                                              ▓▓▓▓▓▓▓▓▓          │   │
│ │   │                                         ▓▓▓▓▓▒▒▒▒▒▒▒▒▓▓▓▓       │   │
│ │400─                                    ▓▓▓▓▓▒▒▒▒▒░░░░░░░░▒▒▒▒▓▓▓▓  │   │
│ │   │                               ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░▒▒▒▒▒▒▒▒│   │
│ │300─ - - - - - - - - - - - ▓▓▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░│   │
│ │   │                  ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│   │
│ │200─             ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│   │
│ │   │        ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│   │
│ │100─   ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│   │
│ │   ├────────────────────────────────────────────────────────────┤   │
│ │   0h          6h           12h          18h          24h        │   │
│ └──────────────────────────────────────────────────────────────────────┘   │
│ ▓ Stuck  ▒ In Progress  ░ Pending                                        │
├────────────────────────────────────────────────────────────────────────────┤
│ PROCESSING RATE (Last 24h)                                                │
│ ┌──────────────────────────────────────────────────────────────────────┐   │
│ │100─                            ●●●●●●●                               │   │
│ │   │                        ●●●●      ●●●●                            │   │
│ │ 80─                    ●●●●              ●●●●                        │   │
│ │   │                ●●●●                      ●●●●                    │   │
│ │ 60─            ●●●●                              ●●●●                │   │
│ │   │        ●●●●                                      ●●●●            │   │
│ │ 40─    ●●●●                                              ●●●●        │   │
│ │   ├────────────────────────────────────────────────────────────┤   │
│ │   0h          6h           12h          18h          24h        │   │
│ └──────────────────────────────────────────────────────────────────────┘   │
│ — invoice  — receipt  — transaction                                       │
├────────────────────────────────────────────────────────────────────────────┤
│ QUEUES (10)                                            Sort: Status ▼     │
├───────────────────────────────────┬────────────────────────────────────────┤
│ ┌─────────────────────────────┐   │ ┌────────────────────────────────────┐│
│ │ Invoice Processing    🟡 ⚠️ │   │ │ Receipt OCR          🟢 HEALTHY   ││
│ ├─────────────────────────────┤   │ ├────────────────────────────────────┤│
│ │ Pending: 156  In Prog: 12   │   │ │ Pending: 23  In Prog: 5           ││
│ │ Stuck: 3  Rate: 23/min      │   │ │ Stuck: 0  Rate: 45/min            ││
│ │ Oldest: 45 min              │   │ │ Oldest: 5 min                     ││
│ │                             │   │ │                                    ││
│ │ Workers: ████████░░ 75%     │   │ │ Workers: ██████████ 100%          ││
│ │                             │   │ │                                    ││
│ │ [Details] [View Stuck (3)] │   │ │ [Details] [Pause]                 ││
│ └─────────────────────────────┘   │ └────────────────────────────────────┘│
│                                   │                                        │
│ ┌─────────────────────────────┐   │ ┌────────────────────────────────────┐│
│ │ Transaction Import  🟢 OK   │   │ │ Email Parser        🟢 HEALTHY    ││
│ ├─────────────────────────────┤   │ ├────────────────────────────────────┤│
│ │ Pending: 45  In Prog: 8     │   │ │ Pending: 12  In Prog: 3           ││
│ │ Stuck: 0  Rate: 67/min      │   │ │ Stuck: 0  Rate: 89/min            ││
│ │ Oldest: 3 min               │   │ │ Oldest: 1 min                     ││
│ │                             │   │ │                                    ││
│ │ [Details] [Pause]           │   │ │ [Details] [Pause]                 ││
│ └─────────────────────────────┘   │ └────────────────────────────────────┘│
└───────────────────────────────────┴────────────────────────────────────────┘
```

### Wireframe 2: Stuck Documents Modal

```
┌──────────────────────────────────────────────────────────────────────┐
│ Stuck Documents - Invoice Processing Queue                     [×]  │
├──────────────────────────────────────────────────────────────────────┤
│ 3 documents have been stuck for >30 minutes                          │
├──────────────────────────────────────────────────────────────────────┤
│ ┌────────────┬──────────┬───────┬─────────────────────┬─────────┐   │
│ │ Document ID│ Age      │ Retry │ Last Error          │ Actions │   │
│ ├────────────┼──────────┼───────┼─────────────────────┼─────────┤   │
│ │☑ doc_abc123│ 45 min   │ 2     │ PDF Parse Timeout   │ [Retry] │   │
│ │            │          │       │                     │ [Skip]  │   │
│ ├────────────┼──────────┼───────┼─────────────────────┼─────────┤   │
│ │☑ doc_def456│ 38 min   │ 1     │ Network Error       │ [Retry] │   │
│ │            │          │       │                     │ [Skip]  │   │
│ ├────────────┼──────────┼───────┼─────────────────────┼─────────┤   │
│ │☑ doc_ghi789│ 32 min   │ 3     │ Out of Memory       │ [Retry] │   │
│ │            │          │       │                     │ [Skip]  │   │
│ └────────────┴──────────┴───────┴─────────────────────┴─────────┘   │
│                                                                      │
│ ☑ Select All (3 selected)                                           │
│                                                                      │
│ [Export List] [Retry Selected (3)] [Skip Selected (3)] [Close]     │
└──────────────────────────────────────────────────────────────────────┘
```

### Wireframe 3: Mobile View (Compact Layout)

```
┌──────────────────────────────────────┐
│ Queue Monitor           [☰] [⟳]     │
├──────────────────────────────────────┤
│ Auto: ●30s | Updated: Just now      │
├──────────────────────────────────────┤
│ 🔴 3 docs stuck in invoice queue     │
├──────────────────────────────────────┤
│ ┌──────────┬──────────┬──────────┐  │
│ │ 10       │ 🟢 8     │ 23       │  │
│ │ Queues   │ Healthy  │ Stuck    │  │
│ └──────────┴──────────┴──────────┘  │
├──────────────────────────────────────┤
│ QUEUE DEPTH (24h)                    │
│ ┌──────────────────────────────────┐ │
│ │500─                    ▓▓▓▓▓▓▓▓ │ │
│ │   │               ▓▓▓▓▓▒▒▒▒▒▒▒▒│ │
│ │300─ - - - - ▓▓▓▓▓▓▒▒▒▒▒░░░░░░░░│ │
│ │   │    ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░│ │
│ │100─▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░░│ │
│ │   ├──────────────────────────┤ │
│ │   0h       12h          24h   │ │
│ └──────────────────────────────────┘ │
├──────────────────────────────────────┤
│ QUEUES (10)                          │
│                                      │
│ ┌──────────────────────────────────┐ │
│ │ Invoice Processing    🟡 WARNING │ │
│ │ Pending: 156  In Prog: 12        │ │
│ │ Stuck: 3  Rate: 23/min           │ │
│ │ [Details] [View Stuck]           │ │
│ └──────────────────────────────────┘ │
│                                      │
│ ┌──────────────────────────────────┐ │
│ │ Receipt OCR           🟢 HEALTHY │ │
│ │ Pending: 23  In Prog: 5          │ │
│ │ Stuck: 0  Rate: 45/min           │ │
│ │ [Details] [Pause]                │ │
│ └──────────────────────────────────┘ │
│                                      │
│ [Load More Queues]                   │
└──────────────────────────────────────┘
```

---

## States & Lifecycle

### Component States

```typescript
type PanelState =
  | 'loading'              // Fetching initial metrics
  | 'ready'                // Metrics loaded, panel interactive
  | 'refreshing'           // Auto-refresh in progress
  | 'pausing_queue'        // Pausing a queue
  | 'resuming_queue'       // Resuming a queue
  | 'retrying_documents'   // Retrying stuck documents
  | 'error'                // Error occurred
```

### State Transitions

```
┌──────────┐
│ Loading  │ ← Component mounts
└────┬─────┘
     │
     │ Fetch queue metrics
     ▼
┌──────────┐
│  Ready   │ ← Panel rendered with metrics
└────┬─────┘
     │
     ├─ Auto-refresh timer (30s) ──────────┐
     │                                     │
     │ User clicks "Pause Queue"           ▼
     ▼                              ┌──────────────┐
┌────────────────┐                 │ Refreshing   │
│ Pausing Queue  │                 └──────┬───────┘
└────┬───────────┘                        │
     │                                    │ Success
     │ Success                            ▼
     ▼                              ┌──────────┐
┌──────────┐                        │  Ready   │
│  Ready   │                        └──────────┘
└────┬─────┘
     │
     │ User clicks "Retry Stuck Docs"
     ▼
┌─────────────────────┐
│ Retrying Documents  │
└─────────┬───────────┘
          │
          │ Success
          ▼
     ┌──────────┐
     │  Ready   │
     └──────────┘
```

---

## Multi-Domain Examples

### 1. Finance: Transaction Processing Queues

**Context:** Monitor queues for processing transactions from various payment gateways.

**Queues:**
- `stripe-transactions`
- `paypal-transactions`
- `crypto-transactions`
- `bank-transfers`

**Configuration:**

```typescript
<QueueMonitorPanel
  queues={['stripe-transactions', 'paypal-transactions', 'crypto-transactions']}
  queueConfig={{
    'stripe-transactions': {
      displayName: 'Stripe Transactions',
      priority: 'high',
      expectedRate: 100,
      maxDepth: 200
    },
    'paypal-transactions': {
      displayName: 'PayPal Transactions',
      priority: 'medium',
      expectedRate: 50
    }
  }}
  alertThresholds={{
    pending: { warning: 100, critical: 500 },
    stuckDocumentMinutes: 30,
    processingRate: { warning: 10, critical: 5 }
  }}
  enableQueueControls={true}
  enableStuckDocActions={true}
/>
```

### 2. Healthcare: Claims Processing Queues

**Queues:**
- `edi-837-claims`
- `hcfa-1500-claims`
- `claims-validation`
- `adjudication-queue`

**Configuration:**

```typescript
<QueueMonitorPanel
  queues={['edi-837-claims', 'hcfa-1500-claims', 'claims-validation']}
  alertThresholds={{
    pending: { warning: 50, critical: 200 },
    stuckDocumentMinutes: 60,
    queueAgeMinutes: 120
  }}
/>
```

### 3. Legal: Document Processing Queues

**Queues:**
- `contract-ocr`
- `deposition-transcription`
- `document-classification`
- `metadata-extraction`

### 4. Research: Data Pipeline Queues

**Queues:**
- `genomic-data-processing`
- `image-analysis`
- `data-validation`

### 5. E-commerce: Order Processing Queues

**Queues:**
- `order-validation`
- `inventory-check`
- `payment-processing`
- `shipping-label-generation`

### 6. SaaS: API Request Queues

**Queues:**
- `webhook-delivery`
- `email-sending`
- `report-generation`
- `data-export`

### 7. Insurance: Claims Queues

**Queues:**
- `fnol-processing` (First Notice of Loss)
- `claims-validation`
- `fraud-detection`
- `adjudication`

---

## User Interactions

### 1. Viewing Stuck Documents

**Trigger:** User clicks "View Stuck Docs (3)" button

**Flow:**
1. User clicks button on queue card
2. Open modal/drawer
3. Fetch stuck documents: `GET /api/queues/{queueName}/stuck`
4. Show table with stuck documents
5. Display: Document ID, Age, Retry Count, Last Error
6. User can select documents and retry/skip

### 2. Retrying Stuck Documents

**Trigger:** User clicks "Retry" button on stuck document

**Flow:**
1. User selects document(s) and clicks "Retry"
2. Show confirmation: "Retry 3 documents?"
3. Call API: `POST /api/queues/{queueName}/retry` with document IDs
4. Show progress indicator
5. On success:
   - Show success toast: "3 documents requeued"
   - Refresh queue metrics
   - Close modal
6. On error:
   - Show error message
   - Keep modal open

### 3. Pausing a Queue

**Trigger:** User clicks "Pause" button

**Flow:**
1. User clicks "Pause" button on queue card
2. Show confirmation dialog: "Pause invoice-processing queue? No new documents will be processed."
3. User clicks "Confirm"
4. Call API: `POST /api/queues/{queueName}/pause`
5. Update queue card: Show "⏸ PAUSED" badge
6. Change button to "Resume"

---

## Accessibility

### WCAG 2.1 AA Compliance

#### Keyboard Navigation

| Action | Shortcut | Behavior |
|--------|----------|----------|
| Navigate queue cards | `Tab` | Focus moves between cards |
| Select queue | `Enter` | Opens queue details |
| Pause/resume queue | `Space` | Toggles queue state |
| View stuck docs | `S` | Opens stuck docs modal |
| Refresh metrics | `R` | Manually refreshes data |

#### Screen Reader Support

```html
<div role="region" aria-label="Queue Monitor Panel">
  <div role="group" aria-label="Summary Metrics">
    <div aria-label="10 total queues">
      <span aria-hidden="true">10</span>
      <span>Total Queues</span>
    </div>
  </div>

  <div role="list" aria-label="Queue status list">
    <div role="listitem" aria-label="Invoice processing queue, warning status, 156 pending, 3 stuck">
      <h3>Invoice Processing Queue</h3>
      <span role="status" aria-label="Warning">🟡</span>
      <p>Pending: 156, Stuck: 3</p>
    </div>
  </div>
</div>
```

---

## Performance Optimizations

### 1. Efficient Chart Rendering

Use memoization for charts:

```typescript
const QueueDepthChart = React.memo(({ data, height }) => {
  return <AreaChart data={data} height={height} />
}, (prev, next) => prev.data === next.data)
```

### 2. Debounced Updates

For real-time WebSocket updates, debounce to avoid excessive re-renders.

---

## Integration Examples

### Example 1: Basic Usage

```typescript
import { QueueMonitorPanel } from '@/components/il/QueueMonitorPanel'

export default function MonitoringPage() {
  const handleQueueClick = (queueName: string) => {
    router.push(`/queues/${queueName}`)
  }

  return (
    <QueueMonitorPanel
      queues={['invoice-processing', 'receipt-ocr', 'transaction-import']}
      refreshInterval={30}
      enableRealTime={false}
      onQueueClick={handleQueueClick}
    />
  )
}
```

---

## Related Components

### 1. PerformanceDashboard
QueueMonitorPanel is embedded in PerformanceDashboard as the Queue Health panel.

### 2. QueueDetailView
Drill-down page showing detailed queue metrics and execution logs.

---

## References

- [Bull Queue Documentation](https://github.com/OptimalBits/bull)
- [Redis Queue Patterns](https://redis.io/topics/pubsub)
- [React Query](https://tanstack.com/query/latest)

---

**Document Version:** 1.0
**Word Count:** ~7,000 words
**Last Updated:** 2025-10-25
**Maintained By:** Platform Team
