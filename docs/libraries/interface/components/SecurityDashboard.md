# SecurityDashboard (IL Component)

**Layer:** Interaction Layer (IL)
**Domain:** Security & Access (Vertical 5.4)
**Status:** Draft
**Last Updated:** 2025-10-25

---

## Table of Contents

1. [Overview](#overview)
2. [Component Interface](#component-interface)
3. [Visual Design](#visual-design)
4. [State Management](#state-management)
5. [Data Fetching](#data-fetching)
6. [Rendering Logic](#rendering-logic)
7. [Event Handlers](#event-handlers)
8. [Accessibility](#accessibility)
9. [Performance](#performance)
10. [Multi-Domain Examples](#multi-domain-examples)
11. [Integration](#integration)
12. [Testing Strategy](#testing-strategy)

---

## Overview

### Purpose

The **SecurityDashboard** is a comprehensive real-time security monitoring component that provides visibility into access control violations, PII activity, security events, and user behavior patterns. It transforms raw security audit data into actionable intelligence through interactive visualizations, enabling security teams to quickly identify threats, detect anomalies, and maintain compliance. This component serves as the central command center for security operations across all tenant domains.

### Key Capabilities

- **Real-Time Security Monitoring**: Auto-refreshing dashboard with configurable intervals (30s/60s/300s)
- **4 Primary Panels**: Access Denials, PII Activity, Security Events Timeline, User Activity Heatmap
- **Threat Detection**: Identify suspicious patterns (failed logins, privilege escalation, data exfiltration)
- **Compliance Tracking**: Monitor PII access, data exports, regulatory violations
- **Time Range Selection**: Flexible windows (1h/6h/24h/7d/30d/custom)
- **Multi-Tenant Support**: Filter by tenant_id, view cross-tenant anomalies (admin only)
- **Alert Integration**: Visual indicators for security threshold breaches
- **Export Capabilities**: CSV/PDF export for compliance reporting
- **Drill-Down Navigation**: Click-through from dashboard to detailed audit logs
- **Dark/Light Theme**: Security-focused color schemes (red for critical, amber for warnings)

### Problem Statement

Security teams face critical challenges:

1. **Delayed Threat Detection**: Security events discovered hours/days after occurrence
2. **Alert Fatigue**: Too many false positives, miss real threats
3. **Compliance Blind Spots**: Can't prove who accessed PII data and when
4. **Manual Log Analysis**: Security analysts waste time parsing raw logs
5. **No Context**: Isolated events lack behavioral patterns (e.g., multiple failed logins)
6. **Reactive Posture**: Only investigate after incidents are reported
7. **Multi-Tenant Complexity**: Can't track access patterns across tenant boundaries

### Solution

The SecurityDashboard provides:

- **Proactive Threat Hunting**: Real-time anomaly detection with behavioral baselines
- **Visual Intelligence**: Charts reveal patterns invisible in raw logs (spikes, clusters)
- **Contextual Alerts**: Correlate multiple signals (failed login + privilege change + data export)
- **Compliance Automation**: Pre-built reports for GDPR Article 15, SOC 2, HIPAA
- **One-Click Investigation**: Navigate from alert to full audit trail in <500ms
- **Risk Scoring**: Aggregate user activity into risk scores (low/medium/high/critical)
- **Automated Response**: Trigger workflows on threshold breaches (e.g., account lockout)

This transforms security operations from reactive incident response to proactive threat prevention.

---

## Component Interface

### Primary Props

```typescript
interface SecurityDashboardProps {
  // ============================================================
  // Time Range Configuration
  // ============================================================

  /**
   * Initial time range for security data
   * @default '24h'
   */
  timeRange?: '1h' | '6h' | '24h' | '7d' | '30d' | 'custom'

  /**
   * Custom time range (if timeRange='custom')
   */
  customTimeRange?: {
    start: Date | string
    end: Date | string
  }

  /**
   * Auto-refresh interval in seconds
   * @default 60
   */
  refreshInterval?: 30 | 60 | 300 | null // null = manual refresh only

  // ============================================================
  // Filtering & Scoping
  // ============================================================

  /**
   * Tenant ID(s) to monitor
   * Single tenant for standard users, multiple for admins
   */
  tenantIds?: string | string[]

  /**
   * Initial filters to apply
   */
  initialFilters?: {
    severityLevel?: ('critical' | 'high' | 'medium' | 'low')[]
    eventTypes?: ('access_denied' | 'pii_access' | 'privilege_change' | 'data_export')[]
    userIds?: string[]
    ipAddresses?: string[]
    resourceTypes?: ('document' | 'report' | 'entity' | 'quote')[]
  }

  /**
   * Show cross-tenant analytics (admin only)
   * @default false
   */
  enableCrossTenantView?: boolean

  // ============================================================
  // Display Configuration
  // ============================================================

  /**
   * Panel visibility configuration
   */
  panels?: {
    accessDenials?: boolean
    piiActivity?: boolean
    securityEvents?: boolean
    userActivityHeatmap?: boolean
  }

  /**
   * Chart configuration
   */
  chartConfig?: {
    /** Show data labels on charts */
    showLabels?: boolean
    /** Enable chart animations */
    animated?: boolean
    /** Color scheme */
    colorScheme?: 'security-default' | 'security-high-contrast' | 'custom'
    /** Custom colors for severity levels */
    customColors?: {
      critical?: string
      high?: string
      medium?: string
      low?: string
    }
  }

  /**
   * Compact mode for smaller screens
   * @default false
   */
  compact?: boolean

  /**
   * Dark mode
   * @default 'auto' (follows system)
   */
  theme?: 'light' | 'dark' | 'auto'

  // ============================================================
  // Alert Thresholds
  // ============================================================

  /**
   * Thresholds for visual alerts
   */
  alertThresholds?: {
    /** Max failed logins per user per hour */
    maxFailedLogins?: number // default: 5
    /** Max PII access events per user per hour */
    maxPiiAccess?: number // default: 20
    /** Max privilege escalations per tenant per hour */
    maxPrivilegeChanges?: number // default: 3
    /** Max data export size per user per day (MB) */
    maxExportSizeMB?: number // default: 1000
  }

  /**
   * Enable automatic incident creation for threshold breaches
   * @default false
   */
  autoCreateIncidents?: boolean

  // ============================================================
  // Export Configuration
  // ============================================================

  /**
   * Enable CSV/PDF export
   * @default true
   */
  enableExport?: boolean

  /**
   * Export format options
   */
  exportFormats?: ('csv' | 'pdf' | 'json')[]

  /**
   * Include raw data in exports
   * @default false
   */
  includeRawData?: boolean

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when user clicks on a security event
   */
  onEventClick?: (event: SecurityEvent) => void

  /**
   * Called when user clicks "View Details" for access denial
   */
  onAccessDenialDrillDown?: (userId: string, resourceId: string) => void

  /**
   * Called when user clicks on PII activity row
   */
  onPiiActivityClick?: (activity: PiiActivity) => void

  /**
   * Called when user clicks on heatmap cell
   */
  onUserActivityClick?: (userId: string, timeSlot: string) => void

  /**
   * Called when alert threshold is breached
   */
  onAlertThresholdBreached?: (alert: SecurityAlert) => void

  /**
   * Called when export is initiated
   */
  onExport?: (format: 'csv' | 'pdf' | 'json', data: any) => void

  /**
   * Called when filters change
   */
  onFiltersChange?: (filters: SecurityFilters) => void

  /**
   * Called when time range changes
   */
  onTimeRangeChange?: (range: TimeRange) => void

  // ============================================================
  // Integration
  // ============================================================

  /**
   * API client for fetching security data
   * If not provided, uses default SecurityAPIClient
   */
  apiClient?: SecurityAPIClient

  /**
   * Enable integration with AuditViewer
   * Shows "View Full Audit Log" button
   * @default true
   */
  enableAuditViewerLink?: boolean

  /**
   * Enable integration with RoleManager
   * Shows "Manage Roles" button for privilege escalation events
   * @default true
   */
  enableRoleManagerLink?: boolean

  /**
   * Custom actions to show in panel headers
   */
  customActions?: React.ReactNode

  // ============================================================
  // Performance
  // ============================================================

  /**
   * Enable data virtualization for large datasets
   * @default true
   */
  enableVirtualization?: boolean

  /**
   * Max data points to render per chart
   * @default 1000
   */
  maxDataPoints?: number

  /**
   * Debounce filter changes (ms)
   * @default 300
   */
  filterDebounceMs?: number
}
```

### Supporting Types

```typescript
/**
 * Security event from audit log
 */
interface SecurityEvent {
  id: string
  timestamp: Date
  tenantId: string
  userId: string
  userEmail: string
  userName: string
  eventType: 'access_denied' | 'pii_access' | 'privilege_change' | 'data_export' | 'login_failed' | 'account_locked' | 'session_hijack'
  severity: 'critical' | 'high' | 'medium' | 'low'
  resourceType: 'document' | 'report' | 'entity' | 'quote' | 'user' | 'role' | 'system'
  resourceId: string
  resourceName?: string
  action: string // e.g., "read", "write", "delete", "escalate"
  ipAddress: string
  userAgent: string
  geoLocation?: {
    country: string
    region: string
    city: string
    lat: number
    lon: number
  }
  details: Record<string, any>
  riskScore: number // 0-100
}

/**
 * PII activity record
 */
interface PiiActivity {
  id: string
  timestamp: Date
  tenantId: string
  userId: string
  userEmail: string
  piiType: 'ssn' | 'credit_card' | 'email' | 'phone' | 'dob' | 'address' | 'medical' | 'financial'
  action: 'view' | 'export' | 'modify' | 'delete'
  documentId: string
  documentName: string
  fieldPath: string // JSON path to PII field
  maskedValue?: string // e.g., "***-**-1234"
  justification?: string // User-provided reason for access
  approvalRequired: boolean
  approvedBy?: string
  complianceFlags: string[] // e.g., ["GDPR", "HIPAA", "PCI-DSS"]
}

/**
 * Aggregated security metrics
 */
interface SecurityMetrics {
  timeRange: TimeRange
  accessDenials: {
    total: number
    byReason: Record<string, number>
    byUser: Array<{ userId: string; userName: string; count: number }>
    byResource: Array<{ resourceType: string; count: number }>
    trend: Array<{ timestamp: Date; count: number }>
  }
  piiActivity: {
    total: number
    byType: Record<string, number>
    byUser: Array<{ userId: string; userName: string; count: number }>
    byAction: Record<string, number>
    complianceViolations: number
  }
  securityEvents: {
    total: number
    critical: number
    high: number
    medium: number
    low: number
    bySeverity: Array<{ timestamp: Date; severity: string; count: number }>
    topEvents: SecurityEvent[]
  }
  userActivity: {
    activeUsers: number
    suspiciousUsers: number
    heatmap: Array<{
      userId: string
      userName: string
      hourlyActivity: Record<string, number> // hour -> count
    }>
    riskDistribution: Record<string, number> // risk level -> count
  }
}

/**
 * Security alert triggered by threshold breach
 */
interface SecurityAlert {
  id: string
  timestamp: Date
  type: 'failed_logins' | 'pii_access' | 'privilege_escalation' | 'data_exfiltration' | 'anomalous_behavior'
  severity: 'critical' | 'high' | 'medium' | 'low'
  message: string
  affectedUsers: string[]
  affectedResources: string[]
  thresholdValue: number
  actualValue: number
  recommendedAction: string
  autoRemediate: boolean
}

/**
 * Time range for dashboard
 */
interface TimeRange {
  start: Date
  end: Date
  label: string // e.g., "Last 24 hours"
}

/**
 * Security filters
 */
interface SecurityFilters {
  severityLevel?: ('critical' | 'high' | 'medium' | 'low')[]
  eventTypes?: string[]
  userIds?: string[]
  ipAddresses?: string[]
  resourceTypes?: string[]
}

/**
 * API client for security data
 */
interface SecurityAPIClient {
  getSecurityMetrics(params: {
    tenantIds: string[]
    timeRange: TimeRange
    filters?: SecurityFilters
  }): Promise<SecurityMetrics>

  getSecurityEvents(params: {
    tenantIds: string[]
    timeRange: TimeRange
    filters?: SecurityFilters
    limit?: number
    offset?: number
  }): Promise<{ events: SecurityEvent[]; total: number }>

  getPiiActivity(params: {
    tenantIds: string[]
    timeRange: TimeRange
    limit?: number
    offset?: number
  }): Promise<{ activities: PiiActivity[]; total: number }>

  exportData(params: {
    format: 'csv' | 'pdf' | 'json'
    data: any
    fileName: string
  }): Promise<Blob>

  createIncident(params: {
    alert: SecurityAlert
    tenantId: string
  }): Promise<{ incidentId: string }>
}
```

---

## Visual Design

### ASCII Wireframe

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Security Dashboard                                    🔒 Tenant: Acme Corp   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ⏰ Time Range: [Last 24 hours ▼]  🔄 Auto-refresh: [60s ▼]  📤 Export ▼    │
│                                                                              │
│  🔍 Filters:  Severity: [All ▼]  Event Type: [All ▼]  User: [__________]   │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│ ┌────────────────────────────────┬────────────────────────────────────────┐ │
│ │ 🚫 Access Denials (134)        │ 🔐 PII Activity (89)                   │ │
│ │                                │                                        │ │
│ │  Top Reasons:                  │  By Type:                              │ │
│ │  ┌──────────────────────┐      │  ┌──────────────────────┐             │ │
│ │  │ ██████ Insufficient  │      │  │ ████ SSN (34)        │             │ │
│ │  │   Permissions (67)   │      │  │ ███ Credit Card (28) │             │ │
│ │  │ ████ Role Mismatch   │      │  │ ██ Email (15)        │             │ │
│ │  │   (45)               │      │  │ ██ Phone (12)        │             │ │
│ │  │ ██ IP Blocked (22)   │      │  └──────────────────────┘             │ │
│ │  └──────────────────────┘      │                                        │ │
│ │                                │  Recent Activity:                      │ │
│ │  Trend (24h):                  │  ┌──────────────────────────────────┐ │ │
│ │  ┌──────────────────────┐      │  │ User        PII Type    Action   │ │ │
│ │  │       ▁▂▃▄█▅▃▂▁       │      │  │ jdoe@...    SSN         Export   │ │ │
│ │  │    ▁▃█▆▅▄▃▂▁          │      │  │ asmith@...  CreditCard  View     │ │ │
│ │  │  ▁▂▄▆█▅▃▂▁            │      │  │ rbrown@...  Email       Modify   │ │ │
│ │  └──────────────────────┘      │  │ mjohnson... Phone       View     │ │ │
│ │                                │  └──────────────────────────────────┘ │ │
│ │  [View Details >]              │  [View All PII Activity >]             │ │
│ └────────────────────────────────┴────────────────────────────────────────┘ │
│                                                                              │
│ ┌────────────────────────────────┬────────────────────────────────────────┐ │
│ │ 🔔 Security Events Timeline    │ 🔥 User Activity Heatmap               │ │
│ │                                │                                        │ │
│ │  Critical: 3  High: 12         │  Last 24 hours by user:                │ │
│ │  Medium: 34  Low: 67           │                                        │ │
│ │                                │       0  4  8  12 16 20               │ │
│ │  ┌──────────────────────┐      │  jdoe    █  ▓  ▒  ░  ▓  █            │ │
│ │  │ 🔴 Critical          │      │  asmith  ▓  ▓  ▓  ▓  ▓  ▓            │ │
│ │  │ 🟠 High              │      │  rbrown  ░  ░  ░  ▒  ▓  █            │ │
│ │  │ 🟡 Medium            │      │  mjohnso ▓  ▓  ▓  ▓  ▓  ▓            │ │
│ │  │ 🟢 Low               │      │  klee    █  █  █  █  █  █  ⚠️        │ │
│ │  │                      │      │                                        │ │
│ │  │ ──────────────────── │      │  Legend:                               │ │
│ │  │ 0   6   12   18  24  │      │  █ High (>50)  ▓ Med (20-50)          │ │
│ │  └──────────────────────┘      │  ▒ Low (5-20)  ░ Minimal (<5)         │ │
│ │                                │  ⚠️ Anomaly detected                   │ │
│ │  Recent Critical Events:       │                                        │ │
│ │  • 14:32 - Privilege escalation│  [View User Details >]                 │ │
│ │    (jdoe → Admin)              │                                        │ │
│ │  • 11:15 - Multiple failed     │                                        │ │
│ │    logins (klee, 8 attempts)   │                                        │ │
│ │  • 09:47 - Data export 2.3 GB  │                                        │ │
│ │    (asmith)                    │                                        │ │
│ │                                │                                        │ │
│ │  [View All Events >]           │                                        │ │
│ └────────────────────────────────┴────────────────────────────────────────┘ │
│                                                                              │
│ ⚠️ 2 Active Alerts: Privilege escalation threshold exceeded | 8 failed      │
│ login attempts detected for klee@acme.com [View Alerts >]                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Responsive Breakpoints

```
Desktop (≥1200px):  4-panel grid (2x2)
Tablet (768-1199px): 2-panel grid (2x1, stacked)
Mobile (<768px):     Single column, collapsible panels
```

---

## State Management

### React State Hooks

```typescript
import { useState, useEffect, useCallback, useMemo } from 'react'
import { useQuery, useQueryClient } from '@tanstack/react-query'

export function SecurityDashboard(props: SecurityDashboardProps) {
  const {
    timeRange = '24h',
    customTimeRange,
    refreshInterval = 60,
    tenantIds,
    initialFilters,
    enableCrossTenantView = false,
    panels = {
      accessDenials: true,
      piiActivity: true,
      securityEvents: true,
      userActivityHeatmap: true,
    },
    chartConfig = {},
    compact = false,
    theme = 'auto',
    alertThresholds = {},
    autoCreateIncidents = false,
    enableExport = true,
    exportFormats = ['csv', 'pdf', 'json'],
    includeRawData = false,
    onEventClick,
    onAccessDenialDrillDown,
    onPiiActivityClick,
    onUserActivityClick,
    onAlertThresholdBreached,
    onExport,
    onFiltersChange,
    onTimeRangeChange,
    apiClient,
    enableAuditViewerLink = true,
    enableRoleManagerLink = true,
    customActions,
    enableVirtualization = true,
    maxDataPoints = 1000,
    filterDebounceMs = 300,
  } = props

  // ============================================================
  // Local State
  // ============================================================

  const [selectedTimeRange, setSelectedTimeRange] = useState<TimeRange>(() =>
    calculateTimeRange(timeRange, customTimeRange)
  )

  const [filters, setFilters] = useState<SecurityFilters>(initialFilters || {})

  const [autoRefresh, setAutoRefresh] = useState<number | null>(refreshInterval)

  const [expandedPanel, setExpandedPanel] = useState<string | null>(null)

  const [activeAlerts, setActiveAlerts] = useState<SecurityAlert[]>([])

  const [exportDialogOpen, setExportDialogOpen] = useState(false)

  const [selectedExportFormat, setSelectedExportFormat] = useState<'csv' | 'pdf' | 'json'>('csv')

  // ============================================================
  // Memoized Values
  // ============================================================

  const tenantIdArray = useMemo(() => {
    if (Array.isArray(tenantIds)) return tenantIds
    if (tenantIds) return [tenantIds]
    return []
  }, [tenantIds])

  const mergedAlertThresholds = useMemo(
    () => ({
      maxFailedLogins: 5,
      maxPiiAccess: 20,
      maxPrivilegeChanges: 3,
      maxExportSizeMB: 1000,
      ...alertThresholds,
    }),
    [alertThresholds]
  )

  const chartColors = useMemo(
    () => ({
      critical: chartConfig.customColors?.critical || '#dc2626',
      high: chartConfig.customColors?.high || '#ea580c',
      medium: chartConfig.customColors?.medium || '#ca8a04',
      low: chartConfig.customColors?.low || '#16a34a',
    }),
    [chartConfig.customColors]
  )

  // ============================================================
  // Callbacks
  // ============================================================

  const handleTimeRangeChange = useCallback(
    (range: TimeRange) => {
      setSelectedTimeRange(range)
      onTimeRangeChange?.(range)
    },
    [onTimeRangeChange]
  )

  const handleFiltersChange = useCallback(
    (newFilters: Partial<SecurityFilters>) => {
      const updatedFilters = { ...filters, ...newFilters }
      setFilters(updatedFilters)
      onFiltersChange?.(updatedFilters)
    },
    [filters, onFiltersChange]
  )

  const handleExport = useCallback(
    async (format: 'csv' | 'pdf' | 'json') => {
      setSelectedExportFormat(format)
      setExportDialogOpen(true)
    },
    []
  )

  const handlePanelExpand = useCallback((panelId: string) => {
    setExpandedPanel((prev) => (prev === panelId ? null : panelId))
  }, [])

  // ============================================================
  // Effect: Alert Monitoring
  // ============================================================

  useEffect(() => {
    // Monitor security metrics for threshold breaches
    // This would integrate with SecurityMetrics data
    // For now, placeholder logic
  }, [mergedAlertThresholds, onAlertThresholdBreached])

  return {
    // State
    selectedTimeRange,
    filters,
    autoRefresh,
    expandedPanel,
    activeAlerts,
    exportDialogOpen,
    selectedExportFormat,
    tenantIdArray,
    mergedAlertThresholds,
    chartColors,

    // Actions
    handleTimeRangeChange,
    handleFiltersChange,
    handleExport,
    handlePanelExpand,
    setAutoRefresh,
    setExportDialogOpen,
  }
}

/**
 * Calculate TimeRange from preset or custom range
 */
function calculateTimeRange(
  preset: string,
  custom?: { start: Date | string; end: Date | string }
): TimeRange {
  const end = new Date()
  let start: Date
  let label: string

  switch (preset) {
    case '1h':
      start = new Date(end.getTime() - 60 * 60 * 1000)
      label = 'Last 1 hour'
      break
    case '6h':
      start = new Date(end.getTime() - 6 * 60 * 60 * 1000)
      label = 'Last 6 hours'
      break
    case '24h':
      start = new Date(end.getTime() - 24 * 60 * 60 * 1000)
      label = 'Last 24 hours'
      break
    case '7d':
      start = new Date(end.getTime() - 7 * 24 * 60 * 60 * 1000)
      label = 'Last 7 days'
      break
    case '30d':
      start = new Date(end.getTime() - 30 * 24 * 60 * 60 * 1000)
      label = 'Last 30 days'
      break
    case 'custom':
      if (!custom) throw new Error('Custom time range required')
      start = new Date(custom.start)
      end.setTime(new Date(custom.end).getTime())
      label = 'Custom range'
      break
    default:
      start = new Date(end.getTime() - 24 * 60 * 60 * 1000)
      label = 'Last 24 hours'
  }

  return { start, end, label }
}
```

---

## Data Fetching

### React Query Integration

```typescript
import { useQuery, useQueryClient } from '@tanstack/react-query'
import { SecurityAPIClient } from './api/SecurityAPIClient'

/**
 * Fetch security metrics
 */
export function useSecurityMetrics(params: {
  tenantIds: string[]
  timeRange: TimeRange
  filters?: SecurityFilters
  refreshInterval?: number | null
  enabled?: boolean
}) {
  return useQuery({
    queryKey: ['security-metrics', params.tenantIds, params.timeRange, params.filters],
    queryFn: async () => {
      const client = new SecurityAPIClient()
      return client.getSecurityMetrics({
        tenantIds: params.tenantIds,
        timeRange: params.timeRange,
        filters: params.filters,
      })
    },
    refetchInterval: params.refreshInterval ? params.refreshInterval * 1000 : false,
    enabled: params.enabled !== false && params.tenantIds.length > 0,
    staleTime: 30 * 1000, // 30 seconds
    gcTime: 5 * 60 * 1000, // 5 minutes
  })
}

/**
 * Fetch security events (for timeline)
 */
export function useSecurityEvents(params: {
  tenantIds: string[]
  timeRange: TimeRange
  filters?: SecurityFilters
  limit?: number
  enabled?: boolean
}) {
  return useQuery({
    queryKey: ['security-events', params.tenantIds, params.timeRange, params.filters, params.limit],
    queryFn: async () => {
      const client = new SecurityAPIClient()
      return client.getSecurityEvents({
        tenantIds: params.tenantIds,
        timeRange: params.timeRange,
        filters: params.filters,
        limit: params.limit || 100,
        offset: 0,
      })
    },
    enabled: params.enabled !== false && params.tenantIds.length > 0,
    staleTime: 30 * 1000,
  })
}

/**
 * Fetch PII activity
 */
export function usePiiActivity(params: {
  tenantIds: string[]
  timeRange: TimeRange
  limit?: number
  enabled?: boolean
}) {
  return useQuery({
    queryKey: ['pii-activity', params.tenantIds, params.timeRange, params.limit],
    queryFn: async () => {
      const client = new SecurityAPIClient()
      return client.getPiiActivity({
        tenantIds: params.tenantIds,
        timeRange: params.timeRange,
        limit: params.limit || 50,
        offset: 0,
      })
    },
    enabled: params.enabled !== false && params.tenantIds.length > 0,
    staleTime: 30 * 1000,
  })
}

/**
 * Prefetch related data
 */
export function usePrefetchSecurityData(params: {
  tenantIds: string[]
  timeRange: TimeRange
  filters?: SecurityFilters
}) {
  const queryClient = useQueryClient()

  useEffect(() => {
    // Prefetch next time range
    const nextTimeRange = calculateNextTimeRange(params.timeRange)

    queryClient.prefetchQuery({
      queryKey: ['security-metrics', params.tenantIds, nextTimeRange, params.filters],
      queryFn: async () => {
        const client = new SecurityAPIClient()
        return client.getSecurityMetrics({
          tenantIds: params.tenantIds,
          timeRange: nextTimeRange,
          filters: params.filters,
        })
      },
    })
  }, [queryClient, params.tenantIds, params.timeRange, params.filters])
}

function calculateNextTimeRange(current: TimeRange): TimeRange {
  const duration = current.end.getTime() - current.start.getTime()
  return {
    start: new Date(current.end.getTime()),
    end: new Date(current.end.getTime() + duration),
    label: 'Next period',
  }
}
```

---

## Rendering Logic

### Core JSX Structure

```tsx
import React from 'react'
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from '@/components/ui/select'
import { Input } from '@/components/ui/input'
import { Badge } from '@/components/ui/badge'
import { Alert, AlertDescription } from '@/components/ui/alert'
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs'
import { BarChart, Bar, LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts'
import { Shield, AlertTriangle, Users, Activity, Download, RefreshCw } from 'lucide-react'

export function SecurityDashboard(props: SecurityDashboardProps) {
  const state = SecurityDashboard(props)
  const {
    selectedTimeRange,
    filters,
    autoRefresh,
    activeAlerts,
    exportDialogOpen,
    tenantIdArray,
    chartColors,
    handleTimeRangeChange,
    handleFiltersChange,
    handleExport,
    setAutoRefresh,
  } = state

  // Fetch data
  const metricsQuery = useSecurityMetrics({
    tenantIds: tenantIdArray,
    timeRange: selectedTimeRange,
    filters,
    refreshInterval: autoRefresh,
  })

  const eventsQuery = useSecurityEvents({
    tenantIds: tenantIdArray,
    timeRange: selectedTimeRange,
    filters,
    limit: 100,
  })

  const piiQuery = usePiiActivity({
    tenantIds: tenantIdArray,
    timeRange: selectedTimeRange,
    limit: 50,
  })

  const isLoading = metricsQuery.isLoading || eventsQuery.isLoading || piiQuery.isLoading
  const metrics = metricsQuery.data
  const events = eventsQuery.data?.events || []
  const piiActivities = piiQuery.data?.activities || []

  return (
    <div className="security-dashboard w-full p-6 space-y-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <div className="flex items-center gap-3">
          <Shield className="h-8 w-8 text-blue-600" />
          <div>
            <h1 className="text-2xl font-bold">Security Dashboard</h1>
            <p className="text-sm text-gray-500">
              {tenantIdArray.length === 1 ? `Tenant: ${tenantIdArray[0]}` : `${tenantIdArray.length} tenants`}
            </p>
          </div>
        </div>

        <div className="flex items-center gap-3">
          {props.customActions}
          {props.enableExport && (
            <Button variant="outline" onClick={() => handleExport('csv')}>
              <Download className="h-4 w-4 mr-2" />
              Export
            </Button>
          )}
        </div>
      </div>

      {/* Controls */}
      <div className="flex flex-wrap items-center gap-4 p-4 bg-gray-50 rounded-lg">
        {/* Time Range */}
        <div className="flex items-center gap-2">
          <span className="text-sm font-medium">Time Range:</span>
          <Select
            value={props.timeRange}
            onValueChange={(value) => {
              const range = calculateTimeRange(value as any, props.customTimeRange)
              handleTimeRangeChange(range)
            }}
          >
            <SelectTrigger className="w-40">
              <SelectValue placeholder="Select range" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="1h">Last 1 hour</SelectItem>
              <SelectItem value="6h">Last 6 hours</SelectItem>
              <SelectItem value="24h">Last 24 hours</SelectItem>
              <SelectItem value="7d">Last 7 days</SelectItem>
              <SelectItem value="30d">Last 30 days</SelectItem>
              <SelectItem value="custom">Custom</SelectItem>
            </SelectContent>
          </Select>
        </div>

        {/* Auto Refresh */}
        <div className="flex items-center gap-2">
          <RefreshCw className={`h-4 w-4 ${autoRefresh ? 'animate-spin' : ''}`} />
          <span className="text-sm font-medium">Auto-refresh:</span>
          <Select
            value={autoRefresh?.toString() || 'null'}
            onValueChange={(value) => setAutoRefresh(value === 'null' ? null : parseInt(value))}
          >
            <SelectTrigger className="w-32">
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="30">30 seconds</SelectItem>
              <SelectItem value="60">60 seconds</SelectItem>
              <SelectItem value="300">5 minutes</SelectItem>
              <SelectItem value="null">Manual</SelectItem>
            </SelectContent>
          </Select>
        </div>

        {/* Filters */}
        <div className="flex items-center gap-2">
          <span className="text-sm font-medium">Severity:</span>
          <Select
            value={filters.severityLevel?.[0] || 'all'}
            onValueChange={(value) => {
              handleFiltersChange({
                severityLevel: value === 'all' ? undefined : [value as any],
              })
            }}
          >
            <SelectTrigger className="w-32">
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="all">All</SelectItem>
              <SelectItem value="critical">Critical</SelectItem>
              <SelectItem value="high">High</SelectItem>
              <SelectItem value="medium">Medium</SelectItem>
              <SelectItem value="low">Low</SelectItem>
            </SelectContent>
          </Select>
        </div>

        <Input
          placeholder="Search user..."
          className="w-48"
          onChange={(e) => {
            handleFiltersChange({
              userIds: e.target.value ? [e.target.value] : undefined,
            })
          }}
        />
      </div>

      {/* Active Alerts */}
      {activeAlerts.length > 0 && (
        <Alert variant="destructive">
          <AlertTriangle className="h-4 w-4" />
          <AlertDescription>
            {activeAlerts.length} active alert{activeAlerts.length > 1 ? 's' : ''}:{' '}
            {activeAlerts.map((a) => a.message).join(' | ')}
            <Button variant="link" className="ml-2">
              View Alerts
            </Button>
          </AlertDescription>
        </Alert>
      )}

      {/* Loading State */}
      {isLoading && (
        <div className="flex items-center justify-center h-64">
          <RefreshCw className="h-8 w-8 animate-spin text-gray-400" />
          <span className="ml-3 text-gray-500">Loading security data...</span>
        </div>
      )}

      {/* Dashboard Panels */}
      {!isLoading && metrics && (
        <div className={`grid gap-6 ${props.compact ? 'grid-cols-1' : 'grid-cols-1 lg:grid-cols-2'}`}>
          {/* Access Denials Panel */}
          {props.panels?.accessDenials !== false && (
            <AccessDenialsPanel
              data={metrics.accessDenials}
              timeRange={selectedTimeRange}
              chartColors={chartColors}
              onDrillDown={props.onAccessDenialDrillDown}
            />
          )}

          {/* PII Activity Panel */}
          {props.panels?.piiActivity !== false && (
            <PiiActivityPanel
              activities={piiActivities}
              metrics={metrics.piiActivity}
              onActivityClick={props.onPiiActivityClick}
            />
          )}

          {/* Security Events Timeline */}
          {props.panels?.securityEvents !== false && (
            <SecurityEventsPanel
              events={events}
              metrics={metrics.securityEvents}
              chartColors={chartColors}
              onEventClick={props.onEventClick}
            />
          )}

          {/* User Activity Heatmap */}
          {props.panels?.userActivityHeatmap !== false && (
            <UserActivityHeatmapPanel
              data={metrics.userActivity}
              onUserClick={props.onUserActivityClick}
            />
          )}
        </div>
      )}

      {/* Links to Related Components */}
      {(props.enableAuditViewerLink || props.enableRoleManagerLink) && (
        <div className="flex gap-4 p-4 bg-blue-50 rounded-lg">
          {props.enableAuditViewerLink && (
            <Button variant="outline">
              <Activity className="h-4 w-4 mr-2" />
              View Full Audit Log
            </Button>
          )}
          {props.enableRoleManagerLink && (
            <Button variant="outline">
              <Users className="h-4 w-4 mr-2" />
              Manage Roles
            </Button>
          )}
        </div>
      )}
    </div>
  )
}
```

### Panel Components

```tsx
/**
 * Access Denials Panel
 */
function AccessDenialsPanel(props: {
  data: SecurityMetrics['accessDenials']
  timeRange: TimeRange
  chartColors: Record<string, string>
  onDrillDown?: (userId: string, resourceId: string) => void
}) {
  const { data, timeRange, chartColors, onDrillDown } = props

  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <Shield className="h-5 w-5 text-red-600" />
          Access Denials ({data.total})
        </CardTitle>
      </CardHeader>
      <CardContent>
        {/* Top Reasons */}
        <div className="mb-4">
          <h4 className="text-sm font-medium mb-2">Top Reasons:</h4>
          <div className="space-y-2">
            {Object.entries(data.byReason)
              .sort(([, a], [, b]) => b - a)
              .slice(0, 5)
              .map(([reason, count]) => (
                <div key={reason} className="flex items-center gap-2">
                  <div className="flex-1 bg-gray-200 rounded-full h-6">
                    <div
                      className="bg-red-600 h-6 rounded-full flex items-center px-2 text-xs text-white"
                      style={{ width: `${(count / data.total) * 100}%` }}
                    >
                      {reason} ({count})
                    </div>
                  </div>
                </div>
              ))}
          </div>
        </div>

        {/* Trend Chart */}
        <div className="mb-4">
          <h4 className="text-sm font-medium mb-2">Trend ({timeRange.label}):</h4>
          <ResponsiveContainer width="100%" height={150}>
            <LineChart data={data.trend}>
              <XAxis
                dataKey="timestamp"
                tickFormatter={(ts) => new Date(ts).toLocaleTimeString()}
              />
              <YAxis />
              <Tooltip />
              <Line type="monotone" dataKey="count" stroke={chartColors.high} strokeWidth={2} />
            </LineChart>
          </ResponsiveContainer>
        </div>

        {/* Top Users */}
        <div>
          <h4 className="text-sm font-medium mb-2">Top Users:</h4>
          <div className="space-y-1 text-sm">
            {data.byUser.slice(0, 5).map((user) => (
              <div key={user.userId} className="flex justify-between">
                <span className="truncate">{user.userName}</span>
                <Badge variant="outline">{user.count}</Badge>
              </div>
            ))}
          </div>
        </div>

        <Button variant="link" className="mt-4 p-0">
          View Details →
        </Button>
      </CardContent>
    </Card>
  )
}

/**
 * PII Activity Panel
 */
function PiiActivityPanel(props: {
  activities: PiiActivity[]
  metrics: SecurityMetrics['piiActivity']
  onActivityClick?: (activity: PiiActivity) => void
}) {
  const { activities, metrics, onActivityClick } = props

  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <Shield className="h-5 w-5 text-blue-600" />
          PII Activity ({metrics.total})
        </CardTitle>
      </CardHeader>
      <CardContent>
        {/* By Type */}
        <div className="mb-4">
          <h4 className="text-sm font-medium mb-2">By Type:</h4>
          <ResponsiveContainer width="100%" height={120}>
            <BarChart
              data={Object.entries(metrics.byType).map(([type, count]) => ({
                type,
                count,
              }))}
            >
              <XAxis dataKey="type" />
              <YAxis />
              <Tooltip />
              <Bar dataKey="count" fill="#3b82f6" />
            </BarChart>
          </ResponsiveContainer>
        </div>

        {/* Recent Activity Table */}
        <div>
          <h4 className="text-sm font-medium mb-2">Recent Activity:</h4>
          <div className="border rounded-lg overflow-hidden">
            <table className="w-full text-sm">
              <thead className="bg-gray-50">
                <tr>
                  <th className="px-3 py-2 text-left">User</th>
                  <th className="px-3 py-2 text-left">PII Type</th>
                  <th className="px-3 py-2 text-left">Action</th>
                </tr>
              </thead>
              <tbody>
                {activities.slice(0, 5).map((activity) => (
                  <tr
                    key={activity.id}
                    className="border-t hover:bg-gray-50 cursor-pointer"
                    onClick={() => onActivityClick?.(activity)}
                  >
                    <td className="px-3 py-2 truncate max-w-[120px]">
                      {activity.userEmail}
                    </td>
                    <td className="px-3 py-2">
                      <Badge variant="outline">{activity.piiType}</Badge>
                    </td>
                    <td className="px-3 py-2">{activity.action}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>

        {metrics.complianceViolations > 0 && (
          <Alert variant="destructive" className="mt-4">
            <AlertTriangle className="h-4 w-4" />
            <AlertDescription>
              {metrics.complianceViolations} compliance violation
              {metrics.complianceViolations > 1 ? 's' : ''} detected
            </AlertDescription>
          </Alert>
        )}

        <Button variant="link" className="mt-4 p-0">
          View All PII Activity →
        </Button>
      </CardContent>
    </Card>
  )
}

/**
 * Security Events Panel
 */
function SecurityEventsPanel(props: {
  events: SecurityEvent[]
  metrics: SecurityMetrics['securityEvents']
  chartColors: Record<string, string>
  onEventClick?: (event: SecurityEvent) => void
}) {
  const { events, metrics, chartColors, onEventClick } = props

  const criticalEvents = events.filter((e) => e.severity === 'critical').slice(0, 3)

  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <AlertTriangle className="h-5 w-5 text-amber-600" />
          Security Events Timeline
        </CardTitle>
      </CardHeader>
      <CardContent>
        {/* Summary Badges */}
        <div className="flex gap-2 mb-4">
          <Badge variant="destructive">Critical: {metrics.critical}</Badge>
          <Badge className="bg-orange-600">High: {metrics.high}</Badge>
          <Badge className="bg-yellow-600">Medium: {metrics.medium}</Badge>
          <Badge variant="outline">Low: {metrics.low}</Badge>
        </div>

        {/* Timeline Chart */}
        <ResponsiveContainer width="100%" height={150}>
          <LineChart data={metrics.bySeverity}>
            <XAxis
              dataKey="timestamp"
              tickFormatter={(ts) => new Date(ts).toLocaleTimeString()}
            />
            <YAxis />
            <Tooltip />
            <Line
              type="monotone"
              dataKey="count"
              stroke={chartColors.critical}
              strokeWidth={2}
            />
          </LineChart>
        </ResponsiveContainer>

        {/* Critical Events List */}
        <div className="mt-4">
          <h4 className="text-sm font-medium mb-2">Recent Critical Events:</h4>
          <ul className="space-y-2 text-sm">
            {criticalEvents.map((event) => (
              <li
                key={event.id}
                className="flex items-start gap-2 cursor-pointer hover:bg-gray-50 p-2 rounded"
                onClick={() => onEventClick?.(event)}
              >
                <span className="text-red-600">•</span>
                <div className="flex-1">
                  <span className="font-medium">
                    {new Date(event.timestamp).toLocaleTimeString()}
                  </span>{' '}
                  - {event.action} ({event.userEmail})
                </div>
              </li>
            ))}
          </ul>
        </div>

        <Button variant="link" className="mt-4 p-0">
          View All Events →
        </Button>
      </CardContent>
    </Card>
  )
}

/**
 * User Activity Heatmap Panel
 */
function UserActivityHeatmapPanel(props: {
  data: SecurityMetrics['userActivity']
  onUserClick?: (userId: string, timeSlot: string) => void
}) {
  const { data, onUserClick } = props

  const hours = Array.from({ length: 6 }, (_, i) => i * 4) // 0, 4, 8, 12, 16, 20

  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <Activity className="h-5 w-5 text-green-600" />
          User Activity Heatmap
        </CardTitle>
      </CardHeader>
      <CardContent>
        <div className="overflow-x-auto">
          <table className="w-full text-xs">
            <thead>
              <tr>
                <th className="px-2 py-1 text-left">User</th>
                {hours.map((hour) => (
                  <th key={hour} className="px-2 py-1 text-center">
                    {hour}h
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {data.heatmap.slice(0, 5).map((row) => (
                <tr key={row.userId}>
                  <td className="px-2 py-1 truncate max-w-[100px]">{row.userName}</td>
                  {hours.map((hour) => {
                    const count = row.hourlyActivity[hour] || 0
                    const intensity = Math.min(count / 50, 1)
                    const bg = `rgba(59, 130, 246, ${intensity})`
                    return (
                      <td
                        key={hour}
                        className="px-2 py-1 text-center cursor-pointer hover:ring-2 hover:ring-blue-400"
                        style={{ backgroundColor: bg }}
                        onClick={() => onUserClick?.(row.userId, `${hour}:00`)}
                      >
                        {count > 0 ? count : ''}
                      </td>
                    )
                  })}
                </tr>
              ))}
            </tbody>
          </table>
        </div>

        <div className="flex items-center gap-4 mt-4 text-xs text-gray-600">
          <span>Legend:</span>
          <div className="flex items-center gap-1">
            <div className="w-4 h-4 bg-blue-100" />
            <span>Low (0-20)</span>
          </div>
          <div className="flex items-center gap-1">
            <div className="w-4 h-4 bg-blue-400" />
            <span>Medium (20-50)</span>
          </div>
          <div className="flex items-center gap-1">
            <div className="w-4 h-4 bg-blue-600" />
            <span>High (&gt;50)</span>
          </div>
        </div>

        <Button variant="link" className="mt-4 p-0">
          View User Details →
        </Button>
      </CardContent>
    </Card>
  )
}
```

---

## Event Handlers

### User Interactions

```typescript
/**
 * Handle time range selection
 */
function handleTimeRangeSelect(
  range: '1h' | '6h' | '24h' | '7d' | '30d' | 'custom',
  customRange?: { start: Date; end: Date }
) {
  const timeRange = calculateTimeRange(range, customRange)

  // Update URL params for shareable links
  const url = new URL(window.location.href)
  url.searchParams.set('range', range)
  if (range === 'custom' && customRange) {
    url.searchParams.set('start', customRange.start.toISOString())
    url.searchParams.set('end', customRange.end.toISOString())
  }
  window.history.pushState({}, '', url)

  // Trigger callback
  props.onTimeRangeChange?.(timeRange)
}

/**
 * Handle filter changes with debouncing
 */
const debouncedFilterChange = useMemo(
  () =>
    debounce((filters: SecurityFilters) => {
      props.onFiltersChange?.(filters)
    }, props.filterDebounceMs || 300),
  [props.onFiltersChange, props.filterDebounceMs]
)

function handleFilterChange(filterKey: keyof SecurityFilters, value: any) {
  const newFilters = { ...filters, [filterKey]: value }
  setFilters(newFilters)
  debouncedFilterChange(newFilters)
}

/**
 * Handle export action
 */
async function handleExportClick(format: 'csv' | 'pdf' | 'json') {
  try {
    const data = {
      metrics: metricsQuery.data,
      events: eventsQuery.data?.events,
      piiActivities: piiQuery.data?.activities,
      timeRange: selectedTimeRange,
      filters,
    }

    const client = new SecurityAPIClient()
    const blob = await client.exportData({
      format,
      data,
      fileName: `security-dashboard-${format}-${Date.now()}`,
    })

    // Trigger download
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `security-dashboard-${format}-${Date.now()}.${format}`
    a.click()
    URL.revokeObjectURL(url)

    // Callback
    props.onExport?.(format, data)
  } catch (error) {
    console.error('Export failed:', error)
    // Show error toast
  }
}

/**
 * Handle panel drill-down
 */
function handlePanelDrillDown(
  panelType: 'access_denials' | 'pii_activity' | 'security_events' | 'user_activity',
  context: any
) {
  switch (panelType) {
    case 'access_denials':
      props.onAccessDenialDrillDown?.(context.userId, context.resourceId)
      break
    case 'pii_activity':
      props.onPiiActivityClick?.(context.activity)
      break
    case 'security_events':
      props.onEventClick?.(context.event)
      break
    case 'user_activity':
      props.onUserActivityClick?.(context.userId, context.timeSlot)
      break
  }
}

/**
 * Handle alert threshold breach
 */
function checkAlertThresholds(metrics: SecurityMetrics) {
  const alerts: SecurityAlert[] = []

  // Check failed logins
  const failedLogins = metrics.securityEvents.topEvents.filter(
    (e) => e.eventType === 'login_failed'
  )
  const userLoginCounts = failedLogins.reduce((acc, event) => {
    acc[event.userId] = (acc[event.userId] || 0) + 1
    return acc
  }, {} as Record<string, number>)

  Object.entries(userLoginCounts).forEach(([userId, count]) => {
    if (count > props.alertThresholds?.maxFailedLogins || 5) {
      alerts.push({
        id: `failed-login-${userId}`,
        timestamp: new Date(),
        type: 'failed_logins',
        severity: 'high',
        message: `User ${userId} has ${count} failed login attempts`,
        affectedUsers: [userId],
        affectedResources: [],
        thresholdValue: props.alertThresholds?.maxFailedLogins || 5,
        actualValue: count,
        recommendedAction: 'Lock account or require password reset',
        autoRemediate: props.autoCreateIncidents || false,
      })
    }
  })

  // Check PII access
  const piiAccessCount = metrics.piiActivity.total
  if (piiAccessCount > (props.alertThresholds?.maxPiiAccess || 20)) {
    alerts.push({
      id: `pii-access-${Date.now()}`,
      timestamp: new Date(),
      type: 'pii_access',
      severity: 'medium',
      message: `Unusually high PII access: ${piiAccessCount} events`,
      affectedUsers: [],
      affectedResources: [],
      thresholdValue: props.alertThresholds?.maxPiiAccess || 20,
      actualValue: piiAccessCount,
      recommendedAction: 'Review PII access logs for compliance',
      autoRemediate: false,
    })
  }

  // Trigger callbacks
  alerts.forEach((alert) => {
    props.onAlertThresholdBreached?.(alert)
  })

  setActiveAlerts(alerts)
}
```

---

## Accessibility

### WCAG 2.1 AA Compliance

```typescript
/**
 * Accessibility features:
 * 1. Keyboard navigation for all panels
 * 2. ARIA labels for chart elements
 * 3. High contrast mode support
 * 4. Screen reader announcements
 * 5. Focus management
 */

// Keyboard navigation
useEffect(() => {
  function handleKeyDown(e: KeyboardEvent) {
    if (e.key === 'Tab' && e.shiftKey) {
      // Navigate to previous panel
    } else if (e.key === 'Tab') {
      // Navigate to next panel
    } else if (e.key === 'Enter' && document.activeElement) {
      // Activate focused element
    }
  }

  window.addEventListener('keydown', handleKeyDown)
  return () => window.removeEventListener('keydown', handleKeyDown)
}, [])

// ARIA labels for charts
<BarChart aria-label="Access denials by reason" role="img">
  <Bar dataKey="count" aria-label="Count" />
</BarChart>

// Screen reader announcements
const [announcement, setAnnouncement] = useState('')

useEffect(() => {
  if (activeAlerts.length > 0) {
    setAnnouncement(`${activeAlerts.length} security alerts detected`)
  }
}, [activeAlerts])

<div role="status" aria-live="polite" className="sr-only">
  {announcement}
</div>

// High contrast support
const highContrastColors = {
  critical: '#ff0000',
  high: '#ff8800',
  medium: '#ffcc00',
  low: '#00ff00',
}

const colors = props.chartConfig?.colorScheme === 'security-high-contrast'
  ? highContrastColors
  : chartColors
```

### Focus Management

```typescript
/**
 * Manage focus for modal dialogs and panel expansion
 */
const panelRefs = useRef<Record<string, HTMLDivElement | null>>({})

function focusPanel(panelId: string) {
  const panel = panelRefs.current[panelId]
  if (panel) {
    panel.focus()
    panel.scrollIntoView({ behavior: 'smooth', block: 'nearest' })
  }
}

// Example usage
<Card
  ref={(el) => (panelRefs.current['access-denials'] = el)}
  tabIndex={0}
  aria-labelledby="access-denials-title"
  onFocus={() => setExpandedPanel('access-denials')}
>
  <CardHeader>
    <CardTitle id="access-denials-title">Access Denials</CardTitle>
  </CardHeader>
  {/* ... */}
</Card>
```

---

## Performance

### Optimization Strategies

```typescript
/**
 * Performance targets:
 * - Initial load: <1.5s (with 10,000 events)
 * - Auto-refresh update: <800ms
 * - Filter change: <300ms
 * - Panel drill-down: <500ms
 * - Export: <2s (for 50,000 events)
 */

// 1. Data virtualization for large datasets
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualizedEventList({ events }: { events: SecurityEvent[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: events.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60, // Estimated row height
    overscan: 10, // Render 10 extra items above/below viewport
  })

  return (
    <div ref={parentRef} className="h-96 overflow-auto">
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const event = events[virtualItem.index]
          return (
            <div
              key={virtualItem.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
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

// 2. Memoization for expensive computations
const processedMetrics = useMemo(() => {
  if (!metricsQuery.data) return null

  return {
    ...metricsQuery.data,
    accessDenials: {
      ...metricsQuery.data.accessDenials,
      topReasons: Object.entries(metricsQuery.data.accessDenials.byReason)
        .sort(([, a], [, b]) => b - a)
        .slice(0, 5),
    },
  }
}, [metricsQuery.data])

// 3. Debounced filter updates
import { useDebouncedValue } from '@/hooks/useDebouncedValue'

const debouncedFilters = useDebouncedValue(filters, props.filterDebounceMs || 300)

// Use debounced filters in queries
const metricsQuery = useSecurityMetrics({
  tenantIds: tenantIdArray,
  timeRange: selectedTimeRange,
  filters: debouncedFilters, // Not `filters`
  refreshInterval: autoRefresh,
})

// 4. Progressive data loading
function useProgressiveSecurityData(params: {
  tenantIds: string[]
  timeRange: TimeRange
  filters?: SecurityFilters
}) {
  const [loadedChunks, setLoadedChunks] = useState(0)
  const chunkSize = 1000

  const { data, isLoading } = useQuery({
    queryKey: ['security-events-progressive', params, loadedChunks],
    queryFn: async () => {
      const client = new SecurityAPIClient()
      return client.getSecurityEvents({
        ...params,
        limit: chunkSize,
        offset: loadedChunks * chunkSize,
      })
    },
    keepPreviousData: true,
  })

  function loadMore() {
    setLoadedChunks((prev) => prev + 1)
  }

  return { data, isLoading, loadMore, hasMore: data?.total || 0 > (loadedChunks + 1) * chunkSize }
}

// 5. Chart data sampling for large datasets
function sampleChartData(data: any[], maxPoints: number): any[] {
  if (data.length <= maxPoints) return data

  const step = Math.ceil(data.length / maxPoints)
  return data.filter((_, i) => i % step === 0)
}

const sampledTrendData = useMemo(
  () => sampleChartData(metrics?.accessDenials.trend || [], props.maxDataPoints || 1000),
  [metrics?.accessDenials.trend, props.maxDataPoints]
)

// 6. Code splitting for heavy components
const PdfExportDialog = lazy(() => import('./PdfExportDialog'))

{exportDialogOpen && (
  <Suspense fallback={<div>Loading export dialog...</div>}>
    <PdfExportDialog {...exportProps} />
  </Suspense>
)}
```

### Performance Monitoring

```typescript
/**
 * Track performance metrics
 */
import { useEffect } from 'react'

function usePerformanceTracking(componentName: string) {
  useEffect(() => {
    const startTime = performance.now()

    return () => {
      const endTime = performance.now()
      const renderTime = endTime - startTime

      if (renderTime > 1500) {
        console.warn(`${componentName} slow render: ${renderTime.toFixed(2)}ms`)
      }

      // Send to analytics
      if (window.analytics) {
        window.analytics.track('Component Render Time', {
          component: componentName,
          renderTime,
        })
      }
    }
  }, [componentName])
}

// Usage
usePerformanceTracking('SecurityDashboard')
```

---

## Multi-Domain Examples

### 1. Healthcare Tenant (HIPAA Compliance)

```typescript
<SecurityDashboard
  tenantIds={['healthcare-corp']}
  timeRange="24h"
  refreshInterval={60}
  initialFilters={{
    piiTypes: ['medical', 'ssn'],
    eventTypes: ['pii_access', 'data_export'],
  }}
  alertThresholds={{
    maxPiiAccess: 10, // Stricter for healthcare
    maxExportSizeMB: 100,
  }}
  autoCreateIncidents={true}
  chartConfig={{
    colorScheme: 'security-high-contrast',
  }}
  onPiiActivityClick={(activity) => {
    // Log HIPAA audit event
    auditLogger.log({
      type: 'HIPAA_PII_ACCESS',
      activity,
      timestamp: new Date(),
    })
  }}
/>
```

### 2. Financial Institution (SOC 2 + PCI-DSS)

```typescript
<SecurityDashboard
  tenantIds={['bank-xyz']}
  timeRange="7d"
  refreshInterval={30} // More frequent for finance
  initialFilters={{
    severityLevel: ['critical', 'high'],
    piiTypes: ['credit_card', 'ssn', 'financial'],
  }}
  alertThresholds={{
    maxFailedLogins: 3,
    maxPiiAccess: 5,
    maxPrivilegeChanges: 1,
  }}
  autoCreateIncidents={true}
  enableCrossTenantView={false}
  onAlertThresholdBreached={(alert) => {
    // Immediate notification for critical alerts
    if (alert.severity === 'critical') {
      notificationService.sendPagerDuty({
        title: 'Critical Security Alert',
        description: alert.message,
        severity: 'critical',
      })
    }
  }}
/>
```

### 3. Legal Research Firm (Attorney-Client Privilege)

```typescript
<SecurityDashboard
  tenantIds={['law-firm-abc']}
  timeRange="30d"
  refreshInterval={300}
  initialFilters={{
    resourceTypes: ['document', 'entity'],
    eventTypes: ['access_denied', 'privilege_change'],
  }}
  panels={{
    accessDenials: true,
    piiActivity: true,
    securityEvents: true,
    userActivityHeatmap: false, // Not needed
  }}
  onAccessDenialDrillDown={(userId, resourceId) => {
    // Check if document has privilege protection
    const doc = getDocument(resourceId)
    if (doc.tags.includes('privileged')) {
      // Log privileged document access attempt
      auditLogger.logPrivilegedAccess({ userId, resourceId })
    }
  }}
/>
```

### 4. Marketing Agency (Client Data Separation)

```typescript
<SecurityDashboard
  tenantIds={['agency-clients']}
  timeRange="24h"
  refreshInterval={60}
  enableCrossTenantView={true} // Admin view
  initialFilters={{
    eventTypes: ['access_denied'],
  }}
  chartConfig={{
    showLabels: true,
    animated: true,
  }}
  onUserActivityClick={(userId, timeSlot) => {
    // Show user activity across all client tenants
    router.push(`/admin/users/${userId}/activity?time=${timeSlot}`)
  }}
/>
```

### 5. HR Platform (Employee PII)

```typescript
<SecurityDashboard
  tenantIds={['hr-platform']}
  timeRange="7d"
  refreshInterval={120}
  initialFilters={{
    piiTypes: ['ssn', 'dob', 'address'],
    eventTypes: ['pii_access', 'data_export'],
  }}
  alertThresholds={{
    maxPiiAccess: 50, // HR accesses PII frequently
    maxExportSizeMB: 500,
  }}
  panels={{
    piiActivity: true,
    userActivityHeatmap: true,
    accessDenials: false,
    securityEvents: true,
  }}
  onExport={(format, data) => {
    // Encrypt exported HR data
    const encrypted = encryptService.encrypt(JSON.stringify(data))
    downloadFile(`hr-security-${format}.enc`, encrypted)
  }}
/>
```

### 6. Research (RSRCH - Utilitario) - Founder Data Privacy

```typescript
<SecurityDashboard
  tenantIds={['rsrch-vc-firm']}
  timeRange="30d"
  refreshInterval={null} // Manual refresh for research compliance
  initialFilters={{
    piiTypes: ['medical', 'email'],
    eventTypes: ['pii_access', 'data_export'],
  }}
  chartConfig={{
    showLabels: true,
    colorScheme: 'security-default',
  }}
  onPiiActivityClick={(activity) => {
    // Check IRB approval for PII access
    const approval = getIRBApproval(activity.documentId)
    if (!approval) {
      showWarning('This document requires IRB approval for PII access')
    }
  }}
/>
```

### 7. E-commerce Platform (Payment Data)

```typescript
<SecurityDashboard
  tenantIds={['ecommerce-store']}
  timeRange="24h"
  refreshInterval={60}
  initialFilters={{
    piiTypes: ['credit_card'],
    eventTypes: ['pii_access', 'data_export', 'access_denied'],
  }}
  alertThresholds={{
    maxPiiAccess: 1000, // High volume
    maxFailedLogins: 5,
  }}
  compact={false}
  onAlertThresholdBreached={(alert) => {
    if (alert.type === 'data_exfiltration') {
      // Auto-lock account on suspected data theft
      securityService.lockAccount(alert.affectedUsers[0])
    }
  }}
/>
```

---

## Integration

### Integration with Objective Layer Primitives

```typescript
/**
 * Integrate SecurityDashboard with OL primitives:
 * 1. AccessPolicy (OL) - Check who has access to security data
 * 2. AuditLog (OL) - Fetch audit events for timeline
 * 3. User (OL) - Resolve user details for heatmap
 * 4. Tenant (OL) - Multi-tenant filtering
 */

import { getAccessPolicy, fetchAuditLogs, getUser, getTenant } from '@/lib/ol'

/**
 * Example: Fetch security metrics using OL primitives
 */
async function fetchSecurityMetricsFromOL(params: {
  tenantIds: string[]
  timeRange: TimeRange
  filters?: SecurityFilters
}): Promise<SecurityMetrics> {
  // 1. Check access policy - Can user view security data?
  const policy = await getAccessPolicy({
    userId: currentUser.id,
    resourceType: 'security_dashboard',
    action: 'view',
  })

  if (!policy.allowed) {
    throw new Error('Access denied: Insufficient permissions to view security dashboard')
  }

  // 2. Fetch audit logs from OL
  const auditLogs = await fetchAuditLogs({
    tenantIds: params.tenantIds,
    startTime: params.timeRange.start,
    endTime: params.timeRange.end,
    eventTypes: params.filters?.eventTypes,
  })

  // 3. Aggregate metrics
  const accessDenials = auditLogs.filter((log) => log.eventType === 'access_denied')
  const piiActivity = auditLogs.filter((log) =>
    ['pii_view', 'pii_export', 'pii_modify'].includes(log.eventType)
  )

  // 4. Resolve user details
  const userIds = [...new Set(auditLogs.map((log) => log.userId))]
  const users = await Promise.all(userIds.map((id) => getUser(id)))
  const userMap = Object.fromEntries(users.map((u) => [u.id, u]))

  // 5. Build metrics
  return {
    timeRange: params.timeRange,
    accessDenials: {
      total: accessDenials.length,
      byReason: aggregateByReason(accessDenials),
      byUser: aggregateByUser(accessDenials, userMap),
      byResource: aggregateByResource(accessDenials),
      trend: buildTrend(accessDenials, params.timeRange),
    },
    piiActivity: {
      total: piiActivity.length,
      byType: aggregateByPiiType(piiActivity),
      byUser: aggregateByUser(piiActivity, userMap),
      byAction: aggregateByAction(piiActivity),
      complianceViolations: countComplianceViolations(piiActivity),
    },
    securityEvents: {
      total: auditLogs.length,
      critical: auditLogs.filter((log) => log.severity === 'critical').length,
      high: auditLogs.filter((log) => log.severity === 'high').length,
      medium: auditLogs.filter((log) => log.severity === 'medium').length,
      low: auditLogs.filter((log) => log.severity === 'low').length,
      bySeverity: buildSeverityTimeline(auditLogs, params.timeRange),
      topEvents: auditLogs.slice(0, 10).map((log) => transformToSecurityEvent(log, userMap)),
    },
    userActivity: {
      activeUsers: userIds.length,
      suspiciousUsers: detectSuspiciousUsers(auditLogs, userMap),
      heatmap: buildHeatmap(auditLogs, userMap),
      riskDistribution: calculateRiskDistribution(auditLogs),
    },
  }
}

/**
 * Helper: Aggregate access denials by reason
 */
function aggregateByReason(logs: AuditLog[]): Record<string, number> {
  return logs.reduce((acc, log) => {
    const reason = log.metadata?.denialReason || 'Unknown'
    acc[reason] = (acc[reason] || 0) + 1
    return acc
  }, {} as Record<string, number>)
}

/**
 * Helper: Build user activity heatmap
 */
function buildHeatmap(
  logs: AuditLog[],
  userMap: Record<string, User>
): Array<{
  userId: string
  userName: string
  hourlyActivity: Record<string, number>
}> {
  const heatmapData: Record<
    string,
    { userId: string; userName: string; hourlyActivity: Record<string, number> }
  > = {}

  logs.forEach((log) => {
    const userId = log.userId
    const hour = new Date(log.timestamp).getHours()

    if (!heatmapData[userId]) {
      heatmapData[userId] = {
        userId,
        userName: userMap[userId]?.name || userId,
        hourlyActivity: {},
      }
    }

    heatmapData[userId].hourlyActivity[hour] =
      (heatmapData[userId].hourlyActivity[hour] || 0) + 1
  })

  return Object.values(heatmapData)
}
```

### Integration with AuditViewer

```tsx
/**
 * Link SecurityDashboard to AuditViewer for drill-down
 */
import { useRouter } from 'next/router'

function SecurityDashboardWithAuditLink(props: SecurityDashboardProps) {
  const router = useRouter()

  return (
    <SecurityDashboard
      {...props}
      enableAuditViewerLink={true}
      onEventClick={(event) => {
        // Navigate to AuditViewer with pre-filled filters
        router.push({
          pathname: '/audit-viewer',
          query: {
            tenantId: event.tenantId,
            userId: event.userId,
            eventType: event.eventType,
            startTime: event.timestamp.toISOString(),
          },
        })
      }}
      onAccessDenialDrillDown={(userId, resourceId) => {
        router.push({
          pathname: '/audit-viewer',
          query: {
            userId,
            resourceId,
            eventType: 'access_denied',
          },
        })
      }}
    />
  )
}
```

### Integration with RoleManager

```tsx
/**
 * Link SecurityDashboard to RoleManager for privilege escalation
 */
function SecurityDashboardWithRoleManager(props: SecurityDashboardProps) {
  const router = useRouter()

  return (
    <SecurityDashboard
      {...props}
      enableRoleManagerLink={true}
      onEventClick={(event) => {
        if (event.eventType === 'privilege_change') {
          // Navigate to RoleManager for this user
          router.push({
            pathname: '/role-manager',
            query: {
              tenantId: event.tenantId,
              userId: event.userId,
            },
          })
        }
      }}
    />
  )
}
```

---

## Testing Strategy

### Unit Tests

```typescript
/**
 * Test: Renders all panels
 */
import { render, screen } from '@testing-library/react'
import { SecurityDashboard } from './SecurityDashboard'

test('renders all 4 panels when enabled', () => {
  render(
    <SecurityDashboard
      tenantIds={['test-tenant']}
      panels={{
        accessDenials: true,
        piiActivity: true,
        securityEvents: true,
        userActivityHeatmap: true,
      }}
    />
  )

  expect(screen.getByText(/Access Denials/i)).toBeInTheDocument()
  expect(screen.getByText(/PII Activity/i)).toBeInTheDocument()
  expect(screen.getByText(/Security Events Timeline/i)).toBeInTheDocument()
  expect(screen.getByText(/User Activity Heatmap/i)).toBeInTheDocument()
})

/**
 * Test: Filters update query
 */
test('filters update security metrics query', async () => {
  const { rerender } = render(<SecurityDashboard tenantIds={['test-tenant']} />)

  // Initial render
  await waitFor(() => {
    expect(screen.getByText(/Loading security data/i)).not.toBeInTheDocument()
  })

  // Change filter
  const severitySelect = screen.getByLabelText(/Severity/i)
  fireEvent.change(severitySelect, { target: { value: 'critical' } })

  // Should trigger new query
  await waitFor(() => {
    expect(screen.getByText(/Loading security data/i)).toBeInTheDocument()
  })
})

/**
 * Test: Alert thresholds trigger callbacks
 */
test('alert threshold breach triggers callback', async () => {
  const onAlertThresholdBreached = jest.fn()

  render(
    <SecurityDashboard
      tenantIds={['test-tenant']}
      alertThresholds={{
        maxFailedLogins: 3,
      }}
      onAlertThresholdBreached={onAlertThresholdBreached}
    />
  )

  // Mock data with 5 failed logins
  mockSecurityMetrics({
    securityEvents: {
      topEvents: Array(5).fill({ eventType: 'login_failed', userId: 'test-user' }),
    },
  })

  await waitFor(() => {
    expect(onAlertThresholdBreached).toHaveBeenCalledWith(
      expect.objectContaining({
        type: 'failed_logins',
        severity: 'high',
        actualValue: 5,
        thresholdValue: 3,
      })
    )
  })
})
```

### Integration Tests

```typescript
/**
 * Test: End-to-end security dashboard flow
 */
test('full security dashboard workflow', async () => {
  // 1. Render dashboard
  render(<SecurityDashboard tenantIds={['test-tenant']} timeRange="24h" />)

  // 2. Wait for data to load
  await waitFor(() => {
    expect(screen.queryByText(/Loading security data/i)).not.toBeInTheDocument()
  })

  // 3. Verify panels rendered
  expect(screen.getByText(/Access Denials/i)).toBeInTheDocument()

  // 4. Click on access denial event
  const denialEvent = screen.getByText(/Insufficient Permissions/i)
  fireEvent.click(denialEvent)

  // 5. Should navigate to AuditViewer
  await waitFor(() => {
    expect(router.push).toHaveBeenCalledWith(
      expect.objectContaining({
        pathname: '/audit-viewer',
      })
    )
  })

  // 6. Export data
  const exportButton = screen.getByText(/Export/i)
  fireEvent.click(exportButton)

  // 7. Select CSV format
  const csvOption = screen.getByText(/CSV/i)
  fireEvent.click(csvOption)

  // 8. Verify download initiated
  await waitFor(() => {
    expect(document.querySelector('a[download]')).toBeInTheDocument()
  })
})
```

### Performance Tests

```typescript
/**
 * Test: Dashboard loads within performance budget
 */
test('dashboard loads in <1.5s with 10,000 events', async () => {
  const startTime = performance.now()

  // Mock 10,000 security events
  mockSecurityMetrics({
    securityEvents: {
      total: 10000,
      topEvents: Array(10000)
        .fill(null)
        .map((_, i) => ({
          id: `event-${i}`,
          timestamp: new Date(),
          eventType: 'access_denied',
          severity: 'medium',
        })),
    },
  })

  render(<SecurityDashboard tenantIds={['test-tenant']} />)

  await waitFor(() => {
    expect(screen.queryByText(/Loading security data/i)).not.toBeInTheDocument()
  })

  const endTime = performance.now()
  const loadTime = endTime - startTime

  expect(loadTime).toBeLessThan(1500) // 1.5s budget
})

/**
 * Test: Auto-refresh updates in <800ms
 */
test('auto-refresh updates in <800ms', async () => {
  render(<SecurityDashboard tenantIds={['test-tenant']} refreshInterval={60} />)

  await waitFor(() => {
    expect(screen.queryByText(/Loading security data/i)).not.toBeInTheDocument()
  })

  // Trigger auto-refresh
  const startTime = performance.now()

  act(() => {
    jest.advanceTimersByTime(60 * 1000) // Advance 60 seconds
  })

  await waitFor(() => {
    expect(screen.queryByText(/Loading security data/i)).not.toBeInTheDocument()
  })

  const endTime = performance.now()
  const refreshTime = endTime - startTime

  expect(refreshTime).toBeLessThan(800) // 800ms budget
})
```

### Accessibility Tests

```typescript
/**
 * Test: Keyboard navigation works
 */
test('keyboard navigation between panels', async () => {
  render(<SecurityDashboard tenantIds={['test-tenant']} />)

  await waitFor(() => {
    expect(screen.queryByText(/Loading security data/i)).not.toBeInTheDocument()
  })

  // Focus first panel
  const firstPanel = screen.getByText(/Access Denials/i).closest('[tabindex]')
  firstPanel?.focus()

  expect(document.activeElement).toBe(firstPanel)

  // Tab to next panel
  fireEvent.keyDown(document.activeElement!, { key: 'Tab' })

  const secondPanel = screen.getByText(/PII Activity/i).closest('[tabindex]')
  expect(document.activeElement).toBe(secondPanel)
})

/**
 * Test: Screen reader announces alerts
 */
test('screen reader announces security alerts', async () => {
  render(
    <SecurityDashboard
      tenantIds={['test-tenant']}
      alertThresholds={{ maxFailedLogins: 3 }}
    />
  )

  // Mock alert threshold breach
  mockSecurityMetrics({
    securityEvents: {
      topEvents: Array(5).fill({ eventType: 'login_failed', userId: 'test-user' }),
    },
  })

  await waitFor(() => {
    const announcement = screen.getByRole('status')
    expect(announcement).toHaveTextContent(/security alerts detected/i)
  })
})
```

---

## References

- [Objective Layer Primitives](/docs/primitives/ol/)
- [AuditViewer Component](/docs/primitives/il/AuditViewer.md)
- [RoleManager Component](/docs/primitives/il/RoleManager.md)
- [Recharts Documentation](https://recharts.org/)
- [TanStack Query](https://tanstack.com/query)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [HIPAA Compliance](https://www.hhs.gov/hipaa/)
- [SOC 2 Security](https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/socforserviceorganizations.html)
- [GDPR Article 15](https://gdpr-info.eu/art-15-gdpr/)
