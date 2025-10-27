# SeriesManager (IL Component)

## Definition

**SeriesManager** is a full CRUD UI component for managing recurring payment series (create, edit, archive, view instances). It provides a complete interface for series lifecycle management with status badges, variance alerts, instance history, filtering, and search capabilities.

**Problem it solves:**
- No standardized UI for managing recurring payment expectations across different domains
- Difficult to track which recurring payments are missing, late, or have amount variances
- No unified pattern for viewing instance history (expected vs actual)
- Scattered recurring payment management across multiple screens
- No clear visual status indicators for series health
- Hard to identify patterns in recurring payment issues

**Solution:**
- Single component for all series CRUD operations
- Visual status badges (✅ Paid, ⚠️ Variance, 🔴 Missing, 📅 Upcoming)
- Instance history view showing last 12 months of expected vs actual
- Variance alerts panel highlighting issues requiring attention
- Integrated RecurrenceConfigDialog for frequency pattern definition
- Search, filter, and group capabilities
- Archive confirmation dialog (soft delete preserves history)
- Responsive design (table on desktop, cards on mobile)
- Keyboard shortcuts for power users

---

## Interface Contract

```typescript
interface SeriesManagerProps {
  // Data
  series: Series[];  // All series (active + archived)
  instances: Record<string, SeriesInstance[]>;  // Instances per series (last 12 months)
  accounts: Account[];  // For filtering and display
  counterparties: Counterparty[];  // For display
  categories: string[];  // Available category options
  currentUserId: string;  // For ownership checks

  // Callbacks
  onSeriesCreate: (series: CreateSeriesInput) => Promise<Series>;
  onSeriesUpdate: (seriesId: string, updates: Partial<Series>) => Promise<Series>;
  onSeriesArchive: (seriesId: string, endDate: string) => Promise<void>;
  onSeriesUnarchive: (seriesId: string) => Promise<void>;
  onInstanceLink: (seriesId: string, transactionId: string, force?: boolean) => Promise<SeriesInstance>;
  onInstanceUnlink: (instanceId: string) => Promise<void>;
  onInstanceSkip: (instanceId: string) => Promise<SeriesInstance>;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Filters & Display
  initialFilter?: SeriesFilter;  // Pre-applied filter
  showArchived?: boolean;  // Show archived series (default: false)
  allowCreate?: boolean;  // Show "New Series" button (default: true)
  allowEdit?: boolean;  // Allow editing series (default: true)
  allowArchive?: boolean;  // Allow archiving series (default: true)
  showVariancePanel?: boolean;  // Show variance alerts panel (default: true)

  // Customization
  theme?: "light" | "dark";
  compact?: boolean;  // Compact view for smaller spaces
  groupBy?: "category" | "account" | "status" | "none";  // Group series in list
}

interface Series {
  series_id: string;
  user_id: string;
  name: string;
  account_id: string;
  counterparty_id: string;
  expected_amount: number;  // Negative for expenses, positive for income
  tolerance: number;  // Amount variance allowed (e.g., 2.00 means ±$2)
  frequency: FrequencyPattern;
  start_date: string;  // ISO date
  end_date: string | null;  // ISO date, null if ongoing
  category: string;  // "subscriptions", "bills", "income", "other"
  is_active: boolean;
  next_expected_date: string | null;  // ISO date
  created_at: string;
  updated_at: string;

  // Optional metadata
  notes?: string;
  icon?: string;  // Emoji
  color?: string;  // Hex color
}

interface SeriesInstance {
  instance_id: string;
  series_id: string;
  user_id: string;
  transaction_id: string | null;
  expected_date: string;  // ISO date
  actual_date: string | null;  // ISO date
  expected_amount: number;
  actual_amount: number | null;
  status: InstanceStatus;
  variance: number | null;  // Difference between expected and actual amount
  link_type: "auto" | "manual" | null;
  created_at: string;
}

type FrequencyPattern =
  | { type: "daily"; interval: number }  // Every N days
  | { type: "weekly"; day_of_week: 0 | 1 | 2 | 3 | 4 | 5 | 6; interval: number }  // Every N weeks on day (0=Monday)
  | { type: "monthly"; day_of_month: 1 | 2 | ... | 31; interval: number }  // Every N months on day
  | { type: "yearly"; month: 1 | 2 | ... | 12; day: 1 | 2 | ... | 31 }  // Yearly on specific date
  | { type: "custom"; dates: string[] };  // Explicit list of dates

type InstanceStatus =
  | "matched"           // Auto-linked, within tolerance
  | "matched_manual"    // Manually linked
  | "variance"          // Matched but amount outside tolerance
  | "missing"           // Expected date passed, no match
  | "upcoming"          // Expected in future
  | "skipped";          // User marked as intentionally skipped

interface CreateSeriesInput {
  name: string;
  account_id: string;
  counterparty_id: string;
  expected_amount: number;
  tolerance: number;
  frequency: FrequencyPattern;
  start_date: string;
  category: string;
  notes?: string;
  icon?: string;
  color?: string;
}

interface SeriesFilter {
  searchQuery?: string;  // Search by series name
  accountIds?: string[];  // Filter by account
  categories?: string[];  // Filter by category
  statuses?: InstanceStatus[];  // Filter by last instance status
  showArchived?: boolean;  // Include archived series
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo } from 'react';

export const SeriesManager: React.FC<SeriesManagerProps> = ({
  series,
  instances,
  accounts,
  counterparties,
  categories,
  currentUserId,
  onSeriesCreate,
  onSeriesUpdate,
  onSeriesArchive,
  onSeriesUnarchive,
  onInstanceLink,
  onInstanceUnlink,
  onInstanceSkip,
  loading = false,
  error = null,
  initialFilter = {},
  showArchived = false,
  allowCreate = true,
  allowEdit = true,
  allowArchive = true,
  showVariancePanel = true,
  theme = "light",
  compact = false,
  groupBy = "category"
}) => {
  const [filter, setFilter] = useState<SeriesFilter>({
    showArchived,
    ...initialFilter
  });
  const [sortBy, setSortBy] = useState<"name" | "next_expected" | "category" | "updated_at">("name");
  const [sortDirection, setSortDirection] = useState<"asc" | "desc">("asc");
  const [showCreateModal, setShowCreateModal] = useState(false);
  const [editingSeries, setEditingSeries] = useState<Series | null>(null);
  const [archiveConfirm, setArchiveConfirm] = useState<Series | null>(null);
  const [viewingSeriesId, setViewingSeriesId] = useState<string | null>(null);
  const [showVariances, setShowVariances] = useState(showVariancePanel);

  // Calculate series status based on last instance
  const getSeriesStatus = (seriesId: string): InstanceStatus | null => {
    const seriesInstances = instances[seriesId] || [];
    if (seriesInstances.length === 0) return null;

    // Find most recent instance
    const sortedInstances = [...seriesInstances].sort(
      (a, b) => new Date(b.expected_date).getTime() - new Date(a.expected_date).getTime()
    );

    return sortedInstances[0]?.status || null;
  };

  // Filter series
  const filteredSeries = useMemo(() => {
    let result = series;

    // Filter by archived status
    if (!filter.showArchived) {
      result = result.filter(s => s.is_active);
    }

    // Search by name
    if (filter.searchQuery) {
      const query = filter.searchQuery.toLowerCase();
      result = result.filter(s =>
        s.name.toLowerCase().includes(query)
      );
    }

    // Filter by account
    if (filter.accountIds && filter.accountIds.length > 0) {
      result = result.filter(s => filter.accountIds!.includes(s.account_id));
    }

    // Filter by category
    if (filter.categories && filter.categories.length > 0) {
      result = result.filter(s => filter.categories!.includes(s.category));
    }

    // Filter by status
    if (filter.statuses && filter.statuses.length > 0) {
      result = result.filter(s => {
        const status = getSeriesStatus(s.series_id);
        return status && filter.statuses!.includes(status);
      });
    }

    return result;
  }, [series, filter, instances]);

  // Sort series
  const sortedSeries = useMemo(() => {
    const sorted = [...filteredSeries];
    sorted.sort((a, b) => {
      let comparison = 0;

      switch (sortBy) {
        case "name":
          comparison = a.name.localeCompare(b.name);
          break;
        case "next_expected":
          const dateA = a.next_expected_date ? new Date(a.next_expected_date).getTime() : Infinity;
          const dateB = b.next_expected_date ? new Date(b.next_expected_date).getTime() : Infinity;
          comparison = dateA - dateB;
          break;
        case "category":
          comparison = a.category.localeCompare(b.category);
          break;
        case "updated_at":
          comparison = new Date(a.updated_at).getTime() - new Date(b.updated_at).getTime();
          break;
      }

      return sortDirection === "asc" ? comparison : -comparison;
    });

    return sorted;
  }, [filteredSeries, sortBy, sortDirection]);

  // Group series
  const groupedSeries = useMemo(() => {
    if (groupBy === "none") {
      return { "All Series": sortedSeries };
    }

    const groups: Record<string, Series[]> = {};
    sortedSeries.forEach(s => {
      let key: string;

      switch (groupBy) {
        case "category":
          key = s.category;
          break;
        case "account":
          const account = accounts.find(a => a.account_id === s.account_id);
          key = account?.name || "Unknown Account";
          break;
        case "status":
          const status = getSeriesStatus(s.series_id);
          key = formatStatus(status);
          break;
        default:
          key = "All Series";
      }

      if (!groups[key]) groups[key] = [];
      groups[key].push(s);
    });

    return groups;
  }, [sortedSeries, groupBy, accounts, instances]);

  // Get variance instances (missing + variance status)
  const varianceInstances = useMemo(() => {
    const variances: Array<{ series: Series; instance: SeriesInstance }> = [];

    sortedSeries.forEach(s => {
      const seriesInstances = instances[s.series_id] || [];
      seriesInstances.forEach(inst => {
        if (inst.status === "missing" || inst.status === "variance") {
          variances.push({ series: s, instance: inst });
        }
      });
    });

    // Sort by expected date (most recent first)
    return variances.sort((a, b) =>
      new Date(b.instance.expected_date).getTime() - new Date(a.instance.expected_date).getTime()
    );
  }, [sortedSeries, instances]);

  // Keyboard shortcuts
  React.useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      // N = New series
      if (e.key === 'n' && !e.ctrlKey && !e.metaKey && allowCreate) {
        const activeElement = document.activeElement as HTMLElement;
        if (activeElement.tagName !== 'INPUT' && activeElement.tagName !== 'TEXTAREA') {
          e.preventDefault();
          setShowCreateModal(true);
        }
      }

      // ESC = Close modals
      if (e.key === 'Escape') {
        setShowCreateModal(false);
        setEditingSeries(null);
        setArchiveConfirm(null);
        setViewingSeriesId(null);
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [allowCreate]);

  // Loading state
  if (loading) {
    return (
      <div className={`series-manager theme-${theme} loading`}>
        <div className="skeleton skeleton-header"></div>
        <div className="skeleton skeleton-list"></div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`series-manager theme-${theme} error`}>
        <div className="error-icon">⚠️</div>
        <div className="error-message">{error}</div>
      </div>
    );
  }

  return (
    <div className={`series-manager theme-${theme} ${compact ? 'compact' : ''}`}>
      {/* Header */}
      <div className="manager-header">
        <h2>Recurring Payment Series</h2>
        <div className="header-actions">
          {allowCreate && (
            <button
              className="create-button"
              onClick={() => setShowCreateModal(true)}
              title="New series (keyboard: N)"
            >
              + New Series
            </button>
          )}
          {showVariancePanel && varianceInstances.length > 0 && (
            <button
              className={`variance-toggle ${showVariances ? 'active' : ''}`}
              onClick={() => setShowVariances(!showVariances)}
              title="Show/hide variance alerts"
            >
              ⚠️ Alerts ({varianceInstances.length})
            </button>
          )}
        </div>
      </div>

      {/* Search & Filters */}
      <div className="manager-filters">
        <input
          type="text"
          placeholder="Search series by name..."
          value={filter.searchQuery || ""}
          onChange={(e) => setFilter({ ...filter, searchQuery: e.target.value })}
          className="search-input"
          aria-label="Search series"
        />

        <select
          value={groupBy}
          onChange={(e) => setGroupBy(e.target.value as any)}
          className="group-by-select"
          aria-label="Group by"
        >
          <option value="none">No Grouping</option>
          <option value="category">Group by Category</option>
          <option value="account">Group by Account</option>
          <option value="status">Group by Status</option>
        </select>

        <select
          value={sortBy}
          onChange={(e) => setSortBy(e.target.value as any)}
          className="sort-select"
          aria-label="Sort by"
        >
          <option value="name">Sort by Name</option>
          <option value="next_expected">Sort by Next Expected</option>
          <option value="category">Sort by Category</option>
          <option value="updated_at">Sort by Last Updated</option>
        </select>

        <button
          className="sort-direction-button"
          onClick={() => setSortDirection(sortDirection === "asc" ? "desc" : "asc")}
          title={`Sort ${sortDirection === "asc" ? "descending" : "ascending"}`}
          aria-label={`Toggle sort direction (currently ${sortDirection})`}
        >
          {sortDirection === "asc" ? "↑" : "↓"}
        </button>

        <label className="show-archived-toggle">
          <input
            type="checkbox"
            checked={filter.showArchived}
            onChange={(e) => setFilter({ ...filter, showArchived: e.target.checked })}
          />
          Show Archived
        </label>
      </div>

      {/* Main Content Area */}
      <div className="manager-content">
        {/* Variance Alerts Panel */}
        {showVariances && varianceInstances.length > 0 && (
          <div className="variance-panel">
            <div className="panel-header">
              <h3>Requires Attention</h3>
              <span className="variance-count">{varianceInstances.length} issue{varianceInstances.length > 1 ? 's' : ''}</span>
            </div>
            <div className="variance-list">
              {varianceInstances.map(({ series: s, instance: inst }) => (
                <VarianceAlertCard
                  key={inst.instance_id}
                  series={s}
                  instance={inst}
                  account={accounts.find(a => a.account_id === s.account_id)}
                  onViewSeries={() => setViewingSeriesId(s.series_id)}
                  onSkip={() => onInstanceSkip(inst.instance_id)}
                  compact={compact}
                  theme={theme}
                />
              ))}
            </div>
          </div>
        )}

        {/* Series List */}
        <div className="series-list-container">
          {Object.keys(groupedSeries).length === 0 || sortedSeries.length === 0 ? (
            <div className="empty-state">
              <div className="empty-icon">📅</div>
              <div className="empty-message">
                {filter.searchQuery
                  ? `No series found matching "${filter.searchQuery}"`
                  : "No recurring payment series yet. Create your first series to track subscriptions, bills, and recurring income."}
              </div>
              {allowCreate && !filter.searchQuery && (
                <button
                  className="create-button-secondary"
                  onClick={() => setShowCreateModal(true)}
                >
                  Create Series
                </button>
              )}
            </div>
          ) : (
            <div className="series-groups">
              {Object.entries(groupedSeries).map(([groupName, groupSeries]) => (
                <div key={groupName} className="series-group">
                  {groupBy !== "none" && (
                    <div className="group-header">
                      {groupName}
                      <span className="group-count">({groupSeries.length})</span>
                    </div>
                  )}

                  <div className="series-list">
                    {groupSeries.map(s => (
                      <SeriesCard
                        key={s.series_id}
                        series={s}
                        instances={instances[s.series_id] || []}
                        account={accounts.find(a => a.account_id === s.account_id)}
                        counterparty={counterparties.find(c => c.counterparty_id === s.counterparty_id)}
                        onView={() => setViewingSeriesId(s.series_id)}
                        onEdit={allowEdit ? () => setEditingSeries(s) : undefined}
                        onArchive={allowArchive && s.is_active ? () => setArchiveConfirm(s) : undefined}
                        onUnarchive={allowArchive && !s.is_active ? () => onSeriesUnarchive(s.series_id) : undefined}
                        compact={compact}
                        theme={theme}
                      />
                    ))}
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      </div>

      {/* Create Series Modal */}
      {showCreateModal && (
        <SeriesCreateModal
          accounts={accounts}
          counterparties={counterparties}
          categories={categories}
          onClose={() => setShowCreateModal(false)}
          onCreate={async (data) => {
            await onSeriesCreate(data);
            setShowCreateModal(false);
          }}
          theme={theme}
        />
      )}

      {/* Edit Series Modal */}
      {editingSeries && (
        <SeriesEditModal
          series={editingSeries}
          categories={categories}
          onClose={() => setEditingSeries(null)}
          onUpdate={async (updates) => {
            await onSeriesUpdate(editingSeries.series_id, updates);
            setEditingSeries(null);
          }}
          theme={theme}
        />
      )}

      {/* Archive Confirmation Modal */}
      {archiveConfirm && (
        <ArchiveConfirmModal
          series={archiveConfirm}
          instanceCount={instances[archiveConfirm.series_id]?.length || 0}
          onClose={() => setArchiveConfirm(null)}
          onConfirm={async (endDate) => {
            await onSeriesArchive(archiveConfirm.series_id, endDate);
            setArchiveConfirm(null);
          }}
          theme={theme}
        />
      )}

      {/* Series Detail View (with instance history) */}
      {viewingSeriesId && (
        <SeriesDetailDialog
          series={series.find(s => s.series_id === viewingSeriesId)!}
          instances={instances[viewingSeriesId] || []}
          account={accounts.find(a => a.account_id === series.find(s => s.series_id === viewingSeriesId)?.account_id)}
          counterparty={counterparties.find(c => c.counterparty_id === series.find(s => s.series_id === viewingSeriesId)?.counterparty_id)}
          onClose={() => setViewingSeriesId(null)}
          onInstanceLink={onInstanceLink}
          onInstanceUnlink={onInstanceUnlink}
          onInstanceSkip={onInstanceSkip}
          theme={theme}
        />
      )}
    </div>
  );
};

// Individual series card component
const SeriesCard: React.FC<{
  series: Series;
  instances: SeriesInstance[];
  account?: Account;
  counterparty?: Counterparty;
  onView: () => void;
  onEdit?: () => void;
  onArchive?: () => void;
  onUnarchive?: () => void;
  compact: boolean;
  theme: "light" | "dark";
}> = ({ series, instances, account, counterparty, onView, onEdit, onArchive, onUnarchive, compact, theme }) => {
  // Get most recent instance for status
  const latestInstance = useMemo(() => {
    if (instances.length === 0) return null;
    return [...instances].sort(
      (a, b) => new Date(b.expected_date).getTime() - new Date(a.expected_date).getTime()
    )[0];
  }, [instances]);

  const status = latestInstance?.status || "upcoming";
  const statusBadge = getStatusBadge(status);

  const formattedAmount = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: account?.currency || 'USD',
    minimumFractionDigits: 2
  }).format(Math.abs(series.expected_amount));

  const nextExpected = series.next_expected_date
    ? new Date(series.next_expected_date).toLocaleDateString()
    : "N/A";

  return (
    <div
      className={`series-card ${series.is_archived ? 'archived' : ''} ${compact ? 'compact' : ''}`}
      onClick={onView}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => {
        if (e.key === 'Enter' || e.key === ' ') {
          e.preventDefault();
          onView();
        }
      }}
      aria-label={`${series.name}, ${formatFrequency(series.frequency)}, ${formattedAmount}, Status: ${formatStatus(status)}`}
    >
      <div className="series-icon" style={{ background: series.color || '#3b82f6' }}>
        {series.icon || getDefaultSeriesIcon(series.category)}
      </div>

      <div className="series-info">
        <div className="series-name">
          {series.name}
          {series.is_archived && <span className="archived-badge">Archived</span>}
          {statusBadge}
        </div>
        <div className="series-meta">
          <span className="series-frequency">{formatFrequency(series.frequency)}</span>
          <span className="separator">•</span>
          <span className="series-amount">{formattedAmount}</span>
          {account && (
            <>
              <span className="separator">•</span>
              <span className="series-account">{account.name}</span>
            </>
          )}
        </div>
        {!compact && counterparty && (
          <div className="series-counterparty">
            To/From: {counterparty.canonical_name}
          </div>
        )}
      </div>

      <div className="series-next">
        <div className="next-label">Next Expected</div>
        <div className="next-date">{nextExpected}</div>
      </div>

      <div className="series-actions" onClick={(e) => e.stopPropagation()}>
        {onEdit && !series.is_archived && (
          <button className="action-button edit" onClick={onEdit} title="Edit series">
            ✏️
          </button>
        )}
        {onArchive && !series.is_archived && (
          <button className="action-button archive" onClick={onArchive} title="Archive series">
            📦
          </button>
        )}
        {onUnarchive && series.is_archived && (
          <button className="action-button restore" onClick={onUnarchive} title="Restore series">
            ↺
          </button>
        )}
      </div>
    </div>
  );
};

// Variance alert card component
const VarianceAlertCard: React.FC<{
  series: Series;
  instance: SeriesInstance;
  account?: Account;
  onViewSeries: () => void;
  onSkip: () => void;
  compact: boolean;
  theme: "light" | "dark";
}> = ({ series, instance, account, onViewSeries, onSkip, compact, theme }) => {
  const isMissing = instance.status === "missing";
  const isVariance = instance.status === "variance";

  const expectedDate = new Date(instance.expected_date).toLocaleDateString();
  const daysOverdue = isMissing
    ? Math.floor((Date.now() - new Date(instance.expected_date).getTime()) / (1000 * 60 * 60 * 24))
    : 0;

  const formattedExpected = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: account?.currency || 'USD',
    minimumFractionDigits: 2
  }).format(Math.abs(instance.expected_amount));

  const formattedActual = instance.actual_amount
    ? new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: account?.currency || 'USD',
        minimumFractionDigits: 2
      }).format(Math.abs(instance.actual_amount))
    : null;

  const formattedVariance = instance.variance
    ? new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: account?.currency || 'USD',
        minimumFractionDigits: 2,
        signDisplay: 'always'
      }).format(instance.variance)
    : null;

  return (
    <div className={`variance-alert-card ${isMissing ? 'missing' : 'variance'} ${compact ? 'compact' : ''}`}>
      <div className="alert-icon">
        {isMissing ? '🔴' : '⚠️'}
      </div>

      <div className="alert-info">
        <div className="alert-series-name">{series.name}</div>
        <div className="alert-details">
          {isMissing ? (
            <>
              <span className="alert-status">Missing payment</span>
              <span className="separator">•</span>
              <span className="alert-date">Expected: {expectedDate}</span>
              <span className="separator">•</span>
              <span className="alert-overdue">{daysOverdue} days overdue</span>
            </>
          ) : (
            <>
              <span className="alert-status">Amount variance</span>
              <span className="separator">•</span>
              <span className="alert-expected">Expected: {formattedExpected}</span>
              <span className="separator">•</span>
              <span className="alert-actual">Actual: {formattedActual}</span>
              <span className="separator">•</span>
              <span className="alert-variance">Variance: {formattedVariance}</span>
            </>
          )}
        </div>
      </div>

      <div className="alert-actions">
        <button className="alert-button view" onClick={onViewSeries}>
          View Series
        </button>
        {isMissing && (
          <button className="alert-button skip" onClick={(e) => {
            e.stopPropagation();
            onSkip();
          }}>
            Mark Skipped
          </button>
        )}
      </div>
    </div>
  );
};

// Helper functions
function getStatusBadge(status: InstanceStatus | null) {
  if (!status) return null;

  const badges: Record<InstanceStatus, { icon: string; label: string; className: string }> = {
    matched: { icon: '✅', label: 'Paid', className: 'status-matched' },
    matched_manual: { icon: '✅', label: 'Paid', className: 'status-matched' },
    variance: { icon: '⚠️', label: 'Variance', className: 'status-variance' },
    missing: { icon: '🔴', label: 'Missing', className: 'status-missing' },
    upcoming: { icon: '📅', label: 'Upcoming', className: 'status-upcoming' },
    skipped: { icon: '⏭️', label: 'Skipped', className: 'status-skipped' }
  };

  const badge = badges[status];
  return (
    <span className={`status-badge ${badge.className}`}>
      {badge.icon} {badge.label}
    </span>
  );
}

function formatStatus(status: InstanceStatus | null): string {
  if (!status) return "No instances";

  const labels: Record<InstanceStatus, string> = {
    matched: "Paid on time",
    matched_manual: "Paid (manual)",
    variance: "Amount variance",
    missing: "Payment missing",
    upcoming: "Upcoming",
    skipped: "Skipped"
  };

  return labels[status] || status;
}

function formatFrequency(frequency: FrequencyPattern): string {
  switch (frequency.type) {
    case "daily":
      return frequency.interval === 1 ? "Daily" : `Every ${frequency.interval} days`;
    case "weekly":
      const days = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'];
      return frequency.interval === 1
        ? `Weekly on ${days[frequency.day_of_week]}`
        : `Every ${frequency.interval} weeks on ${days[frequency.day_of_week]}`;
    case "monthly":
      return frequency.interval === 1
        ? `Monthly on day ${frequency.day_of_month}`
        : `Every ${frequency.interval} months on day ${frequency.day_of_month}`;
    case "yearly":
      return `Yearly on ${frequency.month}/${frequency.day}`;
    case "custom":
      return `Custom (${frequency.dates.length} dates)`;
    default:
      return "Unknown";
  }
}

function getDefaultSeriesIcon(category: string): string {
  const icons: Record<string, string> = {
    subscriptions: "📺",
    bills: "📄",
    income: "💰",
    other: "🔄"
  };
  return icons[category] || "🔄";
}
```

---

## Create Series Modal

```tsx
const SeriesCreateModal: React.FC<{
  accounts: Account[];
  counterparties: Counterparty[];
  categories: string[];
  onClose: () => void;
  onCreate: (data: CreateSeriesInput) => Promise<void>;
  theme: "light" | "dark";
}> = ({ accounts, counterparties, categories, onClose, onCreate, theme }) => {
  const [formData, setFormData] = useState<CreateSeriesInput>({
    name: "",
    account_id: accounts[0]?.account_id || "",
    counterparty_id: counterparties[0]?.counterparty_id || "",
    expected_amount: 0,
    tolerance: 2.0,
    frequency: { type: "monthly", day_of_month: 1, interval: 1 },
    start_date: new Date().toISOString().split('T')[0],
    category: categories[0] || "subscriptions",
    notes: "",
    icon: "",
    color: "#3b82f6"
  });
  const [showRecurrenceDialog, setShowRecurrenceDialog] = useState(false);
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);
    setError(null);

    try {
      // Validation
      if (!formData.name.trim()) {
        throw new Error("Series name is required");
      }
      if (formData.expected_amount === 0) {
        throw new Error("Expected amount cannot be zero");
      }
      if (!formData.account_id) {
        throw new Error("Account is required");
      }
      if (!formData.counterparty_id) {
        throw new Error("Counterparty is required");
      }

      await onCreate(formData);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to create series");
      setSubmitting(false);
    }
  };

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content series-modal" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Create New Recurring Series</h3>
          <button className="close-button" onClick={onClose} aria-label="Close">×</button>
        </div>

        <form onSubmit={handleSubmit} className="series-form">
          {error && <div className="form-error">{error}</div>}

          <div className="form-group">
            <label htmlFor="name">Series Name *</label>
            <input
              type="text"
              id="name"
              value={formData.name}
              onChange={(e) => setFormData({ ...formData, name: e.target.value })}
              placeholder="e.g., Netflix Subscription, Monthly Rent, Salary"
              required
              autoFocus
            />
          </div>

          <div className="form-row">
            <div className="form-group">
              <label htmlFor="account">Account *</label>
              <select
                id="account"
                value={formData.account_id}
                onChange={(e) => setFormData({ ...formData, account_id: e.target.value })}
                required
              >
                {accounts.map(acc => (
                  <option key={acc.account_id} value={acc.account_id}>
                    {acc.name} ({acc.currency})
                  </option>
                ))}
              </select>
            </div>

            <div className="form-group">
              <label htmlFor="counterparty">Counterparty *</label>
              <select
                id="counterparty"
                value={formData.counterparty_id}
                onChange={(e) => setFormData({ ...formData, counterparty_id: e.target.value })}
                required
              >
                {counterparties.map(cp => (
                  <option key={cp.counterparty_id} value={cp.counterparty_id}>
                    {cp.canonical_name}
                  </option>
                ))}
              </select>
            </div>
          </div>

          <div className="form-row">
            <div className="form-group">
              <label htmlFor="expected_amount">Expected Amount *</label>
              <input
                type="number"
                id="expected_amount"
                value={formData.expected_amount}
                onChange={(e) => setFormData({ ...formData, expected_amount: parseFloat(e.target.value) || 0 })}
                step="0.01"
                placeholder="-15.99 for expenses, +2000.00 for income"
                required
              />
              <small>Negative for expenses, positive for income</small>
            </div>

            <div className="form-group">
              <label htmlFor="tolerance">Tolerance (±)</label>
              <input
                type="number"
                id="tolerance"
                value={formData.tolerance}
                onChange={(e) => setFormData({ ...formData, tolerance: parseFloat(e.target.value) || 0 })}
                step="0.01"
                min="0"
                placeholder="2.00"
              />
              <small>Amount variance allowed (e.g., 2.00 means ±$2)</small>
            </div>
          </div>

          <div className="form-group">
            <label>Recurrence Pattern *</label>
            <div className="recurrence-preview">
              <div className="recurrence-summary">
                {formatFrequency(formData.frequency)}
              </div>
              <button
                type="button"
                className="recurrence-edit-button"
                onClick={() => setShowRecurrenceDialog(true)}
              >
                Configure
              </button>
            </div>
          </div>

          <div className="form-row">
            <div className="form-group">
              <label htmlFor="start_date">Start Date *</label>
              <input
                type="date"
                id="start_date"
                value={formData.start_date}
                onChange={(e) => setFormData({ ...formData, start_date: e.target.value })}
                max={new Date().toISOString().split('T')[0]}
                required
              />
            </div>

            <div className="form-group">
              <label htmlFor="category">Category *</label>
              <select
                id="category"
                value={formData.category}
                onChange={(e) => setFormData({ ...formData, category: e.target.value })}
                required
              >
                {categories.map(cat => (
                  <option key={cat} value={cat}>
                    {cat.charAt(0).toUpperCase() + cat.slice(1)}
                  </option>
                ))}
              </select>
            </div>
          </div>

          <div className="form-group">
            <label htmlFor="notes">Notes (Optional)</label>
            <textarea
              id="notes"
              value={formData.notes}
              onChange={(e) => setFormData({ ...formData, notes: e.target.value })}
              rows={3}
              placeholder="Add notes about this series..."
            />
          </div>

          <div className="form-row">
            <div className="form-group">
              <label htmlFor="icon">Icon (Emoji)</label>
              <input
                type="text"
                id="icon"
                value={formData.icon}
                onChange={(e) => setFormData({ ...formData, icon: e.target.value })}
                placeholder="e.g., 📺"
                maxLength={2}
              />
            </div>

            <div className="form-group">
              <label htmlFor="color">Color</label>
              <input
                type="color"
                id="color"
                value={formData.color}
                onChange={(e) => setFormData({ ...formData, color: e.target.value })}
              />
            </div>
          </div>

          <div className="form-actions">
            <button type="button" onClick={onClose} className="cancel-button" disabled={submitting}>
              Cancel
            </button>
            <button type="submit" className="submit-button" disabled={submitting}>
              {submitting ? "Creating..." : "Create Series"}
            </button>
          </div>
        </form>

        {/* Recurrence Config Dialog */}
        {showRecurrenceDialog && (
          <RecurrenceConfigDialog
            frequency={formData.frequency}
            onClose={() => setShowRecurrenceDialog(false)}
            onConfirm={(frequency) => {
              setFormData({ ...formData, frequency });
              setShowRecurrenceDialog(false);
            }}
            theme={theme}
          />
        )}
      </div>
    </div>
  );
};
```

---

## Series Detail Dialog (with Instance History)

```tsx
const SeriesDetailDialog: React.FC<{
  series: Series;
  instances: SeriesInstance[];
  account?: Account;
  counterparty?: Counterparty;
  onClose: () => void;
  onInstanceLink: (seriesId: string, transactionId: string, force?: boolean) => Promise<SeriesInstance>;
  onInstanceUnlink: (instanceId: string) => Promise<void>;
  onInstanceSkip: (instanceId: string) => Promise<SeriesInstance>;
  theme: "light" | "dark";
}> = ({ series, instances, account, counterparty, onClose, onInstanceLink, onInstanceUnlink, onInstanceSkip, theme }) => {
  const sortedInstances = useMemo(() => {
    return [...instances].sort(
      (a, b) => new Date(b.expected_date).getTime() - new Date(a.expected_date).getTime()
    );
  }, [instances]);

  const formattedAmount = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: account?.currency || 'USD',
    minimumFractionDigits: 2
  }).format(Math.abs(series.expected_amount));

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content series-detail-modal" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <div>
            <h3>{series.name}</h3>
            <div className="series-detail-meta">
              {formatFrequency(series.frequency)} • {formattedAmount}
            </div>
          </div>
          <button className="close-button" onClick={onClose} aria-label="Close">×</button>
        </div>

        <div className="series-detail-body">
          {/* Series Info Section */}
          <div className="detail-section">
            <h4>Series Information</h4>
            <div className="detail-grid">
              <div className="detail-item">
                <span className="detail-label">Account:</span>
                <span className="detail-value">{account?.name || "Unknown"}</span>
              </div>
              <div className="detail-item">
                <span className="detail-label">Counterparty:</span>
                <span className="detail-value">{counterparty?.canonical_name || "Unknown"}</span>
              </div>
              <div className="detail-item">
                <span className="detail-label">Expected Amount:</span>
                <span className="detail-value">{formattedAmount}</span>
              </div>
              <div className="detail-item">
                <span className="detail-label">Tolerance:</span>
                <span className="detail-value">±{series.tolerance}</span>
              </div>
              <div className="detail-item">
                <span className="detail-label">Frequency:</span>
                <span className="detail-value">{formatFrequency(series.frequency)}</span>
              </div>
              <div className="detail-item">
                <span className="detail-label">Next Expected:</span>
                <span className="detail-value">
                  {series.next_expected_date
                    ? new Date(series.next_expected_date).toLocaleDateString()
                    : "N/A"}
                </span>
              </div>
              <div className="detail-item">
                <span className="detail-label">Category:</span>
                <span className="detail-value">{series.category}</span>
              </div>
              <div className="detail-item">
                <span className="detail-label">Status:</span>
                <span className="detail-value">{series.is_active ? "Active" : "Archived"}</span>
              </div>
            </div>
            {series.notes && (
              <div className="detail-notes">
                <span className="detail-label">Notes:</span>
                <p>{series.notes}</p>
              </div>
            )}
          </div>

          {/* Instance History Section */}
          <div className="detail-section">
            <h4>Instance History (Last 12 Months)</h4>
            {sortedInstances.length === 0 ? (
              <div className="empty-instances">
                No instances yet. Instances will appear here as expected dates occur.
              </div>
            ) : (
              <div className="instance-timeline">
                {sortedInstances.map(instance => (
                  <InstanceRow
                    key={instance.instance_id}
                    instance={instance}
                    currency={account?.currency || 'USD'}
                    onUnlink={() => onInstanceUnlink(instance.instance_id)}
                    onSkip={() => onInstanceSkip(instance.instance_id)}
                    theme={theme}
                  />
                ))}
              </div>
            )}
          </div>
        </div>
      </div>
    </div>
  );
};

// Instance row in timeline
const InstanceRow: React.FC<{
  instance: SeriesInstance;
  currency: string;
  onUnlink: () => void;
  onSkip: () => void;
  theme: "light" | "dark";
}> = ({ instance, currency, onUnlink, onSkip, theme }) => {
  const expectedDate = new Date(instance.expected_date).toLocaleDateString();
  const actualDate = instance.actual_date
    ? new Date(instance.actual_date).toLocaleDateString()
    : null;

  const formattedExpected = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
    minimumFractionDigits: 2
  }).format(Math.abs(instance.expected_amount));

  const formattedActual = instance.actual_amount
    ? new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency,
        minimumFractionDigits: 2
      }).format(Math.abs(instance.actual_amount))
    : null;

  const statusBadge = getStatusBadge(instance.status);

  return (
    <div className={`instance-row status-${instance.status}`}>
      <div className="instance-status-indicator">
        {statusBadge}
      </div>

      <div className="instance-info">
        <div className="instance-expected">
          <span className="instance-label">Expected:</span>
          <span className="instance-date">{expectedDate}</span>
          <span className="instance-amount">{formattedExpected}</span>
        </div>
        {instance.status !== "upcoming" && instance.status !== "missing" && (
          <div className="instance-actual">
            <span className="instance-label">Actual:</span>
            <span className="instance-date">{actualDate || "N/A"}</span>
            <span className="instance-amount">{formattedActual || "N/A"}</span>
          </div>
        )}
        {instance.variance !== null && instance.variance !== 0 && (
          <div className="instance-variance">
            <span className="instance-label">Variance:</span>
            <span className={`variance-value ${instance.variance > 0 ? 'positive' : 'negative'}`}>
              {new Intl.NumberFormat('en-US', {
                style: 'currency',
                currency,
                minimumFractionDigits: 2,
                signDisplay: 'always'
              }).format(instance.variance)}
            </span>
          </div>
        )}
      </div>

      <div className="instance-actions">
        {instance.status === "missing" && (
          <button className="instance-button skip" onClick={onSkip}>
            Mark Skipped
          </button>
        )}
        {instance.transaction_id && (
          <button className="instance-button unlink" onClick={onUnlink}>
            Unlink
          </button>
        )}
      </div>
    </div>
  );
};
```

---

## Visual Wireframes

### Main View with Variance Panel

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Recurring Payment Series                [+ New Series] [⚠️ Alerts (5)]     │
├────────────────────────────────────────────────────────────────────────────┤
│ [Search series...] [Group by Category ▼] [Sort by Name ▼] [↑] [☐ Archived]│
├──────────────────────────┬─────────────────────────────────────────────────┤
│ Requires Attention       │ SUBSCRIPTIONS (8)                               │
│ 5 issues                 │                                                 │
│                          │ [📺] Netflix            ✅ Paid                 │
│ 🔴 Netflix               │      Monthly on day 15 • $15.99                 │
│ Missing payment          │      Chase Credit • Netflix Inc                 │
│ Expected: Feb 15, 2024   │      Next: Mar 15          [✏️] [📦]            │
│ 3 days overdue           │                                                 │
│ [View Series][Skip]      │ [📺] Disney+            📅 Upcoming             │
│                          │      Monthly on day 1 • $13.99                  │
│ ⚠️ Spotify               │      Chase Credit • Disney                      │
│ Amount variance          │      Next: Mar 1           [✏️] [📦]            │
│ Expected: $9.99          │                                                 │
│ Actual: $10.99           │ BILLS (4)                                       │
│ Variance: +$1.00         │                                                 │
│ [View Series]            │ [📄] Electric Bill      ✅ Paid                 │
│                          │      Monthly on day 5 • $120.00 ± $20           │
│                          │      Chase Checking • Power Company             │
│                          │      Next: Mar 5           [✏️] [📦]            │
└──────────────────────────┴─────────────────────────────────────────────────┘
```

### Series Detail View (Instance History)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Netflix Subscription                                                    [×]│
│ Monthly on day 15 • $15.99                                                 │
├────────────────────────────────────────────────────────────────────────────┤
│ Series Information                                                         │
│ ┌────────────────────────────────────────────────────────────────────────┐│
│ │ Account:         Chase Credit Card                                     ││
│ │ Counterparty:    Netflix Inc                                           ││
│ │ Expected Amount: $15.99                                                ││
│ │ Tolerance:       ±$2.00                                                ││
│ │ Frequency:       Monthly on day 15                                     ││
│ │ Next Expected:   Mar 15, 2024                                          ││
│ │ Category:        subscriptions                                         ││
│ │ Status:          Active                                                ││
│ └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│ Instance History (Last 12 Months)                                          │
│ ┌────────────────────────────────────────────────────────────────────────┐│
│ │ 🔴 Missing                                                             ││
│ │ Expected: Feb 15, 2024 • $15.99                                        ││
│ │                                                 [Mark Skipped]         ││
│ │                                                                        ││
│ │ ✅ Paid                                                                ││
│ │ Expected: Jan 15, 2024 • $15.99                                        ││
│ │ Actual:   Jan 15, 2024 • $15.99                                        ││
│ │                                                 [Unlink]               ││
│ │                                                                        ││
│ │ ⚠️ Variance                                                            ││
│ │ Expected: Dec 15, 2023 • $15.99                                        ││
│ │ Actual:   Dec 16, 2023 • $17.99                                        ││
│ │ Variance: +$2.00                                                       ││
│ │                                                 [Unlink]               ││
│ │                                                                        ││
│ │ ✅ Paid                                                                ││
│ │ Expected: Nov 15, 2023 • $15.99                                        ││
│ │ Actual:   Nov 15, 2023 • $15.99                                        ││
│ │                                                 [Unlink]               ││
│ └────────────────────────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Multi-Domain Applicability

### Finance Domain
```tsx
<SeriesManager
  series={financeSeries}
  instances={financeInstances}
  accounts={financeAccounts}
  counterparties={financeCounterparties}
  categories={["subscriptions", "bills", "income", "other"]}
  currentUserId="user_123"
  onSeriesCreate={createSeries}
  onSeriesUpdate={updateSeries}
  onSeriesArchive={archiveSeries}
  onSeriesUnarchive={unarchiveSeries}
  onInstanceLink={linkInstance}
  onInstanceUnlink={unlinkInstance}
  onInstanceSkip={skipInstance}
  groupBy="category"
/>
// Series: Netflix ($15.99/mo), Rent ($1200/mo), Salary (+$5000/mo bi-weekly)
// Use case: Track subscriptions, bills, and income
```

### Healthcare Domain
```tsx
<SeriesManager
  series={healthcareSeries}
  instances={healthcareInstances}
  accounts={healthcareAccounts}
  counterparties={healthcareProviders}
  categories={["insurance", "prescriptions", "appointments", "other"]}
  currentUserId="patient_456"
  onSeriesCreate={createHealthSeries}
  onSeriesUpdate={updateHealthSeries}
  onSeriesArchive={archiveHealthSeries}
  onSeriesUnarchive={unarchiveHealthSeries}
  onInstanceLink={linkHealthInstance}
  onInstanceUnlink={unlinkHealthInstance}
  onInstanceSkip={skipHealthInstance}
  groupBy="category"
/>
// Series: Insurance Premium ($350/mo), Blood Pressure Medication ($25/mo), Physical Therapy (2x/week)
// Use case: Track recurring healthcare costs and appointments
```

### Legal Domain
```tsx
<SeriesManager
  series={legalSeries}
  instances={legalInstances}
  accounts={legalAccounts}
  counterparties={legalEntities}
  categories={["retainers", "fees", "disbursements", "other"]}
  currentUserId="attorney_789"
  onSeriesCreate={createLegalSeries}
  onSeriesUpdate={updateLegalSeries}
  onSeriesArchive={archiveLegalSeries}
  onSeriesUnarchive={unarchiveLegalSeries}
  onInstanceLink={linkLegalInstance}
  onInstanceUnlink={unlinkLegalInstance}
  onInstanceSkip={skipLegalInstance}
  groupBy="category"
/>
// Series: Monthly Retainer ($2500/mo), Court Filing Fees ($350/quarter), Lease Payment ($3000/mo)
// Use case: Track recurring legal fees and obligations
```

### Research Domain (RSRCH - Utilitario)
```tsx
<SeriesManager
  series={rsrchSeries}
  instances={rsrchInstances}
  accounts={rsrchAccounts}
  counterparties={rsrchProviders}
  categories={["api_subscriptions", "scraping_services", "infrastructure", "other"]}
  currentUserId="rsrch_analyst_101"
  onSeriesCreate={createRSRCHSeries}
  onSeriesUpdate={updateRSRCHSeries}
  onSeriesArchive={archiveRSRCHSeries}
  onSeriesUnarchive={unarchiveRSRCHSeries}
  onInstanceLink={linkRSRCHInstance}
  onInstanceUnlink={unlinkRSRCHInstance}
  onInstanceSkip={skipRSRCHInstance}
  groupBy="category"
/>
// Series: Twitter API Subscription ($5000/month), Web Scraping Service ($2500/month), Cloud Infrastructure ($800/month)
// Use case: Track recurring API costs and web scraping service expenses
```

---

## User Interactions

### 1. Create New Series
**User Flow:**
1. User clicks "+ New Series" button (or presses "N" keyboard shortcut)
2. SeriesCreateModal opens
3. User fills in series details:
   - Name (e.g., "Netflix Subscription")
   - Account (dropdown)
   - Counterparty (dropdown)
   - Expected Amount (-15.99)
   - Tolerance (±2.00)
   - Start Date (date picker)
   - Category (dropdown)
4. User clicks "Configure" to open RecurrenceConfigDialog
5. User selects frequency pattern (Monthly on day 15)
6. User confirms frequency pattern
7. User submits form
8. System validates and creates series
9. Series appears in list with "Upcoming" status

### 2. View Series Instance History
**User Flow:**
1. User clicks on series card in list
2. SeriesDetailDialog opens showing:
   - Series information (account, counterparty, frequency, etc.)
   - Instance history (last 12 months)
3. User sees timeline of expected vs actual:
   - ✅ Paid instances (green)
   - ⚠️ Variance instances (yellow)
   - 🔴 Missing instances (red)
   - 📅 Upcoming instances (blue)
4. User can unlink instances or mark missing ones as skipped

### 3. Handle Missing Payment Alert
**User Flow:**
1. User sees variance panel with missing payment alert
2. Alert shows: "Netflix - Missing payment - Expected: Feb 15, 2024 - 3 days overdue"
3. User clicks "View Series" to see instance history
4. User investigates:
   - Option A: Payment actually occurred, manually link transaction
   - Option B: Payment genuinely missing, mark as skipped or wait
   - Option C: Series no longer needed, archive series

### 4. Edit Series
**User Flow:**
1. User clicks edit button (✏️) on series card
2. SeriesEditModal opens with pre-filled data
3. User can edit:
   - Name
   - Expected Amount
   - Tolerance
   - Frequency
   - Category
   - Notes
4. **Cannot edit:** Account, Counterparty (immutable to preserve instance integrity)
5. User submits changes
6. System validates and updates series

### 5. Archive Series
**User Flow:**
1. User clicks archive button (📦) on series card
2. ArchiveConfirmModal opens showing:
   - Series name
   - Number of instances that will be preserved
   - End date selector (defaults to today)
3. User confirms archive
4. Series is soft-deleted (is_active = false, end_date set)
5. Instance history is preserved
6. Series hidden from main list unless "Show Archived" is checked

---

## State Management

### Local State (Component)
```typescript
interface SeriesManagerState {
  filter: SeriesFilter;
  sortBy: "name" | "next_expected" | "category" | "updated_at";
  sortDirection: "asc" | "desc";
  showCreateModal: boolean;
  editingSeries: Series | null;
  archiveConfirm: Series | null;
  viewingSeriesId: string | null;
  showVariances: boolean;
}
```

### Derived State (Computed)
```typescript
// Series status (from latest instance)
const seriesStatus = useMemo(() => {
  return getSeriesStatus(seriesId);
}, [instances, seriesId]);

// Filtered series
const filteredSeries = useMemo(() => {
  // Apply search, account, category, status filters
}, [series, filter]);

// Sorted series
const sortedSeries = useMemo(() => {
  // Apply sort by name, next_expected, category, updated_at
}, [filteredSeries, sortBy, sortDirection]);

// Grouped series
const groupedSeries = useMemo(() => {
  // Group by category, account, status, or none
}, [sortedSeries, groupBy]);

// Variance instances (missing + variance)
const varianceInstances = useMemo(() => {
  // Collect all instances with status "missing" or "variance"
  // Sort by expected date (most recent first)
}, [instances, series]);
```

---

## API Integration

### GET /api/series
```typescript
// Fetch all series for user
const response = await fetch('/api/series?is_active=true', {
  headers: { 'Authorization': `Bearer ${token}` }
});
const { series } = await response.json();
```

### POST /api/series
```typescript
// Create new series
const response = await fetch('/api/series', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(createSeriesInput)
});
const newSeries = await response.json();
```

### PUT /api/series/{series_id}
```typescript
// Update series
const response = await fetch(`/api/series/${seriesId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(updates)
});
const updatedSeries = await response.json();
```

### POST /api/series/{series_id}/archive
```typescript
// Archive series
const response = await fetch(`/api/series/${seriesId}/archive`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ end_date: '2024-02-20' })
});
```

### GET /api/series/{series_id}/instances
```typescript
// Fetch instances for series (last 12 months)
const response = await fetch(`/api/series/${seriesId}/instances?limit=12`, {
  headers: { 'Authorization': `Bearer ${token}` }
});
const { instances } = await response.json();
```

### POST /api/series/{series_id}/link
```typescript
// Manually link transaction to series
const response = await fetch(`/api/series/${seriesId}/link`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ transaction_id: txnId, force: false })
});
const instance = await response.json();
```

### POST /api/instances/{instance_id}/skip
```typescript
// Mark instance as skipped
const response = await fetch(`/api/instances/${instanceId}/skip`, {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` }
});
const updatedInstance = await response.json();
```

---

## Accessibility

```tsx
// Main container
<div
  className="series-manager"
  role="region"
  aria-label="Recurring payment series manager"
>
  {/* Series card */}
  <div
    className="series-card"
    role="button"
    tabIndex={0}
    onClick={onView}
    onKeyDown={(e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        onView();
      }
    }}
    aria-label={`${series.name}, ${formatFrequency(series.frequency)}, ${formattedAmount}, Status: ${formatStatus(status)}`}
  >
    {/* Card content */}
  </div>

  {/* Search input */}
  <input
    type="text"
    placeholder="Search series by name..."
    aria-label="Search series"
  />

  {/* Filter selects */}
  <select aria-label="Group by">
    <option value="none">No Grouping</option>
    <option value="category">Group by Category</option>
  </select>

  {/* Action buttons */}
  <button
    className="action-button edit"
    onClick={onEdit}
    title="Edit series"
    aria-label={`Edit ${series.name}`}
  >
    ✏️
  </button>

  {/* Status badges */}
  <span
    className="status-badge status-matched"
    aria-label="Payment matched and paid on time"
  >
    ✅ Paid
  </span>

  {/* Empty state */}
  <div className="empty-state" role="status">
    <div className="empty-message">
      No recurring payment series yet. Create your first series to track subscriptions, bills, and recurring income.
    </div>
  </div>
}
```

**ARIA Labels:**
- All interactive elements have aria-label or aria-labelledby
- Status badges have descriptive aria-labels
- Empty states have role="status"
- Modals have role="dialog" and aria-labelledby pointing to title

**Keyboard Navigation:**
- Tab: Navigate through interactive elements
- Enter/Space: Activate buttons and cards
- Escape: Close modals
- N: Open create series modal (global shortcut)
- ↑↓: Navigate through series list (future enhancement)

**Screen Reader Support:**
- Series cards announce: name, frequency, amount, and status
- Variance alerts announce: type (missing/variance), series name, details
- Loading states announce "Loading series..."
- Error states announce error message

---

## Related Components

**Uses (Dependencies):**
- RecurrenceConfigDialog (for frequency pattern UI)
- AccountSelector (in create/edit modals)
- CounterpartySelector (in create/edit modals)

**Used By:**
- Dashboard (shows series status widget)
- Transaction Detail (links to series via SeriesSelector)
- Settings Page (series management)

**Related Components:**
- SeriesSelector (dropdown for selecting series to link)
- RecurrenceConfigDialog (frequency pattern configuration)
- AccountManager (manage accounts referenced by series)
- CounterpartyManager (manage counterparties referenced by series)

---

## Edge Cases

### EC1: Last Day of Month
**Scenario:** Series set to monthly on day 31, but February only has 28/29 days
**Handling:** RecurrenceEngine automatically adjusts to last day of month
- Jan 31 → Feb 28/29 → Mar 31 → Apr 30 → May 31
**UI Indication:** Show warning in RecurrenceConfigDialog: "Day 31 will adjust to last day of month in shorter months"

### EC2: Multiple Transactions Match Same Expected Instance
**Scenario:** Two transactions (pending and posted) match same expected date
**Handling:** InstanceTracker links to first chronologically, ignores duplicate
**UI Indication:** User can manually unlink and relink if incorrect match

### EC3: Amount Variance Outside Tolerance
**Scenario:** Expected $1200 ± $50, actual $1300 (outside tolerance)
**Handling:** Auto-link fails, instance remains "missing", user gets variance alert
**UI Actions:**
- Option A: Update series expected_amount to $1300
- Option B: Force link with variance flag
- Option C: Ignore (leave as missing)

### EC4: Series Created After Transactions Exist
**Scenario:** User creates series on Dec 31 for Netflix, but has 12 months of past Netflix transactions
**Handling:** System only auto-links future transactions
**UI Action:** User can manually link past transactions via Transaction List

### EC5: Series Archived Mid-Year
**Scenario:** Gym membership cancelled in June, series archived
**Handling:** end_date = Jun 30, is_active = false, instances Jan-Jun preserved
**UI Display:** Series hidden unless "Show Archived" checked, instance history still viewable

---

## Performance Considerations

**Lazy Loading:**
- Load instance history only when SeriesDetailDialog is opened
- Load variance instances only when variance panel is expanded

**Memoization:**
- Filter, sort, group operations are memoized to avoid re-computation
- Series status derived from latest instance (memoized)

**Debouncing:**
- Search input debounced (300ms) to reduce filter computations

**Pagination (Future):**
- If user has >100 series, implement virtual scrolling or pagination
- Load instances in chunks (12 months at a time)

**Caching:**
- Cache series list for 5 minutes
- Invalidate cache on create/update/archive

---

## Testing Strategy

### Unit Tests
```typescript
// Filter tests
test('filters series by search query', () => {
  const result = filterSeries(series, { searchQuery: 'netflix' });
  expect(result).toHaveLength(1);
  expect(result[0].name).toBe('Netflix');
});

test('filters series by account', () => {
  const result = filterSeries(series, { accountIds: ['acc_1'] });
  expect(result.every(s => s.account_id === 'acc_1')).toBe(true);
});

test('filters series by status', () => {
  const result = filterSeries(series, { statuses: ['missing'] });
  expect(result.every(s => getSeriesStatus(s.series_id) === 'missing')).toBe(true);
});

// Sort tests
test('sorts series by name ascending', () => {
  const result = sortSeries(series, 'name', 'asc');
  expect(result[0].name).toBe('Disney+');
  expect(result[1].name).toBe('Netflix');
});

test('sorts series by next_expected date', () => {
  const result = sortSeries(series, 'next_expected', 'asc');
  const dates = result.map(s => s.next_expected_date);
  expect(dates).toEqual([...dates].sort());
});

// Group tests
test('groups series by category', () => {
  const result = groupSeries(series, 'category');
  expect(result).toHaveProperty('subscriptions');
  expect(result).toHaveProperty('bills');
});

// Variance detection tests
test('detects missing instances', () => {
  const variances = getVarianceInstances(series, instances);
  const missing = variances.filter(v => v.instance.status === 'missing');
  expect(missing.length).toBeGreaterThan(0);
});
```

### Integration Tests
```typescript
test('creates series and shows in list', async () => {
  render(<SeriesManager {...props} />);

  fireEvent.click(screen.getByText('+ New Series'));

  fireEvent.change(screen.getByLabelText('Series Name'), {
    target: { value: 'Netflix' }
  });
  // ... fill other fields

  fireEvent.click(screen.getByText('Create Series'));

  await waitFor(() => {
    expect(screen.getByText('Netflix')).toBeInTheDocument();
  });
});

test('archives series and hides from list', async () => {
  render(<SeriesManager {...props} />);

  fireEvent.click(screen.getByTitle('Archive series'));
  fireEvent.click(screen.getByText('Confirm Archive'));

  await waitFor(() => {
    expect(screen.queryByText('Netflix')).not.toBeInTheDocument();
  });
});

test('filters series by search query', () => {
  render(<SeriesManager {...props} />);

  fireEvent.change(screen.getByPlaceholderText('Search series...'), {
    target: { value: 'netflix' }
  });

  expect(screen.getByText('Netflix')).toBeInTheDocument();
  expect(screen.queryByText('Disney+')).not.toBeInTheDocument();
});
```

---

## Summary

SeriesManager provides a comprehensive UI for managing recurring payment series across all domains. Key features include:

✅ **Full CRUD operations** (create, edit, archive, restore)
✅ **Visual status indicators** (✅ Paid, ⚠️ Variance, 🔴 Missing, 📅 Upcoming)
✅ **Variance alerts panel** highlighting issues requiring attention
✅ **Instance history view** showing 12 months of expected vs actual
✅ **Search, filter, and group** capabilities
✅ **Keyboard shortcuts** for power users (N=new, ESC=close)
✅ **Responsive design** (desktop/mobile)
✅ **Multi-domain support** (finance, healthcare, legal, research)
✅ **Accessible** (ARIA labels, keyboard nav, screen reader support)

This component is the primary interface for users to manage their recurring payment expectations and monitor series health.
