# AuditViewer (IL Component)

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

The **AuditViewer** is a comprehensive audit log exploration component that provides forensic-level visibility into all system activities, user actions, and data changes. It enables security teams, compliance officers, and investigators to query, filter, and export audit trails for compliance reporting (SOC 2, HIPAA, GDPR Article 15), incident investigation, and historical analysis. This component transforms raw audit logs into actionable intelligence with advanced filtering, timeline visualization, and cross-referenced event correlation.

### Key Capabilities

- **Paginated Audit Log Table**: High-performance virtualized table for 1M+ events
- **Advanced Filtering**: 20+ filter dimensions (user, action, resource, time, tenant, IP, etc.)
- **Date Range Picker**: Flexible time windows with presets and custom ranges
- **Full-Text Search**: Search across event metadata, user details, resource names
- **Event Details Panel**: Expandable rows showing full event context and diff viewer
- **Timeline Visualization**: Interactive timeline showing event clusters and patterns
- **CSV/PDF Export**: One-click export for compliance reporting
- **Diff Viewer**: Before/after comparison for data modification events
- **Related Events**: Auto-link related events (e.g., create → update → delete)
- **Bookmark & Share**: Save filter combinations, share via URL
- **Real-Time Updates**: Auto-refresh for monitoring live activity
- **Multi-Tenant Support**: Filter by tenant or cross-tenant view (admin only)

### Problem Statement

Audit log analysis is critical but painful:

1. **Data Overload**: 100K+ events per day, impossible to manually review
2. **Poor Searchability**: Raw logs lack indexing, takes minutes to find events
3. **No Context**: Event shows "document updated" but not what changed
4. **Compliance Gaps**: Can't prove user actions for SOC 2, HIPAA audits
5. **Slow Investigations**: Incident response teams spend hours reconstructing timelines
6. **Export Hell**: Exporting to Excel loses formatting, metadata
7. **No Correlation**: Can't link related events (e.g., login → access → export)
8. **Retention Issues**: Logs deleted prematurely, can't investigate old incidents

### Solution

The AuditViewer provides:

- **Instant Search**: Sub-second search across millions of events with Elasticsearch
- **Visual Timeline**: See event patterns at a glance (spikes, anomalies)
- **Contextual Details**: Click event to see full diff, related events, user context
- **Smart Filters**: Pre-built compliance filters (GDPR, HIPAA, SOC 2)
- **Export Automation**: Schedule daily/weekly CSV exports, auto-upload to S3
- **Event Correlation**: Auto-link create/update/delete for same resource
- **Compliance Reports**: One-click SOC 2 Type II audit trail export
- **Immutable Logs**: Write-once storage with cryptographic verification

This transforms audit log analysis from reactive forensics to proactive compliance.

---

## Component Interface

### Primary Props

```typescript
interface AuditViewerProps {
  // ============================================================
  // Tenant & User Context
  // ============================================================

  /**
   * Tenant ID(s) to show audit logs for
   */
  tenantId?: string | string[]

  /**
   * Current user (for permission checks)
   */
  currentUser: {
    id: string
    email: string
    permissions: string[]
  }

  /**
   * Enable cross-tenant view (admin only)
   * @default false
   */
  enableCrossTenantView?: boolean

  // ============================================================
  // Initial State
  // ============================================================

  /**
   * Initial date range
   * @default 'last_24h'
   */
  initialDateRange?: '1h' | '24h' | '7d' | '30d' | '90d' | 'custom'

  /**
   * Custom date range (if initialDateRange='custom')
   */
  customDateRange?: {
    start: Date | string
    end: Date | string
  }

  /**
   * Initial filters
   */
  initialFilters?: AuditFilters

  /**
   * Initial search query
   */
  initialSearchQuery?: string

  /**
   * Auto-refresh interval (seconds)
   * @default null (no auto-refresh)
   */
  autoRefreshInterval?: 30 | 60 | 300 | null

  // ============================================================
  // Display Configuration
  // ============================================================

  /**
   * Table layout
   * @default 'comfortable'
   */
  density?: 'compact' | 'comfortable' | 'spacious'

  /**
   * Visible columns
   */
  visibleColumns?: ('timestamp' | 'user' | 'action' | 'resource' | 'ip' | 'details')[]

  /**
   * Show timeline visualization
   * @default true
   */
  showTimeline?: boolean

  /**
   * Show diff viewer for modification events
   * @default true
   */
  showDiffViewer?: boolean

  /**
   * Show related events panel
   * @default true
   */
  showRelatedEvents?: boolean

  /**
   * Enable event details expansion
   * @default true
   */
  enableDetailsExpansion?: boolean

  /**
   * Theme
   * @default 'auto'
   */
  theme?: 'light' | 'dark' | 'auto'

  // ============================================================
  // Filtering & Search
  // ============================================================

  /**
   * Enable advanced filters
   * @default true
   */
  enableAdvancedFilters?: boolean

  /**
   * Enable full-text search
   * @default true
   */
  enableFullTextSearch?: boolean

  /**
   * Available action types to filter
   * If not provided, fetched from API
   */
  availableActions?: string[]

  /**
   * Available resource types to filter
   */
  availableResourceTypes?: string[]

  /**
   * Pre-defined filter presets
   */
  filterPresets?: FilterPreset[]

  /**
   * Debounce search input (ms)
   * @default 300
   */
  searchDebounceMs?: number

  // ============================================================
  // Export Configuration
  // ============================================================

  /**
   * Enable CSV/PDF export
   * @default true
   */
  enableExport?: boolean

  /**
   * Export formats
   * @default ['csv', 'pdf', 'json']
   */
  exportFormats?: ('csv' | 'pdf' | 'json')[]

  /**
   * Include full event metadata in exports
   * @default false
   */
  includeFullMetadata?: boolean

  /**
   * Max events per export
   * @default 10000
   */
  maxExportEvents?: number

  // ============================================================
  // Pagination
  // ============================================================

  /**
   * Page size
   * @default 50
   */
  pageSize?: 25 | 50 | 100 | 200

  /**
   * Enable infinite scroll
   * @default false (use pagination)
   */
  enableInfiniteScroll?: boolean

  // ============================================================
  // Performance
  // ============================================================

  /**
   * Enable virtualization for large datasets
   * @default true
   */
  enableVirtualization?: boolean

  /**
   * Prefetch next page
   * @default true
   */
  prefetchNextPage?: boolean

  // ============================================================
  // Callbacks
  // ============================================================

  /**
   * Called when event is clicked
   */
  onEventClick?: (event: AuditEvent) => void

  /**
   * Called when filters change
   */
  onFiltersChange?: (filters: AuditFilters) => void

  /**
   * Called when date range changes
   */
  onDateRangeChange?: (range: DateRange) => void

  /**
   * Called when export is initiated
   */
  onExport?: (format: 'csv' | 'pdf' | 'json', events: AuditEvent[]) => void

  /**
   * Called when user clicks "View User Details"
   */
  onUserClick?: (userId: string) => void

  /**
   * Called when user clicks "View Resource Details"
   */
  onResourceClick?: (resourceType: string, resourceId: string) => void

  /**
   * Called when bookmark is created
   */
  onBookmarkCreate?: (bookmark: FilterBookmark) => void

  // ============================================================
  // Integration
  // ============================================================

  /**
   * API client for audit data
   */
  apiClient?: AuditAPIClient

  /**
   * Enable integration with SecurityDashboard
   * @default true
   */
  enableSecurityDashboardLink?: boolean

  /**
   * Enable integration with RoleManager
   * @default true
   */
  enableRoleManagerLink?: boolean

  /**
   * Custom actions to show in toolbar
   */
  customActions?: React.ReactNode
}
```

### Supporting Types

```typescript
/**
 * Audit event entry
 */
interface AuditEvent {
  id: string
  timestamp: Date
  tenantId: string
  userId: string
  userEmail: string
  userName: string
  action: string // e.g., 'document.create', 'user.login', 'role.assign'
  actionCategory: 'create' | 'read' | 'update' | 'delete' | 'auth' | 'admin' | 'system'
  resourceType: string // e.g., 'document', 'report', 'user', 'role'
  resourceId: string
  resourceName?: string
  ipAddress: string
  userAgent: string
  geoLocation?: {
    country: string
    region: string
    city: string
  }
  success: boolean
  errorMessage?: string
  duration_ms?: number
  metadata: Record<string, any> // Free-form event metadata
  beforeState?: any // For update events
  afterState?: any // For update events
  relatedEventIds?: string[] // Linked events
  severity: 'info' | 'warning' | 'error' | 'critical'
  complianceTags?: string[] // e.g., ['GDPR', 'HIPAA', 'SOC2']
}

/**
 * Audit filters
 */
interface AuditFilters {
  userIds?: string[]
  actions?: string[]
  actionCategories?: ('create' | 'read' | 'update' | 'delete' | 'auth' | 'admin' | 'system')[]
  resourceTypes?: string[]
  resourceIds?: string[]
  ipAddresses?: string[]
  success?: boolean
  severity?: ('info' | 'warning' | 'error' | 'critical')[]
  complianceTags?: string[]
  hasMetadataKey?: string // Filter events with specific metadata key
}

/**
 * Date range
 */
interface DateRange {
  start: Date
  end: Date
  label: string // e.g., "Last 24 hours"
}

/**
 * Filter preset (pre-configured filters)
 */
interface FilterPreset {
  id: string
  name: string
  description: string
  icon?: string
  filters: AuditFilters
  dateRange?: DateRange
}

/**
 * Filter bookmark (saved by user)
 */
interface FilterBookmark {
  id: string
  name: string
  filters: AuditFilters
  dateRange: DateRange
  createdAt: Date
  createdBy: string
}

/**
 * API client interface
 */
interface AuditAPIClient {
  getEvents(params: {
    tenantIds: string[]
    dateRange: DateRange
    filters?: AuditFilters
    search?: string
    page?: number
    pageSize?: number
    sortBy?: 'timestamp' | 'user' | 'action'
    sortOrder?: 'asc' | 'desc'
  }): Promise<{
    events: AuditEvent[]
    total: number
    hasMore: boolean
  }>

  getEventDetails(eventId: string): Promise<AuditEvent>

  getRelatedEvents(eventId: string): Promise<AuditEvent[]>

  exportEvents(params: {
    format: 'csv' | 'pdf' | 'json'
    events: AuditEvent[]
    fileName: string
  }): Promise<Blob>

  getAvailableActions(tenantId: string): Promise<string[]>

  getAvailableResourceTypes(tenantId: string): Promise<string[]>

  saveBookmark(bookmark: Omit<FilterBookmark, 'id' | 'createdAt'>): Promise<FilterBookmark>

  getBookmarks(userId: string): Promise<FilterBookmark[]>
}
```

---

## Visual Design

### ASCII Wireframe (Desktop)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Audit Viewer                                          🏢 Tenant: Acme Corp   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  📅 [Last 24 hours ▼]  🔍 [Search events...        ]  🔖 Bookmarks ▼        │
│                                                                              │
│  Filters: User: [All ▼] Action: [All ▼] Resource: [All ▼] [+ Advanced]     │
│  Presets: [GDPR Requests] [SOC 2 Changes] [Failed Logins] [PII Access]     │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│ 📊 Timeline (Last 24 hours)                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │       ▁▂▃▅█▆▄▃▂▁  ▁▃▅▇█▆▄▂▁  ▁▂▄▆█▅▃▁                                  │ │
│  │    ▁▃▅▇█████████████████████████████▇▅▃▁                                │ │
│  │  ──────────────────────────────────────────────────────────            │ │
│  │  0h      6h      12h      18h      24h                                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│ 📋 Audit Log (1,234 events)                                    📤 Export ▼  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ Timestamp         User            Action           Resource      IP    │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │ 2025-10-25 14:32  jdoe@acme.com   document.update  Report #123  1.2... │ │
│  │   ▶ Details: Modified 3 fields (title, status, assignee)              │ │
│  │                                                                        │ │
│  │ 2025-10-25 14:28  asmith@acme.com user.login      —            8.8... │ │
│  │                                                                        │ │
│  │ 2025-10-25 14:15  rbrown@acme.com role.assign     User: jdoe    1.2...│ │
│  │   ▶ Details: Assigned "Editor" role to jdoe@acme.com                  │ │
│  │                                                                        │ │
│  │ 2025-10-25 13:47  jdoe@acme.com   fact.delete     Fact #456    1.2... │ │
│  │   ⚠️ Critical: Deleted founder fact with 12 source references          │ │
│  │                                                                        │ │
│  │ 2025-10-25 13:22  klee@acme.com   document.read   Doc #789     5.6... │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  < 1 2 3 ... 25 >                                 Showing 1-50 of 1,234     │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ 🔍 Event Details: document.update (Report #123)                             │
│                                                                              │
│  User: jdoe@acme.com (John Doe)                    [View User Profile]      │
│  Time: 2025-10-25 14:32:15 UTC                                              │
│  IP: 1.2.3.4 (San Francisco, CA, US)                                        │
│  Duration: 342ms                                                             │
│  Success: ✓                                                                  │
│                                                                              │
│  Changes:                                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ Field      Before                  After                               │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │ title      "Q4 Report"             "Q4 Financial Report"               │ │
│  │ status     "draft"                 "published"                         │ │
│  │ assignee   null                    "asmith@acme.com"                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Related Events (3):                                                         │
│  • 2025-10-25 10:15 - document.create (Report #123)                         │
│  • 2025-10-25 12:00 - document.update (Report #123)                         │
│  • 2025-10-25 14:32 - document.update (Report #123) ← Current               │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Compact Layout (Mobile)

```
┌─────────────────────────────┐
│ Audit Viewer          ≡     │
├─────────────────────────────┤
│                             │
│ 📅 [Last 24h ▼]            │
│ 🔍 [Search...        ]     │
│                             │
│ 📋 1,234 events             │
│ ┌─────────────────────────┐ │
│ │ 14:32 jdoe@acme.com     │ │
│ │ document.update         │ │
│ │ Report #123             │ │
│ ├─────────────────────────┤ │
│ │ 14:28 asmith@acme.com   │ │
│ │ user.login              │ │
│ │ —                       │ │
│ └─────────────────────────┘ │
│                             │
│ [Load More]                 │
│                             │
└─────────────────────────────┘
```

---

## State Management

### React State Hooks

```typescript
import { useState, useEffect, useCallback, useMemo } from 'react'
import { useQuery, useQueryClient } from '@tanstack/react-query'

export function useAuditViewer(props: AuditViewerProps) {
  const {
    tenantId,
    currentUser,
    enableCrossTenantView = false,
    initialDateRange = '24h',
    customDateRange,
    initialFilters,
    initialSearchQuery = '',
    autoRefreshInterval = null,
    showTimeline = true,
    showDiffViewer = true,
    showRelatedEvents = true,
    enableDetailsExpansion = true,
    enableAdvancedFilters = true,
    enableFullTextSearch = true,
    filterPresets = [],
    pageSize = 50,
    enableInfiniteScroll = false,
    enableVirtualization = true,
    prefetchNextPage = true,
    onEventClick,
    onFiltersChange,
    onDateRangeChange,
    onExport,
    onUserClick,
    onResourceClick,
    apiClient,
    searchDebounceMs = 300,
  } = props

  // ============================================================
  // Local State
  // ============================================================

  const [dateRange, setDateRange] = useState<DateRange>(() =>
    calculateDateRange(initialDateRange, customDateRange)
  )

  const [filters, setFilters] = useState<AuditFilters>(initialFilters || {})

  const [searchQuery, setSearchQuery] = useState(initialSearchQuery)

  const [currentPage, setCurrentPage] = useState(1)

  const [expandedEventId, setExpandedEventId] = useState<string | null>(null)

  const [selectedEvent, setSelectedEvent] = useState<AuditEvent | null>(null)

  const [sortBy, setSortBy] = useState<'timestamp' | 'user' | 'action'>('timestamp')

  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('desc')

  const [bookmarks, setBookmarks] = useState<FilterBookmark[]>([])

  const [exportDialogOpen, setExportDialogOpen] = useState(false)

  // ============================================================
  // Memoized Values
  // ============================================================

  const tenantIdArray = useMemo(() => {
    if (!tenantId) return []
    return Array.isArray(tenantId) ? tenantId : [tenantId]
  }, [tenantId])

  const canViewCrossTenant = useMemo(() => {
    return (
      enableCrossTenantView &&
      currentUser.permissions.includes('view_cross_tenant_audit')
    )
  }, [enableCrossTenantView, currentUser.permissions])

  // ============================================================
  // Callbacks
  // ============================================================

  const handleDateRangeChange = useCallback(
    (range: DateRange) => {
      setDateRange(range)
      setCurrentPage(1) // Reset to first page
      onDateRangeChange?.(range)
    },
    [onDateRangeChange]
  )

  const handleFiltersChange = useCallback(
    (newFilters: Partial<AuditFilters>) => {
      const updatedFilters = { ...filters, ...newFilters }
      setFilters(updatedFilters)
      setCurrentPage(1)
      onFiltersChange?.(updatedFilters)
    },
    [filters, onFiltersChange]
  )

  const handleSearchChange = useCallback(
    (query: string) => {
      setSearchQuery(query)
      setCurrentPage(1)
    },
    []
  )

  const handleEventClick = useCallback(
    (event: AuditEvent) => {
      setSelectedEvent(event)
      setExpandedEventId(event.id)
      onEventClick?.(event)
    },
    [onEventClick]
  )

  const handleApplyPreset = useCallback(
    (preset: FilterPreset) => {
      setFilters(preset.filters)
      if (preset.dateRange) {
        setDateRange(preset.dateRange)
      }
      setCurrentPage(1)
    },
    []
  )

  const handleBookmarkCreate = useCallback(
    async (name: string) => {
      const bookmark: Omit<FilterBookmark, 'id' | 'createdAt'> = {
        name,
        filters,
        dateRange,
        createdBy: currentUser.id,
      }

      const client = apiClient || new AuditAPIClient()
      const saved = await client.saveBookmark(bookmark)
      setBookmarks((prev) => [...prev, saved])
    },
    [filters, dateRange, currentUser.id, apiClient]
  )

  return {
    // State
    dateRange,
    filters,
    searchQuery,
    currentPage,
    expandedEventId,
    selectedEvent,
    sortBy,
    sortOrder,
    bookmarks,
    exportDialogOpen,
    tenantIdArray,
    canViewCrossTenant,

    // Actions
    setDateRange: handleDateRangeChange,
    setFilters: handleFiltersChange,
    setSearchQuery: handleSearchChange,
    setCurrentPage,
    setExpandedEventId,
    setSelectedEvent: handleEventClick,
    setSortBy,
    setSortOrder,
    setExportDialogOpen,
    applyPreset: handleApplyPreset,
    createBookmark: handleBookmarkCreate,
  }
}

/**
 * Calculate DateRange from preset
 */
function calculateDateRange(
  preset: string,
  custom?: { start: Date | string; end: Date | string }
): DateRange {
  const end = new Date()
  let start: Date
  let label: string

  switch (preset) {
    case '1h':
      start = new Date(end.getTime() - 60 * 60 * 1000)
      label = 'Last 1 hour'
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
    case '90d':
      start = new Date(end.getTime() - 90 * 24 * 60 * 60 * 1000)
      label = 'Last 90 days'
      break
    case 'custom':
      if (!custom) throw new Error('Custom date range required')
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
import { useQuery, useInfiniteQuery, useQueryClient } from '@tanstack/react-query'
import { AuditAPIClient } from './api/AuditAPIClient'

/**
 * Fetch audit events (paginated)
 */
export function useAuditEvents(params: {
  tenantIds: string[]
  dateRange: DateRange
  filters?: AuditFilters
  search?: string
  page?: number
  pageSize?: number
  sortBy?: 'timestamp' | 'user' | 'action'
  sortOrder?: 'asc' | 'desc'
  autoRefresh?: number | null
}) {
  return useQuery({
    queryKey: ['audit-events', params],
    queryFn: async () => {
      const client = new AuditAPIClient()
      return client.getEvents({
        tenantIds: params.tenantIds,
        dateRange: params.dateRange,
        filters: params.filters,
        search: params.search,
        page: params.page || 1,
        pageSize: params.pageSize || 50,
        sortBy: params.sortBy || 'timestamp',
        sortOrder: params.sortOrder || 'desc',
      })
    },
    enabled: params.tenantIds.length > 0,
    keepPreviousData: true, // Keep old data while fetching new page
    refetchInterval: params.autoRefresh ? params.autoRefresh * 1000 : false,
    staleTime: 30 * 1000, // 30 seconds
  })
}

/**
 * Fetch audit events (infinite scroll)
 */
export function useInfiniteAuditEvents(params: {
  tenantIds: string[]
  dateRange: DateRange
  filters?: AuditFilters
  search?: string
  pageSize?: number
}) {
  return useInfiniteQuery({
    queryKey: ['audit-events-infinite', params],
    queryFn: async ({ pageParam = 1 }) => {
      const client = new AuditAPIClient()
      return client.getEvents({
        tenantIds: params.tenantIds,
        dateRange: params.dateRange,
        filters: params.filters,
        search: params.search,
        page: pageParam,
        pageSize: params.pageSize || 50,
      })
    },
    enabled: params.tenantIds.length > 0,
    getNextPageParam: (lastPage, allPages) => {
      return lastPage.hasMore ? allPages.length + 1 : undefined
    },
  })
}

/**
 * Fetch event details
 */
export function useEventDetails(eventId: string | null) {
  return useQuery({
    queryKey: ['event-details', eventId],
    queryFn: async () => {
      if (!eventId) return null
      const client = new AuditAPIClient()
      return client.getEventDetails(eventId)
    },
    enabled: !!eventId,
  })
}

/**
 * Fetch related events
 */
export function useRelatedEvents(eventId: string | null) {
  return useQuery({
    queryKey: ['related-events', eventId],
    queryFn: async () => {
      if (!eventId) return []
      const client = new AuditAPIClient()
      return client.getRelatedEvents(eventId)
    },
    enabled: !!eventId,
  })
}

/**
 * Prefetch next page
 */
export function usePrefetchNextPage(params: {
  tenantIds: string[]
  dateRange: DateRange
  filters?: AuditFilters
  search?: string
  currentPage: number
  pageSize?: number
  enabled?: boolean
}) {
  const queryClient = useQueryClient()

  useEffect(() => {
    if (!params.enabled) return

    const nextPage = params.currentPage + 1

    queryClient.prefetchQuery({
      queryKey: [
        'audit-events',
        {
          ...params,
          page: nextPage,
        },
      ],
      queryFn: async () => {
        const client = new AuditAPIClient()
        return client.getEvents({
          tenantIds: params.tenantIds,
          dateRange: params.dateRange,
          filters: params.filters,
          search: params.search,
          page: nextPage,
          pageSize: params.pageSize || 50,
        })
      },
    })
  }, [queryClient, params])
}
```

---

## Rendering Logic

### Core JSX Structure

```tsx
import React from 'react'
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from '@/components/ui/select'
import { Badge } from '@/components/ui/badge'
import { Calendar, Search, Download, Bookmark, ChevronDown, ChevronRight } from 'lucide-react'

export function AuditViewer(props: AuditViewerProps) {
  const state = useAuditViewer(props)
  const {
    dateRange,
    filters,
    searchQuery,
    currentPage,
    expandedEventId,
    selectedEvent,
    sortBy,
    sortOrder,
    bookmarks,
    exportDialogOpen,
    tenantIdArray,
    setDateRange,
    setFilters,
    setSearchQuery,
    setCurrentPage,
    setExpandedEventId,
    setSelectedEvent,
    setSortBy,
    setSortOrder,
    setExportDialogOpen,
    applyPreset,
    createBookmark,
  } = state

  // Fetch data
  const eventsQuery = props.enableInfiniteScroll
    ? useInfiniteAuditEvents({
        tenantIds: tenantIdArray,
        dateRange,
        filters,
        search: searchQuery,
        pageSize: props.pageSize,
      })
    : useAuditEvents({
        tenantIds: tenantIdArray,
        dateRange,
        filters,
        search: searchQuery,
        page: currentPage,
        pageSize: props.pageSize,
        sortBy,
        sortOrder,
        autoRefresh: props.autoRefreshInterval,
      })

  const relatedEventsQuery = useRelatedEvents(expandedEventId)

  const events = props.enableInfiniteScroll
    ? (eventsQuery as any).data?.pages.flatMap((p: any) => p.events) || []
    : (eventsQuery as any).data?.events || []

  const totalEvents = (eventsQuery as any).data?.total || 0
  const isLoading = eventsQuery.isLoading

  // Prefetch next page
  usePrefetchNextPage({
    tenantIds: tenantIdArray,
    dateRange,
    filters,
    search: searchQuery,
    currentPage,
    pageSize: props.pageSize,
    enabled: props.prefetchNextPage && !props.enableInfiniteScroll,
  })

  return (
    <div className="audit-viewer w-full p-6 space-y-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <div className="flex items-center gap-3">
          <Search className="h-8 w-8 text-blue-600" />
          <div>
            <h1 className="text-2xl font-bold">Audit Viewer</h1>
            <p className="text-sm text-gray-500">
              View and search audit logs
            </p>
          </div>
        </div>

        <div className="flex items-center gap-3">
          {props.customActions}
          {props.enableExport && (
            <Button variant="outline" onClick={() => setExportDialogOpen(true)}>
              <Download className="h-4 w-4 mr-2" />
              Export
            </Button>
          )}
        </div>
      </div>

      {/* Controls */}
      <div className="space-y-4">
        {/* Date Range & Search */}
        <div className="flex items-center gap-4">
          <Select
            value={props.initialDateRange}
            onValueChange={(value) => {
              const range = calculateDateRange(value as any, props.customDateRange)
              setDateRange(range)
            }}
          >
            <SelectTrigger className="w-48">
              <Calendar className="h-4 w-4 mr-2" />
              <SelectValue placeholder="Select date range" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="1h">Last 1 hour</SelectItem>
              <SelectItem value="24h">Last 24 hours</SelectItem>
              <SelectItem value="7d">Last 7 days</SelectItem>
              <SelectItem value="30d">Last 30 days</SelectItem>
              <SelectItem value="90d">Last 90 days</SelectItem>
              <SelectItem value="custom">Custom range</SelectItem>
            </SelectContent>
          </Select>

          {props.enableFullTextSearch && (
            <Input
              placeholder="Search events..."
              value={searchQuery}
              onChange={(e) => setSearchQuery(e.target.value)}
              className="max-w-md"
            />
          )}

          <Button variant="outline" onClick={() => createBookmark('New Bookmark')}>
            <Bookmark className="h-4 w-4 mr-2" />
            Save Filters
          </Button>
        </div>

        {/* Filters */}
        {props.enableAdvancedFilters && (
          <div className="flex flex-wrap items-center gap-3 p-4 bg-gray-50 rounded-lg">
            <span className="text-sm font-medium">Filters:</span>

            <Select
              value={filters.userIds?.[0] || 'all'}
              onValueChange={(value) => {
                setFilters({
                  userIds: value === 'all' ? undefined : [value],
                })
              }}
            >
              <SelectTrigger className="w-40">
                <SelectValue placeholder="User" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Users</SelectItem>
                {/* Dynamically populated */}
              </SelectContent>
            </Select>

            <Select
              value={filters.actionCategories?.[0] || 'all'}
              onValueChange={(value) => {
                setFilters({
                  actionCategories: value === 'all' ? undefined : [value as any],
                })
              }}
            >
              <SelectTrigger className="w-40">
                <SelectValue placeholder="Action" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Actions</SelectItem>
                <SelectItem value="create">Create</SelectItem>
                <SelectItem value="read">Read</SelectItem>
                <SelectItem value="update">Update</SelectItem>
                <SelectItem value="delete">Delete</SelectItem>
                <SelectItem value="auth">Auth</SelectItem>
                <SelectItem value="admin">Admin</SelectItem>
              </SelectContent>
            </Select>

            <Select
              value={filters.resourceTypes?.[0] || 'all'}
              onValueChange={(value) => {
                setFilters({
                  resourceTypes: value === 'all' ? undefined : [value],
                })
              }}
            >
              <SelectTrigger className="w-40">
                <SelectValue placeholder="Resource" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Resources</SelectItem>
                <SelectItem value="document">Document</SelectItem>
                <SelectItem value="report">Report</SelectItem>
                <SelectItem value="user">User</SelectItem>
                <SelectItem value="role">Role</SelectItem>
              </SelectContent>
            </Select>
          </div>
        )}

        {/* Filter Presets */}
        {props.filterPresets && props.filterPresets.length > 0 && (
          <div className="flex items-center gap-2">
            <span className="text-sm font-medium">Presets:</span>
            {props.filterPresets.map((preset) => (
              <Button
                key={preset.id}
                variant="outline"
                size="sm"
                onClick={() => applyPreset(preset)}
              >
                {preset.name}
              </Button>
            ))}
          </div>
        )}
      </div>

      {/* Timeline */}
      {props.showTimeline && (
        <Card>
          <CardHeader>
            <CardTitle>Timeline ({dateRange.label})</CardTitle>
          </CardHeader>
          <CardContent>
            <AuditTimeline events={events} dateRange={dateRange} />
          </CardContent>
        </Card>
      )}

      {/* Audit Log Table */}
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center justify-between">
            <span>Audit Log ({totalEvents.toLocaleString()} events)</span>
            {props.autoRefreshInterval && (
              <Badge variant="outline">Auto-refresh: {props.autoRefreshInterval}s</Badge>
            )}
          </CardTitle>
        </CardHeader>
        <CardContent>
          {isLoading && (
            <div className="flex items-center justify-center h-64">
              <span className="text-gray-500">Loading events...</span>
            </div>
          )}

          {!isLoading && events.length === 0 && (
            <div className="flex items-center justify-center h-64">
              <span className="text-gray-500">No events found</span>
            </div>
          )}

          {!isLoading && events.length > 0 && (
            <AuditEventTable
              events={events}
              expandedEventId={expandedEventId}
              onEventClick={setSelectedEvent}
              onEventExpand={setExpandedEventId}
              showDiffViewer={props.showDiffViewer}
              density={props.density}
              visibleColumns={props.visibleColumns}
              enableVirtualization={props.enableVirtualization}
            />
          )}

          {/* Pagination */}
          {!props.enableInfiniteScroll && (
            <div className="flex items-center justify-between mt-4">
              <Button
                variant="outline"
                disabled={currentPage === 1}
                onClick={() => setCurrentPage((p) => p - 1)}
              >
                Previous
              </Button>
              <span className="text-sm text-gray-600">
                Page {currentPage} of {Math.ceil(totalEvents / (props.pageSize || 50))}
              </span>
              <Button
                variant="outline"
                disabled={currentPage * (props.pageSize || 50) >= totalEvents}
                onClick={() => setCurrentPage((p) => p + 1)}
              >
                Next
              </Button>
            </div>
          )}

          {/* Infinite Scroll Loader */}
          {props.enableInfiniteScroll && (eventsQuery as any).hasNextPage && (
            <div className="flex justify-center mt-4">
              <Button onClick={() => (eventsQuery as any).fetchNextPage()}>
                Load More
              </Button>
            </div>
          )}
        </CardContent>
      </Card>

      {/* Event Details Panel */}
      {props.showRelatedEvents && selectedEvent && (
        <EventDetailsPanel
          event={selectedEvent}
          relatedEvents={relatedEventsQuery.data || []}
          showDiffViewer={props.showDiffViewer}
          onUserClick={props.onUserClick}
          onResourceClick={props.onResourceClick}
        />
      )}
    </div>
  )
}
```

### Sub-Components

```tsx
/**
 * Audit Event Table
 */
function AuditEventTable(props: {
  events: AuditEvent[]
  expandedEventId: string | null
  onEventClick: (event: AuditEvent) => void
  onEventExpand: (eventId: string | null) => void
  showDiffViewer?: boolean
  density?: 'compact' | 'comfortable' | 'spacious'
  visibleColumns?: string[]
  enableVirtualization?: boolean
}) {
  const { events, expandedEventId, onEventClick, onEventExpand, showDiffViewer } = props

  const rowHeight = props.density === 'compact' ? 40 : props.density === 'spacious' ? 80 : 60

  return (
    <div className="border rounded-lg overflow-hidden">
      <table className="w-full text-sm">
        <thead className="bg-gray-50">
          <tr>
            <th className="px-3 py-2 text-left w-10"></th>
            <th className="px-3 py-2 text-left">Timestamp</th>
            <th className="px-3 py-2 text-left">User</th>
            <th className="px-3 py-2 text-left">Action</th>
            <th className="px-3 py-2 text-left">Resource</th>
            <th className="px-3 py-2 text-left">IP</th>
          </tr>
        </thead>
        <tbody>
          {events.map((event) => (
            <React.Fragment key={event.id}>
              <tr
                className="border-t hover:bg-gray-50 cursor-pointer"
                onClick={() => onEventClick(event)}
              >
                <td className="px-3 py-2">
                  <button
                    onClick={(e) => {
                      e.stopPropagation()
                      onEventExpand(expandedEventId === event.id ? null : event.id)
                    }}
                  >
                    {expandedEventId === event.id ? (
                      <ChevronDown className="h-4 w-4" />
                    ) : (
                      <ChevronRight className="h-4 w-4" />
                    )}
                  </button>
                </td>
                <td className="px-3 py-2">
                  {new Date(event.timestamp).toLocaleString()}
                </td>
                <td className="px-3 py-2 truncate max-w-[150px]">
                  {event.userEmail}
                </td>
                <td className="px-3 py-2">
                  <Badge variant="outline">{event.action}</Badge>
                </td>
                <td className="px-3 py-2 truncate max-w-[150px]">
                  {event.resourceName || event.resourceId}
                </td>
                <td className="px-3 py-2 text-xs text-gray-500">
                  {event.ipAddress}
                </td>
              </tr>

              {/* Expanded Details */}
              {expandedEventId === event.id && (
                <tr className="border-t bg-blue-50">
                  <td colSpan={6} className="px-12 py-4">
                    <EventDetailsInline event={event} showDiffViewer={showDiffViewer} />
                  </td>
                </tr>
              )}
            </React.Fragment>
          ))}
        </tbody>
      </table>
    </div>
  )
}

/**
 * Inline event details (expanded row)
 */
function EventDetailsInline(props: { event: AuditEvent; showDiffViewer?: boolean }) {
  const { event, showDiffViewer } = props

  return (
    <div className="space-y-3 text-sm">
      <div>
        <span className="font-medium">Details:</span> {event.metadata?.description || 'No description'}
      </div>

      {showDiffViewer && event.beforeState && event.afterState && (
        <div>
          <span className="font-medium">Changes:</span>
          <DiffViewer before={event.beforeState} after={event.afterState} />
        </div>
      )}

      {event.errorMessage && (
        <div className="text-red-600">
          <span className="font-medium">Error:</span> {event.errorMessage}
        </div>
      )}
    </div>
  )
}

/**
 * Diff Viewer
 */
function DiffViewer(props: { before: any; after: any }) {
  const { before, after } = props

  const changes = Object.keys({ ...before, ...after }).map((key) => ({
    field: key,
    before: before[key],
    after: after[key],
    changed: before[key] !== after[key],
  }))

  return (
    <div className="mt-2 border rounded overflow-hidden">
      <table className="w-full text-xs">
        <thead className="bg-gray-100">
          <tr>
            <th className="px-2 py-1 text-left">Field</th>
            <th className="px-2 py-1 text-left">Before</th>
            <th className="px-2 py-1 text-left">After</th>
          </tr>
        </thead>
        <tbody>
          {changes.filter((c) => c.changed).map((change) => (
            <tr key={change.field} className="border-t">
              <td className="px-2 py-1 font-medium">{change.field}</td>
              <td className="px-2 py-1 text-red-600 line-through">
                {JSON.stringify(change.before)}
              </td>
              <td className="px-2 py-1 text-green-600">
                {JSON.stringify(change.after)}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}

/**
 * Event Details Panel (full panel, not inline)
 */
function EventDetailsPanel(props: {
  event: AuditEvent
  relatedEvents: AuditEvent[]
  showDiffViewer?: boolean
  onUserClick?: (userId: string) => void
  onResourceClick?: (resourceType: string, resourceId: string) => void
}) {
  const { event, relatedEvents, showDiffViewer, onUserClick, onResourceClick } = props

  return (
    <Card>
      <CardHeader>
        <CardTitle>
          Event Details: {event.action} ({event.resourceName || event.resourceId})
        </CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div className="grid grid-cols-2 gap-4 text-sm">
          <div>
            <span className="font-medium">User:</span>{' '}
            <Button variant="link" onClick={() => onUserClick?.(event.userId)}>
              {event.userEmail}
            </Button>
          </div>
          <div>
            <span className="font-medium">Time:</span>{' '}
            {new Date(event.timestamp).toLocaleString()}
          </div>
          <div>
            <span className="font-medium">IP:</span> {event.ipAddress}
            {event.geoLocation && ` (${event.geoLocation.city}, ${event.geoLocation.country})`}
          </div>
          <div>
            <span className="font-medium">Duration:</span> {event.duration_ms}ms
          </div>
          <div>
            <span className="font-medium">Success:</span>{' '}
            {event.success ? '✓' : '✗'}
          </div>
        </div>

        {showDiffViewer && event.beforeState && event.afterState && (
          <div>
            <h4 className="font-medium mb-2">Changes:</h4>
            <DiffViewer before={event.beforeState} after={event.afterState} />
          </div>
        )}

        {relatedEvents.length > 0 && (
          <div>
            <h4 className="font-medium mb-2">Related Events ({relatedEvents.length}):</h4>
            <ul className="space-y-1 text-sm">
              {relatedEvents.map((re) => (
                <li key={re.id}>
                  • {new Date(re.timestamp).toLocaleString()} - {re.action}
                  {re.id === event.id && ' ← Current'}
                </li>
              ))}
            </ul>
          </div>
        )}
      </CardContent>
    </Card>
  )
}

/**
 * Audit Timeline (simple sparkline)
 */
function AuditTimeline(props: { events: AuditEvent[]; dateRange: DateRange }) {
  const { events, dateRange } = props

  // Group events by hour
  const hourlyBuckets = groupEventsByHour(events, dateRange)

  return (
    <div className="h-24 flex items-end gap-1">
      {hourlyBuckets.map((bucket, i) => {
        const height = (bucket.count / Math.max(...hourlyBuckets.map((b) => b.count))) * 100
        return (
          <div
            key={i}
            className="flex-1 bg-blue-500 rounded-t"
            style={{ height: `${height}%` }}
            title={`${bucket.hour}: ${bucket.count} events`}
          />
        )
      })}
    </div>
  )
}

function groupEventsByHour(events: AuditEvent[], dateRange: DateRange) {
  const hours = Math.ceil((dateRange.end.getTime() - dateRange.start.getTime()) / (60 * 60 * 1000))
  const buckets = Array.from({ length: hours }, (_, i) => ({
    hour: new Date(dateRange.start.getTime() + i * 60 * 60 * 1000).toISOString(),
    count: 0,
  }))

  events.forEach((event) => {
    const eventHour = Math.floor(
      (new Date(event.timestamp).getTime() - dateRange.start.getTime()) / (60 * 60 * 1000)
    )
    if (eventHour >= 0 && eventHour < hours) {
      buckets[eventHour].count++
    }
  })

  return buckets
}
```

---

## Event Handlers

### Export Handling

```typescript
/**
 * Export audit events
 */
async function handleExport(format: 'csv' | 'pdf' | 'json', events: AuditEvent[]) {
  const client = new AuditAPIClient()

  const blob = await client.exportEvents({
    format,
    events,
    fileName: `audit-log-${format}-${Date.now()}`,
  })

  // Trigger download
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `audit-log-${format}-${Date.now()}.${format}`
  a.click()
  URL.revokeObjectURL(url)

  props.onExport?.(format, events)
}
```

---

## Accessibility

### WCAG 2.1 AA Compliance

```typescript
// Keyboard navigation for table
<table role="grid" aria-label="Audit events">
  <tbody>
    {events.map((event, i) => (
      <tr
        key={event.id}
        role="row"
        tabIndex={0}
        aria-rowindex={i + 1}
        onKeyDown={(e) => {
          if (e.key === 'Enter') onEventClick(event)
        }}
      >
        {/* ... */}
      </tr>
    ))}
  </tbody>
</table>
```

---

## Performance

### Optimization Strategies

```typescript
// 1. Virtualization for large tables (handled by TanStack Virtual)
// 2. Debounced search (handled in state management)
// 3. Prefetch next page (handled in data fetching)
// 4. Memoized filter presets

const memoizedPresets = useMemo(
  () => props.filterPresets || DEFAULT_PRESETS,
  [props.filterPresets]
)
```

---

## Multi-Domain Examples

### 1. Healthcare (HIPAA Audit)

```tsx
<AuditViewer
  tenantId="healthcare-corp"
  currentUser={currentUser}
  filterPresets={[
    {
      id: 'hipaa-phi-access',
      name: 'HIPAA PHI Access',
      filters: { complianceTags: ['HIPAA'], actions: ['document.read', 'document.export'] },
    },
  ]}
/>
```

### 2. Financial (SOC 2)

```tsx
<AuditViewer
  tenantId="bank-xyz"
  currentUser={currentUser}
  filterPresets={[
    {
      id: 'soc2-admin-changes',
      name: 'SOC 2 Admin Changes',
      filters: { actionCategories: ['admin'], complianceTags: ['SOC2'] },
    },
  ]}
/>
```

---

## Integration

### Integration with OL

```typescript
import { fetchAuditLogs } from '@/lib/ol'

async function fetchAuditEventsFromOL(params: any): Promise<AuditEvent[]> {
  const logs = await fetchAuditLogs(params)
  return logs.map((log) => ({
    id: log.id,
    timestamp: new Date(log.timestamp),
    tenantId: log.tenant_id,
    userId: log.user_id,
    userEmail: log.user_email,
    userName: log.user_name,
    action: log.action,
    actionCategory: log.action_category,
    resourceType: log.resource_type,
    resourceId: log.resource_id,
    ipAddress: log.ip_address,
    userAgent: log.user_agent,
    success: log.success,
    metadata: log.metadata,
  }))
}
```

---

## Testing Strategy

```typescript
test('renders audit log table', () => {
  render(<AuditViewer tenantId="test" currentUser={mockUser} />)
  expect(screen.getByText(/Audit Log/i)).toBeInTheDocument()
})

test('filters events by action', async () => {
  render(<AuditViewer tenantId="test" currentUser={mockUser} />)
  const actionFilter = screen.getByLabelText(/Action/i)
  fireEvent.change(actionFilter, { target: { value: 'create' } })
  await waitFor(() => {
    expect(screen.queryByText(/delete/i)).not.toBeInTheDocument()
  })
})
```

---

## References

- [SecurityDashboard Component](/docs/primitives/il/SecurityDashboard.md)
- [RoleManager Component](/docs/primitives/il/RoleManager.md)
- [TanStack Table](https://tanstack.com/table)
- [SOC 2 Compliance](https://www.aicpa.org/)
