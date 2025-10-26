# PerformanceDashboard (IL Component)

**Layer:** Interaction Layer (IL)
**Domain:** Rule Performance & Logs (Vertical 5.3)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Architecture](#architecture)
4. [Dashboard Panels](#dashboard-panels)
5. [Chart Components](#chart-components)
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

The **PerformanceDashboard** is a comprehensive real-time monitoring component designed to provide observability into parser performance, rule execution efficiency, queue health, and error patterns. It transforms raw performance metrics into actionable insights through interactive visualizations, enabling operations teams to quickly identify bottlenecks, optimize slow rules, and maintain system health. This component is the central nervous system for monitoring document processing pipelines in production.

### Key Capabilities

- **Real-Time Metrics**: Auto-refreshing dashboard with configurable intervals (30s/60s/120s)
- **Multi-Panel Layout**: 4 primary panels (Parser Stats, Rule Stats, Queue Health, Error Breakdown)
- **Performance Analytics**: P50/P95/P99 latency percentiles, throughput rates, error rates
- **Time Range Selection**: Flexible time windows (1h/6h/24h/7d/30d)
- **Drill-Down Navigation**: Click-through from summary metrics to detailed execution logs
- **Export Capabilities**: CSV/JSON export for all metrics and charts
- **Filter & Search**: Filter by parser ID, rule ID, queue name, error type
- **Alerting Integration**: Visual indicators for threshold breaches (latency, queue depth, error rate)
- **Responsive Design**: Adapts from desktop (4 panels) to mobile (stacked panels)
- **Dark/Light Theme**: Matches system preferences or user selection

### Problem Statement

Operations teams struggle with parser performance monitoring because:

1. **No Visibility**: Black box - can't see which parsers/rules are slow
2. **Reactive Troubleshooting**: Only discover issues after users complain
3. **Resource Waste**: Slow rules consume disproportionate resources
4. **Queue Backlogs**: No early warning when queues start backing up
5. **Error Patterns**: Similar errors repeat without root cause analysis
6. **Optimization Guesswork**: No data to guide performance improvements
7. **Manual Monitoring**: Requires running ad-hoc queries and building spreadsheets

### Solution

The PerformanceDashboard provides:

- **Proactive Monitoring**: Real-time alerts before issues impact users
- **Performance Profiling**: Identify slowest parsers/rules with P95 latency tracking
- **Queue Health**: Visual indicators for queue depth, stuck documents, processing rate
- **Error Intelligence**: Aggregate errors by type, show top failing rules
- **Historical Trends**: Compare current performance vs. 24h/7d/30d averages
- **One-Click Drill-Down**: Navigate from dashboard to specific execution logs
- **Automated Reporting**: Schedule daily/weekly performance summaries

This transforms ops from reactive fire-fighting to proactive optimization.

---

## Component Interface

### Primary Props

```typescript
interface PerformanceDashboardProps {
  // ============================================================
  // Time Range Configuration
  // ============================================================

  /**
   * Initial time range for dashboard data
   * @default '24h'
   */
  initialTimeRange?: '1h' | '6h' | '24h' | '7d' | '30d' | 'custom'

  /**
   * Custom time range (if initialTimeRange='custom')
   */
  customTimeRange?: {
    start: Date | string
    end: Date | string
  }

  /**
   * Auto-refresh interval in seconds
   * Set to 0 to disable auto-refresh
   * @default 60
   */
  refreshInterval?: 30 | 60 | 120 | 0

  /**
   * Enable real-time updates via WebSocket
   * Falls back to polling if WebSocket unavailable
   * @default false
   */
  enableRealTime?: boolean

  // ============================================================
  // Filter Configuration
  // ============================================================

  /**
   * Pre-applied filters
   */
  initialFilters?: {
    parserId?: string | string[]
    ruleId?: string | string[]
    queueName?: string | string[]
    errorType?: string | string[]
    statusCode?: number | number[]
  }

  /**
   * Available parsers for filter dropdown
   * If not provided, fetched from API
   */
  availableParsers?: ParserInfo[]

  /**
   * Available queues for filter dropdown
   */
  availableQueues?: QueueInfo[]

  /**
   * Show filter bar
   * @default true
   */
  showFilters?: boolean

  /**
   * Enable search across all metrics
   * @default true
   */
  enableSearch?: boolean

  // ============================================================
  // Panel Configuration
  // ============================================================

  /**
   * Which panels to display
   * @default all panels enabled
   */
  panels?: {
    parserStats?: boolean        // Default: true
    ruleStats?: boolean          // Default: true
    queueHealth?: boolean        // Default: true
    errorBreakdown?: boolean     // Default: true
  }

  /**
   * Panel layout mode
   * @default 'grid'
   */
  layoutMode?: 'grid' | 'stacked' | 'tabs'

  /**
   * Allow users to rearrange panels (drag & drop)
   * @default false
   */
  allowPanelReordering?: boolean

  /**
   * Panel size/height configuration
   */
  panelHeight?: {
    parserStats?: string         // Default: '400px'
    ruleStats?: string           // Default: '400px'
    queueHealth?: string         // Default: '350px'
    errorBreakdown?: string      // Default: '350px'
  }

  // ============================================================
  // Alert Thresholds
  // ============================================================

  /**
   * Performance alert thresholds
   */
  thresholds?: {
    // Parser latency thresholds (milliseconds)
    parserLatency?: {
      warning: number              // Default: 2000 (2s)
      critical: number             // Default: 5000 (5s)
    }

    // Rule execution time thresholds (milliseconds)
    ruleExecutionTime?: {
      warning: number              // Default: 500 (500ms)
      critical: number             // Default: 1000 (1s)
    }

    // Queue depth thresholds (document count)
    queueDepth?: {
      warning: number              // Default: 100
      critical: number             // Default: 500
    }

    // Error rate thresholds (percentage)
    errorRate?: {
      warning: number              // Default: 5 (5%)
      critical: number             // Default: 10 (10%)
    }

    // Stuck documents threshold (minutes)
    stuckDocumentMinutes?: number  // Default: 30
  }

  /**
   * Show visual alerts when thresholds exceeded
   * @default true
   */
  showAlerts?: boolean

  /**
   * Play sound on critical alerts
   * @default false
   */
  playSoundAlerts?: boolean

  // ============================================================
  // Chart Configuration
  // ============================================================

  /**
   * Chart library to use
   * @default 'recharts'
   */
  chartLibrary?: 'recharts' | 'chart.js' | 'victory'

  /**
   * Chart color scheme
   * @default 'default'
   */
  colorScheme?: 'default' | 'colorblind' | 'high-contrast' | 'custom'

  /**
   * Custom colors (if colorScheme='custom')
   */
  customColors?: {
    primary?: string
    success?: string
    warning?: string
    critical?: string
    neutral?: string
  }

  /**
   * Show chart legends
   * @default true
   */
  showChartLegends?: boolean

  /**
   * Enable chart tooltips
   * @default true
   */
  enableChartTooltips?: boolean

  /**
   * Chart animation duration (milliseconds)
   * Set to 0 to disable animations
   * @default 300
   */
  chartAnimationDuration?: number

  // ============================================================
  // Data Source Configuration
  // ============================================================

  /**
   * API endpoint for fetching metrics
   * @example '/api/metrics/performance'
   */
  metricsApiEndpoint?: string

  /**
   * WebSocket URL for real-time updates
   * @example 'wss://api.example.com/metrics/stream'
   */
  websocketUrl?: string

  /**
   * Custom data fetcher function
   * Allows using custom API client or mocked data
   */
  dataFetcher?: (params: MetricsQueryParams) => Promise<PerformanceMetrics>

  /**
   * Enable data caching
   * @default true
   */
  enableCaching?: boolean

  /**
   * Cache TTL in seconds
   * @default 300 (5 minutes)
   */
  cacheTtl?: number

  // ============================================================
  // Export Configuration
  // ============================================================

  /**
   * Enable CSV export
   * @default true
   */
  enableCsvExport?: boolean

  /**
   * Enable JSON export
   * @default true
   */
  enableJsonExport?: boolean

  /**
   * Enable PDF export (requires jsPDF)
   * @default false
   */
  enablePdfExport?: boolean

  /**
   * Custom export filename prefix
   * @default 'performance-metrics'
   */
  exportFilenamePrefix?: string

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when time range changes
   * @param timeRange - New time range
   */
  onTimeRangeChange?: (timeRange: TimeRange) => void

  /**
   * Called when filters are updated
   * @param filters - New filter values
   */
  onFilterChange?: (filters: DashboardFilters) => void

  /**
   * Called when user drills down into specific metric
   * @param drillDownContext - Context for drill-down (parser/rule/queue)
   */
  onDrillDown?: (drillDownContext: DrillDownContext) => void

  /**
   * Called when alert threshold is breached
   * @param alert - Alert details
   */
  onAlertTriggered?: (alert: AlertEvent) => void

  /**
   * Called when user exports data
   * @param format - Export format (csv/json/pdf)
   * @param data - Exported data
   */
  onExport?: (format: 'csv' | 'json' | 'pdf', data: any) => void

  /**
   * Called when dashboard refresh completes
   * @param metrics - Updated metrics
   */
  onRefresh?: (metrics: PerformanceMetrics) => void

  /**
   * Called on API errors
   * @param error - Error object
   */
  onError?: (error: Error) => void

  // ============================================================
  // UI Options
  // ============================================================

  /**
   * Dashboard title
   * @default 'Performance Dashboard'
   */
  title?: string

  /**
   * Show dashboard header
   * @default true
   */
  showHeader?: boolean

  /**
   * Show time range picker in header
   * @default true
   */
  showTimeRangePicker?: boolean

  /**
   * Show refresh button/interval selector
   * @default true
   */
  showRefreshControls?: boolean

  /**
   * Show export buttons
   * @default true
   */
  showExportButtons?: boolean

  /**
   * Show full-screen toggle
   * @default true
   */
  showFullScreenToggle?: boolean

  /**
   * Theme (light/dark/auto)
   * @default 'auto'
   */
  theme?: 'light' | 'dark' | 'auto'

  /**
   * Compact mode (smaller panels, less padding)
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
   * Enable debug mode (shows query times, cache hits, etc.)
   * @default false
   */
  debugMode?: boolean

  /**
   * Enable performance profiling (React DevTools)
   * @default false
   */
  enableProfiling?: boolean

  /**
   * Locale for number/date formatting
   * @default 'en-US'
   */
  locale?: string

  /**
   * Timezone for timestamp display
   * @default 'UTC'
   */
  timezone?: string

  /**
   * Custom metric formatters
   */
  formatters?: {
    latency?: (ms: number) => string
    throughput?: (count: number) => string
    errorRate?: (rate: number) => string
    timestamp?: (date: Date) => string
  }
}

// ============================================================
// Supporting Types
// ============================================================

interface ParserInfo {
  id: string
  name: string
  version: string
  enabled: boolean
}

interface QueueInfo {
  name: string
  displayName: string
  priority: number
}

interface TimeRange {
  preset?: '1h' | '6h' | '24h' | '7d' | '30d'
  start?: Date
  end?: Date
}

interface DashboardFilters {
  parserId?: string[]
  ruleId?: string[]
  queueName?: string[]
  errorType?: string[]
  statusCode?: number[]
}

interface DrillDownContext {
  type: 'parser' | 'rule' | 'queue' | 'error'
  id: string
  name: string
  timeRange: TimeRange
  filters?: DashboardFilters
}

interface AlertEvent {
  id: string
  type: 'parser_latency' | 'rule_execution_time' | 'queue_depth' | 'error_rate' | 'stuck_documents'
  severity: 'warning' | 'critical'
  message: string
  value: number
  threshold: number
  timestamp: Date
  context?: any
}

interface PerformanceMetrics {
  parserStats: ParserStats
  ruleStats: RuleStats
  queueHealth: QueueHealth
  errorBreakdown: ErrorBreakdown
  metadata: {
    timeRange: TimeRange
    generatedAt: Date
    dataPoints: number
  }
}

interface ParserStats {
  summary: {
    totalExecutions: number
    avgLatency: number
    p50Latency: number
    p95Latency: number
    p99Latency: number
    throughput: number          // docs/second
    errorRate: number           // percentage
  }
  byParser: Array<{
    parserId: string
    parserName: string
    executions: number
    avgLatency: number
    p95Latency: number
    errorCount: number
    errorRate: number
    trend: 'up' | 'down' | 'stable'
  }>
  latencyTimeSeries: Array<{
    timestamp: Date
    p50: number
    p95: number
    p99: number
  }>
}

interface RuleStats {
  summary: {
    totalRuleExecutions: number
    avgExecutionTime: number
    slowRulesCount: number      // Rules exceeding threshold
    matchRate: number           // percentage
  }
  byRule: Array<{
    ruleId: string
    ruleName: string
    ruleType: 'exact' | 'regex' | 'fuzzy'
    executions: number
    avgTime: number
    p95Time: number
    matchCount: number
    matchRate: number
  }>
  executionTimeSeries: Array<{
    timestamp: Date
    avgTime: number
    maxTime: number
  }>
  ruleTypeDistribution: Array<{
    type: 'exact' | 'regex' | 'fuzzy'
    count: number
    percentage: number
  }>
}

interface QueueHealth {
  summary: {
    totalQueues: number
    healthyQueues: number
    warningQueues: number
    criticalQueues: number
    totalPending: number
    totalInProgress: number
    totalStuck: number
  }
  byQueue: Array<{
    queueName: string
    status: 'healthy' | 'warning' | 'critical'
    pending: number
    inProgress: number
    stuck: number
    processingRate: number      // docs/minute
    avgProcessingTime: number   // milliseconds
    oldestDocumentAge: number   // minutes
  }>
  depthTimeSeries: Array<{
    timestamp: Date
    pending: number
    inProgress: number
    stuck: number
  }>
  processingRateTimeSeries: Array<{
    timestamp: Date
    rate: number
  }>
}

interface ErrorBreakdown {
  summary: {
    totalErrors: number
    uniqueErrorTypes: number
    errorRate: number           // percentage
  }
  byType: Array<{
    errorType: string
    errorMessage: string
    count: number
    percentage: number
    firstSeen: Date
    lastSeen: Date
    trend: 'increasing' | 'decreasing' | 'stable'
  }>
  byParser: Array<{
    parserId: string
    parserName: string
    errorCount: number
    topErrors: Array<{
      type: string
      count: number
    }>
  }>
  errorTimeSeries: Array<{
    timestamp: Date
    errorCount: number
  }>
}

interface MetricsQueryParams {
  timeRange: TimeRange
  filters?: DashboardFilters
  granularity?: 'minute' | 'hour' | 'day'
}

// ============================================================
// Component State
// ============================================================

interface PerformanceDashboardState {
  // Data state
  metrics: PerformanceMetrics | null
  loading: boolean
  error: Error | null
  lastUpdated: Date | null

  // UI state
  timeRange: TimeRange
  filters: DashboardFilters
  selectedPanel: string | null
  expandedPanel: string | null

  // Real-time state
  isRealTimeEnabled: boolean
  websocketConnected: boolean
  refreshCountdown: number        // Seconds until next refresh

  // Alert state
  activeAlerts: AlertEvent[]
  mutedAlerts: Set<string>        // Alert IDs that are muted

  // Export state
  exporting: boolean
  exportFormat: 'csv' | 'json' | 'pdf' | null

  // Layout state
  panelLayout: PanelLayout[]
  fullScreen: boolean
}

interface PanelLayout {
  id: string
  name: string
  order: number
  visible: boolean
  collapsed: boolean
}
```

---

## Architecture

### Component Hierarchy

```
PerformanceDashboard
├── DashboardHeader
│   ├── TitleSection (title, subtitle)
│   ├── TimeRangePicker (1h/6h/24h/7d/30d)
│   ├── RefreshControls (auto-refresh toggle, interval selector)
│   ├── FilterBar
│   │   ├── ParserFilter (multi-select dropdown)
│   │   ├── RuleFilter (multi-select dropdown)
│   │   ├── QueueFilter (multi-select dropdown)
│   │   └── SearchInput (search across all metrics)
│   └── ActionButtons
│       ├── ExportButton (CSV/JSON/PDF)
│       ├── FullScreenToggle
│       └── RefreshButton
├── AlertBanner (shows active alerts)
├── DashboardGrid (4 panels in 2x2 layout)
│   ├── ParserStatsPanel
│   │   ├── PanelHeader (title, collapse toggle)
│   │   ├── SummaryMetrics
│   │   │   ├── MetricCard (Total Executions)
│   │   │   ├── MetricCard (Avg Latency)
│   │   │   ├── MetricCard (P95 Latency)
│   │   │   └── MetricCard (Error Rate)
│   │   ├── LatencyChart (line chart with p50/p95/p99)
│   │   └── ParserTable (sortable, paginated)
│   ├── RuleStatsPanel
│   │   ├── PanelHeader
│   │   ├── SummaryMetrics
│   │   │   ├── MetricCard (Total Rule Executions)
│   │   │   ├── MetricCard (Avg Execution Time)
│   │   │   ├── MetricCard (Slow Rules Count)
│   │   │   └── MetricCard (Match Rate)
│   │   ├── ExecutionTimeChart (area chart)
│   │   ├── RuleTable (with drill-down)
│   │   └── RuleTypeDistribution (pie chart)
│   ├── QueueHealthPanel
│   │   ├── PanelHeader
│   │   ├── SummaryMetrics
│   │   │   ├── MetricCard (Healthy Queues)
│   │   │   ├── MetricCard (Warning Queues)
│   │   │   ├── MetricCard (Critical Queues)
│   │   │   └── MetricCard (Total Stuck)
│   │   ├── QueueDepthChart (stacked area chart)
│   │   ├── ProcessingRateChart (line chart)
│   │   └── QueueList (cards with health badges)
│   └── ErrorBreakdownPanel
│       ├── PanelHeader
│       ├── SummaryMetrics
│       │   ├── MetricCard (Total Errors)
│       │   ├── MetricCard (Unique Error Types)
│       │   └── MetricCard (Error Rate)
│       ├── ErrorTimeSeriesChart (line chart)
│       ├── ErrorTypeTable (sortable by count)
│       └── ErrorDistributionPieChart
└── DashboardFooter
    ├── StatusBar (last updated, data points, connection status)
    └── LegendPanel (color coding reference)
```

### Data Flow

```
Component Mount
    ↓
Fetch Initial Metrics
    ↓
    ├── API Call: GET /api/metrics/performance?timeRange=24h
    │   └── Returns: ParserStats, RuleStats, QueueHealth, ErrorBreakdown
    ↓
Parse & Transform Data
    ↓
    ├── Calculate percentiles (p50, p95, p99)
    ├── Aggregate by parser/rule/queue
    ├── Detect threshold breaches
    └── Format time series for charts
    ↓
Render Dashboard
    ↓
    ├── Render 4 panels with charts
    ├── Show alerts if thresholds exceeded
    └── Start auto-refresh timer
    ↓
Auto-Refresh Loop (every 60s)
    ↓
    ├── Fetch updated metrics
    ├── Merge with cached data
    ├── Update charts with new data points
    └── Check for new alerts

User Interaction: Time Range Change
    ↓
Update timeRange state
    ↓
Re-fetch metrics with new time range
    ↓
Update all charts

User Interaction: Filter Change
    ↓
Update filters state
    ↓
Filter metrics client-side (or re-fetch from API)
    ↓
Update charts with filtered data

User Interaction: Drill-Down
    ↓
Capture click event (e.g., click on parser row)
    ↓
Call onDrillDown(context)
    ↓
External handler navigates to detail view
```

### State Management

The PerformanceDashboard uses React hooks and context for state management:

1. **Local State**: Manages UI state (filters, time range, panel expansion)
2. **React Query**: Handles data fetching, caching, and auto-refresh
3. **WebSocket Context**: Manages real-time updates (optional)
4. **Alert Context**: Tracks active alerts and muted alerts across sessions

---

## Dashboard Panels

### 1. Parser Stats Panel

Displays performance metrics for document parsers.

**Metrics:**
- **Total Executions**: Count of parser invocations in time range
- **Avg Latency**: Mean processing time across all parsers
- **P95 Latency**: 95th percentile latency (most important for SLA)
- **Error Rate**: Percentage of failed parser executions

**Charts:**
- **Latency Time Series**: Line chart with 3 lines (p50, p95, p99)
  - X-axis: Time
  - Y-axis: Latency (ms)
  - Colors: Green (p50), Yellow (p95), Red (p99)
  - Threshold lines: Warning (2s), Critical (5s)

- **Parser Table**: Sortable table with columns:
  - Parser Name
  - Executions (count)
  - Avg Latency (ms)
  - P95 Latency (ms)
  - Error Count
  - Error Rate (%)
  - Trend (↑/↓/→)
  - Actions (drill-down button)

**Interactions:**
- Click parser row → drill down to parser detail page
- Sort table by column (latency, error rate, etc.)
- Filter by parser name (search)

### 2. Rule Stats Panel

Displays performance metrics for normalization rules (merchant matching, categorization).

**Metrics:**
- **Total Rule Executions**: Count of rule evaluations
- **Avg Execution Time**: Mean time to execute a rule
- **Slow Rules Count**: Number of rules exceeding threshold (>500ms)
- **Match Rate**: Percentage of successful rule matches

**Charts:**
- **Execution Time Area Chart**: Shows execution time trends
  - X-axis: Time
  - Y-axis: Execution time (ms)
  - Areas: Min, Avg, Max execution times

- **Rule Table**: Sortable table with columns:
  - Rule Name
  - Type (exact/regex/fuzzy)
  - Executions (count)
  - Avg Time (ms)
  - P95 Time (ms)
  - Match Count
  - Match Rate (%)
  - Actions (optimize button)

- **Rule Type Distribution Pie Chart**:
  - Segments: Exact (green), Regex (yellow), Fuzzy (red)
  - Tooltip: Count and percentage

**Interactions:**
- Click rule row → drill down to rule detail page
- Filter by rule type (exact/regex/fuzzy)
- Sort by execution time or match rate

### 3. Queue Health Panel

Monitors document processing queues.

**Metrics:**
- **Healthy Queues**: Count of queues with depth < warning threshold
- **Warning Queues**: Count of queues with depth between warning and critical
- **Critical Queues**: Count of queues with depth > critical threshold
- **Total Stuck**: Count of documents stuck for >30 minutes

**Charts:**
- **Queue Depth Stacked Area Chart**:
  - X-axis: Time
  - Y-axis: Document count
  - Areas: Pending (blue), In Progress (yellow), Stuck (red)

- **Processing Rate Line Chart**:
  - X-axis: Time
  - Y-axis: Documents per minute
  - Line: Current processing rate
  - Reference line: Expected rate

- **Queue List (Cards)**:
  - Each queue is a card with:
    - Queue name
    - Health badge (🟢 Healthy / 🟡 Warning / 🔴 Critical)
    - Pending count
    - In progress count
    - Stuck count
    - Processing rate (docs/min)
    - Oldest document age
    - Actions (view stuck docs, pause/resume)

**Interactions:**
- Click queue card → drill down to queue detail page
- Filter queues by status (healthy/warning/critical)
- Click "View Stuck Docs" → modal with list of stuck documents

### 4. Error Breakdown Panel

Analyzes error patterns across the system.

**Metrics:**
- **Total Errors**: Count of errors in time range
- **Unique Error Types**: Number of distinct error types
- **Error Rate**: Percentage of failed operations

**Charts:**
- **Error Time Series Line Chart**:
  - X-axis: Time
  - Y-axis: Error count
  - Line: Total errors over time
  - Shaded area: Error trend

- **Error Type Table**: Sortable table with columns:
  - Error Type
  - Error Message (truncated with tooltip)
  - Count
  - Percentage
  - First Seen
  - Last Seen
  - Trend (increasing/decreasing/stable)
  - Actions (view logs button)

- **Error Distribution Pie Chart**:
  - Segments: Top 5 error types + "Other"
  - Colors: Distinct color per error type
  - Tooltip: Error type, count, percentage

**Interactions:**
- Click error type row → drill down to error logs
- Filter by parser or time range
- Export error report (CSV)

---

## Chart Components

### Chart Configuration (Recharts)

All charts use Recharts library with consistent styling:

```typescript
const chartConfig = {
  margin: { top: 10, right: 30, left: 0, bottom: 0 },
  colors: {
    primary: '#3B82F6',      // Blue
    success: '#10B981',      // Green
    warning: '#F59E0B',      // Yellow
    critical: '#EF4444',     // Red
    neutral: '#6B7280'       // Gray
  },
  fontSize: 12,
  fontFamily: 'Inter, sans-serif',
  grid: {
    stroke: '#E5E7EB',
    strokeDasharray: '3 3'
  },
  tooltip: {
    backgroundColor: '#1F2937',
    borderRadius: 8,
    border: 'none',
    color: '#FFFFFF'
  }
}
```

### Responsive Chart Sizing

```typescript
const useChartDimensions = () => {
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 })

  useEffect(() => {
    const updateDimensions = () => {
      const container = document.getElementById('chart-container')
      if (container) {
        setDimensions({
          width: container.offsetWidth,
          height: container.offsetHeight
        })
      }
    }

    updateDimensions()
    window.addEventListener('resize', updateDimensions)
    return () => window.removeEventListener('resize', updateDimensions)
  }, [])

  return dimensions
}
```

---

## Visual Wireframes

### Wireframe 1: Default Dashboard View (Desktop)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Performance Dashboard                                    [Full Screen] [⟳] │
├────────────────────────────────────────────────────────────────────────────┤
│ Time Range: [1h] [6h] [●24h] [7d] [30d] | Auto-refresh: ●60s | Last updated: Just now
│ Filters: Parser [All ▼] Rule [All ▼] Queue [All ▼] | Search: [             ] [CSV] [JSON]
├────────────────────────────────────────────────────────────────────────────┤
│ 🟡 WARNING: 2 queues have high pending counts (>100 documents)              │
├───────────────────────────────────┬────────────────────────────────────────┤
│ PARSER STATS                   [↕]│ RULE STATS                          [↕]│
├───────────────────────────────────┼────────────────────────────────────────┤
│ ┌─────────┬─────────┬─────────┐  │ ┌─────────┬─────────┬─────────┐       │
│ │ 45,678  │ 1.2s    │ 2.8s    │  │ │ 123,456 │ 245ms   │ 12      │       │
│ │ Total   │ Avg     │ P95     │  │ │ Total   │ Avg     │ Slow    │       │
│ │ Execs   │ Latency │ Latency │  │ │ Execs   │ Time    │ Rules   │       │
│ └─────────┴─────────┴─────────┘  │ └─────────┴─────────┴─────────┘       │
│                                   │                                        │
│ Latency Over Time (24h)           │ Execution Time (24h)                   │
│ ┌─────────────────────────────┐   │ ┌─────────────────────────────────┐   │
│ │3s─                       ●──│   │ │800─            ▓▓▓▓             │   │
│ │  │                     ●●   │   │ │   │         ▓▓▓    ▓▓▓          │   │
│ │2s─ - - - - - - - - ●●●      │   │ │600─      ▓▓▓          ▓▓▓       │   │
│ │  │               ●●         │   │ │   │   ▓▓▓                ▓▓▓    │   │
│ │1s─         ●●●●●            │   │ │400─▓▓▓                      ▓▓  │   │
│ │  │  ●●●●●●●                 │   │ │   ├────────────────────────────┤   │
│ │0s─●●                        │   │ │   0h    6h    12h   18h   24h   │   │
│ │  ├────────────────────────┐│   │ └─────────────────────────────────┘   │
│ │  0h   6h   12h  18h   24h ││   │                                        │
│ └─────────────────────────────┘   │ Top Slow Rules                         │
│ — P99  — P95  — P50               │ ┌─────────────────────────┬────────┐  │
│                                   │ │ Merchant Fuzzy Match    │ 1,234ms│  │
│ By Parser                         │ │ Category Regex          │   892ms│  │
│ ┌──────────────────┬───────────┐  │ │ Tax Rule Evaluator      │   756ms│  │
│ │ PDF Parser    ↑  │ 2.4s  3.2%│  │ │ Currency Converter      │   623ms│  │
│ │ CSV Parser    →  │ 0.8s  0.5%│  │ └─────────────────────────┴────────┘  │
│ │ XML Parser    ↓  │ 1.1s  1.2%│  │                                        │
│ └──────────────────┴───────────┘  │                                        │
├───────────────────────────────────┼────────────────────────────────────────┤
│ QUEUE HEALTH                   [↕]│ ERROR BREAKDOWN                     [↕]│
├───────────────────────────────────┼────────────────────────────────────────┤
│ ┌─────────┬─────────┬─────────┐  │ ┌─────────┬─────────┬─────────┐       │
│ │ 🟢 8    │ 🟡 2    │ 🔴 0    │  │ │ 1,234   │ 47      │ 2.7%    │       │
│ │ Healthy │ Warning │ Critical│  │ │ Total   │ Unique  │ Error   │       │
│ │ Queues  │ Queues  │ Queues  │  │ │ Errors  │ Types   │ Rate    │       │
│ └─────────┴─────────┴─────────┘  │ └─────────┴─────────┴─────────┘       │
│                                   │                                        │
│ Queue Depth (24h)                 │ Errors Over Time (24h)                 │
│ ┌─────────────────────────────┐   │ ┌─────────────────────────────────┐   │
│ │500─                         │   │ │100─              ●              │   │
│ │   │                    ▓▓▓▓▓│   │ │   │            ●   ●            │   │
│ │400─               ▓▓▓▓▓▒▒▒▒▒│   │ │ 80─          ●       ●          │   │
│ │   │          ▓▓▓▓▓▒▒▒▒▒░░░░░│   │ │   │        ●           ●        │   │
│ │300─     ▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░│   │ │ 60─      ●               ●      │   │
│ │   │▓▓▓▓▓▒▒▒▒▒░░░░░░░░░░░░░░░│   │ │   │    ●                   ●    │   │
│ │200─▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░│   │ │ 40─  ●                       ●  │   │
│ │   ├────────────────────────┐│   │ │   ├────────────────────────────┤   │
│ │   0h   6h   12h  18h   24h ││   │ │   0h    6h    12h   18h   24h   │   │
│ └─────────────────────────────┘   │ └─────────────────────────────────┘   │
│ ▓ Stuck ▒ In Progress ░ Pending   │                                        │
│                                   │ Top Errors                             │
│ Queues                            │ ┌────────────────────────┬────────┐   │
│ ┌─────────────────────────────┐   │ │ PDF Parse Timeout      │ 456 ●  │   │
│ │ 🟢 invoice-processing       │   │ │ Invalid Date Format    │ 234 ●  │   │
│ │    Pending: 45  In Prog: 12 │   │ │ Missing Required Field │ 189 ●  │   │
│ │ 🟡 receipt-ocr  WARNING     │   │ │ Network Timeout        │ 123 ●  │   │
│ │    Pending: 156  In Prog: 8 │   │ │ File Not Found         │  89 ●  │   │
│ │ 🟢 transaction-import       │   │ └────────────────────────┴────────┘   │
│ │    Pending: 23  In Prog: 5  │   │ ● Increasing  ● Stable  ● Decreasing  │
│ └─────────────────────────────┘   │                                        │
└───────────────────────────────────┴────────────────────────────────────────┘
 Last updated: 2025-10-25 14:32:15 UTC | 8,456 data points | ● Connected
```

### Wireframe 2: Mobile View (Stacked Panels)

```
┌──────────────────────────────────────┐
│ Performance Dashboard           [☰] │
├──────────────────────────────────────┤
│ [●24h▼] | Auto: ●60s | [⟳]          │
│ Filters: [All ▼] [All ▼]            │
├──────────────────────────────────────┤
│ 🟡 2 queues have high pending counts │
├──────────────────────────────────────┤
│ PARSER STATS                      [↕]│
├──────────────────────────────────────┤
│ ┌──────────┬──────────┬──────────┐  │
│ │ 45,678   │ 1.2s     │ 2.8s     │  │
│ │ Execs    │ Avg      │ P95      │  │
│ └──────────┴──────────┴──────────┘  │
│                                      │
│ Latency (24h)                        │
│ ┌──────────────────────────────────┐ │
│ │3s─                        ●●●●   │ │
│ │  │                    ●●●●       │ │
│ │2s─ - - - - - - - ●●●●            │ │
│ │  │            ●●●●               │ │
│ │1s─      ●●●●●●                   │ │
│ │  │●●●●●●                         │ │
│ │  ├──────────────────────────────┤ │
│ │  0h        12h             24h   │ │
│ └──────────────────────────────────┘ │
│                                      │
│ [View Details →]                     │
├──────────────────────────────────────┤
│ RULE STATS                        [↕]│
├──────────────────────────────────────┤
│ ┌──────────┬──────────┬──────────┐  │
│ │ 123,456  │ 245ms    │ 12       │  │
│ │ Execs    │ Avg      │ Slow     │  │
│ └──────────┴──────────┴──────────┘  │
│                                      │
│ Top Slow Rules                       │
│ ┌────────────────────────┬────────┐ │
│ │ Merchant Fuzzy Match   │ 1,234ms│ │
│ │ Category Regex         │   892ms│ │
│ │ Tax Rule Evaluator     │   756ms│ │
│ └────────────────────────┴────────┘ │
│                                      │
│ [View Details →]                     │
├──────────────────────────────────────┤
│ QUEUE HEALTH                      [↕]│
├──────────────────────────────────────┤
│ ┌──────────┬──────────┬──────────┐  │
│ │ 🟢 8     │ 🟡 2     │ 🔴 0     │  │
│ │ Healthy  │ Warning  │ Critical │  │
│ └──────────┴──────────┴──────────┘  │
│                                      │
│ ┌──────────────────────────────────┐ │
│ │ 🟢 invoice-processing            │ │
│ │    Pending: 45  In Prog: 12      │ │
│ │ 🟡 receipt-ocr  WARNING          │ │
│ │    Pending: 156  In Prog: 8      │ │
│ └──────────────────────────────────┘ │
│                                      │
│ [View Details →]                     │
├──────────────────────────────────────┤
│ ERROR BREAKDOWN                   [↕]│
├──────────────────────────────────────┤
│ ┌──────────┬──────────┬──────────┐  │
│ │ 1,234    │ 47       │ 2.7%     │  │
│ │ Errors   │ Types    │ Rate     │  │
│ └──────────┴──────────┴──────────┘  │
│                                      │
│ Top Errors                           │
│ ┌────────────────────────┬────────┐ │
│ │ PDF Parse Timeout      │ 456 ●  │ │
│ │ Invalid Date Format    │ 234 ●  │ │
│ │ Missing Required Field │ 189 ●  │ │
│ └────────────────────────┴────────┘ │
│                                      │
│ [View Details →]                     │
└──────────────────────────────────────┘
```

### Wireframe 3: Drill-Down View (Parser Detail)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ ← Back to Dashboard | PDF Parser Performance                               │
├────────────────────────────────────────────────────────────────────────────┤
│ Time Range: [●24h] [7d] [30d] | Export: [CSV] [JSON]                      │
├────────────────────────────────────────────────────────────────────────────┤
│ OVERVIEW                                                                   │
├────────────────────────────────────────────────────────────────────────────┤
│ ┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐      │
│ │ 12,345   │ 2.4s     │ 3.8s     │ 5.2s     │ 389      │ 3.2%     │      │
│ │ Execs    │ Avg      │ P95      │ P99      │ Errors   │ Error %  │      │
│ └──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘      │
│                                                                            │
│ LATENCY PERCENTILES (24h)                                                 │
│ ┌──────────────────────────────────────────────────────────────────────┐  │
│ │ 6s─                                                            ●●    │  │
│ │   │                                                        ●●●●      │  │
│ │ 5s─ - - - - - - - - - - - - - - - - - - - - - - - - - ●●●●          │  │
│ │   │                                                  ●●●●             │  │
│ │ 4s─                                            ●●●●●●                 │  │
│ │   │                                      ●●●●●●                       │  │
│ │ 3s─                                ●●●●●●                             │  │
│ │   │                          ●●●●●●                                   │  │
│ │ 2s─ - - - - - - - - - - ●●●●●●                                        │  │
│ │   │                ●●●●●●                                             │  │
│ │ 1s─          ●●●●●●                                                   │  │
│ │   │    ●●●●●●                                                         │  │
│ │ 0s─●●●●                                                               │  │
│ │   ├──────────────────────────────────────────────────────────────────┤  │
│ │   0h        6h         12h         18h         24h                    │  │
│ └──────────────────────────────────────────────────────────────────────┘  │
│ — P99  — P95  — P50                                                       │
│                                                                            │
│ SLOWEST EXECUTIONS (Last 24h)                                             │
│ ┌─────────────────────┬────────┬────────┬─────────────────────────────┐  │
│ │ Document ID         │ Size   │ Latency│ Timestamp                   │  │
│ ├─────────────────────┼────────┼────────┼─────────────────────────────┤  │
│ │ doc_abc123          │ 12.3MB │ 8,234ms│ 2025-10-25 14:15:23 UTC  [↗]│  │
│ │ doc_def456          │  9.8MB │ 7,892ms│ 2025-10-25 13:42:11 UTC  [↗]│  │
│ │ doc_ghi789          │ 15.1MB │ 7,654ms│ 2025-10-25 12:33:45 UTC  [↗]│  │
│ │ doc_jkl012          │ 11.2MB │ 7,123ms│ 2025-10-25 11:28:09 UTC  [↗]│  │
│ │ doc_mno345          │ 13.5MB │ 6,987ms│ 2025-10-25 10:15:32 UTC  [↗]│  │
│ └─────────────────────┴────────┴────────┴─────────────────────────────┘  │
│                                                                            │
│ ERROR BREAKDOWN                                                            │
│ ┌────────────────────────────────┬────────┬────────────────────────────┐  │
│ │ Error Type                     │ Count  │ Last Seen                  │  │
│ ├────────────────────────────────┼────────┼────────────────────────────┤  │
│ │ PDF Parse Timeout              │    234 │ 2025-10-25 14:30:12 UTC [↗]│  │
│ │ Corrupted PDF Structure        │     89 │ 2025-10-25 14:25:45 UTC [↗]│  │
│ │ Unsupported PDF Version        │     45 │ 2025-10-25 14:18:33 UTC [↗]│  │
│ │ Out of Memory                  │     21 │ 2025-10-25 13:55:22 UTC [↗]│  │
│ └────────────────────────────────┴────────┴────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## States & Lifecycle

### Component States

```typescript
type DashboardState =
  | 'loading'           // Initial data fetch
  | 'ready'             // Data loaded, dashboard interactive
  | 'refreshing'        // Auto-refresh in progress
  | 'filtering'         // Applying filters
  | 'exporting'         // Exporting data
  | 'error'             // API error occurred
  | 'disconnected'      // WebSocket disconnected
```

### State Transitions

```
┌──────────┐
│ Loading  │ ← Component mounts, fetch initial metrics
└────┬─────┘
     │
     │ Data fetched successfully
     ▼
┌──────────┐
│  Ready   │ ← Dashboard interactive, charts rendered
└────┬─────┘
     │
     ├─ Auto-refresh timer (60s) ──────────┐
     │                                     │
     │ User changes time range             ▼
     ▼                             ┌──────────────┐
┌──────────┐                       │ Refreshing   │
│Filtering │                       └──────┬───────┘
└────┬─────┘                              │
     │                                    │ Success
     │ Filters applied                    ▼
     ▼                             ┌──────────┐
┌──────────┐                       │  Ready   │
│  Ready   │                       └──────────┘
└────┬─────┘
     │
     │ User clicks export
     ▼
┌──────────┐
│Exporting │
└────┬─────┘
     │
     │ Export complete
     ▼
┌──────────┐
│  Ready   │
└──────────┘

Error Path:
API Error → Error State → Show error message → Retry button → Loading
```

### Auto-Refresh Countdown

Visual countdown in header:

```
Auto-refresh in: 59s... 58s... 57s... → 0s → [Refreshing...] → Ready
```

---

## Multi-Domain Examples

### 1. Finance: Transaction Processing Dashboard

**Context:** A fintech company monitors transaction processing performance across multiple payment gateways.

**Configuration:**

```typescript
<PerformanceDashboard
  title="Transaction Processing Performance"
  initialTimeRange="24h"
  refreshInterval={60}
  thresholds={{
    parserLatency: { warning: 1000, critical: 3000 },
    queueDepth: { warning: 50, critical: 200 }
  }}
  initialFilters={{
    parserId: ['stripe_parser', 'paypal_parser', 'crypto_parser']
  }}
  panels={{
    parserStats: true,
    ruleStats: true,
    queueHealth: true,
    errorBreakdown: true
  }}
/>
```

**Sample Metrics:**

```json
{
  "parserStats": {
    "summary": {
      "totalExecutions": 456789,
      "avgLatency": 234,
      "p95Latency": 892,
      "errorRate": 0.5
    },
    "byParser": [
      {
        "parserId": "stripe_parser",
        "parserName": "Stripe Transaction Parser",
        "executions": 289456,
        "avgLatency": 189,
        "p95Latency": 678,
        "errorRate": 0.3
      },
      {
        "parserId": "paypal_parser",
        "parserName": "PayPal Transaction Parser",
        "executions": 123456,
        "avgLatency": 312,
        "p95Latency": 1234,
        "errorRate": 0.8
      }
    ]
  },
  "ruleStats": {
    "byRule": [
      {
        "ruleId": "merchant_normalization",
        "ruleName": "Merchant Name Normalization",
        "ruleType": "fuzzy",
        "executions": 456789,
        "avgTime": 45,
        "matchRate": 87.5
      }
    ]
  }
}
```

**Use Case:**
- Monitor payment gateway latency
- Identify slow merchant normalization rules
- Track transaction queue backlogs during peak hours
- Analyze error patterns by payment method

### 2. Healthcare: Claim Processing Dashboard

**Context:** A health insurance company processes medical claims from various sources (EDI, portal, fax).

**Configuration:**

```typescript
<PerformanceDashboard
  title="Claims Processing Performance"
  initialTimeRange="7d"
  refreshInterval={120}
  thresholds={{
    parserLatency: { warning: 5000, critical: 10000 },
    queueDepth: { warning: 500, critical: 2000 }
  }}
  initialFilters={{
    parserId: ['edi_837_parser', 'hcfa_1500_parser', 'ub_04_parser']
  }}
/>
```

**Sample Metrics:**

```json
{
  "parserStats": {
    "byParser": [
      {
        "parserId": "edi_837_parser",
        "parserName": "EDI 837 Claims Parser",
        "executions": 89234,
        "avgLatency": 1234,
        "p95Latency": 3456,
        "errorRate": 1.2
      },
      {
        "parserId": "hcfa_1500_parser",
        "parserName": "HCFA-1500 Form Parser",
        "executions": 45678,
        "avgLatency": 2345,
        "p95Latency": 5678,
        "errorRate": 3.5
      }
    ]
  },
  "errorBreakdown": {
    "byType": [
      {
        "errorType": "MISSING_DIAGNOSIS_CODE",
        "count": 234,
        "percentage": 15.2
      },
      {
        "errorType": "INVALID_PROVIDER_NPI",
        "count": 189,
        "percentage": 12.3
      }
    ]
  }
}
```

**Use Case:**
- Monitor claims processing latency by source type
- Identify validation rules causing delays
- Track queue backlogs during open enrollment
- Analyze error patterns (missing fields, invalid codes)

### 3. Legal: Document Processing Dashboard

**Context:** A law firm processes legal documents (contracts, depositions, discovery) for case management.

**Configuration:**

```typescript
<PerformanceDashboard
  title="Legal Document Processing"
  initialTimeRange="24h"
  refreshInterval={60}
  thresholds={{
    parserLatency: { warning: 3000, critical: 8000 },
    queueDepth: { warning: 20, critical: 100 }
  }}
  initialFilters={{
    parserId: ['contract_parser', 'deposition_parser', 'email_parser']
  }}
/>
```

**Sample Metrics:**

```json
{
  "parserStats": {
    "byParser": [
      {
        "parserId": "contract_parser",
        "parserName": "Contract Document Parser",
        "executions": 1234,
        "avgLatency": 4567,
        "p95Latency": 8912,
        "errorRate": 2.1
      }
    ]
  },
  "ruleStats": {
    "byRule": [
      {
        "ruleId": "party_name_extraction",
        "ruleName": "Contract Party Name Extraction",
        "ruleType": "regex",
        "executions": 1234,
        "avgTime": 123,
        "matchRate": 92.3
      }
    ]
  }
}
```

**Use Case:**
- Monitor contract parsing latency (often large PDFs)
- Track entity extraction rule performance (parties, dates, amounts)
- Identify stuck documents in OCR queue
- Analyze parsing errors by document type

### 4. Research (RSRCH - Utilitario): Web Scraping Pipeline Dashboard

**Context:** RSRCH team monitors fact extraction from web sources (TechCrunch, podcasts, Twitter).

**Configuration:**

```typescript
<PerformanceDashboard
  title="RSRCH Fact Discovery Pipeline"
  initialTimeRange="7d"
  refreshInterval={300}
  thresholds={{
    parserLatency: { warning: 10000, critical: 30000 },
    queueDepth: { warning: 10, critical: 50 }
  }}
  initialFilters={{
    parserId: ['techcrunch_web_parser', 'podcast_transcript_parser', 'twitter_api_parser']
  }}
/>
```

**Sample Metrics:**

```json
{
  "parserStats": {
    "byParser": [
      {
        "parserId": "genomic_data_parser",
        "parserName": "Genomic Sequencing Data Parser",
        "executions": 234,
        "avgLatency": 45678,
        "p95Latency": 89234,
        "errorRate": 1.5
      }
    ]
  }
}
```

**Use Case:**
- Monitor large file processing (genomic data, imaging files)
- Track data validation rule performance
- Identify slow quality control checks
- Analyze error patterns in data ingestion

### 5. E-commerce: Order Processing Dashboard

**Context:** An e-commerce platform processes orders from multiple sales channels (web, mobile, marketplace).

**Configuration:**

```typescript
<PerformanceDashboard
  title="Order Processing Performance"
  initialTimeRange="24h"
  refreshInterval={30}
  thresholds={{
    parserLatency: { warning: 500, critical: 2000 },
    queueDepth: { warning: 200, critical: 1000 }
  }}
  initialFilters={{
    parserId: ['shopify_parser', 'amazon_parser', 'ebay_parser']
  }}
/>
```

**Sample Metrics:**

```json
{
  "parserStats": {
    "byParser": [
      {
        "parserId": "shopify_parser",
        "parserName": "Shopify Order Parser",
        "executions": 123456,
        "avgLatency": 234,
        "p95Latency": 567,
        "errorRate": 0.3
      }
    ]
  },
  "queueHealth": {
    "byQueue": [
      {
        "queueName": "order_validation",
        "status": "healthy",
        "pending": 45,
        "inProgress": 12,
        "stuck": 0,
        "processingRate": 234
      }
    ]
  }
}
```

**Use Case:**
- Monitor order parsing latency by sales channel
- Track validation rule performance (inventory check, fraud detection)
- Identify queue backlogs during flash sales
- Analyze error patterns (invalid addresses, out-of-stock items)

### 6. SaaS: API Performance Dashboard

**Context:** A SaaS platform monitors API request processing performance.

**Configuration:**

```typescript
<PerformanceDashboard
  title="API Performance Dashboard"
  initialTimeRange="1h"
  refreshInterval={30}
  thresholds={{
    parserLatency: { warning: 200, critical: 1000 },
    queueDepth: { warning: 100, critical: 500 }
  }}
  initialFilters={{
    parserId: ['rest_api_parser', 'graphql_parser', 'webhook_parser']
  }}
/>
```

**Sample Metrics:**

```json
{
  "parserStats": {
    "byParser": [
      {
        "parserId": "graphql_parser",
        "parserName": "GraphQL Query Parser",
        "executions": 345678,
        "avgLatency": 45,
        "p95Latency": 123,
        "errorRate": 0.5
      }
    ]
  }
}
```

**Use Case:**
- Monitor API endpoint latency
- Track query parsing and validation performance
- Identify slow rate limiting rules
- Analyze error patterns by API version

### 7. Insurance: Claims Processing Dashboard

**Context:** An insurance company processes claims from multiple sources (online portal, agent submissions, third-party integrations).

**Configuration:**

```typescript
<PerformanceDashboard
  title="Insurance Claims Performance"
  initialTimeRange="24h"
  refreshInterval={60}
  thresholds={{
    parserLatency: { warning: 2000, critical: 5000 },
    queueDepth: { warning: 100, critical: 500 }
  }}
  initialFilters={{
    parserId: ['acord_parser', 'claims_portal_parser', 'fnol_parser']
  }}
/>
```

**Sample Metrics:**

```json
{
  "parserStats": {
    "byParser": [
      {
        "parserId": "acord_parser",
        "parserName": "ACORD XML Claims Parser",
        "executions": 12345,
        "avgLatency": 1234,
        "p95Latency": 2345,
        "errorRate": 1.8
      }
    ]
  },
  "ruleStats": {
    "byRule": [
      {
        "ruleId": "claim_amount_validation",
        "ruleName": "Claim Amount Validation",
        "ruleType": "exact",
        "executions": 12345,
        "avgTime": 12,
        "matchRate": 98.7
      }
    ]
  }
}
```

**Use Case:**
- Monitor claims parsing latency by submission method
- Track validation rule performance (coverage limits, deductibles)
- Identify stuck claims in fraud detection queue
- Analyze error patterns (missing policy numbers, invalid dates)

---

## User Interactions

### 1. Changing Time Range

**Trigger:** User clicks time range button (1h/6h/24h/7d/30d)

**Flow:**
1. User clicks "7d" button
2. Update `timeRange` state to `{ preset: '7d' }`
3. Show loading overlay on all panels
4. Fetch metrics: `GET /api/metrics/performance?timeRange=7d`
5. Parse response and update charts
6. Hide loading overlay
7. Update "Last updated" timestamp
8. Call `onTimeRangeChange({ preset: '7d' })`

**Visual Feedback:**
- Active button highlighted (blue background)
- Loading skeleton on charts
- Progress bar in header

**Error Handling:**
- If API fails, show error toast
- Keep previous data visible
- Show "Retry" button

### 2. Applying Filters

**Trigger:** User selects parser from filter dropdown

**Flow:**
1. User opens "Parser" filter dropdown
2. Multi-select dropdown shows available parsers
3. User checks "PDF Parser" and "CSV Parser"
4. Click "Apply" button
5. Update `filters` state: `{ parserId: ['pdf_parser', 'csv_parser'] }`
6. Filter metrics client-side (or re-fetch from API)
7. Update all charts with filtered data
8. Show filter badges in header: `[PDF Parser ×] [CSV Parser ×]`
9. Call `onFilterChange({ parserId: ['pdf_parser', 'csv_parser'] })`

**Clear Filters:**
- Click "×" on filter badge
- Click "Clear All Filters" button

### 3. Drilling Down to Parser Details

**Trigger:** User clicks parser row in Parser Stats table

**Flow:**
1. User clicks "PDF Parser" row
2. Capture click event with parser context
3. Call `onDrillDown({ type: 'parser', id: 'pdf_parser', name: 'PDF Parser', timeRange: currentTimeRange })`
4. External handler (parent component) navigates to parser detail page
5. Parser detail page shows:
   - Latency percentiles chart (p50/p95/p99)
   - Slowest executions table
   - Error breakdown by error type
   - Performance trends (compare to 24h/7d ago)

**Alternative Flow:**
- If no `onDrillDown` handler, open detail view in modal/drawer

### 4. Auto-Refresh

**Trigger:** Auto-refresh timer (60s)

**Flow:**
1. Component mounts, start 60s countdown timer
2. Show countdown in header: "Auto-refresh in: 59s"
3. Every second, decrement countdown
4. When countdown reaches 0:
   - Set state to `refreshing`
   - Show spinner in header
   - Fetch updated metrics: `GET /api/metrics/performance?timeRange=24h`
   - Merge new data with existing data (append to time series)
   - Update charts with smooth transition animation
   - Reset countdown to 60s
   - Call `onRefresh(metrics)`

**Manual Refresh:**
- User clicks "⟳" refresh button
- Reset countdown and immediately fetch new data

**Pause Auto-Refresh:**
- User toggles auto-refresh switch to OFF
- Stop countdown timer
- Show "Auto-refresh: OFF" in header

### 5. Exporting Data

**Trigger:** User clicks "Export" button and selects format (CSV/JSON)

**Flow:**
1. User clicks "Export" button
2. Dropdown shows options: [CSV] [JSON] [PDF]
3. User selects "CSV"
4. Set state: `{ exporting: true, exportFormat: 'csv' }`
5. Show progress indicator
6. Generate CSV:
   ```
   Parser Name,Executions,Avg Latency (ms),P95 Latency (ms),Error Rate (%)
   PDF Parser,12345,2400,3800,3.2
   CSV Parser,8901,800,1200,0.5
   ```
7. Trigger browser download: `performance-metrics-2025-10-25.csv`
8. Set state: `{ exporting: false }`
9. Call `onExport('csv', csvData)`

**Export All Data:**
- Includes summary metrics + time series data
- Large exports (7d/30d) may take 5-10 seconds

### 6. Responding to Alerts

**Trigger:** Queue depth exceeds warning threshold

**Flow:**
1. Auto-refresh fetches new metrics
2. Detect threshold breach:
   ```typescript
   if (queue.pending > thresholds.queueDepth.warning) {
     triggerAlert({
       type: 'queue_depth',
       severity: 'warning',
       message: 'Invoice processing queue has 156 pending documents',
       value: 156,
       threshold: 100
     })
   }
   ```
3. Add alert to `activeAlerts` array
4. Show alert banner at top of dashboard:
   ```
   🟡 WARNING: Invoice processing queue has 156 pending documents
   [View Queue] [Mute]
   ```
5. Call `onAlertTriggered(alert)`
6. If `playSoundAlerts=true`, play beep sound

**Mute Alert:**
- User clicks "Mute" button
- Add alert ID to `mutedAlerts` set
- Hide alert banner
- Don't show this alert again until value drops below threshold

### 7. Collapsing/Expanding Panels

**Trigger:** User clicks collapse toggle on panel header

**Flow:**
1. User clicks "↕" collapse button on "Parser Stats" panel
2. Toggle `collapsed` state for panel
3. Animate panel height from 400px → 60px (header only)
4. Hide panel content (chart + table)
5. Change collapse icon: "↕" → "↓"
6. Save layout preference to localStorage

**Expand:**
- Click "↓" button
- Animate panel height from 60px → 400px
- Show panel content with fade-in animation

### 8. Searching Across Metrics

**Trigger:** User types in search input

**Flow:**
1. User types "timeout" in search box
2. Debounce input (300ms)
3. Filter all visible metrics:
   - Parser names containing "timeout"
   - Rule names containing "timeout"
   - Error types containing "timeout"
4. Highlight matching rows in tables
5. Show search results count: "3 results found"
6. Show "Clear search" button (×)

**Clear Search:**
- Click "×" button or clear input
- Remove filters and show all data

### 9. Hovering Over Charts

**Trigger:** User hovers mouse over latency chart

**Flow:**
1. User moves mouse over chart area
2. Show vertical crosshair line at mouse position
3. Show tooltip with data point values:
   ```
   ┌──────────────────────┐
   │ 2025-10-25 14:30     │
   │ P50: 1,234ms         │
   │ P95: 2,345ms         │
   │ P99: 3,456ms         │
   └──────────────────────┘
   ```
4. Highlight data points on all 3 lines (p50/p95/p99)
5. Update cursor to crosshair

**Mouse Leave:**
- Hide tooltip
- Remove crosshair
- Reset cursor

### 10. Viewing Stuck Documents

**Trigger:** User clicks "View Stuck Docs" on queue card

**Flow:**
1. User clicks "View Stuck Docs" button on "invoice-processing" queue card
2. Open modal/drawer with stuck documents list
3. Fetch stuck documents: `GET /api/queues/invoice-processing/stuck`
4. Show table:
   ```
   ┌─────────────┬──────────┬─────────────────────────┬────────┐
   │ Document ID │ Age      │ Last Updated            │ Action │
   ├─────────────┼──────────┼─────────────────────────┼────────┤
   │ doc_abc123  │ 45 mins  │ 2025-10-25 13:45:00 UTC │ [Retry]│
   │ doc_def456  │ 38 mins  │ 2025-10-25 13:52:00 UTC │ [Retry]│
   │ doc_ghi789  │ 32 mins  │ 2025-10-25 13:58:00 UTC │ [Retry]│
   └─────────────┴──────────┴─────────────────────────┴────────┘
   ```
5. User can click "Retry" to requeue document

**Manual Retry:**
- Click "Retry" button on document row
- Call API: `POST /api/queues/invoice-processing/retry` with document ID
- Show success toast: "Document requeued successfully"
- Remove from stuck list

---

## Accessibility

### WCAG 2.1 AA Compliance

#### 1. Keyboard Navigation

All dashboard interactions are keyboard-accessible:

| Action | Keyboard Shortcut | Focus Behavior |
|--------|------------------|----------------|
| Navigate panels | `Tab` | Focus moves through panels in order |
| Expand/collapse panel | `Enter` or `Space` | Toggles panel expansion |
| Navigate time range buttons | `Arrow Left/Right` | Moves focus between time range options |
| Select time range | `Enter` or `Space` | Applies selected time range |
| Open filter dropdown | `Enter` or `Space` | Opens dropdown, focus on first option |
| Navigate filter options | `Arrow Up/Down` | Moves through filter options |
| Select filter option | `Space` | Checks/unchecks option |
| Apply filters | `Enter` | Applies selected filters |
| Export data | `Ctrl+E` | Opens export menu |
| Refresh dashboard | `Ctrl+R` or `F5` | Manually refreshes data |
| Toggle full screen | `F11` | Enters/exits full screen mode |
| Close modal | `Esc` | Closes open modal/drawer |

#### 2. Screen Reader Support

**ARIA Labels:**

```html
<div role="region" aria-label="Performance Dashboard">
  <div role="banner" aria-label="Dashboard Controls">
    <div role="group" aria-label="Time Range Selector">
      <button aria-label="Show data for last 1 hour" aria-pressed="false">1h</button>
      <button aria-label="Show data for last 6 hours" aria-pressed="false">6h</button>
      <button aria-label="Show data for last 24 hours" aria-pressed="true">24h</button>
    </div>
  </div>

  <div role="alert" aria-live="assertive" aria-atomic="true">
    2 queues have high pending counts exceeding 100 documents
  </div>

  <div role="region" aria-label="Parser Statistics Panel">
    <div role="group" aria-label="Summary Metrics">
      <div role="text" aria-label="Total Executions: 45,678">
        <span aria-hidden="true">45,678</span>
        <span aria-hidden="true">Total Executions</span>
      </div>
    </div>

    <div role="img" aria-label="Latency over time chart showing p50, p95, and p99 latency percentiles">
      <!-- Chart rendered here -->
      <div class="visually-hidden" aria-live="polite">
        Chart data: At 6am, p50 latency was 800 milliseconds, p95 was 1200 milliseconds, p99 was 2000 milliseconds.
      </div>
    </div>

    <table role="table" aria-label="Parser performance table">
      <thead>
        <tr>
          <th scope="col" aria-sort="none">Parser Name</th>
          <th scope="col" aria-sort="descending">Average Latency</th>
          <th scope="col" aria-sort="none">Error Rate</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>PDF Parser</td>
          <td aria-label="2400 milliseconds">2.4s</td>
          <td aria-label="3.2 percent">3.2%</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>
```

**Live Regions:**

Auto-refresh announcements:
```html
<div aria-live="polite" aria-atomic="true" class="visually-hidden">
  Dashboard refreshed. Last updated: October 25, 2025 at 2:32 PM UTC.
  Total executions: 45,678. Average latency: 1.2 seconds. 2 active alerts.
</div>
```

Alert announcements:
```html
<div aria-live="assertive" aria-atomic="true" role="alert">
  Warning: Invoice processing queue has 156 pending documents, exceeding threshold of 100.
</div>
```

#### 3. Color Contrast

All colors meet WCAG AA standards:

| Element | Foreground | Background | Ratio | Status |
|---------|-----------|------------|-------|--------|
| Body text | #1F2937 | #FFFFFF | 15.3:1 | ✅ AAA |
| Secondary text | #6B7280 | #FFFFFF | 4.6:1 | ✅ AA |
| Success indicator | #059669 | #FFFFFF | 4.7:1 | ✅ AA |
| Warning indicator | #D97706 | #FFFFFF | 4.9:1 | ✅ AA |
| Critical indicator | #DC2626 | #FFFFFF | 5.8:1 | ✅ AA |
| Link text | #2563EB | #FFFFFF | 6.1:1 | ✅ AA |
| Chart lines | Various | #FFFFFF | 4.5:1+ | ✅ AA |

**High Contrast Mode:**

When user enables high contrast (prefers-contrast: high):
```css
@media (prefers-contrast: high) {
  .dashboard {
    --color-success: #008000;      /* Pure green */
    --color-warning: #FFA500;      /* Orange */
    --color-critical: #FF0000;     /* Pure red */
    --border-width: 2px;           /* Thicker borders */
  }

  .metric-card {
    border: 2px solid #000000;
  }

  .chart-line {
    stroke-width: 3px;             /* Thicker lines */
  }
}
```

#### 4. Focus Indicators

All interactive elements have visible focus indicators:

```css
button:focus-visible,
a:focus-visible,
input:focus-visible,
select:focus-visible {
  outline: 2px solid #2563EB;
  outline-offset: 2px;
  border-radius: 4px;
}

.chart-element:focus-visible {
  outline: 3px solid #2563EB;
  outline-offset: 4px;
}
```

#### 5. Chart Accessibility

Charts provide alternative text and keyboard navigation:

```typescript
<ResponsiveContainer width="100%" height={300}>
  <LineChart data={latencyData} aria-label="Parser latency over time">
    <XAxis
      dataKey="timestamp"
      aria-label="Time axis"
      tickFormatter={(value) => format(value, 'HH:mm')}
    />
    <YAxis
      aria-label="Latency in milliseconds"
      tickFormatter={(value) => `${value}ms`}
    />
    <Tooltip
      content={<CustomTooltip />}
      aria-live="polite"
    />
    <Legend
      aria-label="Chart legend"
      wrapperStyle={{ fontSize: 14 }}
    />
    <Line
      type="monotone"
      dataKey="p50"
      stroke="#10B981"
      name="P50 Latency"
      aria-label="50th percentile latency"
    />
    <Line
      type="monotone"
      dataKey="p95"
      stroke="#F59E0B"
      name="P95 Latency"
      aria-label="95th percentile latency"
    />
    <Line
      type="monotone"
      dataKey="p99"
      stroke="#EF4444"
      name="P99 Latency"
      aria-label="99th percentile latency"
    />
  </LineChart>
</ResponsiveContainer>

<!-- Alternative text for screen readers -->
<div class="visually-hidden" aria-live="polite">
  Chart showing parser latency over the last 24 hours.
  At 6 AM, p50 was 800ms, p95 was 1200ms, p99 was 2000ms.
  At 12 PM, p50 was 900ms, p95 was 1400ms, p99 was 2500ms.
  Current values: p50 is 850ms, p95 is 1300ms, p99 is 2200ms.
</div>
```

#### 6. Text Resizing

Dashboard supports text resizing up to 200%:

```css
.dashboard {
  font-size: 1rem;                 /* 16px base */
}

@media (min-resolution: 2dppx) {
  .dashboard {
    font-size: 0.875rem;           /* 14px on high-DPI displays */
  }
}

/* Scalable text */
.metric-value {
  font-size: clamp(1.5rem, 5vw, 3rem);
}

.metric-label {
  font-size: clamp(0.875rem, 2vw, 1rem);
}
```

#### 7. Error Identification

Errors are identified through multiple cues (not just color):

```html
<div class="error-row" role="alert">
  <span class="error-icon" aria-hidden="true">🔴</span>
  <span class="error-label">Error:</span>
  <span class="error-message">PDF Parse Timeout</span>
  <span class="error-count" aria-label="456 occurrences">456</span>
</div>
```

Visual indicators:
- Icon: 🔴 (red circle)
- Text label: "Error"
- Color: Red background (#FEE2E2)
- Border: Red left border (4px)

---

## Performance Optimizations

### 1. Data Fetching & Caching

Use React Query for efficient data fetching:

```typescript
import { useQuery } from '@tanstack/react-query'

const useDashboardMetrics = (timeRange: TimeRange, filters: DashboardFilters) => {
  return useQuery({
    queryKey: ['dashboard-metrics', timeRange, filters],
    queryFn: () => fetchMetrics({ timeRange, filters }),
    staleTime: 60000,              // Data fresh for 60s
    cacheTime: 300000,             // Cache for 5 minutes
    refetchInterval: 60000,        // Auto-refresh every 60s
    refetchOnWindowFocus: true,    // Refresh when tab regains focus
    keepPreviousData: true,        // Show old data while fetching new
  })
}
```

### 2. Chart Rendering Optimization

Memoize chart components to prevent unnecessary re-renders:

```typescript
const LatencyChart = React.memo(({ data, height }: LatencyChartProps) => {
  return (
    <ResponsiveContainer width="100%" height={height}>
      <LineChart data={data}>
        {/* Chart configuration */}
      </LineChart>
    </ResponsiveContainer>
  )
}, (prevProps, nextProps) => {
  // Custom comparison: only re-render if data or height changed
  return (
    prevProps.data === nextProps.data &&
    prevProps.height === nextProps.height
  )
})
```

### 3. Virtualized Tables

For large datasets (>100 rows), use virtualization:

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

const ParserTable = ({ data }: { data: ParserStat[] }) => {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: data.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,        // Row height
    overscan: 10                   // Render 10 extra rows
  })

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px` }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`
            }}
          >
            <ParserRow data={data[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

### 4. Debounced Filtering

Debounce filter inputs to reduce re-renders:

```typescript
import { useDebouncedValue } from '@/hooks/useDebouncedValue'

const FilterBar = () => {
  const [searchInput, setSearchInput] = useState('')
  const debouncedSearch = useDebouncedValue(searchInput, 300)

  useEffect(() => {
    if (debouncedSearch) {
      // Trigger filter update
      applyFilters({ search: debouncedSearch })
    }
  }, [debouncedSearch])

  return (
    <input
      type="text"
      value={searchInput}
      onChange={(e) => setSearchInput(e.target.value)}
      placeholder="Search..."
    />
  )
}
```

### 5. Lazy Loading Panels

Load panel components on-demand:

```typescript
const ParserStatsPanel = lazy(() => import('./ParserStatsPanel'))
const RuleStatsPanel = lazy(() => import('./RuleStatsPanel'))
const QueueHealthPanel = lazy(() => import('./QueueHealthPanel'))
const ErrorBreakdownPanel = lazy(() => import('./ErrorBreakdownPanel'))

const Dashboard = () => {
  return (
    <Suspense fallback={<PanelSkeleton />}>
      <ParserStatsPanel />
      <RuleStatsPanel />
      <QueueHealthPanel />
      <ErrorBreakdownPanel />
    </Suspense>
  )
}
```

### 6. Web Workers for Data Processing

Offload heavy calculations to Web Worker:

```typescript
// metrics.worker.ts
self.addEventListener('message', (e) => {
  const { rawData } = e.data

  const processed = {
    percentiles: calculatePercentiles(rawData),
    aggregations: aggregateByParser(rawData),
    timeSeries: buildTimeSeries(rawData)
  }

  self.postMessage(processed)
})

// In component
const metricsWorker = useRef<Worker>()

useEffect(() => {
  metricsWorker.current = new Worker('/metrics.worker.js')

  metricsWorker.current.onmessage = (e) => {
    setProcessedMetrics(e.data)
  }

  return () => metricsWorker.current?.terminate()
}, [])

// Send raw data to worker
useEffect(() => {
  if (rawMetrics) {
    metricsWorker.current?.postMessage({ rawData: rawMetrics })
  }
}, [rawMetrics])
```

### 7. Incremental Data Updates

For real-time updates, merge new data instead of replacing:

```typescript
const mergeMetrics = (
  existing: PerformanceMetrics,
  incoming: PerformanceMetrics
): PerformanceMetrics => {
  return {
    ...existing,
    parserStats: {
      ...existing.parserStats,
      latencyTimeSeries: [
        ...existing.parserStats.latencyTimeSeries,
        ...incoming.parserStats.latencyTimeSeries
      ].slice(-100)  // Keep last 100 data points
    }
  }
}
```

### 8. Code Splitting

Split large dependencies:

```typescript
// Chart library loaded on-demand
const Recharts = lazy(() => import('recharts'))

// PDF export loaded only when needed
const exportToPdf = async () => {
  const jsPDF = (await import('jspdf')).default
  const pdf = new jsPDF()
  // ... generate PDF
}
```

---

## Integration Examples

### Example 1: React Integration with React Query

```typescript
import { PerformanceDashboard } from '@/components/il/PerformanceDashboard'
import { useQuery } from '@tanstack/react-query'

export default function DashboardPage() {
  const [timeRange, setTimeRange] = useState<TimeRange>({ preset: '24h' })
  const [filters, setFilters] = useState<DashboardFilters>({})

  const { data: metrics, isLoading, error } = useQuery({
    queryKey: ['performance-metrics', timeRange, filters],
    queryFn: () => fetchPerformanceMetrics({ timeRange, filters }),
    refetchInterval: 60000
  })

  const handleDrillDown = (context: DrillDownContext) => {
    if (context.type === 'parser') {
      router.push(`/parsers/${context.id}/performance`)
    } else if (context.type === 'queue') {
      router.push(`/queues/${context.name}/monitoring`)
    }
  }

  const handleAlertTriggered = (alert: AlertEvent) => {
    // Send to monitoring system
    monitoring.sendAlert({
      severity: alert.severity,
      message: alert.message,
      context: alert.context
    })

    // Show browser notification
    if (Notification.permission === 'granted') {
      new Notification('Performance Alert', {
        body: alert.message,
        icon: '/alert-icon.png'
      })
    }
  }

  return (
    <PerformanceDashboard
      initialTimeRange="24h"
      refreshInterval={60}
      enableRealTime={false}
      thresholds={{
        parserLatency: { warning: 2000, critical: 5000 },
        ruleExecutionTime: { warning: 500, critical: 1000 },
        queueDepth: { warning: 100, critical: 500 },
        errorRate: { warning: 5, critical: 10 }
      }}
      onTimeRangeChange={setTimeRange}
      onFilterChange={setFilters}
      onDrillDown={handleDrillDown}
      onAlertTriggered={handleAlertTriggered}
      dataFetcher={async (params) => {
        const response = await fetch('/api/metrics/performance', {
          method: 'POST',
          body: JSON.stringify(params)
        })
        return response.json()
      }}
    />
  )
}
```

### Example 2: With WebSocket Real-Time Updates

```typescript
import { useWebSocket } from '@/hooks/useWebSocket'

export function RealTimeDashboard() {
  const [metrics, setMetrics] = useState<PerformanceMetrics | null>(null)

  const { connected, sendMessage } = useWebSocket('wss://api.example.com/metrics/stream', {
    onMessage: (data) => {
      if (data.type === 'metrics_update') {
        setMetrics((prev) => mergeMetrics(prev, data.payload))
      }
    },
    onConnect: () => {
      sendMessage({ type: 'subscribe', channels: ['parser_stats', 'queue_health'] })
    }
  })

  return (
    <PerformanceDashboard
      enableRealTime={true}
      websocketUrl="wss://api.example.com/metrics/stream"
      refreshInterval={0}  // Disable polling, use WebSocket only
      dataFetcher={async () => {
        // Fetch initial data, then rely on WebSocket for updates
        const response = await fetch('/api/metrics/performance?timeRange=24h')
        return response.json()
      }}
    />
  )
}
```

### Example 3: Custom Formatters

```typescript
<PerformanceDashboard
  formatters={{
    latency: (ms) => {
      if (ms < 1000) return `${ms}ms`
      return `${(ms / 1000).toFixed(2)}s`
    },
    throughput: (count) => {
      if (count < 1000) return `${count}/s`
      if (count < 1000000) return `${(count / 1000).toFixed(1)}K/s`
      return `${(count / 1000000).toFixed(1)}M/s`
    },
    errorRate: (rate) => {
      return `${rate.toFixed(2)}%`
    },
    timestamp: (date) => {
      return format(date, 'MMM d, HH:mm', { timeZone: 'America/New_York' })
    }
  }}
  locale="en-US"
  timezone="America/New_York"
/>
```

### Example 4: Custom Alert Handling

```typescript
const handleAlertTriggered = async (alert: AlertEvent) => {
  // Log to analytics
  analytics.track('performance_alert_triggered', {
    type: alert.type,
    severity: alert.severity,
    value: alert.value,
    threshold: alert.threshold
  })

  // Send to Slack
  if (alert.severity === 'critical') {
    await fetch('/api/slack/notify', {
      method: 'POST',
      body: JSON.stringify({
        channel: '#ops-alerts',
        message: `🚨 ${alert.message}`,
        context: alert.context
      })
    })
  }

  // Send to PagerDuty
  if (alert.severity === 'critical') {
    await fetch('/api/pagerduty/incident', {
      method: 'POST',
      body: JSON.stringify({
        title: alert.message,
        urgency: 'high',
        details: alert.context
      })
    })
  }
}

<PerformanceDashboard
  onAlertTriggered={handleAlertTriggered}
  playSoundAlerts={true}
/>
```

---

## Related Components

### 1. RuleOptimizer

**Relationship:** PerformanceDashboard → RuleOptimizer

When user clicks on a slow rule in the Rule Stats panel, they can navigate to RuleOptimizer for detailed analysis and optimization recommendations.

**Integration:**
```typescript
<PerformanceDashboard
  onDrillDown={(context) => {
    if (context.type === 'rule') {
      router.push(`/rules/${context.id}/optimizer`)
    }
  }}
/>

{/* On optimizer page */}
<RuleOptimizer
  ruleId={ruleId}
  timeRange="7d"
  showOptimizationRecommendations={true}
/>
```

### 2. QueueMonitorPanel

**Relationship:** PerformanceDashboard uses QueueMonitorPanel internally

The Queue Health panel is extracted to QueueMonitorPanel for use in other contexts.

**Integration:**
```typescript
<PerformanceDashboard
  panels={{
    parserStats: true,
    ruleStats: true,
    queueHealth: true,  // Uses QueueMonitorPanel internally
    errorBreakdown: true
  }}
/>
```

### 3. ParserDetailView

**Relationship:** PerformanceDashboard → ParserDetailView

Drill-down from parser row to detailed parser performance view.

**Integration:**
```typescript
<PerformanceDashboard
  onDrillDown={(context) => {
    if (context.type === 'parser') {
      setSelectedParser(context.id)
      setShowDetailView(true)
    }
  }}
/>

{showDetailView && (
  <ParserDetailView
    parserId={selectedParser}
    timeRange={timeRange}
    onClose={() => setShowDetailView(false)}
  />
)}
```

### 4. AlertNotificationCenter

**Relationship:** PerformanceDashboard → AlertNotificationCenter

Alerts triggered by dashboard are sent to centralized notification system.

**Integration:**
```typescript
<PerformanceDashboard
  onAlertTriggered={(alert) => {
    alertNotificationCenter.addAlert(alert)
  }}
/>

<AlertNotificationCenter
  alerts={alerts}
  onDismiss={(alertId) => {
    alertNotificationCenter.dismissAlert(alertId)
  }}
/>
```

---

## References

### Performance Monitoring

- [Prometheus Monitoring](https://prometheus.io/docs/introduction/overview/)
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
- [OpenTelemetry Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Datadog Dashboards](https://docs.datadoghq.com/dashboards/)

### Chart Libraries

- [Recharts Documentation](https://recharts.org/en-US/)
- [Chart.js](https://www.chartjs.org/docs/latest/)
- [Victory Charts](https://formidable.com/open-source/victory/)
- [D3.js](https://d3js.org/)

### Real-Time Data

- [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [React Query](https://tanstack.com/query/latest)

### Accessibility

- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [Chart Accessibility](https://www.w3.org/WAI/tutorials/images/complex/)

### Performance Optimization

- [React Performance Optimization](https://react.dev/learn/render-and-commit)
- [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [React Virtualization](https://tanstack.com/virtual/latest)

---

**Document Version:** 1.0
**Word Count:** ~9,500 words
**Last Updated:** 2025-10-25
**Maintained By:** Platform Team
