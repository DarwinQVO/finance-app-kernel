# TransactionTable (IL Component)

## Definition

**TransactionTable** is a reusable data grid component for displaying canonical records with sorting, pagination, and row selection. It provides a consistent tabular interface across all exploration views.

**Problem it solves:**
- Table UI implemented differently across pages (inconsistent UX)
- Sorting logic duplicated (client-side vs server-side confusion)
- Pagination controls reinvented (different styles, behaviors)
- Responsive design hard (tables break on mobile)
- Loading states inconsistent (spinners vs skeletons)

**Solution:**
- Single, reusable table component
- Server-side sorting/pagination (via TransactionQuery)
- Responsive design (horizontal scroll + mobile-friendly columns)
- Built-in loading skeletons
- Accessibility (keyboard navigation, screen readers)

---

## Interface Contract

```typescript
// React/TypeScript example

interface Column {
  key: string;                // Field name (e.g., "transaction_date")
  label: string;              // Display name (e.g., "Date")
  sortable: boolean;          // Can user sort by this column?
  render?: (value: any, row: any) => React.ReactNode; // Custom renderer
  align?: "left" | "right" | "center";
  width?: string;             // e.g., "120px", "auto"
}

interface TransactionTableProps {
  // Data
  rows: Array<Record<string, any>>;  // Transaction data
  columns: Column[];                  // Column definitions

  // Sorting
  sortField?: string;                 // Current sort field
  sortOrder?: "asc" | "desc";        // Current sort order
  onSort?: (field: string, order: "asc" | "desc") => void;

  // Pagination
  pagination?: {
    has_next: boolean;
    has_prev: boolean;
    next_cursor?: string;
    prev_cursor?: string;
    total_count?: number;
    current_start: number;            // e.g., 1 (for "Showing 1-50")
    current_end: number;              // e.g., 50
  };
  onPageChange?: (cursor: string, direction: "next" | "prev") => void;

  // Selection
  selectable?: boolean;               // Show checkboxes?
  selectedRows?: Set<string>;         // Selected row IDs
  onSelectionChange?: (selected: Set<string>) => void;

  // Row actions
  onRowClick?: (row: any) => void;   // Click handler (drill-down)
  rowActions?: Array<{
    label: string;
    icon?: React.ReactNode;
    onClick: (row: any) => void;
  }>;

  // UI states
  loading?: boolean;                  // Show skeleton
  emptyMessage?: string;              // "No transactions found"
  error?: string;                     // Error message
}

const TransactionTable: React.FC<TransactionTableProps> = (props) => {
  // Implementation
};
```

---

## Multi-Domain Applicability

**Universal Pattern:** Display structured records in tabular format

### Finance Domain
```tsx
<TransactionTable
  columns={[
    { key: "transaction_date", label: "Date", sortable: true },
    { key: "merchant", label: "Merchant", sortable: true },
    {
      key: "amount",
      label: "Amount",
      sortable: true,
      align: "right",
      render: (value, row) => (
        <Currency amount={value} currency={row.currency} />
      )
    },
    { key: "category", label: "Category", sortable: false },
    { key: "account_id", label: "Account", sortable: true }
  ]}
  rows={transactions}
  sortField="transaction_date"
  sortOrder="desc"
  onSort={(field, order) => refetch({ sort: { field, order } })}
  pagination={{
    has_next: true,
    next_cursor: "eyJ...",
    current_start: 1,
    current_end: 50,
    total_count: 127
  }}
  onPageChange={(cursor, direction) => fetchPage(cursor)}
  onRowClick={(row) => navigate(`/transactions/${row.canonical_id}`)}
/>
```

### Healthcare Domain
```tsx
<TransactionTable
  columns={[
    { key: "test_date", label: "Test Date", sortable: true },
    { key: "test_name", label: "Test", sortable: true },
    {
      key: "value",
      label: "Result",
      sortable: true,
      render: (value, row) => (
        <LabResult
          value={value}
          normal={row.is_normal}
          referenceRange={row.reference_range}
        />
      )
    },
    { key: "ordering_physician", label: "Doctor", sortable: true }
  ]}
  rows={labResults}
  onSort={(field, order) => refetch({ sort: { field, order } })}
/>
```

### Legal Domain
```tsx
<TransactionTable
  columns={[
    { key: "contract_id", label: "Contract", sortable: true },
    { key: "clause_text", label: "Clause", sortable: false },
    {
      key: "risk_level",
      label: "Risk",
      sortable: true,
      render: (value) => <RiskBadge level={value} />
    },
    { key: "counterparty", label: "Party", sortable: true }
  ]}
  rows={clauses}
  selectable={true}
  selectedRows={selectedClauses}
  onSelectionChange={setSelectedClauses}
/>
```

### Research Domain
```tsx
<TransactionTable
  columns={[
    { key: "title", label: "Title", sortable: true },
    { key: "author", label: "Author", sortable: true },
    { key: "publication_year", label: "Year", sortable: true },
    { key: "journal", label: "Journal", sortable: false },
    {
      key: "doi",
      label: "DOI",
      sortable: false,
      render: (value) => <a href={`https://doi.org/${value}`}>{value}</a>
    }
  ]}
  rows={citations}
/>
```

### Manufacturing Domain
```tsx
<TransactionTable
  columns={[
    { key: "measurement_date", label: "Date", sortable: true },
    { key: "product_id", label: "Product", sortable: true },
    { key: "batch_number", label: "Batch", sortable: true },
    {
      key: "passed_qc",
      label: "Status",
      sortable: true,
      render: (value) => (
        <StatusBadge status={value ? "pass" : "fail"} />
      )
    }
  ]}
  rows={qcMeasurements}
/>
```

### Media Domain
```tsx
<TransactionTable
  columns={[
    { key: "timestamp", label: "Time", sortable: true },
    { key: "speaker_id", label: "Speaker", sortable: true },
    { key: "text", label: "Transcript", sortable: false },
    {
      key: "sentiment_score",
      label: "Sentiment",
      sortable: true,
      render: (value) => <SentimentBar score={value} />
    }
  ]}
  rows={snippets}
/>
```

---

## Responsibilities

✅ **DOES:**
- Render table with column headers
- Show sortable column indicators (↑↓)
- Handle sort clicks (call onSort callback)
- Render pagination controls (Prev/Next buttons, "Showing X-Y of Z")
- Show loading skeleton (not spinner)
- Show empty state ("No results")
- Show error state
- Handle row clicks (drill-down)
- Render custom column content (via render prop)
- Support row selection (checkboxes)
- Keyboard navigation (Tab, Enter)
- Responsive design (horizontal scroll on mobile)
- Accessibility (ARIA labels, screen reader support)

---

## NOT Responsibilities

❌ **DOES NOT:**
- Fetch data (receives rows prop)
- Execute queries (parent component handles)
- Manage pagination state (receives pagination prop)
- Store sort state (receives sortField/sortOrder)
- Filter data (receives pre-filtered rows)
- Calculate summary metrics (receives pre-calculated)

---

## Implementation Notes

### Responsive Design

**Desktop (>768px):**
```
┌────────────────────────────────────────────────────┐
│ Date       │ Merchant     │ Amount  │ Category     │
├────────────────────────────────────────────────────┤
│ 05/05/25   │ OpenAI       │ $200.00 │ Software     │
│ 05/01/25   │ Perplexity   │ $20.00  │ Software     │
└────────────────────────────────────────────────────┘
```

**Mobile (<768px):**
```
┌──────────────────────────┐
│ OpenAI                   │
│ $200.00 • 05/05/25      │
│ Software                 │
├──────────────────────────┤
│ Perplexity              │
│ $20.00 • 05/01/25       │
│ Software                 │
└──────────────────────────┘
```

**Or horizontal scroll:**
```
┌──────────────→
│ Date  │ Merchant │ ...
│ 05/05 │ OpenAI   │ ...
```

---

### Loading State (Skeleton)

**Not this (spinner):**
```
┌────────────────────────┐
│         ⌛            │
│    Loading...          │
└────────────────────────┘
```

**This (skeleton):**
```
┌────────────────────────────────────────┐
│ ████████  ████████████  ███████  ████ │
│ ████████  ████████████  ███████  ████ │
│ ████████  ████████████  ███████  ████ │
└────────────────────────────────────────┘
```

**Implementation:**
```tsx
{loading ? (
  <tbody>
    {Array.from({ length: 10 }).map((_, i) => (
      <tr key={i}>
        <td><Skeleton width="80px" /></td>
        <td><Skeleton width="120px" /></td>
        <td><Skeleton width="60px" /></td>
        <td><Skeleton width="100px" /></td>
      </tr>
    ))}
  </tbody>
) : (
  <tbody>
    {rows.map(row => <TableRow row={row} />)}
  </tbody>
)}
```

---

### Sort Indicators

**Unsorted column:**
```
Merchant ↕
```

**Sorted ascending:**
```
Amount ↑
```

**Sorted descending:**
```
Date ↓
```

**Implementation:**
```tsx
<th onClick={() => onSort("merchant", sortOrder === "asc" ? "desc" : "asc")}>
  Merchant
  {sortField === "merchant" ? (
    sortOrder === "asc" ? "↑" : "↓"
  ) : (
    "↕"
  )}
</th>
```

---

### Pagination Controls

**Layout:**
```
┌─────────────────────────────────────────────────┐
│ Showing 1-50 of 127    [ ← Prev ] [ Next → ]   │
└─────────────────────────────────────────────────┘
```

**States:**
- First page: Prev disabled
- Last page: Next disabled
- Loading: Both disabled + spinner

**Implementation:**
```tsx
<div className="pagination">
  <span>Showing {current_start}-{current_end} of {total_count}</span>
  <div className="buttons">
    <button
      disabled={!has_prev || loading}
      onClick={() => onPageChange(prev_cursor!, "prev")}
    >
      ← Prev
    </button>
    <button
      disabled={!has_next || loading}
      onClick={() => onPageChange(next_cursor!, "next")}
    >
      Next →
    </button>
  </div>
</div>
```

---

### Empty State

**Message:**
```
┌────────────────────────────────────┐
│                                    │
│         📊                         │
│   No transactions found            │
│                                    │
│   Try adjusting your filters       │
│   [ Reset Filters ]                │
│                                    │
└────────────────────────────────────┘
```

---

### Accessibility

**ARIA attributes:**
```tsx
<table role="table" aria-label="Transaction list">
  <thead>
    <tr role="row">
      <th
        role="columnheader"
        aria-sort={sortField === "date" ? sortOrder : "none"}
        tabIndex={0}
        onKeyDown={(e) => e.key === "Enter" && onSort("date")}
      >
        Date
      </th>
    </tr>
  </thead>
</table>
```

**Keyboard navigation:**
- Tab: Move between sortable headers
- Enter/Space: Toggle sort
- Arrow keys: Navigate cells (optional)

---

## Related Components

**Uses:**
- **Skeleton** (loading state)
- **Badge** (status indicators)
- **Currency** (amount formatting)
- **Tooltip** (hover info)

**Used by:**
- Transaction List page (vertical 2.1)
- OL Exploration (vertical 2.2)
- Finance Dashboard (vertical 2.3)
- Transfer linking views (vertical 3.5)

---

## Version History

- **v1.0** (2025-05-23): Initial release
  - Server-side sorting/pagination
  - Skeleton loading
  - Responsive design
  - Accessibility support
