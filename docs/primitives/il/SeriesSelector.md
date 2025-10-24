# SeriesSelector (IL Component)

## Definition

**SeriesSelector** is a reusable dropdown component for selecting a recurring payment series to link to a transaction. It provides search, grouping by category, inline series creation, and displays next expected date and amount for each series option.

**Problem it solves:**
- No standardized UI for selecting series when linking transactions
- Hard to find the right series from a long list
- No context about series details (next expected date, amount) when selecting
- No quick way to create a new series inline
- Inconsistent series selection across different features

**Solution:**
- Searchable dropdown with fuzzy matching
- Grouped by category (subscriptions, bills, income, other)
- Shows next expected date and amount for each option
- "Create new series" inline action
- Optional filtering by account
- Optional display of transaction count per series
- Keyboard navigation (↑↓ select, Enter confirm, Esc close)
- Loading and empty states
- Used in Transaction Detail and Manual Link Dialog

---

## Interface Contract

```typescript
interface SeriesSelectorProps {
  // Data
  series: Series[];  // Available series to select from
  accounts?: Account[];  // For display and filtering
  transactionCounts?: Record<string, number>;  // Transaction count per series (optional)

  // Current selection
  value: string | null;  // Currently selected series_id
  onChange: (seriesId: string | null) => void;  // Callback when selection changes

  // Filtering
  accountId?: string;  // Filter series by account (optional)
  excludeArchived?: boolean;  // Exclude archived series (default: true)
  categoryFilter?: string[];  // Filter by specific categories

  // UI states
  loading?: boolean;
  disabled?: boolean;
  error?: string | null;

  // Actions
  onCreateNew?: () => void;  // Callback for "Create new series" action
  allowCreateNew?: boolean;  // Show "Create new series" option (default: true)

  // Display options
  placeholder?: string;
  showTransactionCount?: boolean;  // Show transaction count per series (default: false)
  showNextExpected?: boolean;  // Show next expected date/amount (default: true)
  groupByCategory?: boolean;  // Group series by category (default: true)

  // Customization
  theme?: "light" | "dark";
  size?: "small" | "medium" | "large";
}

interface Series {
  series_id: string;
  name: string;
  account_id: string;
  counterparty_id: string;
  expected_amount: number;
  frequency: FrequencyPattern;
  next_expected_date: string | null;
  category: string;
  is_active: boolean;
}

interface Account {
  account_id: string;
  name: string;
  currency: string;
}

type FrequencyPattern =
  | { type: "daily"; interval: number }
  | { type: "weekly"; day_of_week: 0-6; interval: number }
  | { type: "monthly"; day_of_month: 1-31; interval: number }
  | { type: "yearly"; month: 1-12; day: 1-31 }
  | { type: "custom"; dates: string[] };
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useRef, useEffect } from 'react';

export const SeriesSelector: React.FC<SeriesSelectorProps> = ({
  series,
  accounts = [],
  transactionCounts = {},
  value,
  onChange,
  accountId,
  excludeArchived = true,
  categoryFilter,
  loading = false,
  disabled = false,
  error = null,
  onCreateNew,
  allowCreateNew = true,
  placeholder = "Select a series...",
  showTransactionCount = false,
  showNextExpected = true,
  groupByCategory = true,
  theme = "light",
  size = "medium"
}) => {
  const [isOpen, setIsOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState("");
  const [highlightedIndex, setHighlightedIndex] = useState(-1);
  const dropdownRef = useRef<HTMLDivElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  // Get selected series
  const selectedSeries = useMemo(() => {
    return series.find(s => s.series_id === value) || null;
  }, [series, value]);

  // Filter series
  const filteredSeries = useMemo(() => {
    let result = series;

    // Filter by archived status
    if (excludeArchived) {
      result = result.filter(s => s.is_active);
    }

    // Filter by account
    if (accountId) {
      result = result.filter(s => s.account_id === accountId);
    }

    // Filter by category
    if (categoryFilter && categoryFilter.length > 0) {
      result = result.filter(s => categoryFilter.includes(s.category));
    }

    // Filter by search query
    if (searchQuery.trim()) {
      const query = searchQuery.toLowerCase();
      result = result.filter(s =>
        s.name.toLowerCase().includes(query) ||
        s.category.toLowerCase().includes(query)
      );
    }

    return result;
  }, [series, excludeArchived, accountId, categoryFilter, searchQuery]);

  // Group series by category
  const groupedSeries = useMemo(() => {
    if (!groupByCategory) {
      return { "All": filteredSeries };
    }

    const groups: Record<string, Series[]> = {};
    filteredSeries.forEach(s => {
      const categoryLabel = formatCategory(s.category);
      if (!groups[categoryLabel]) groups[categoryLabel] = [];
      groups[categoryLabel].push(s);
    });

    // Sort groups: subscriptions, bills, income, other
    const sortedGroups: Record<string, Series[]> = {};
    const order = ["Subscriptions", "Bills", "Income", "Other"];
    order.forEach(cat => {
      if (groups[cat]) sortedGroups[cat] = groups[cat];
    });

    return sortedGroups;
  }, [filteredSeries, groupByCategory]);

  // Flatten grouped series for keyboard navigation
  const flattenedSeries = useMemo(() => {
    return Object.values(groupedSeries).flat();
  }, [groupedSeries]);

  // Handle click outside to close dropdown
  useEffect(() => {
    const handleClickOutside = (event: MouseEvent) => {
      if (dropdownRef.current && !dropdownRef.current.contains(event.target as Node)) {
        setIsOpen(false);
        setSearchQuery("");
        setHighlightedIndex(-1);
      }
    };

    document.addEventListener('mousedown', handleClickOutside);
    return () => document.removeEventListener('mousedown', handleClickOutside);
  }, []);

  // Keyboard navigation
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (!isOpen && (e.key === 'Enter' || e.key === ' ')) {
      e.preventDefault();
      setIsOpen(true);
      return;
    }

    if (!isOpen) return;

    switch (e.key) {
      case 'Escape':
        e.preventDefault();
        setIsOpen(false);
        setSearchQuery("");
        setHighlightedIndex(-1);
        break;

      case 'ArrowDown':
        e.preventDefault();
        setHighlightedIndex(prev =>
          prev < flattenedSeries.length - 1 ? prev + 1 : prev
        );
        break;

      case 'ArrowUp':
        e.preventDefault();
        setHighlightedIndex(prev => (prev > 0 ? prev - 1 : prev));
        break;

      case 'Enter':
        e.preventDefault();
        if (highlightedIndex >= 0 && highlightedIndex < flattenedSeries.length) {
          const selectedSeries = flattenedSeries[highlightedIndex];
          handleSelect(selectedSeries.series_id);
        } else if (allowCreateNew && flattenedSeries.length === 0 && searchQuery.trim()) {
          handleCreateNew();
        }
        break;
    }
  };

  const handleSelect = (seriesId: string) => {
    onChange(seriesId);
    setIsOpen(false);
    setSearchQuery("");
    setHighlightedIndex(-1);
  };

  const handleClear = (e: React.MouseEvent) => {
    e.stopPropagation();
    onChange(null);
    setSearchQuery("");
  };

  const handleCreateNew = () => {
    if (onCreateNew) {
      onCreateNew();
      setIsOpen(false);
      setSearchQuery("");
    }
  };

  // Get account for currency formatting
  const getAccount = (accountId: string) => {
    return accounts.find(a => a.account_id === accountId);
  };

  // Loading state
  if (loading) {
    return (
      <div className={`series-selector theme-${theme} size-${size} loading`}>
        <div className="selector-skeleton"></div>
      </div>
    );
  }

  return (
    <div
      ref={dropdownRef}
      className={`series-selector theme-${theme} size-${size} ${isOpen ? 'open' : ''} ${disabled ? 'disabled' : ''}`}
      onKeyDown={handleKeyDown}
    >
      {/* Trigger */}
      <div
        className="selector-trigger"
        onClick={() => !disabled && setIsOpen(!isOpen)}
        role="button"
        tabIndex={disabled ? -1 : 0}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-disabled={disabled}
        aria-label="Select series"
      >
        {selectedSeries ? (
          <div className="selected-series">
            <span className="selected-name">{selectedSeries.name}</span>
            {showNextExpected && selectedSeries.next_expected_date && (
              <span className="selected-meta">
                {formatFrequency(selectedSeries.frequency)} • {formatAmount(selectedSeries.expected_amount, getAccount(selectedSeries.account_id))}
              </span>
            )}
          </div>
        ) : (
          <span className="placeholder">{placeholder}</span>
        )}

        <div className="selector-actions">
          {selectedSeries && !disabled && (
            <button
              className="clear-button"
              onClick={handleClear}
              aria-label="Clear selection"
              tabIndex={-1}
            >
              ×
            </button>
          )}
          <span className="dropdown-arrow">{isOpen ? '▲' : '▼'}</span>
        </div>
      </div>

      {/* Dropdown */}
      {isOpen && (
        <div className="selector-dropdown" role="listbox">
          {/* Search */}
          <div className="dropdown-search">
            <input
              ref={inputRef}
              type="text"
              placeholder="Search series..."
              value={searchQuery}
              onChange={(e) => {
                setSearchQuery(e.target.value);
                setHighlightedIndex(-1);
              }}
              autoFocus
              aria-label="Search series"
            />
          </div>

          {/* Options */}
          <div className="dropdown-options">
            {Object.keys(groupedSeries).length === 0 ? (
              <div className="empty-state">
                {searchQuery.trim() ? (
                  <>
                    <div className="empty-message">
                      No series found matching "{searchQuery}"
                    </div>
                    {allowCreateNew && onCreateNew && (
                      <button
                        className="create-new-button"
                        onClick={handleCreateNew}
                      >
                        + Create new series "{searchQuery}"
                      </button>
                    )}
                  </>
                ) : (
                  <>
                    <div className="empty-message">
                      No series available
                    </div>
                    {allowCreateNew && onCreateNew && (
                      <button
                        className="create-new-button"
                        onClick={handleCreateNew}
                      >
                        + Create new series
                      </button>
                    )}
                  </>
                )}
              </div>
            ) : (
              <>
                {Object.entries(groupedSeries).map(([categoryLabel, categorySeries]) => (
                  <div key={categoryLabel} className="option-group">
                    {groupByCategory && (
                      <div className="group-label">{categoryLabel}</div>
                    )}
                    {categorySeries.map((s, idx) => {
                      const globalIndex = flattenedSeries.findIndex(fs => fs.series_id === s.series_id);
                      const isHighlighted = globalIndex === highlightedIndex;
                      const isSelected = s.series_id === value;

                      return (
                        <SeriesOption
                          key={s.series_id}
                          series={s}
                          account={getAccount(s.account_id)}
                          transactionCount={transactionCounts[s.series_id]}
                          isHighlighted={isHighlighted}
                          isSelected={isSelected}
                          showNextExpected={showNextExpected}
                          showTransactionCount={showTransactionCount}
                          onClick={() => handleSelect(s.series_id)}
                        />
                      );
                    })}
                  </div>
                ))}

                {/* Create New Option */}
                {allowCreateNew && onCreateNew && (
                  <div className="option-group">
                    <button
                      className="create-new-option"
                      onClick={handleCreateNew}
                    >
                      + Create new series
                    </button>
                  </div>
                )}
              </>
            )}
          </div>
        </div>
      )}

      {/* Error */}
      {error && <div className="selector-error">{error}</div>}
    </div>
  );
};

// Individual series option
const SeriesOption: React.FC<{
  series: Series;
  account?: Account;
  transactionCount?: number;
  isHighlighted: boolean;
  isSelected: boolean;
  showNextExpected: boolean;
  showTransactionCount: boolean;
  onClick: () => void;
}> = ({
  series,
  account,
  transactionCount = 0,
  isHighlighted,
  isSelected,
  showNextExpected,
  showTransactionCount,
  onClick
}) => {
  const formattedAmount = formatAmount(series.expected_amount, account);
  const nextExpected = series.next_expected_date
    ? new Date(series.next_expected_date).toLocaleDateString('en-US', {
        month: 'short',
        day: 'numeric'
      })
    : null;

  return (
    <div
      className={`series-option ${isHighlighted ? 'highlighted' : ''} ${isSelected ? 'selected' : ''}`}
      onClick={onClick}
      role="option"
      aria-selected={isSelected}
    >
      <div className="option-main">
        <div className="option-name">{series.name}</div>
        <div className="option-meta">
          <span className="option-frequency">{formatFrequency(series.frequency)}</span>
          <span className="separator">•</span>
          <span className="option-amount">{formattedAmount}</span>
        </div>
      </div>

      <div className="option-right">
        {showNextExpected && nextExpected && (
          <div className="option-next">
            <div className="next-label">Next</div>
            <div className="next-date">{nextExpected}</div>
          </div>
        )}
        {showTransactionCount && transactionCount > 0 && (
          <div className="option-count">
            {transactionCount} txn{transactionCount > 1 ? 's' : ''}
          </div>
        )}
        {isSelected && (
          <div className="option-checkmark">✓</div>
        )}
      </div>
    </div>
  );
};

// Helper functions
function formatCategory(category: string): string {
  const labels: Record<string, string> = {
    subscriptions: "Subscriptions",
    bills: "Bills",
    income: "Income",
    other: "Other"
  };
  return labels[category] || category;
}

function formatFrequency(frequency: FrequencyPattern): string {
  switch (frequency.type) {
    case "daily":
      return frequency.interval === 1 ? "Daily" : `Every ${frequency.interval}d`;
    case "weekly":
      return frequency.interval === 1 ? "Weekly" : `Every ${frequency.interval}w`;
    case "monthly":
      return frequency.interval === 1 ? "Monthly" : `Every ${frequency.interval}m`;
    case "yearly":
      return "Yearly";
    case "custom":
      return "Custom";
    default:
      return "Unknown";
  }
}

function formatAmount(amount: number, account?: Account): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: account?.currency || 'USD',
    minimumFractionDigits: 2
  }).format(Math.abs(amount));
}
```

---

## Visual Wireframes

### Closed State (No Selection)
```
┌──────────────────────────────────────────────────┐
│ Select a series...                            ▼  │
└──────────────────────────────────────────────────┘
```

### Closed State (With Selection)
```
┌──────────────────────────────────────────────────┐
│ Netflix Subscription                      [×] ▼  │
│ Monthly • $15.99                                 │
└──────────────────────────────────────────────────┘
```

### Open State (With Groups)
```
┌──────────────────────────────────────────────────┐
│ Netflix Subscription                      [×] ▲  │
│ Monthly • $15.99                                 │
├──────────────────────────────────────────────────┤
│ [Search series...]                               │
├──────────────────────────────────────────────────┤
│ SUBSCRIPTIONS                                    │
│ ┌────────────────────────────────────────────┐   │
│ │ Netflix Subscription            Next: Feb 15│   │
│ │ Monthly • $15.99                           │✓ │
│ └────────────────────────────────────────────┘   │
│ ┌────────────────────────────────────────────┐   │
│ │ Disney+                         Next: Mar 1 │   │
│ │ Monthly • $13.99                            │   │
│ └────────────────────────────────────────────┘   │
│ ┌────────────────────────────────────────────┐   │
│ │ Spotify Premium                Next: Feb 20 │   │
│ │ Monthly • $9.99                             │   │
│ └────────────────────────────────────────────┘   │
│                                                  │
│ BILLS                                            │
│ ┌────────────────────────────────────────────┐   │
│ │ Electric Bill                  Next: Mar 5  │   │
│ │ Monthly • $120.00                           │   │
│ └────────────────────────────────────────────┘   │
│ ┌────────────────────────────────────────────┐   │
│ │ Internet Service               Next: Feb 28 │   │
│ │ Monthly • $79.99                            │   │
│ └────────────────────────────────────────────┘   │
│                                                  │
│ [+ Create new series]                            │
└──────────────────────────────────────────────────┘
```

### Search State (Results Found)
```
┌──────────────────────────────────────────────────┐
│ [netflix                                    ]    │
├──────────────────────────────────────────────────┤
│ SUBSCRIPTIONS                                    │
│ ┌────────────────────────────────────────────┐   │
│ │ Netflix Subscription            Next: Feb 15│   │
│ │ Monthly • $15.99                           │✓ │
│ └────────────────────────────────────────────┘   │
│                                                  │
│ [+ Create new series]                            │
└──────────────────────────────────────────────────┘
```

### Search State (No Results)
```
┌──────────────────────────────────────────────────┐
│ [hulu                                       ]    │
├──────────────────────────────────────────────────┤
│ No series found matching "hulu"                  │
│                                                  │
│ [+ Create new series "hulu"]                     │
└──────────────────────────────────────────────────┘
```

---

## Usage Examples

### Transaction Detail Page
```tsx
// User viewing transaction and wants to link to series
<div className="transaction-detail">
  <h3>Transaction Details</h3>

  <div className="field">
    <label>Date</label>
    <div>Feb 15, 2024</div>
  </div>

  <div className="field">
    <label>Counterparty</label>
    <div>Netflix Inc</div>
  </div>

  <div className="field">
    <label>Amount</label>
    <div>-$15.99</div>
  </div>

  <div className="field">
    <label>Link to Series</label>
    <SeriesSelector
      series={allSeries}
      accounts={accounts}
      value={transaction.series_id}
      onChange={async (seriesId) => {
        if (seriesId) {
          await linkTransactionToSeries(transaction.id, seriesId);
        } else {
          await unlinkTransactionFromSeries(transaction.id);
        }
      }}
      accountId={transaction.account_id}
      onCreateNew={() => setShowCreateSeriesModal(true)}
      showNextExpected={true}
      showTransactionCount={false}
      placeholder="Select series to link..."
    />
  </div>
</div>
```

### Manual Link Dialog
```tsx
// User manually linking transaction to series (from SeriesManager)
<dialog open>
  <h3>Link Transaction to Series</h3>

  <div className="dialog-body">
    <div className="transaction-preview">
      <div>Netflix Inc</div>
      <div>Feb 15, 2024</div>
      <div>-$15.99</div>
    </div>

    <div className="field">
      <label>Select Series *</label>
      <SeriesSelector
        series={allSeries}
        accounts={accounts}
        value={selectedSeriesId}
        onChange={setSelectedSeriesId}
        accountId={transaction.account_id}
        onCreateNew={() => {
          setShowCreateSeriesModal(true);
        }}
        showNextExpected={true}
        showTransactionCount={true}
        placeholder="Choose a series..."
      />
    </div>

    <div className="field">
      <label>
        <input type="checkbox" checked={forceLink} onChange={(e) => setForceLink(e.target.checked)} />
        Force link (ignore tolerance)
      </label>
    </div>
  </div>

  <div className="dialog-actions">
    <button onClick={onCancel}>Cancel</button>
    <button onClick={() => onConfirm(selectedSeriesId, forceLink)} disabled={!selectedSeriesId}>
      Link Transaction
    </button>
  </div>
</dialog>
```

### Filtered by Account
```tsx
// Only show series for specific account
<SeriesSelector
  series={allSeries}
  accounts={accounts}
  value={null}
  onChange={(seriesId) => console.log('Selected:', seriesId)}
  accountId="acc_chase_credit_1"  // Only show series for this account
  showNextExpected={true}
  placeholder="Select series for Chase Credit Card..."
/>
```

### With Transaction Count
```tsx
// Show how many transactions are linked to each series
<SeriesSelector
  series={allSeries}
  accounts={accounts}
  transactionCounts={{
    'series_netflix_1': 24,
    'series_spotify_1': 18,
    'series_rent_1': 12
  }}
  value={null}
  onChange={(seriesId) => console.log('Selected:', seriesId)}
  showTransactionCount={true}
  showNextExpected={true}
/>
```

---

## Props API

### Data Props

**series** (required)
- Type: `Series[]`
- Description: Array of series available for selection
- Filtering: Will be filtered by `accountId`, `excludeArchived`, and `categoryFilter` props

**accounts** (optional)
- Type: `Account[]`
- Description: Array of accounts for currency formatting and display
- Default: `[]`

**transactionCounts** (optional)
- Type: `Record<string, number>`
- Description: Map of series_id → transaction count
- Default: `{}`
- Used when: `showTransactionCount={true}`

### Selection Props

**value** (required)
- Type: `string | null`
- Description: Currently selected series_id
- Controlled: Yes, must be managed by parent component

**onChange** (required)
- Type: `(seriesId: string | null) => void`
- Description: Callback when selection changes
- Called with: series_id when selected, null when cleared

### Filtering Props

**accountId** (optional)
- Type: `string`
- Description: Filter series to only show those for specific account
- Use case: Transaction detail page (only show series for transaction's account)

**excludeArchived** (optional)
- Type: `boolean`
- Description: Whether to exclude archived series from list
- Default: `true`

**categoryFilter** (optional)
- Type: `string[]`
- Description: Filter series by specific categories
- Example: `["subscriptions", "bills"]`

### UI State Props

**loading** (optional)
- Type: `boolean`
- Description: Show loading skeleton
- Default: `false`

**disabled** (optional)
- Type: `boolean`
- Description: Disable selector (no interaction)
- Default: `false`

**error** (optional)
- Type: `string | null`
- Description: Error message to display below selector
- Default: `null`

### Action Props

**onCreateNew** (optional)
- Type: `() => void`
- Description: Callback when "Create new series" is clicked
- Required if: `allowCreateNew={true}`

**allowCreateNew** (optional)
- Type: `boolean`
- Description: Show "Create new series" option
- Default: `true`

### Display Props

**placeholder** (optional)
- Type: `string`
- Description: Placeholder text when no selection
- Default: `"Select a series..."`

**showTransactionCount** (optional)
- Type: `boolean`
- Description: Show transaction count per series
- Default: `false`
- Requires: `transactionCounts` prop

**showNextExpected** (optional)
- Type: `boolean`
- Description: Show next expected date and amount
- Default: `true`

**groupByCategory** (optional)
- Type: `boolean`
- Description: Group series by category in dropdown
- Default: `true`

### Customization Props

**theme** (optional)
- Type: `"light" | "dark"`
- Description: Visual theme
- Default: `"light"`

**size** (optional)
- Type: `"small" | "medium" | "large"`
- Description: Size variant
- Default: `"medium"`

---

## Keyboard Navigation

**When Closed:**
- `Tab`: Focus selector
- `Enter` or `Space`: Open dropdown
- `↓`: Open dropdown and focus first option

**When Open:**
- `Escape`: Close dropdown, clear search
- `↑`: Move highlight to previous option
- `↓`: Move highlight to next option
- `Enter`: Select highlighted option (or create new if no results)
- `Type`: Search series by name

**When Option Highlighted:**
- Visual highlight (background color change)
- Scroll into view if out of viewport

---

## Multi-Domain Applicability

### Finance Domain
```tsx
<SeriesSelector
  series={financeSeries}
  accounts={financeAccounts}
  transactionCounts={txCounts}
  value={selectedSeriesId}
  onChange={setSelectedSeriesId}
  accountId={transaction.account_id}
  onCreateNew={openCreateSeriesDialog}
  placeholder="Link to subscription or bill..."
/>
// Series: Netflix, Rent, Salary
// Use case: Link expense/income transaction to recurring series
```

### Healthcare Domain
```tsx
<SeriesSelector
  series={healthcareSeries}
  accounts={healthcareAccounts}
  value={selectedSeriesId}
  onChange={setSelectedSeriesId}
  accountId={claim.account_id}
  categoryFilter={["insurance", "prescriptions"]}
  onCreateNew={openCreateHealthSeriesDialog}
  placeholder="Link to insurance or prescription..."
/>
// Series: Insurance Premium, Blood Pressure Medication, Physical Therapy
// Use case: Link healthcare claim to recurring payment series
```

### Legal Domain
```tsx
<SeriesSelector
  series={legalSeries}
  accounts={legalAccounts}
  value={selectedSeriesId}
  onChange={setSelectedSeriesId}
  accountId={payment.account_id}
  categoryFilter={["retainers", "fees"]}
  onCreateNew={openCreateLegalSeriesDialog}
  placeholder="Link to retainer or fee..."
/>
// Series: Monthly Retainer, Court Filing Fees, Lease Payment
// Use case: Link legal payment to recurring obligation
```

### Research Domain
```tsx
<SeriesSelector
  series={researchSeries}
  accounts={researchAccounts}
  value={selectedSeriesId}
  onChange={setSelectedSeriesId}
  accountId={disbursement.account_id}
  categoryFilter={["grants", "subscriptions"]}
  onCreateNew={openCreateResearchSeriesDialog}
  placeholder="Link to grant or subscription..."
/>
// Series: NSF Grant Disbursement, Journal Subscription, Lab Equipment Maintenance
// Use case: Link research payment to recurring funding or cost
```

---

## Styling

```css
.series-selector {
  position: relative;
  width: 100%;
}

/* Trigger */
.selector-trigger {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 12px;
  background: var(--input-bg);
  border: 1px solid var(--input-border);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}

.selector-trigger:hover {
  border-color: var(--input-border-hover);
}

.series-selector.open .selector-trigger {
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.series-selector.disabled .selector-trigger {
  background: var(--input-disabled-bg);
  cursor: not-allowed;
  opacity: 0.6;
}

.selected-series {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.selected-name {
  font-size: 14px;
  font-weight: 500;
  color: var(--text-color);
}

.selected-meta {
  font-size: 12px;
  color: var(--secondary-color);
}

.placeholder {
  font-size: 14px;
  color: var(--placeholder-color);
}

.selector-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.clear-button {
  padding: 0;
  background: none;
  border: none;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  color: var(--secondary-color);
  transition: color 0.2s;
}

.clear-button:hover {
  color: var(--text-color);
}

.dropdown-arrow {
  font-size: 10px;
  color: var(--secondary-color);
}

/* Dropdown */
.selector-dropdown {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  max-height: 400px;
  background: var(--dropdown-bg);
  border: 1px solid var(--dropdown-border);
  border-radius: 6px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  z-index: 1000;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.dropdown-search {
  padding: 12px;
  border-bottom: 1px solid var(--divider-color);
}

.dropdown-search input {
  width: 100%;
  padding: 8px 12px;
  background: var(--search-input-bg);
  border: 1px solid var(--search-input-border);
  border-radius: 4px;
  font-size: 14px;
  outline: none;
}

.dropdown-search input:focus {
  border-color: var(--primary-color);
}

.dropdown-options {
  overflow-y: auto;
  max-height: 320px;
}

/* Options */
.option-group {
  padding: 8px 0;
}

.option-group:not(:last-child) {
  border-bottom: 1px solid var(--divider-color);
}

.group-label {
  padding: 8px 12px;
  font-size: 11px;
  font-weight: 600;
  color: var(--group-label-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.series-option {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  cursor: pointer;
  transition: background 0.2s;
}

.series-option:hover,
.series-option.highlighted {
  background: var(--option-hover-bg);
}

.series-option.selected {
  background: var(--option-selected-bg);
}

.option-main {
  flex: 1;
  min-width: 0;
}

.option-name {
  font-size: 14px;
  font-weight: 500;
  color: var(--text-color);
  margin-bottom: 2px;
}

.option-meta {
  font-size: 12px;
  color: var(--secondary-color);
  display: flex;
  align-items: center;
  gap: 6px;
}

.separator {
  color: var(--separator-color);
}

.option-right {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-left: 16px;
}

.option-next {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
}

.next-label {
  font-size: 10px;
  color: var(--secondary-color);
  text-transform: uppercase;
}

.next-date {
  font-size: 12px;
  font-weight: 600;
  color: var(--text-color);
}

.option-count {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--count-bg);
  color: var(--count-color);
  border-radius: 4px;
  font-weight: 500;
}

.option-checkmark {
  font-size: 16px;
  color: var(--primary-color);
}

/* Create New */
.create-new-option,
.create-new-button {
  width: 100%;
  padding: 12px;
  background: none;
  border: none;
  text-align: left;
  font-size: 14px;
  font-weight: 500;
  color: var(--primary-color);
  cursor: pointer;
  transition: background 0.2s;
}

.create-new-option:hover,
.create-new-button:hover {
  background: var(--option-hover-bg);
}

/* Empty State */
.empty-state {
  padding: 24px 12px;
  text-align: center;
}

.empty-message {
  font-size: 14px;
  color: var(--secondary-color);
  margin-bottom: 16px;
}

/* Error */
.selector-error {
  margin-top: 4px;
  font-size: 12px;
  color: var(--error-color);
}

/* Loading */
.series-selector.loading .selector-skeleton {
  height: 44px;
  background: linear-gradient(90deg, var(--skeleton-bg) 25%, var(--skeleton-shimmer) 50%, var(--skeleton-bg) 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 6px;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* Size Variants */
.series-selector.size-small .selector-trigger {
  padding: 6px 10px;
}

.series-selector.size-small .selected-name,
.series-selector.size-small .option-name {
  font-size: 13px;
}

.series-selector.size-large .selector-trigger {
  padding: 14px 16px;
}

.series-selector.size-large .selected-name,
.series-selector.size-large .option-name {
  font-size: 16px;
}

/* Theme: Light */
.theme-light {
  --input-bg: #ffffff;
  --input-border: #d1d5db;
  --input-border-hover: #9ca3af;
  --input-disabled-bg: #f3f4f6;
  --text-color: #111827;
  --secondary-color: #6b7280;
  --placeholder-color: #9ca3af;
  --primary-color: #3b82f6;
  --dropdown-bg: #ffffff;
  --dropdown-border: #d1d5db;
  --divider-color: #e5e7eb;
  --search-input-bg: #ffffff;
  --search-input-border: #d1d5db;
  --group-label-color: #6b7280;
  --option-hover-bg: #f3f4f6;
  --option-selected-bg: #e0e7ff;
  --separator-color: #d1d5db;
  --count-bg: #dbeafe;
  --count-color: #1e40af;
  --error-color: #dc2626;
  --skeleton-bg: #e5e7eb;
  --skeleton-shimmer: #f3f4f6;
}

/* Theme: Dark */
.theme-dark {
  --input-bg: #374151;
  --input-border: #4b5563;
  --input-border-hover: #6b7280;
  --input-disabled-bg: #1f2937;
  --text-color: #f3f4f6;
  --secondary-color: #9ca3af;
  --placeholder-color: #6b7280;
  --primary-color: #3b82f6;
  --dropdown-bg: #1f2937;
  --dropdown-border: #374151;
  --divider-color: #374151;
  --search-input-bg: #374151;
  --search-input-border: #4b5563;
  --group-label-color: #9ca3af;
  --option-hover-bg: #374151;
  --option-selected-bg: #1e3a8a;
  --separator-color: #4b5563;
  --count-bg: #1e3a8a;
  --count-color: #93c5fd;
  --error-color: #f87171;
  --skeleton-bg: #374151;
  --skeleton-shimmer: #4b5563;
}
```

---

## Accessibility

```tsx
// Trigger
<div
  className="selector-trigger"
  role="button"
  tabIndex={0}
  aria-haspopup="listbox"
  aria-expanded={isOpen}
  aria-disabled={disabled}
  aria-label="Select series"
>
  {/* Trigger content */}
</div>

// Dropdown
<div className="selector-dropdown" role="listbox" aria-label="Series options">
  {/* Options */}
</div>

// Option
<div
  className="series-option"
  role="option"
  aria-selected={isSelected}
  onClick={onClick}
>
  {/* Option content */}
</div>

// Search input
<input
  type="text"
  placeholder="Search series..."
  aria-label="Search series"
/>

// Clear button
<button
  className="clear-button"
  onClick={handleClear}
  aria-label="Clear selection"
>
  ×
</button>
```

**ARIA Roles:**
- Trigger: `role="button"`
- Dropdown: `role="listbox"`
- Options: `role="option"`

**ARIA States:**
- `aria-haspopup="listbox"`: Indicates dropdown contains a list
- `aria-expanded`: true when open, false when closed
- `aria-selected`: true for selected option
- `aria-disabled`: true when disabled

**Keyboard Support:**
- Full keyboard navigation (↑↓ select, Enter confirm, Esc close)
- Focus management (auto-focus search when opened)
- Scroll into view for highlighted options

**Screen Reader:**
- Announces selection: "Netflix Subscription, Monthly, $15.99, selected"
- Announces options: "Netflix Subscription, Monthly, $15.99, Next: Feb 15"
- Announces empty state: "No series found matching..."

---

## Related Components

**Uses (Dependencies):**
- None (standalone component)

**Used By:**
- Transaction Detail page (link transaction to series)
- Manual Link Dialog (in SeriesManager)
- Bulk Edit Dialog (link multiple transactions)

**Similar Components:**
- AccountSelector (dropdown for selecting account)
- CounterpartySelector (dropdown for selecting counterparty)
- CategorySelector (dropdown for selecting category)

---

## Summary

SeriesSelector provides a powerful, reusable dropdown for selecting recurring payment series with:

✅ **Searchable** (fuzzy matching by name)
✅ **Grouped** (by category: subscriptions, bills, income, other)
✅ **Contextual** (shows next expected date and amount)
✅ **Filterable** (by account, category, archived status)
✅ **Inline creation** ("Create new series" option)
✅ **Keyboard navigable** (↑↓ select, Enter confirm, Esc close)
✅ **Accessible** (ARIA labels, roles, keyboard support)
✅ **Responsive** (adapts to container size)
✅ **Multi-domain** (finance, healthcare, legal, research)

This component standardizes series selection across the application and provides a consistent, user-friendly experience for linking transactions to recurring payment series.
