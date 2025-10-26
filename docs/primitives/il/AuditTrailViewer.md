# AuditTrailViewer (IL Component)

## Definition

**AuditTrailViewer** is a timeline visualization component that displays the complete history of field changes on entities. It shows all corrections, overrides, reverts, and deletions with full context about who made each change, when, and why. The component supports multiple view modes (timeline, table, compact), filtering by user/action/date, and export functionality for compliance and analysis.

**Problem it solves:**
- Users cannot see who changed a field value or when
- No visibility into correction history for compliance/audit requirements
- Cannot track which user made the most corrections or errors
- No way to revert to a previous value after multiple changes
- Missing context on why a field was changed (no reason/notes)
- Cannot export audit trail for regulatory reporting
- Large audit histories (100+ changes) cause UI performance issues
- No filtering by user, action type, or date range

**Solution:**
- Timeline view showing chronological history with visual indicators
- Table view for sorting/filtering large datasets
- Compact view for embedding in small spaces (sidebars, tooltips)
- Color-coded action types (extracted=blue, override=orange, revert=purple, delete=red)
- User avatars and relative timestamps ("2 hours ago")
- Real-time updates when new changes occur
- Export to JSON or CSV for compliance reporting
- Pagination for large histories (100+ entries)
- One-click revert to any previous value
- Empty state when no history exists

---

## Interface Contract

```typescript
interface AuditTrailViewerProps {
  // Data source
  entityId: string;                  // Entity whose history to display
  entityType: string;                // Entity type (transaction, patient, case, etc.)
  fieldName?: string;                // If provided, show only this field's history

  // Display options
  variant: 'timeline' | 'table' | 'compact';
  showIcons?: boolean;               // Show action icons (default: true)
  showTimestamps?: boolean;          // Show timestamps (default: true)
  showUsers?: boolean;               // Show user avatars/names (default: true)
  showReasons?: boolean;             // Show reason for change (default: true)
  maxEntries?: number;               // Pagination limit (default: 50)

  // Filtering
  filterByUser?: string;             // Show only changes by this user
  filterByAction?: AuditAction[];    // Filter by action types
  dateRange?: {                      // Filter by date range
    start: Date;
    end: Date;
  };

  // Actions
  onExport?: (format: 'json' | 'csv') => void;
  onRevert?: (auditId: string) => Promise<void>;
  onViewDetails?: (auditId: string) => void;

  // Real-time updates
  enableRealtime?: boolean;          // Poll for new entries (default: false)
  realtimeInterval?: number;         // Poll interval in ms (default: 5000)

  // UI customization
  theme?: 'light' | 'dark';
  compact?: boolean;                 // Reduced padding/spacing
  height?: string | number;          // Fixed height with scroll (e.g., "400px")
}

interface AuditEntry {
  audit_id: string;                  // Unique identifier
  entity_id: string;                 // Entity being changed
  entity_type: string;               // Entity type
  field_name: string;                // Field being changed

  // Change details
  action: AuditAction;               // Type of change
  old_value: any;                    // Value before change
  new_value: any;                    // Value after change

  // Context
  changed_by: string;                // User ID who made change
  changed_by_name?: string;          // Display name (if available)
  changed_at: string;                // ISO timestamp
  reason?: string;                   // Optional explanation

  // Metadata
  ip_address?: string;               // User IP (for security audits)
  user_agent?: string;               // Browser/client info
  confidence_score?: number;         // For AI-extracted values (0-1)

  // System
  created_at: string;                // Record creation timestamp
}

type AuditAction =
  | 'extracted'                      // Initial value from extraction
  | 'override'                       // User manually corrected
  | 'revert'                         // Reverted to previous value
  | 'delete'                         // Field value deleted
  | 'ai_correction'                  // AI-suggested correction applied
  | 'bulk_update';                   // Part of bulk operation

interface AuditTrailViewerState {
  entries: AuditEntry[];
  loading: boolean;
  error: string | null;
  currentPage: number;
  totalPages: number;
  filters: AuditFilters;
}

interface AuditFilters {
  user?: string;
  actions?: AuditAction[];
  dateRange?: { start: Date; end: Date };
  searchText?: string;
}
```

---

## Component Structure

```tsx
import React, { useState, useEffect, useMemo, useCallback } from 'react';
import { format, formatDistanceToNow } from 'date-fns';
import {
  Clock,
  User,
  Edit,
  Undo,
  Trash,
  Download,
  Filter,
  ChevronDown,
  ChevronUp,
  AlertCircle,
  CheckCircle
} from 'lucide-react';
import {
  ScrollArea,
  Badge,
  Avatar,
  Button,
  Select,
  Card,
  Table,
  Skeleton
} from '@/components/ui';

export const AuditTrailViewer: React.FC<AuditTrailViewerProps> = ({
  entityId,
  entityType,
  fieldName,
  variant = 'timeline',
  showIcons = true,
  showTimestamps = true,
  showUsers = true,
  showReasons = true,
  maxEntries = 50,
  filterByUser,
  filterByAction,
  dateRange,
  onExport,
  onRevert,
  onViewDetails,
  enableRealtime = false,
  realtimeInterval = 5000,
  theme = 'light',
  compact = false,
  height
}) => {
  // State
  const [entries, setEntries] = useState<AuditEntry[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [currentPage, setCurrentPage] = useState(1);
  const [totalEntries, setTotalEntries] = useState(0);
  const [expandedEntries, setExpandedEntries] = useState<Set<string>>(new Set());

  // Filters
  const [filters, setFilters] = useState<AuditFilters>({
    user: filterByUser,
    actions: filterByAction,
    dateRange: dateRange,
    searchText: ''
  });

  // Fetch audit trail
  const fetchAuditTrail = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const params = new URLSearchParams({
        entity_id: entityId,
        entity_type: entityType,
        page: currentPage.toString(),
        limit: maxEntries.toString()
      });

      if (fieldName) params.append('field_name', fieldName);
      if (filters.user) params.append('user_id', filters.user);
      if (filters.actions?.length) {
        filters.actions.forEach(action => params.append('action', action));
      }
      if (filters.dateRange) {
        params.append('start_date', filters.dateRange.start.toISOString());
        params.append('end_date', filters.dateRange.end.toISOString());
      }

      const response = await fetch(`/api/audit-trail?${params}`);
      if (!response.ok) throw new Error('Failed to fetch audit trail');

      const data = await response.json();
      setEntries(data.entries);
      setTotalEntries(data.total);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  }, [entityId, entityType, fieldName, currentPage, maxEntries, filters]);

  // Initial fetch
  useEffect(() => {
    fetchAuditTrail();
  }, [fetchAuditTrail]);

  // Real-time polling
  useEffect(() => {
    if (!enableRealtime) return;

    const interval = setInterval(() => {
      fetchAuditTrail();
    }, realtimeInterval);

    return () => clearInterval(interval);
  }, [enableRealtime, realtimeInterval, fetchAuditTrail]);

  // Calculate total pages
  const totalPages = Math.ceil(totalEntries / maxEntries);

  // Filter entries locally (for search text)
  const filteredEntries = useMemo(() => {
    if (!filters.searchText) return entries;

    const searchLower = filters.searchText.toLowerCase();
    return entries.filter(entry =>
      entry.field_name.toLowerCase().includes(searchLower) ||
      entry.changed_by_name?.toLowerCase().includes(searchLower) ||
      entry.reason?.toLowerCase().includes(searchLower) ||
      String(entry.old_value).toLowerCase().includes(searchLower) ||
      String(entry.new_value).toLowerCase().includes(searchLower)
    );
  }, [entries, filters.searchText]);

  // Action styling
  const getActionStyle = (action: AuditAction) => {
    switch (action) {
      case 'extracted':
        return { color: 'blue', icon: Clock, label: 'Extracted' };
      case 'override':
        return { color: 'orange', icon: Edit, label: 'Overridden' };
      case 'revert':
        return { color: 'purple', icon: Undo, label: 'Reverted' };
      case 'delete':
        return { color: 'red', icon: Trash, label: 'Deleted' };
      case 'ai_correction':
        return { color: 'green', icon: CheckCircle, label: 'AI Corrected' };
      case 'bulk_update':
        return { color: 'gray', icon: AlertCircle, label: 'Bulk Update' };
      default:
        return { color: 'gray', icon: Clock, label: action };
    }
  };

  // Export handler
  const handleExport = (format: 'json' | 'csv') => {
    if (!onExport) return;

    if (format === 'json') {
      const blob = new Blob([JSON.stringify(entries, null, 2)], {
        type: 'application/json'
      });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `audit-trail-${entityId}-${Date.now()}.json`;
      a.click();
      URL.revokeObjectURL(url);
    } else {
      // CSV export
      const headers = ['Timestamp', 'Field', 'Action', 'Old Value', 'New Value', 'User', 'Reason'];
      const rows = entries.map(entry => [
        entry.changed_at,
        entry.field_name,
        entry.action,
        entry.old_value,
        entry.new_value,
        entry.changed_by_name || entry.changed_by,
        entry.reason || ''
      ]);

      const csv = [
        headers.join(','),
        ...rows.map(row => row.map(cell => `"${cell}"`).join(','))
      ].join('\n');

      const blob = new Blob([csv], { type: 'text/csv' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `audit-trail-${entityId}-${Date.now()}.csv`;
      a.click();
      URL.revokeObjectURL(url);
    }

    onExport(format);
  };

  // Toggle entry expansion
  const toggleExpanded = (auditId: string) => {
    setExpandedEntries(prev => {
      const next = new Set(prev);
      if (next.has(auditId)) {
        next.delete(auditId);
      } else {
        next.add(auditId);
      }
      return next;
    });
  };

  // Render loading state
  if (loading && entries.length === 0) {
    return (
      <div className={`audit-trail-viewer theme-${theme} ${compact ? 'compact' : ''}`}>
        <div className="loading-state">
          <Skeleton className="h-12 mb-2" />
          <Skeleton className="h-12 mb-2" />
          <Skeleton className="h-12 mb-2" />
        </div>
      </div>
    );
  }

  // Render error state
  if (error) {
    return (
      <div className={`audit-trail-viewer theme-${theme} ${compact ? 'compact' : ''}`}>
        <div className="error-state">
          <AlertCircle className="error-icon" />
          <p>Failed to load audit trail</p>
          <p className="error-message">{error}</p>
          <Button onClick={fetchAuditTrail}>Retry</Button>
        </div>
      </div>
    );
  }

  // Render empty state
  if (filteredEntries.length === 0) {
    return (
      <div className={`audit-trail-viewer theme-${theme} ${compact ? 'compact' : ''}`}>
        <div className="empty-state">
          <Clock className="empty-icon" />
          <p>No audit history</p>
          <p className="empty-message">
            {filters.searchText || filters.user || filters.actions?.length
              ? 'No entries match your filters'
              : 'No changes have been made to this entity'}
          </p>
        </div>
      </div>
    );
  }

  // Render timeline view
  if (variant === 'timeline') {
    return (
      <div className={`audit-trail-viewer timeline-view theme-${theme} ${compact ? 'compact' : ''}`}>
        {/* Header */}
        <div className="viewer-header">
          <div className="header-title">
            <Clock className="title-icon" />
            <h3>
              Audit Trail
              {fieldName && <span className="field-badge">{fieldName}</span>}
            </h3>
            <Badge variant="outline">{totalEntries} entries</Badge>
          </div>

          <div className="header-actions">
            {/* Search */}
            <input
              type="text"
              className="search-input"
              placeholder="Search history..."
              value={filters.searchText}
              onChange={(e) => setFilters(prev => ({ ...prev, searchText: e.target.value }))}
            />

            {/* Export */}
            {onExport && (
              <div className="export-dropdown">
                <Button variant="outline" size="sm">
                  <Download className="w-4 h-4 mr-2" />
                  Export
                </Button>
                <div className="export-menu">
                  <button onClick={() => handleExport('json')}>Export JSON</button>
                  <button onClick={() => handleExport('csv')}>Export CSV</button>
                </div>
              </div>
            )}
          </div>
        </div>

        {/* Timeline */}
        <ScrollArea className="timeline-content" style={{ height: height || 'auto' }}>
          <div className="timeline-line"></div>

          {filteredEntries.map((entry, index) => {
            const { color, icon: Icon, label } = getActionStyle(entry.action);
            const isExpanded = expandedEntries.has(entry.audit_id);

            return (
              <div key={entry.audit_id} className={`timeline-entry ${color}`}>
                {/* Icon */}
                {showIcons && (
                  <div className={`entry-icon bg-${color}`}>
                    <Icon className="w-4 h-4" />
                  </div>
                )}

                {/* Content */}
                <Card className="entry-card">
                  <div className="entry-header">
                    <div className="entry-meta">
                      {showUsers && (
                        <div className="entry-user">
                          <Avatar
                            name={entry.changed_by_name || entry.changed_by}
                            size="sm"
                          />
                          <span className="user-name">
                            {entry.changed_by_name || entry.changed_by}
                          </span>
                        </div>
                      )}

                      <Badge variant={color}>{label}</Badge>

                      {showTimestamps && (
                        <span className="entry-timestamp" title={format(new Date(entry.changed_at), 'PPpp')}>
                          {formatDistanceToNow(new Date(entry.changed_at), { addSuffix: true })}
                        </span>
                      )}
                    </div>

                    {/* Expand toggle */}
                    <button
                      className="expand-button"
                      onClick={() => toggleExpanded(entry.audit_id)}
                      aria-label={isExpanded ? 'Collapse' : 'Expand'}
                    >
                      {isExpanded ? <ChevronUp className="w-4 h-4" /> : <ChevronDown className="w-4 h-4" />}
                    </button>
                  </div>

                  {/* Summary */}
                  <div className="entry-summary">
                    <span className="field-name">{entry.field_name}</span>
                    <span className="change-arrow">→</span>
                    <span className="new-value">{formatValue(entry.new_value)}</span>
                  </div>

                  {/* Expanded details */}
                  {isExpanded && (
                    <div className="entry-details">
                      <div className="detail-row">
                        <span className="detail-label">Old Value:</span>
                        <span className="detail-value old">{formatValue(entry.old_value)}</span>
                      </div>

                      <div className="detail-row">
                        <span className="detail-label">New Value:</span>
                        <span className="detail-value new">{formatValue(entry.new_value)}</span>
                      </div>

                      {showReasons && entry.reason && (
                        <div className="detail-row">
                          <span className="detail-label">Reason:</span>
                          <span className="detail-value reason">{entry.reason}</span>
                        </div>
                      )}

                      {entry.confidence_score !== undefined && (
                        <div className="detail-row">
                          <span className="detail-label">Confidence:</span>
                          <span className="detail-value">
                            {Math.round(entry.confidence_score * 100)}%
                          </span>
                        </div>
                      )}

                      {/* Actions */}
                      <div className="entry-actions">
                        {onRevert && entry.action !== 'extracted' && (
                          <Button
                            variant="outline"
                            size="sm"
                            onClick={() => onRevert(entry.audit_id)}
                          >
                            <Undo className="w-3 h-3 mr-1" />
                            Revert to this value
                          </Button>
                        )}

                        {onViewDetails && (
                          <Button
                            variant="ghost"
                            size="sm"
                            onClick={() => onViewDetails(entry.audit_id)}
                          >
                            View Details
                          </Button>
                        )}
                      </div>
                    </div>
                  )}
                </Card>
              </div>
            );
          })}
        </ScrollArea>

        {/* Pagination */}
        {totalPages > 1 && (
          <div className="pagination">
            <Button
              variant="outline"
              size="sm"
              disabled={currentPage === 1}
              onClick={() => setCurrentPage(p => p - 1)}
            >
              Previous
            </Button>

            <span className="page-info">
              Page {currentPage} of {totalPages}
            </span>

            <Button
              variant="outline"
              size="sm"
              disabled={currentPage === totalPages}
              onClick={() => setCurrentPage(p => p + 1)}
            >
              Next
            </Button>
          </div>
        )}
      </div>
    );
  }

  // Render table view
  if (variant === 'table') {
    return (
      <div className={`audit-trail-viewer table-view theme-${theme} ${compact ? 'compact' : ''}`}>
        {/* Header (same as timeline) */}
        <div className="viewer-header">
          <div className="header-title">
            <Clock className="title-icon" />
            <h3>Audit Trail</h3>
            <Badge variant="outline">{totalEntries} entries</Badge>
          </div>

          <div className="header-actions">
            <input
              type="text"
              className="search-input"
              placeholder="Search..."
              value={filters.searchText}
              onChange={(e) => setFilters(prev => ({ ...prev, searchText: e.target.value }))}
            />

            {onExport && (
              <Button variant="outline" size="sm" onClick={() => handleExport('csv')}>
                <Download className="w-4 h-4 mr-2" />
                Export CSV
              </Button>
            )}
          </div>
        </div>

        {/* Table */}
        <ScrollArea style={{ height: height || 'auto' }}>
          <Table>
            <thead>
              <tr>
                <th>Timestamp</th>
                {!fieldName && <th>Field</th>}
                <th>Action</th>
                <th>Old Value</th>
                <th>New Value</th>
                {showUsers && <th>User</th>}
                {showReasons && <th>Reason</th>}
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              {filteredEntries.map(entry => {
                const { color, label } = getActionStyle(entry.action);

                return (
                  <tr key={entry.audit_id}>
                    <td className="timestamp-cell">
                      <span title={format(new Date(entry.changed_at), 'PPpp')}>
                        {format(new Date(entry.changed_at), 'PP p')}
                      </span>
                    </td>

                    {!fieldName && (
                      <td className="field-cell">{entry.field_name}</td>
                    )}

                    <td className="action-cell">
                      <Badge variant={color}>{label}</Badge>
                    </td>

                    <td className="value-cell old">{formatValue(entry.old_value)}</td>
                    <td className="value-cell new">{formatValue(entry.new_value)}</td>

                    {showUsers && (
                      <td className="user-cell">
                        <div className="user-info">
                          <Avatar
                            name={entry.changed_by_name || entry.changed_by}
                            size="xs"
                          />
                          <span>{entry.changed_by_name || entry.changed_by}</span>
                        </div>
                      </td>
                    )}

                    {showReasons && (
                      <td className="reason-cell" title={entry.reason}>
                        {entry.reason ? truncate(entry.reason, 50) : '—'}
                      </td>
                    )}

                    <td className="actions-cell">
                      {onRevert && entry.action !== 'extracted' && (
                        <Button
                          variant="ghost"
                          size="xs"
                          onClick={() => onRevert(entry.audit_id)}
                        >
                          <Undo className="w-3 h-3" />
                        </Button>
                      )}
                    </td>
                  </tr>
                );
              })}
            </tbody>
          </Table>
        </ScrollArea>

        {/* Pagination */}
        {totalPages > 1 && (
          <div className="pagination">
            <Button
              variant="outline"
              size="sm"
              disabled={currentPage === 1}
              onClick={() => setCurrentPage(p => p - 1)}
            >
              Previous
            </Button>

            <span className="page-info">
              Page {currentPage} of {totalPages}
            </span>

            <Button
              variant="outline"
              size="sm"
              disabled={currentPage === totalPages}
              onClick={() => setCurrentPage(p => p + 1)}
            >
              Next
            </Button>
          </div>
        )}
      </div>
    );
  }

  // Render compact view
  return (
    <div className={`audit-trail-viewer compact-view theme-${theme}`}>
      <div className="compact-header">
        <span className="compact-title">History ({filteredEntries.length})</span>
        {onExport && (
          <button className="export-icon" onClick={() => handleExport('json')}>
            <Download className="w-3 h-3" />
          </button>
        )}
      </div>

      <ScrollArea className="compact-content" style={{ height: height || '200px' }}>
        {filteredEntries.map(entry => {
          const { color, icon: Icon } = getActionStyle(entry.action);

          return (
            <div key={entry.audit_id} className={`compact-entry ${color}`}>
              <Icon className="compact-icon" />
              <div className="compact-info">
                <span className="compact-change">
                  {entry.field_name}: {formatValue(entry.new_value)}
                </span>
                <span className="compact-meta">
                  {entry.changed_by_name || entry.changed_by} ·{' '}
                  {formatDistanceToNow(new Date(entry.changed_at), { addSuffix: true })}
                </span>
              </div>
            </div>
          );
        })}
      </ScrollArea>
    </div>
  );
};

// Helper functions
function formatValue(value: any): string {
  if (value === null || value === undefined) return '—';
  if (typeof value === 'object') return JSON.stringify(value);
  return String(value);
}

function truncate(text: string, maxLength: number): string {
  if (text.length <= maxLength) return text;
  return text.substring(0, maxLength - 3) + '...';
}
```

---

## UI Wireframes

### Timeline View (Default)

```
┌─────────────────────────────────────────────────────────────────┐
│ Audit Trail                                [Search...] [Export▾] │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ●─────┐  Jane Doe  [Overridden]  2 hours ago              [▾]  │
│  │     │  merchant → Amazon.com                                  │
│  │     └──────────────────────────────────────────────────────  │
│  │                                                                │
│  ●─────┐  AI System  [AI Corrected]  1 day ago             [▾]  │
│  │     │  category → Shopping                                    │
│  │     └──────────────────────────────────────────────────────  │
│  │                                                                │
│  ●─────┐  John Smith  [Overridden]  2 days ago             [▸]  │
│  │     │  amount → $45.00                                        │
│  │     │  ┌─────────────────────────────────────────────┐       │
│  │     │  │ Old Value: $42.50                           │       │
│  │     │  │ New Value: $45.00                           │       │
│  │     │  │ Reason: Corrected from receipt              │       │
│  │     │  │                                              │       │
│  │     │  │ [Revert to this value] [View Details]       │       │
│  │     │  └─────────────────────────────────────────────┘       │
│  │     └──────────────────────────────────────────────────────  │
│  │                                                                │
│  ●─────┐  System  [Extracted]  5 days ago                  [▾]  │
│        │  merchant → AMZN MKTP US*1234                           │
│        └──────────────────────────────────────────────────────  │
│                                                                   │
│                    [← Previous]  Page 1 of 3  [Next →]           │
└─────────────────────────────────────────────────────────────────┘
```

### Table View

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Audit Trail                                         [Search...]  [Export CSV]        │
├────────────┬────────┬────────────┬──────────┬──────────┬──────────┬────────┬────────┤
│ Timestamp  │ Field  │ Action     │ Old      │ New      │ User     │ Reason │ Actions│
├────────────┼────────┼────────────┼──────────┼──────────┼──────────┼────────┼────────┤
│ Oct 24     │merchant│[Overridden]│ AMZN     │Amazon.com│Jane Doe  │Normali │  [↶]   │
│ 2:30 PM    │        │            │          │          │          │...     │        │
├────────────┼────────┼────────────┼──────────┼──────────┼──────────┼────────┼────────┤
│ Oct 23     │category│[AI Correct]│Uncategor │Shopping  │AI System │Auto-de │        │
│ 10:15 AM   │        │            │ized      │          │          │...     │        │
├────────────┼────────┼────────────┼──────────┼──────────┼──────────┼────────┼────────┤
│ Oct 22     │amount  │[Overridden]│ $42.50   │ $45.00   │John Smith│Correct │  [↶]   │
│ 9:00 AM    │        │            │          │          │          │...     │        │
├────────────┼────────┼────────────┼──────────┼──────────┼──────────┼────────┼────────┤
│ Oct 20     │merchant│[Extracted] │ null     │AMZN MKTP │System    │—       │        │
│ 8:00 AM    │        │            │          │US*1234   │          │        │        │
└────────────┴────────┴────────────┴──────────┴──────────┴──────────┴────────┴────────┘
```

### Compact View

```
┌──────────────────────────────────┐
│ History (4)              [↓]     │
├──────────────────────────────────┤
│ ✎  merchant: Amazon.com          │
│    Jane Doe · 2 hours ago        │
├──────────────────────────────────┤
│ ✓  category: Shopping            │
│    AI System · 1 day ago         │
├──────────────────────────────────┤
│ ✎  amount: $45.00                │
│    John Smith · 2 days ago       │
├──────────────────────────────────┤
│ ⏱  merchant: AMZN MKTP US*1234   │
│    System · 5 days ago           │
└──────────────────────────────────┘
```

---

## Multi-Domain Examples

### Example 1: Finance - Transaction Corrections

**Scenario:** User corrects merchant name and category on a bank transaction. View complete history.

```typescript
import { AuditTrailViewer } from '@/components/il';

function TransactionDetailPage({ transactionId }: { transactionId: string }) {
  const handleRevert = async (auditId: string) => {
    const confirmed = confirm('Revert to this value?');
    if (!confirmed) return;

    await fetch(`/api/audit/${auditId}/revert`, { method: 'POST' });
    // Refresh data
  };

  const handleExport = (format: 'json' | 'csv') => {
    console.log(`Exporting audit trail as ${format}`);
  };

  return (
    <div className="transaction-detail">
      <h2>Transaction Details</h2>

      {/* Transaction info */}
      <Card>
        <div className="transaction-fields">
          <Field label="Merchant" value="Amazon.com" />
          <Field label="Amount" value="$45.00" />
          <Field label="Category" value="Shopping" />
        </div>
      </Card>

      {/* Audit trail */}
      <AuditTrailViewer
        entityId={transactionId}
        entityType="transaction"
        variant="timeline"
        showIcons={true}
        showTimestamps={true}
        showUsers={true}
        showReasons={true}
        maxEntries={50}
        onRevert={handleRevert}
        onExport={handleExport}
        enableRealtime={true}
        height="500px"
      />
    </div>
  );
}
```

**Audit Entries:**

```json
[
  {
    "audit_id": "aud_001",
    "entity_id": "txn_abc123",
    "entity_type": "transaction",
    "field_name": "merchant",
    "action": "override",
    "old_value": "AMZN MKTP US*1234",
    "new_value": "Amazon.com",
    "changed_by": "user_jane_doe",
    "changed_by_name": "Jane Doe",
    "changed_at": "2025-10-24T14:30:00Z",
    "reason": "Normalized merchant name for reporting",
    "created_at": "2025-10-24T14:30:00Z"
  },
  {
    "audit_id": "aud_002",
    "entity_id": "txn_abc123",
    "entity_type": "transaction",
    "field_name": "category",
    "action": "ai_correction",
    "old_value": "Uncategorized",
    "new_value": "Shopping",
    "changed_by": "system_ai",
    "changed_by_name": "AI System",
    "changed_at": "2025-10-23T10:15:00Z",
    "confidence_score": 0.92,
    "created_at": "2025-10-23T10:15:00Z"
  },
  {
    "audit_id": "aud_003",
    "entity_id": "txn_abc123",
    "entity_type": "transaction",
    "field_name": "merchant",
    "action": "extracted",
    "old_value": null,
    "new_value": "AMZN MKTP US*1234",
    "changed_by": "system_extraction",
    "changed_by_name": "System",
    "changed_at": "2025-10-20T08:00:00Z",
    "created_at": "2025-10-20T08:00:00Z"
  }
]
```

---

### Example 2: Healthcare - Patient Diagnosis Code Changes

**Scenario:** Medical coder reviews diagnosis code correction history for HIPAA compliance audit.

```typescript
function PatientEncounterAudit({ encounterId }: { encounterId: string }) {
  return (
    <div className="encounter-audit">
      <h2>Diagnosis Code Audit Trail</h2>

      <AuditTrailViewer
        entityId={encounterId}
        entityType="encounter"
        fieldName="diagnosis_code"  // Show only diagnosis_code changes
        variant="table"
        showIcons={true}
        showTimestamps={true}
        showUsers={true}
        showReasons={true}
        maxEntries={100}
        onExport={(format) => {
          // Export for HIPAA audit
          console.log(`Exporting HIPAA audit trail as ${format}`);
        }}
      />
    </div>
  );
}
```

**Use Case:**
- **Compliance Officer** reviews all diagnosis code changes
- **Medical Coder** checks who changed diagnosis from R07.9 to I20.9
- **Auditor** exports CSV for regulatory reporting

---

### Example 3: Legal - Case Number Corrections

**Scenario:** Legal assistant tracks all corrections made to case numbers from OCR scanning.

```typescript
function CaseAuditLog({ caseId }: { caseId: string }) {
  const [userFilter, setUserFilter] = useState<string | undefined>();

  return (
    <div className="case-audit">
      <h2>Case Number Correction History</h2>

      {/* User filter */}
      <Select
        value={userFilter}
        onChange={(e) => setUserFilter(e.target.value || undefined)}
      >
        <option value="">All Users</option>
        <option value="assistant_sarah">Sarah Lee</option>
        <option value="assistant_mike">Mike Chen</option>
      </Select>

      <AuditTrailViewer
        entityId={caseId}
        entityType="legal_case"
        fieldName="case_number"
        variant="timeline"
        filterByUser={userFilter}
        filterByAction={['override', 'revert']}
        showIcons={true}
        showTimestamps={true}
        showUsers={true}
        showReasons={true}
        onViewDetails={(auditId) => {
          // Show detailed modal with IP address, user agent, etc.
          console.log(`Viewing details for ${auditId}`);
        }}
      />
    </div>
  );
}
```

**Entries:**

```json
[
  {
    "audit_id": "aud_legal_001",
    "entity_id": "case_001",
    "entity_type": "legal_case",
    "field_name": "case_number",
    "action": "override",
    "old_value": "2024-CV-12345",
    "new_value": "2024-CV-01234",
    "changed_by": "assistant_sarah",
    "changed_by_name": "Sarah Lee",
    "changed_at": "2025-10-24T16:20:00Z",
    "reason": "OCR misread case number from filed document",
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "created_at": "2025-10-24T16:20:00Z"
  }
]
```

---

### Example 4: Research (RSRCH - Utilitario) - Founder Entity Name Corrections

**Scenario:** RSRCH analyst reviews founder entity name correction history.

```typescript
function FounderFactAuditPanel({ factId }: { factId: string }) {
  return (
    <div className="fact-audit">
      <h3>Entity Name History</h3>

      <AuditTrailViewer
        entityId={factId}
        entityType="fact"
        fieldName="subject_entity"
        variant="compact"
        showIcons={true}
        showTimestamps={true}
        showUsers={false}  // Hide users in compact view
        showReasons={false}
        height="300px"
      />
    </div>
  );
}
```

**Entries:**

```json
[
  {
    "audit_id": "aud_fact_001",
    "entity_id": "fact_sama_helion_001",
    "entity_type": "fact",
    "field_name": "subject_entity",
    "action": "override",
    "old_value": "@sama",
    "new_value": "Sam Altman",
    "changed_by": "rsrch_analyst_alice",
    "changed_by_name": "Alice Wong",
    "changed_at": "2025-10-23T09:00:00Z",
    "reason": "Normalized Twitter handle to canonical name",
    "created_at": "2025-10-23T09:00:00Z"
  },
  {
    "audit_id": "aud_fact_002",
    "entity_id": "fact_sama_helion_001",
    "entity_type": "fact",
    "field_name": "subject_entity",
    "action": "extracted",
    "old_value": null,
    "new_value": "J. Smith, et al.",
    "changed_by": "system_pdf_extract",
    "changed_by_name": "PDF Extractor",
    "changed_at": "2025-10-20T12:00:00Z",
    "created_at": "2025-10-20T12:00:00Z"
  }
]
```

---

### Example 5: E-commerce - Product Price Change History

**Scenario:** Catalog manager audits all price changes on a product for pricing strategy analysis.

```typescript
function ProductPriceHistory({ productId }: { productId: string }) {
  const lastMonth = useMemo(() => ({
    start: subMonths(new Date(), 1),
    end: new Date()
  }), []);

  return (
    <div className="product-price-audit">
      <h2>Price Change History</h2>

      <AuditTrailViewer
        entityId={productId}
        entityType="product"
        fieldName="price"
        variant="table"
        dateRange={lastMonth}
        filterByAction={['override', 'bulk_update']}
        showIcons={true}
        showTimestamps={true}
        showUsers={true}
        showReasons={true}
        maxEntries={100}
        onExport={(format) => {
          // Export for pricing analysis
          console.log(`Exporting price history as ${format}`);
        }}
      />
    </div>
  );
}
```

**Entries:**

```json
[
  {
    "audit_id": "aud_prod_001",
    "entity_id": "prod_abc123",
    "entity_type": "product",
    "field_name": "price",
    "action": "bulk_update",
    "old_value": 29.99,
    "new_value": 24.99,
    "changed_by": "manager_tom",
    "changed_by_name": "Tom Chen",
    "changed_at": "2025-10-24T00:00:00Z",
    "reason": "Black Friday sale - 20% off",
    "created_at": "2025-10-24T00:00:00Z"
  },
  {
    "audit_id": "aud_prod_002",
    "entity_id": "prod_abc123",
    "entity_type": "product",
    "field_name": "price",
    "action": "override",
    "old_value": 34.99,
    "new_value": 29.99,
    "changed_by": "manager_tom",
    "changed_by_name": "Tom Chen",
    "changed_at": "2025-10-15T10:00:00Z",
    "reason": "Price match competitor",
    "created_at": "2025-10-15T10:00:00Z"
  }
]
```

---

### Example 6: SaaS - Subscription MRR Corrections

**Scenario:** Finance team audits MRR corrections for revenue reporting accuracy.

```typescript
function SubscriptionMRRAudit({ subscriptionId }: { subscriptionId: string }) {
  const [actionFilter, setActionFilter] = useState<AuditAction[]>(['override']);

  return (
    <div className="mrr-audit">
      <h2>MRR Correction Audit</h2>

      {/* Action filter */}
      <div className="filter-actions">
        <label>
          <input
            type="checkbox"
            checked={actionFilter.includes('override')}
            onChange={(e) => {
              if (e.target.checked) {
                setActionFilter([...actionFilter, 'override']);
              } else {
                setActionFilter(actionFilter.filter(a => a !== 'override'));
              }
            }}
          />
          Overrides
        </label>

        <label>
          <input
            type="checkbox"
            checked={actionFilter.includes('revert')}
            onChange={(e) => {
              if (e.target.checked) {
                setActionFilter([...actionFilter, 'revert']);
              } else {
                setActionFilter(actionFilter.filter(a => a !== 'revert'));
              }
            }}
          />
          Reverts
        </label>
      </div>

      <AuditTrailViewer
        entityId={subscriptionId}
        entityType="subscription"
        fieldName="mrr"
        variant="timeline"
        filterByAction={actionFilter}
        showIcons={true}
        showTimestamps={true}
        showUsers={true}
        showReasons={true}
        onRevert={async (auditId) => {
          // Revert MRR correction
          await fetch(`/api/subscriptions/${subscriptionId}/revert`, {
            method: 'POST',
            body: JSON.stringify({ audit_id: auditId })
          });
        }}
        onExport={(format) => {
          // Export for finance reporting
          console.log(`Exporting MRR audit as ${format}`);
        }}
      />
    </div>
  );
}
```

**Entries:**

```json
[
  {
    "audit_id": "aud_sub_001",
    "entity_id": "sub_001",
    "entity_type": "subscription",
    "field_name": "mrr",
    "action": "override",
    "old_value": 99.00,
    "new_value": 79.20,
    "changed_by": "cs_manager_lisa",
    "changed_by_name": "Lisa Park",
    "changed_at": "2025-10-22T14:00:00Z",
    "reason": "Applied 20% annual discount negotiated with customer",
    "created_at": "2025-10-22T14:00:00Z"
  },
  {
    "audit_id": "aud_sub_002",
    "entity_id": "sub_001",
    "entity_type": "subscription",
    "field_name": "mrr",
    "action": "extracted",
    "old_value": null,
    "new_value": 99.00,
    "changed_by": "system_stripe_webhook",
    "changed_by_name": "Stripe Webhook",
    "changed_at": "2025-10-01T12:00:00Z",
    "created_at": "2025-10-01T12:00:00Z"
  }
]
```

---

### Example 7: Insurance - Policy Premium Adjustments

**Scenario:** Insurance underwriter reviews premium adjustment history for policy audit.

```typescript
function PolicyPremiumAudit({ policyId }: { policyId: string }) {
  return (
    <div className="policy-audit">
      <h2>Premium Adjustment History</h2>

      <AuditTrailViewer
        entityId={policyId}
        entityType="insurance_policy"
        fieldName="premium"
        variant="table"
        showIcons={true}
        showTimestamps={true}
        showUsers={true}
        showReasons={true}
        maxEntries={50}
        onExport={(format) => {
          // Export for regulatory audit
          console.log(`Exporting premium audit as ${format}`);
        }}
      />
    </div>
  );
}
```

---

## Edge Cases

### Edge Case 1: Very Long Audit Trail (100+ Entries)

**Scenario:** Entity has 500+ audit entries spanning years.

**Solution:** Pagination with configurable page size.

```typescript
function LongAuditTrail({ entityId }: { entityId: string }) {
  return (
    <AuditTrailViewer
      entityId={entityId}
      entityType="transaction"
      variant="table"
      maxEntries={50}  // Show 50 per page
      showIcons={false}  // Reduce visual noise
      showReasons={false}
      height="600px"
    />
  );
}
```

**Performance:**
- Backend returns paginated results (50 per page)
- Frontend only renders visible entries
- ScrollArea virtualizes long lists
- Load subsequent pages on demand

---

### Edge Case 2: No History (Empty State)

**Scenario:** Newly created entity with no changes yet.

**Solution:** Show helpful empty state with guidance.

```typescript
// Component automatically renders empty state
<AuditTrailViewer
  entityId="new_entity_001"
  entityType="transaction"
  variant="timeline"
/>

// Renders:
// ┌─────────────────────────────────┐
// │         ⏱                       │
// │    No audit history             │
// │                                 │
// │ No changes have been made       │
// │ to this entity                  │
// └─────────────────────────────────┘
```

---

### Edge Case 3: Real-Time Updates (New Entry While Viewing)

**Scenario:** User viewing audit trail when another user makes a change.

**Solution:** Enable real-time polling or WebSocket updates.

```typescript
function RealtimeAuditTrail({ entityId }: { entityId: string }) {
  const [showNotification, setShowNotification] = useState(false);

  return (
    <>
      {/* Notification banner */}
      {showNotification && (
        <div className="update-notification">
          New changes detected. <button onClick={() => window.location.reload()}>Refresh</button>
        </div>
      )}

      <AuditTrailViewer
        entityId={entityId}
        entityType="transaction"
        variant="timeline"
        enableRealtime={true}
        realtimeInterval={5000}  // Poll every 5 seconds
        onNewEntries={() => setShowNotification(true)}
      />
    </>
  );
}
```

**Implementation:**

```typescript
// Inside AuditTrailViewer component
useEffect(() => {
  if (!enableRealtime) return;

  const interval = setInterval(async () => {
    const response = await fetch(`/api/audit-trail?entity_id=${entityId}&since=${lastFetchTimestamp}`);
    const newEntries = await response.json();

    if (newEntries.length > 0) {
      setEntries(prev => [...newEntries, ...prev]);
      onNewEntries?.();
    }
  }, realtimeInterval);

  return () => clearInterval(interval);
}, [enableRealtime, realtimeInterval, entityId]);
```

---

### Edge Case 4: Export Large Histories (10,000+ Entries)

**Scenario:** User wants to export entire audit trail with 10,000+ entries.

**Solution:** Async export with progress indicator.

```typescript
function LargeExport({ entityId }: { entityId: string }) {
  const [exporting, setExporting] = useState(false);
  const [progress, setProgress] = useState(0);

  const handleExport = async (format: 'json' | 'csv') => {
    setExporting(true);
    setProgress(0);

    // Request async export
    const response = await fetch('/api/audit-trail/export', {
      method: 'POST',
      body: JSON.stringify({ entity_id: entityId, format })
    });

    const { export_id } = await response.json();

    // Poll for progress
    const pollInterval = setInterval(async () => {
      const statusResponse = await fetch(`/api/exports/${export_id}`);
      const { status, progress: p, download_url } = await statusResponse.json();

      setProgress(p);

      if (status === 'completed') {
        clearInterval(pollInterval);
        setExporting(false);

        // Download file
        window.location.href = download_url;
      }
    }, 1000);
  };

  return (
    <div>
      <AuditTrailViewer
        entityId={entityId}
        entityType="transaction"
        variant="table"
        onExport={handleExport}
      />

      {exporting && (
        <div className="export-progress">
          Exporting... {progress}%
        </div>
      )}
    </div>
  );
}
```

---

### Edge Case 5: Mobile View (Compact Layout)

**Scenario:** User viewing audit trail on mobile device.

**Solution:** Automatically switch to compact view on small screens.

```typescript
function ResponsiveAuditTrail({ entityId }: { entityId: string }) {
  const [isMobile, setIsMobile] = useState(false);

  useEffect(() => {
    const checkMobile = () => {
      setIsMobile(window.innerWidth < 768);
    };

    checkMobile();
    window.addEventListener('resize', checkMobile);
    return () => window.removeEventListener('resize', checkMobile);
  }, []);

  return (
    <AuditTrailViewer
      entityId={entityId}
      entityType="transaction"
      variant={isMobile ? 'compact' : 'timeline'}
      compact={isMobile}
      height={isMobile ? '300px' : '500px'}
    />
  );
}
```

---

### Edge Case 6: Filtering with No Results

**Scenario:** User filters by user who made no changes.

**Solution:** Show filtered empty state.

```typescript
// Component automatically shows:
// ┌─────────────────────────────────┐
// │         ⏱                       │
// │    No audit history             │
// │                                 │
// │ No entries match your filters   │
// │                                 │
// │ [Clear Filters]                 │
// └─────────────────────────────────┘
```

---

### Edge Case 7: Circular Reverts

**Scenario:** User reverts value A→B, then B→A, then A→B again.

**Solution:** Show complete revert chain in timeline.

```json
[
  {
    "audit_id": "aud_005",
    "action": "revert",
    "old_value": "Amazon Inc",
    "new_value": "Amazon.com",
    "changed_at": "2025-10-24T16:00:00Z",
    "reason": "Reverted to standardized name"
  },
  {
    "audit_id": "aud_004",
    "action": "override",
    "old_value": "Amazon.com",
    "new_value": "Amazon Inc",
    "changed_at": "2025-10-24T15:00:00Z",
    "reason": "Changed to legal entity name"
  },
  {
    "audit_id": "aud_003",
    "action": "revert",
    "old_value": "AMZN",
    "new_value": "Amazon.com",
    "changed_at": "2025-10-24T14:00:00Z",
    "reason": "Reverted accidental change"
  }
]
```

**UI shows:**
- Clear indication of revert actions
- Links to previous values
- Warning if value matches an earlier state

---

## Styling (Tailwind CSS)

```css
/* Timeline View */
.audit-trail-viewer.timeline-view {
  @apply bg-white dark:bg-gray-900 rounded-lg border border-gray-200 dark:border-gray-700;
}

.viewer-header {
  @apply flex items-center justify-between p-4 border-b border-gray-200 dark:border-gray-700;
}

.header-title {
  @apply flex items-center gap-2;
}

.title-icon {
  @apply w-5 h-5 text-gray-500;
}

.field-badge {
  @apply ml-2 px-2 py-1 text-xs bg-blue-100 text-blue-700 rounded;
}

.header-actions {
  @apply flex items-center gap-2;
}

.search-input {
  @apply px-3 py-2 text-sm border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500;
}

.timeline-content {
  @apply p-4 overflow-y-auto;
}

.timeline-line {
  @apply absolute left-8 top-0 bottom-0 w-0.5 bg-gray-200 dark:bg-gray-700;
}

.timeline-entry {
  @apply relative mb-4 flex gap-4;
}

.entry-icon {
  @apply relative z-10 flex items-center justify-center w-8 h-8 rounded-full border-2 border-white dark:border-gray-900;
}

.entry-icon.bg-blue {
  @apply bg-blue-500 text-white;
}

.entry-icon.bg-orange {
  @apply bg-orange-500 text-white;
}

.entry-icon.bg-purple {
  @apply bg-purple-500 text-white;
}

.entry-icon.bg-red {
  @apply bg-red-500 text-white;
}

.entry-icon.bg-green {
  @apply bg-green-500 text-white;
}

.entry-icon.bg-gray {
  @apply bg-gray-500 text-white;
}

.entry-card {
  @apply flex-1 p-4 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg shadow-sm;
}

.entry-header {
  @apply flex items-center justify-between mb-2;
}

.entry-meta {
  @apply flex items-center gap-2 flex-wrap;
}

.entry-user {
  @apply flex items-center gap-2;
}

.user-name {
  @apply text-sm font-medium text-gray-900 dark:text-gray-100;
}

.entry-timestamp {
  @apply text-xs text-gray-500 dark:text-gray-400;
}

.expand-button {
  @apply p-1 text-gray-400 hover:text-gray-600 dark:hover:text-gray-300;
}

.entry-summary {
  @apply flex items-center gap-2 text-sm;
}

.field-name {
  @apply font-medium text-gray-700 dark:text-gray-300;
}

.change-arrow {
  @apply text-gray-400;
}

.new-value {
  @apply font-mono text-blue-600 dark:text-blue-400;
}

.entry-details {
  @apply mt-3 pt-3 border-t border-gray-100 dark:border-gray-700 space-y-2;
}

.detail-row {
  @apply flex items-start gap-2 text-sm;
}

.detail-label {
  @apply font-medium text-gray-600 dark:text-gray-400 min-w-24;
}

.detail-value {
  @apply font-mono text-gray-900 dark:text-gray-100;
}

.detail-value.old {
  @apply text-red-600 dark:text-red-400 line-through;
}

.detail-value.new {
  @apply text-green-600 dark:text-green-400;
}

.detail-value.reason {
  @apply font-sans italic text-gray-600 dark:text-gray-400;
}

.entry-actions {
  @apply flex gap-2 mt-3;
}

/* Table View */
.audit-trail-viewer.table-view {
  @apply bg-white dark:bg-gray-900 rounded-lg border border-gray-200 dark:border-gray-700;
}

.audit-trail-viewer.table-view table {
  @apply w-full text-sm;
}

.audit-trail-viewer.table-view th {
  @apply px-4 py-3 text-left font-medium text-gray-700 dark:text-gray-300 bg-gray-50 dark:bg-gray-800 border-b border-gray-200 dark:border-gray-700;
}

.audit-trail-viewer.table-view td {
  @apply px-4 py-3 border-b border-gray-100 dark:border-gray-800;
}

.timestamp-cell {
  @apply text-gray-600 dark:text-gray-400;
}

.field-cell {
  @apply font-medium text-gray-900 dark:text-gray-100;
}

.value-cell {
  @apply font-mono text-sm;
}

.value-cell.old {
  @apply text-red-600 dark:text-red-400;
}

.value-cell.new {
  @apply text-green-600 dark:text-green-400;
}

.user-cell .user-info {
  @apply flex items-center gap-2;
}

.reason-cell {
  @apply text-gray-600 dark:text-gray-400 max-w-xs truncate;
}

.actions-cell {
  @apply text-right;
}

/* Compact View */
.audit-trail-viewer.compact-view {
  @apply bg-white dark:bg-gray-900 rounded-lg border border-gray-200 dark:border-gray-700;
}

.compact-header {
  @apply flex items-center justify-between px-3 py-2 border-b border-gray-200 dark:border-gray-700;
}

.compact-title {
  @apply text-sm font-medium text-gray-700 dark:text-gray-300;
}

.export-icon {
  @apply p-1 text-gray-400 hover:text-gray-600 dark:hover:text-gray-300;
}

.compact-content {
  @apply divide-y divide-gray-100 dark:divide-gray-800;
}

.compact-entry {
  @apply flex items-start gap-2 px-3 py-2 hover:bg-gray-50 dark:hover:bg-gray-800;
}

.compact-icon {
  @apply w-4 h-4 mt-0.5 flex-shrink-0;
}

.compact-entry.blue .compact-icon {
  @apply text-blue-500;
}

.compact-entry.orange .compact-icon {
  @apply text-orange-500;
}

.compact-entry.purple .compact-icon {
  @apply text-purple-500;
}

.compact-entry.red .compact-icon {
  @apply text-red-500;
}

.compact-entry.green .compact-icon {
  @apply text-green-500;
}

.compact-info {
  @apply flex-1 min-w-0;
}

.compact-change {
  @apply block text-sm text-gray-900 dark:text-gray-100 truncate;
}

.compact-meta {
  @apply block text-xs text-gray-500 dark:text-gray-400 truncate;
}

/* Pagination */
.pagination {
  @apply flex items-center justify-between px-4 py-3 border-t border-gray-200 dark:border-gray-700;
}

.page-info {
  @apply text-sm text-gray-600 dark:text-gray-400;
}

/* Empty/Error States */
.empty-state,
.error-state,
.loading-state {
  @apply flex flex-col items-center justify-center py-12 text-center;
}

.empty-icon,
.error-icon {
  @apply w-12 h-12 text-gray-400 mb-4;
}

.empty-message,
.error-message {
  @apply text-sm text-gray-500 dark:text-gray-400 mt-1;
}

/* Compact modifier */
.audit-trail-viewer.compact {
  @apply text-sm;
}

.audit-trail-viewer.compact .entry-card {
  @apply p-3;
}

.audit-trail-viewer.compact .entry-icon {
  @apply w-6 h-6;
}
```

---

## Implementation Notes

### 1. Data Fetching Strategy

```typescript
// API endpoint structure
GET /api/audit-trail?entity_id=xxx&entity_type=xxx&page=1&limit=50

Response:
{
  "entries": AuditEntry[],
  "total": number,
  "page": number,
  "total_pages": number
}
```

### 2. Real-Time Updates

```typescript
// WebSocket approach (preferred)
const socket = new WebSocket('ws://api/audit-trail/subscribe');

socket.send(JSON.stringify({
  action: 'subscribe',
  entity_id: entityId
}));

socket.onmessage = (event) => {
  const newEntry = JSON.parse(event.data);
  setEntries(prev => [newEntry, ...prev]);
};

// Polling approach (fallback)
useInterval(() => {
  fetchNewEntries();
}, 5000);
```

### 3. Export Optimization

```typescript
// For large exports, use server-side generation
async function handleLargeExport(entityId: string, format: 'json' | 'csv') {
  // Request async export job
  const { job_id } = await fetch('/api/audit-trail/export', {
    method: 'POST',
    body: JSON.stringify({ entity_id: entityId, format })
  }).then(r => r.json());

  // Poll for completion
  const checkStatus = async () => {
    const { status, download_url } = await fetch(`/api/export-jobs/${job_id}`)
      .then(r => r.json());

    if (status === 'completed') {
      window.location.href = download_url;
    } else {
      setTimeout(checkStatus, 1000);
    }
  };

  checkStatus();
}
```

### 4. Virtualization for Long Lists

```typescript
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualizedTimeline({ entries }: { entries: AuditEntry[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: entries.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100,
    overscan: 5
  });

  return (
    <div ref={parentRef} style={{ height: '500px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative'
        }}
      >
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualRow.start}px)`
            }}
          >
            <TimelineEntry entry={entries[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Testing Strategy

### Unit Tests

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { AuditTrailViewer } from './AuditTrailViewer';

describe('AuditTrailViewer', () => {
  const mockEntries: AuditEntry[] = [
    {
      audit_id: 'aud_001',
      entity_id: 'txn_001',
      entity_type: 'transaction',
      field_name: 'merchant',
      action: 'override',
      old_value: 'AMZN',
      new_value: 'Amazon.com',
      changed_by: 'user_jane',
      changed_by_name: 'Jane Doe',
      changed_at: '2025-10-24T14:30:00Z',
      reason: 'Normalized name',
      created_at: '2025-10-24T14:30:00Z'
    }
  ];

  beforeEach(() => {
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({
          entries: mockEntries,
          total: 1,
          page: 1,
          total_pages: 1
        })
      })
    ) as jest.Mock;
  });

  it('renders timeline view by default', async () => {
    render(
      <AuditTrailViewer
        entityId="txn_001"
        entityType="transaction"
        variant="timeline"
      />
    );

    await waitFor(() => {
      expect(screen.getByText('Audit Trail')).toBeInTheDocument();
      expect(screen.getByText('merchant')).toBeInTheDocument();
      expect(screen.getByText('Amazon.com')).toBeInTheDocument();
    });
  });

  it('filters entries by search text', async () => {
    render(
      <AuditTrailViewer
        entityId="txn_001"
        entityType="transaction"
        variant="timeline"
      />
    );

    await waitFor(() => {
      expect(screen.getByText('merchant')).toBeInTheDocument();
    });

    const searchInput = screen.getByPlaceholderText('Search history...');
    fireEvent.change(searchInput, { target: { value: 'category' } });

    await waitFor(() => {
      expect(screen.queryByText('merchant')).not.toBeInTheDocument();
    });
  });

  it('calls onRevert when revert button clicked', async () => {
    const onRevert = jest.fn();

    render(
      <AuditTrailViewer
        entityId="txn_001"
        entityType="transaction"
        variant="timeline"
        onRevert={onRevert}
      />
    );

    await waitFor(() => {
      expect(screen.getByText('merchant')).toBeInTheDocument();
    });

    // Expand entry
    const expandButton = screen.getAllByLabelText('Expand')[0];
    fireEvent.click(expandButton);

    // Click revert
    const revertButton = screen.getByText(/Revert to this value/);
    fireEvent.click(revertButton);

    expect(onRevert).toHaveBeenCalledWith('aud_001');
  });

  it('exports to JSON', async () => {
    const onExport = jest.fn();

    render(
      <AuditTrailViewer
        entityId="txn_001"
        entityType="transaction"
        variant="timeline"
        onExport={onExport}
      />
    );

    await waitFor(() => {
      expect(screen.getByText('Export')).toBeInTheDocument();
    });

    // Open export menu
    const exportButton = screen.getByText('Export');
    fireEvent.click(exportButton);

    // Click JSON option
    const jsonOption = screen.getByText('Export JSON');
    fireEvent.click(jsonOption);

    expect(onExport).toHaveBeenCalledWith('json');
  });

  it('shows empty state when no entries', async () => {
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({
          entries: [],
          total: 0,
          page: 1,
          total_pages: 0
        })
      })
    ) as jest.Mock;

    render(
      <AuditTrailViewer
        entityId="txn_001"
        entityType="transaction"
        variant="timeline"
      />
    );

    await waitFor(() => {
      expect(screen.getByText('No audit history')).toBeInTheDocument();
      expect(screen.getByText(/No changes have been made/)).toBeInTheDocument();
    });
  });

  it('shows error state on fetch failure', async () => {
    global.fetch = jest.fn(() =>
      Promise.reject(new Error('Network error'))
    ) as jest.Mock;

    render(
      <AuditTrailViewer
        entityId="txn_001"
        entityType="transaction"
        variant="timeline"
      />
    );

    await waitFor(() => {
      expect(screen.getByText('Failed to load audit trail')).toBeInTheDocument();
      expect(screen.getByText('Network error')).toBeInTheDocument();
    });
  });

  it('paginates results', async () => {
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({
          entries: mockEntries,
          total: 100,
          page: 1,
          total_pages: 2
        })
      })
    ) as jest.Mock;

    render(
      <AuditTrailViewer
        entityId="txn_001"
        entityType="transaction"
        variant="timeline"
        maxEntries={50}
      />
    );

    await waitFor(() => {
      expect(screen.getByText('Page 1 of 2')).toBeInTheDocument();
    });

    const nextButton = screen.getByText('Next');
    expect(nextButton).not.toBeDisabled();

    fireEvent.click(nextButton);

    await waitFor(() => {
      expect(screen.getByText('Page 2 of 2')).toBeInTheDocument();
    });
  });
});
```

---

## Performance Benchmarks

| Metric | Target | Notes |
|--------|--------|-------|
| Initial Load (50 entries) | < 500ms | Includes API fetch + render |
| Pagination Change | < 200ms | Client-side state update |
| Search Filter | < 100ms | Local array filter |
| Export (1000 entries) | < 2s | JSON generation |
| Export (10,000 entries) | < 30s | Async job |
| Real-time Poll | 5s interval | Configurable |
| Memory Usage (1000 entries) | < 50MB | Virtual scrolling helps |

---

## Accessibility

```typescript
// ARIA labels and roles
<div role="region" aria-label="Audit Trail">
  <h3 id="audit-trail-heading">Audit Trail</h3>

  <div role="list" aria-labelledby="audit-trail-heading">
    {entries.map(entry => (
      <div key={entry.audit_id} role="listitem">
        <button
          aria-expanded={isExpanded}
          aria-controls={`details-${entry.audit_id}`}
          aria-label={`Expand details for ${entry.field_name} change`}
        >
          {/* ... */}
        </button>

        {isExpanded && (
          <div id={`details-${entry.audit_id}`}>
            {/* ... */}
          </div>
        )}
      </div>
    ))}
  </div>
</div>

// Keyboard navigation
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    if (e.key === 'ArrowDown') {
      // Navigate to next entry
      focusNextEntry();
    } else if (e.key === 'ArrowUp') {
      // Navigate to previous entry
      focusPreviousEntry();
    } else if (e.key === 'Enter' || e.key === ' ') {
      // Expand/collapse current entry
      toggleCurrentEntry();
    }
  };

  document.addEventListener('keydown', handleKeyDown);
  return () => document.removeEventListener('keydown', handleKeyDown);
}, []);

// Screen reader announcements
<div role="status" aria-live="polite" className="sr-only">
  {entries.length} audit entries loaded
</div>
```

---

## Related Components

- **OverrideStore** (OL): Backend storage for field overrides
- **ValidationEngine** (OL): Validates corrections before saving
- **FieldCorrectionDrawer** (IL): UI for making corrections (generates audit entries)
- **ReconciliationPanel** (IL): Shows reconciliation status with audit trail link

---

## References

- [Objective Layer Architecture](../../architecture/objective-layer.md)
- [Corrections Flow (Vertical 4.3)](../../verticals/corrections-flow.md)
- [OverrideStore OL Primitive](../ol/OverrideStore.md)
- [Field-Level Reconciliation](../../reconciliation/field-level.md)

---

## Changelog

### Version 1.0.0 (2025-10-24)

- Initial specification
- Timeline, table, and compact view variants
- Multi-domain examples (finance, healthcare, legal, research, e-commerce, SaaS)
- Real-time updates support
- Export to JSON/CSV
- Pagination for large histories
- Mobile-responsive design
- Accessibility features (ARIA, keyboard navigation)
- Comprehensive edge case handling

---

**End of AuditTrailViewer IL Component Specification**
