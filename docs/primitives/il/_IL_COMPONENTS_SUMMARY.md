# IL Components Summary

**Status**: Specification complete
**Last Updated**: 2025-10-24
**Verticals covered**: 1.1, 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 4.1, 4.2

---

## Component Catalog

### Vertical 1.1 (Upload Flow)

#### 1. FileUpload ✅
**Full spec**: [FileUpload.md](FileUpload.md)

**Props**: `accept`, `maxSizeBytes`, `onFileSelect`, `onError`
**States**: idle | dragover | validating | error
**Reusable across**: Finance, Health, Legal, Generic uploads

---

### 2. RegistrySelect

**Purpose**: Dropdown for selecting from a registry (e.g., source types)

**Props**:
```typescript
interface RegistrySelectProps {
  registry: Registry  // { id, label, description }[]
  value: string
  onChange: (value: string) => void
  disabled?: boolean
  placeholder?: string
}
```

**Example (Finance)**:
```tsx
<RegistrySelect
  registry={[
    { id: "bofa_pdf", label: "Bank of America PDF", description: "Checking/Savings statement" }
  ]}
  value={sourceType}
  onChange={setSourceType}
  placeholder="Select statement type"
/>
```

**Reusability**: Any dropdown with typed options (parsers, categories, accounts)

---

### 3. UploadButton

**Purpose**: Action button with loading/success/error states

**Props**:
```typescript
interface UploadButtonProps {
  onClick: () => void | Promise<void>
  disabled?: boolean
  loading?: boolean
  children: ReactNode
  variant?: "primary" | "secondary"
}
```

**States**:
- `idle`: Default state
- `loading`: Spinner + "Uploading..."
- `success`: Checkmark + "Uploaded!" (auto-reset after 2s)
- `error`: X icon + "Failed" (stays until next click)

**Visual**:
```
idle:    [ Upload ]
loading: [⏳ Uploading...]
success: [✓ Uploaded!]
error:   [✗ Failed]
```

**Reusability**: Any async action (save, submit, delete, export)

---

### 4. UploadStatus

**Purpose**: Status indicator with icon + text

**Props**:
```typescript
interface UploadStatusProps {
  status: UploadStatus
  error_message?: string
  showTimestamp?: boolean
  timestamp?: ISO8601
}
```

**Visual Mapping**:
```
queued_for_parse → 🕒 Queued
parsing          → ⏳ Parsing...
parsed           → ✓ Parsed
normalizing      → ⏳ Normalizing...
normalized       → ✅ Complete
error            → ❌ Error: {error_message}
```

**Color Scheme**:
```
queued   → Gray
parsing  → Blue (animated)
parsed   → Green
error    → Red
```

**Reusability**: Any status display (job status, pipeline stages, order status)

---

### 5. UploadList

**Purpose**: Paginated list of items with status badges

**Props**:
```typescript
interface UploadListProps {
  uploads: UploadRecord[]
  onSelect?: (upload_id: string) => void
  pageSize?: number
  sortBy?: "created_at" | "status"
  filterBy?: { status?: UploadStatus }
}
```

**Visual**:
```
┌────────────────────────────────────────────────┐
│ statement_jan.pdf       [✓ Parsed]   10:32 AM │
│ statement_feb.pdf       [⏳ Parsing...] now   │
│ large_file.pdf          [❌ Error]    10:15 AM│
└────────────────────────────────────────────────┘

[< Previous] Page 1 of 5 [Next >]
```

**Features**:
- Client-side pagination (for <100 items)
- Server-side pagination (for >100 items)
- Sortable columns
- Status filtering
- Click row → drill-down

**Reusability**: Any list view (transactions, documents, reports, users)

---

### Vertical 2.1 (Transaction List View)

#### 6. TransactionTable ✅
**Full spec**: [TransactionTable.md](TransactionTable.md)

**Purpose**: Data grid with sorting, pagination, and row selection

**Props**: `rows`, `columns`, `sortField`, `sortOrder`, `onSort`, `pagination`, `onPageChange`, `loading`
**Features**: Server-side sorting, cursor-based pagination, loading skeletons
**Reusable across**: Finance transactions, healthcare records, legal documents, research papers

---

### Vertical 2.2 (OL Exploration)

#### 7. DrillDownPanel ✅
**Full spec**: [DrillDownPanel.md](DrillDownPanel.md)

**Purpose**: Slide-in panel for complete data lineage drill-down

**Props**: `drilldownData`, `loading`, `error`, `onClose`, `onDownloadArtifact`, `defaultTab`, `theme`
**Features**: Tabbed interface (Overview, Raw Data, Provenance, Artifact), responsive (panel/full-screen), keyboard navigation
**Reusable across**: Transaction drill-down, lab result inspection, contract clause analysis, citation verification

---

#### 8. ProvenanceTimeline ✅
**Full spec**: [ProvenanceTimeline.md](ProvenanceTimeline.md)

**Purpose**: Visual timeline for provenance chain (upload → parse → normalize)

**Props**: `entries`, `variant`, `showDurations`, `showMetadata`, `maxHeight`, `theme`
**Features**: Color-coded events, duration indicators, expandable metadata, responsive (vertical/horizontal)
**Reusable across**: Any audit trail visualization (upload timeline, approval workflow, processing pipeline)

---

### Vertical 2.3 (Finance Dashboard)

#### 9. DashboardGrid ✅
**Full spec**: [DashboardGrid.md](DashboardGrid.md)

**Purpose**: Responsive grid layout for dashboard widgets

**Props**:
```typescript
interface DashboardGridProps {
  widgets: DashboardWidget[]
  columns?: number  // Default: 12 (desktop), 1 (mobile)
  gap?: number      // Default: 16px
  editable?: boolean  // Allow drag-to-rearrange (v2 feature)
  onWidgetClick?: (widget: DashboardWidget) => void
  loading?: boolean
}

interface DashboardWidget {
  id: string
  type: WidgetType
  position: { row: number; col: number; width: number; height?: number }
  data: any
  loading?: boolean
  error?: string
}

type WidgetType =
  | "income_summary"
  | "expense_summary"
  | "net_summary"
  | "category_breakdown"
  | "top_merchants"
  | "deductible_summary"
```

**Visual (Desktop 12-col grid)**:
```
┌────────────────────────────────────────────────────────┐
│ Grid Container (12 columns)                            │
├────────────────────────────────────────────────────────┤
│ ┌────────┐  ┌────────┐  ┌────────┐                    │
│ │ Widget │  │ Widget │  │ Widget │                    │
│ │ (col 0 │  │ (col 4 │  │ (col 8 │                    │
│ │ width 4)  │ width 4)  │ width 4)                    │
│ └────────┘  └────────┘  └────────┘                    │
│                                                        │
│ ┌──────────────────────────────────────────┐          │
│ │ Widget (col 0, width 12)                 │          │
│ │ Full-width metric card                   │          │
│ └──────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────┘
```

**Mobile (stacked)**:
```
┌──────────────────┐
│ Widget           │
│ (stacked)        │
└──────────────────┘
┌──────────────────┐
│ Widget           │
│ (stacked)        │
└──────────────────┘
┌──────────────────┐
│ Widget           │
│ (stacked)        │
└──────────────────┘
```

**Features**:
- Responsive grid (12-col desktop, 1-col mobile)
- Automatic gap spacing (default 16px)
- Widget positioning via row/col/width
- Loading skeletons per widget
- Error state per widget
- Click handler for drill-down

**Reusability**:
- Finance: Transaction dashboard
- Healthcare: Patient vitals dashboard
- Legal: Case management dashboard
- Research: Research metrics dashboard
- Manufacturing: Production dashboard
- Media: Content analytics dashboard

---

#### 10. MetricCard ✅
**Full spec**: [MetricCard.md](MetricCard.md)

**Purpose**: Reusable widget for displaying single metric (number, trend, breakdown)

**Props**:
```typescript
interface MetricCardProps {
  title: string
  value: number | string
  currency?: string  // "USD", "MXN", etc.
  format?: "currency" | "number" | "percentage"
  trend?: {
    direction: "up" | "down" | "neutral"
    value: number
    comparison: string  // e.g., "from April"
  }
  breakdown?: { label: string; value: number; percentage: number }[]
  loading?: boolean
  error?: string
  onClick?: () => void  // For drill-down
  icon?: ReactNode
  variant?: "default" | "compact" | "detailed"
}
```

**Visual States**:

**Default**:
```
┌─────────────────────────┐
│ 💰 Income               │
├─────────────────────────┤
│ $9,000.00               │
│ ↑ 5% from April         │
└─────────────────────────┘
```

**Detailed (with breakdown)**:
```
┌─────────────────────────────────┐
│ 🍔 Top Categories               │
├─────────────────────────────────┤
│ Food & Drink     $680.00   (8%) │
│ Travel         $4,516.00  (55%) │
│ Software       $2,500.00  (30%) │
│ Health           $551.00   (7%) │
│                                 │
│ [View All →]                    │
└─────────────────────────────────┘
```

**Loading**:
```
┌─────────────────────────┐
│ ▓▓▓▓▓▓▓▓▓▓              │
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓           │
│ ▓▓▓▓▓▓▓▓                │
└─────────────────────────┘
```

**Error**:
```
┌─────────────────────────┐
│ 💰 Income               │
├─────────────────────────┤
│ ⚠️ Failed to load       │
│ [Retry]                 │
└─────────────────────────┘
```

**Features**:
- Automatic number formatting (currency, percentage, etc.)
- Trend indicators with direction arrows (↑ ↓ →)
- Optional breakdown list (top N items)
- Click handler for drill-down navigation
- Loading skeleton
- Error state with retry button
- Responsive sizing (compact on mobile)

**Accessibility**:
- Screen reader announces: "Income metric. Value: $9,000. Up 5% from April. Click to view details."
- Trend uses icon + color (not just color)
- Keyboard focusable with `Tab`
- `Enter` triggers drill-down

**Reusability**:
- Finance: Income card, expense card, net card, deductible card
- Healthcare: Heart rate card, blood pressure card, BMI card
- Legal: Cases won/lost card, hours billed card
- Research: Papers published card, citations card, h-index card
- Manufacturing: Units produced card, defect rate card, yield card
- Media: Views card, engagement rate card, subscriber count card

---

#### 11. SavedViewSelector ✅
**Full spec**: [SavedViewSelector.md](SavedViewSelector.md)

**Purpose**: Dropdown component for selecting and managing saved views (with MRU history)

**Props**:
```typescript
interface SavedViewSelectorProps {
  views: SavedView[]
  currentViewId?: string
  defaultViewId?: string

  onViewSelect: (viewId: string) => void
  onViewCreate?: () => void
  onViewEdit?: (viewId: string) => void
  onViewDelete?: (viewId: string) => void
  onViewSetDefault?: (viewId: string) => void

  showCreateButton?: boolean  // Default: true
  showManageButton?: boolean  // Default: true
  maxRecentViews?: number     // Default: 5
  theme?: "light" | "dark"
}

interface SavedView {
  view_id: string
  name: string
  description?: string
  is_system: boolean   // Cannot delete system views
  is_default: boolean
  created_at: string   // ISO 8601
  updated_at: string   // ISO 8601
}
```

**Visual (Collapsed)**:
```
┌─────────────────────────────────────┐
│ Monthly Overview              [▼]   │
└─────────────────────────────────────┘
```

**Visual (Expanded)**:
```
┌─────────────────────────────────────┐
│ 🔍 Search views...                  │
├─────────────────────────────────────┤
│ RECENT                              │
│ ● Monthly Overview (active)         │
│   Q1 2025 Deductibles               │
│   Annual Summary 2024               │
├─────────────────────────────────────┤
│ SYSTEM VIEWS                        │
│   Monthly Overview              ⭐   │
│   Quarterly Review                  │
│   YTD Summary                       │
├─────────────────────────────────────┤
│ MY VIEWS (3)                        │
│   Q1 2025 Deductibles    [✏️] [🗑️]  │
│   Custom Dashboard       [✏️] [🗑️]  │
│   Tax Prep View          [✏️] [🗑️]  │
├─────────────────────────────────────┤
│ [+ Create New View]                 │
│ [⚙️ Manage Views]                   │
└─────────────────────────────────────┘
```

**Features**:
- MRU (Most Recently Used) history - shows last 5 accessed views at top
- Search/filter by view name
- System vs user views separation (system views cannot be deleted)
- Action buttons per user view: edit, delete, set as default
- Default view marked with star (⭐)
- Active view highlighted
- Keyboard navigation (↑↓ arrows, Enter to select, ESC to close)
- Create new view button
- Manage views button (opens view manager modal)

**States**:
- **Empty** (no user views): Show only system views + "Create your first view" CTA
- **Loading**: Skeleton loader for view list
- **Error**: "Failed to load views" with retry button

**Accessibility**:
- Screen reader announces: "Saved views dropdown. Current view: Monthly Overview. 8 views available."
- Keyboard accessible (Tab to focus, Space/Enter to open, arrows to navigate)
- ARIA labels for all action buttons
- Focus trap when dropdown is open

**Reusability**:
- Finance: Dashboard view selector, saved filter selector, report template selector
- Healthcare: Patient query selector, saved search selector, report view selector
- Legal: Case search selector, document filter selector, custom view selector
- Research: Paper library view selector, citation query selector, saved search selector
- Manufacturing: QC dashboard selector, production view selector, custom report selector
- Media: Analytics view selector, content filter selector, saved segment selector

**Universal Pattern**: Any feature that saves user configurations (filters, layouts, queries, segments)

---

### Vertical 3.1 (Account Registry)

#### 12. AccountManager ✅
**Full spec**: [AccountManager.md](AccountManager.md)

**Purpose**: Full CRUD UI for managing user accounts (create, edit, archive, search, filter)

**Props**:
```typescript
interface AccountManagerProps {
  accounts: Account[];
  onAccountCreate: (account: CreateAccountInput) => Promise<Account>;
  onAccountUpdate: (accountId: string, updates: Partial<Account>) => Promise<Account>;
  onAccountArchive: (accountId: string) => Promise<void>;
  onAccountUnarchive: (accountId: string) => Promise<void>;

  loading?: boolean;
  error?: string | null;

  // UI customization
  showArchived?: boolean;  // Default: false
  groupBy?: "type" | "currency" | null;  // Default: null
  defaultSort?: "name" | "created_at";  // Default: "name"
}

interface Account {
  account_id: string;
  name: string;
  type: AccountType;  // "checking", "savings", "credit", "debit", "investment", "loan"
  currency: string;   // ISO 4217
  institution?: string;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}
```

**Features**:
- List view with search (real-time filtering by name)
- Filter by type, currency, active status
- Sort by name, created date
- Create account modal (name, type, currency, institution)
- Edit account modal (name, institution only - type/currency immutable)
- Archive confirmation dialog (shows transaction count)
- Empty state ("Add your first account" CTA)
- Keyboard shortcuts (N = new account, ESC = close modal)

**Visual (List View)**:
```
┌──────────────────────────────────────────────────────────┐
│ 🔍 Search accounts...                  [+ New Account]   │
├──────────────────────────────────────────────────────────┤
│ CHECKING ACCOUNTS (2)                                    │
│   ┌────────────────────────────────────────────────┐     │
│   │ 🏦 BofA Personal Checking         USD    [⚙️]  │     │
│   │    Bank of America                              │     │
│   └────────────────────────────────────────────────┘     │
│   ┌────────────────────────────────────────────────┐     │
│   │ 🏦 Scotia MXN                     MXN    [⚙️]  │     │
│   │    Scotiabank                                   │     │
│   └────────────────────────────────────────────────┘     │
│                                                          │
│ CREDIT CARDS (2)                                         │
│   ┌────────────────────────────────────────────────┐     │
│   │ 💳 BofA Credit                    USD    [⚙️]  │     │
│   │    Bank of America                              │     │
│   └────────────────────────────────────────────────┘     │
│   ┌────────────────────────────────────────────────┐     │
│   │ 💳 Apple Card                     USD    [⚙️]  │     │
│   │    Apple                                        │     │
│   └────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Manage bank accounts, credit cards
- Healthcare: Manage insurance providers, medical facilities
- Legal: Manage trust accounts, client accounts
- Research: Manage funding sources, grants
- Manufacturing: Manage cost centers, departments
- Media: Manage revenue streams, ad accounts

---

#### 13. AccountSelector ✅
**Full spec**: [AccountSelector.md](AccountSelector.md)

**Purpose**: Reusable dropdown for selecting account (used in transaction editing, filtering, reports)

**Props**:
```typescript
interface AccountSelectorProps {
  accounts: Account[];
  currentAccountId?: string;
  onAccountSelect: (accountId: string) => void;

  // Optional features
  onAccountCreate?: () => void;  // Show "Create New Account" button
  showCreateButton?: boolean;    // Default: false
  groupBy?: "type" | "currency" | null;  // Default: "type"
  showCurrencyBadge?: boolean;   // Default: true
  showInstitution?: boolean;     // Default: true

  // For transfer flows (exclude same account)
  excludeAccountId?: string;     // Don't show this account in list

  // States
  loading?: boolean;
  error?: string | null;
  disabled?: boolean;
  placeholder?: string;  // Default: "Select account..."
}
```

**Visual (Collapsed)**:
```
┌─────────────────────────────────────┐
│ 🏦 BofA Personal Checking     [▼]   │
└─────────────────────────────────────┘
```

**Visual (Expanded with Grouping)**:
```
┌─────────────────────────────────────┐
│ 🔍 Search accounts...               │
├─────────────────────────────────────┤
│ CHECKING ACCOUNTS                   │
│ ● 🏦 BofA Personal Checking   USD   │
│   🏦 Scotia MXN                MXN   │
├─────────────────────────────────────┤
│ CREDIT CARDS                        │
│   💳 BofA Credit               USD   │
│   💳 Apple Card                USD   │
├─────────────────────────────────────┤
│ DEBIT CARDS                         │
│   💳 Wise USD                  USD   │
├─────────────────────────────────────┤
│ [+ Create New Account]              │
└─────────────────────────────────────┘
```

**Features**:
- Search/filter by account name
- Group by type or currency
- Show currency badge
- Show institution name (optional)
- Keyboard navigation (↑↓ arrows, Enter to select, ESC to close)
- Create account inline (opens modal, auto-selects new account)
- Exclude specific accounts (for transfers - can't select same account twice)
- Empty state with "Create first account" CTA

**Reusability**:
- Finance: Select account in transaction edit, filter transactions by account, select account for transfer
- Healthcare: Select insurance provider in claim edit
- Legal: Select trust account in transaction allocation
- Research: Select funding source in expense reporting
- Manufacturing: Select cost center in work order
- Media: Select revenue stream in content analytics

---

## Composition Example (Upload Screen)

```tsx
function UploadScreen() {
  const [sourceType, setSourceType] = useState("")
  const [uploads, setUploads] = useState([])

  return (
    <div>
      <h1>Upload Bank Statement</h1>

      <RegistrySelect
        registry={SOURCE_TYPES}
        value={sourceType}
        onChange={setSourceType}
        placeholder="Select statement type"
      />

      <FileUpload
        accept={["application/pdf"]}
        maxSizeBytes={50 * 1024 * 1024}
        onFileSelect={(file) => {
          uploadFile(file, sourceType).then((record) => {
            setUploads([record, ...uploads])
          })
        }}
        onError={(err) => toast.error(err.message)}
      />

      <h2>Recent Uploads</h2>
      <UploadList
        uploads={uploads}
        onSelect={(id) => router.push(`/uploads/${id}`)}
        sortBy="created_at"
      />
    </div>
  )
}
```

---

## Testing Strategy

**Unit Tests** (per component):
- Props validation
- State transitions
- Event emissions
- Accessibility (keyboard nav, ARIA)

**Integration Tests** (composition):
- FileUpload → UploadButton → UploadList flow
- Select source → upload file → see in list
- Error handling end-to-end

**Visual Regression** (Storybook + Chromatic):
- All states documented
- Light/dark themes
- Responsive breakpoints

---

## Theming Contract

All components accept `className` and respect CSS variables:

```css
:root {
  --color-primary: #4CAF50;
  --color-error: #f44336;
  --color-success: #4CAF50;
  --color-pending: #2196F3;
  --border-radius: 4px;
  --spacing-unit: 8px;
}
```

**Dark mode support**:
```css
[data-theme="dark"] {
  --color-bg: #1e1e1e;
  --color-text: #ffffff;
  --color-border: #444;
}
```

---

**Maturity**: All 13 components specified, ready for implementation

---

## Component Count by Vertical

| Vertical | Components | Status |
|----------|-----------|--------|
| 1.1 Upload Flow | 5 components | ✅ Specified |
| 2.1 Transaction List | 1 component | ✅ Specified |
| 2.2 OL Exploration | 2 components | ✅ Specified |
| 2.3 Finance Dashboard | 3 components | ✅ Specified |
| 3.1 Account Registry | 2 components | ✅ Specified |
| **TOTAL** | **13 components** | |

---

**Next vertical**: 3.2 Counterparty Registry (expected: 2-3 new IL components for open registry management)

### Vertical 3.2 (Counterparty Registry)

#### 14. CounterpartyManager ✅
**Full spec**: [CounterpartyManager.md](CounterpartyManager.md)

**Purpose**: Full CRUD UI for counterparties with duplicate suggestions and bulk merge

**Key Features**:
- List view with search across canonical names AND aliases
- Duplicate suggestions panel with similarity scores (Levenshtein algorithm)
- Multi-select for bulk merge operations
- Edit modal (canonical name, type, aliases, notes)
- Transaction count display per counterparty
- Filter by type (merchant, person, business, government)
- Group by type with smart ordering

**Reusability**:
- Finance: Manage merchants and payees
- Healthcare: Manage healthcare providers
- Legal: Manage opposing parties and law firms
- Research: Manage publishers and co-authors
- Manufacturing: Manage suppliers and vendors
- Media: Manage advertisers and sponsors

---

#### 15. CounterpartySelector ✅
**Full spec**: [CounterpartySelector.md](CounterpartySelector.md)

**Purpose**: Dropdown for selecting counterparty with alias search and grouping

**Key Features**:
- Search across canonical names + aliases with highlighting
- Group by type (person, merchant, business, government)
- Show transaction count (helps users pick the right one)
- Create inline (add new counterparty mid-flow)
- Auto-sort by transaction count (most used first)
- Keyboard navigation
- Size variants (small, medium, large)

**Reusability**:
- Finance: Select merchant in transaction edit, filter by counterparty
- Healthcare: Select provider in claim edit
- Legal: Select opposing party in case allocation
- Research: Select publisher in citation management
- Manufacturing: Select supplier in purchase order
- Media: Select advertiser in campaign analytics

---

#### 16. MergeCounterpartiesDialog ✅
**Full spec**: [MergeCounterpartiesDialog.md](MergeCounterpartiesDialog.md)

**Purpose**: 5-step wizard for merging duplicate counterparties

**Key Features**:
- **Step 1**: Select primary counterparty (keeps its canonical name)
- **Step 2**: Preview combined aliases + total transaction count
- **Step 3**: Confirm with type-to-confirm ("MERGE") + irreversible warning
- **Step 4**: Progress indicator during merge (batch processing)
- **Step 5**: Success screen with stats
- Visual progress indicator at top of dialog
- Transaction migration shown in real-time

**Reusability**:
- Finance: Merge duplicate merchants ("OpenAI" + "OpenAI Inc")
- Healthcare: Merge duplicate providers ("Blue Cross" + "BCBS")
- Legal: Merge duplicate parties ("Smith Law" + "Smith & Associates")
- Research: Merge duplicate publishers ("Nature" + "Nature Publishing")
- Manufacturing: Merge duplicate suppliers ("Acme Corp" + "ACME")
- Media: Merge duplicate advertisers ("YouTube" + "YouTube Inc")

---

### Vertical 3.3 (Series Registry)

#### 17. SeriesManager ✅
**Full spec**: [SeriesManager.md](SeriesManager.md)

**Purpose**: Full CRUD UI for managing recurring payment series with status badges and variance alerts

**Key Features**:
- List view with status badges (✅ Paid on time, ⚠️ Amount variance, 🔴 Missing, 📅 Upcoming)
- Create/Edit modal with RecurrenceConfigDialog integration (2-step flow)
- Instance history view (last 12 months of expected vs actual)
- Variance alerts panel (dashboard widget showing overdue/missing payments)
- Filter by account, category, status
- Search by series name
- Group by category, account, or status
- Archive confirmation dialog (preserves instance history)
- Manual link transaction to series
- Keyboard shortcuts (N = new series, ESC = close)

**Visual (List View with Status)**:
```
┌──────────────────────────────────────────────────────────┐
│ 🔍 Search series...                    [+ New Series]    │
├──────────────────────────────────────────────────────────┤
│ SUBSCRIPTIONS (3)                                        │
│   ┌────────────────────────────────────────────────┐     │
│   │ 💻 OpenAI ChatGPT Plus    ✅ Paid on time      │     │
│   │    $20.00/month · Next: Nov 5, 2024       [⚙️]  │     │
│   └────────────────────────────────────────────────┘     │
│   ┌────────────────────────────────────────────────┐     │
│   │ 📺 Netflix Premium        🔴 Missing (3 days)  │     │
│   │    $15.99/month · Expected: Nov 15         [⚙️]  │     │
│   └────────────────────────────────────────────────┘     │
│   ┌────────────────────────────────────────────────┐     │
│   │ 🎵 Spotify Premium        ⚠️ Amount variance   │     │
│   │    $9.99/month · Paid $10.99              [⚙️]  │     │
│   └────────────────────────────────────────────────┘     │
│                                                          │
│ BILLS (2)                                                │
│   ┌────────────────────────────────────────────────┐     │
│   │ 🏠 Rent - Monthly         ✅ Paid on time      │     │
│   │    $1,200.00/month · Next: Dec 1          [⚙️]  │     │
│   └────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Manage subscriptions, bills, recurring income
- Healthcare: Manage insurance premiums, prescription refills
- Legal: Manage monthly retainers, recurring court fees
- Research: Manage recurring grant disbursements, lab subscriptions
- Manufacturing: Manage equipment lease payments, maintenance schedules
- Media: Manage licensing fees, hosting subscriptions

---

#### 18. SeriesSelector ✅
**Full spec**: [SeriesSelector.md](SeriesSelector.md)

**Purpose**: Dropdown for selecting series to link to transaction (used in transaction detail, manual link dialog)

**Key Features**:
- Search by series name (fuzzy matching)
- Group by category (subscriptions, bills, income, other)
- Show next expected date + amount for each series
- "Create new series" inline option
- Filter by account (optional prop)
- Show transaction count per series (optional)
- Keyboard navigation (↑↓ select, Enter confirm, Esc close)
- Auto-suggest based on counterparty + amount
- Size variants (small, medium, large)

**Visual (Expanded)**:
```
┌─────────────────────────────────────┐
│ 🔍 Search series...                 │
├─────────────────────────────────────┤
│ SUBSCRIPTIONS                       │
│ ● 💻 OpenAI ChatGPT Plus            │
│      $20.00/month · Next: Nov 5     │
│   📺 Netflix Premium                │
│      $15.99/month · Next: Nov 15    │
├─────────────────────────────────────┤
│ BILLS                               │
│   🏠 Rent - Monthly                 │
│      $1,200.00/month · Next: Dec 1  │
│   ⚡ Electricity (CFE)              │
│      ~$80.00/month · Next: Nov 20   │
├─────────────────────────────────────┤
│ [+ Create New Series]               │
└─────────────────────────────────────┘
```

**Reusability**:
- Finance: Select series in transaction edit, link payment to subscription
- Healthcare: Select insurance premium series in claim edit
- Legal: Select retainer series in payment allocation
- Research: Select grant series in expense reporting
- Manufacturing: Select lease series in accounting entry
- Media: Select license series in revenue tracking

---

#### 19. RecurrenceConfigDialog ✅
**Full spec**: [RecurrenceConfigDialog.md](RecurrenceConfigDialog.md)

**Purpose**: Dialog for configuring recurrence patterns (daily, weekly, monthly, yearly, custom)

**Key Features**:
- Frequency type selector with visual icons (Daily, Weekly, Monthly, Yearly, Custom)
- Conditional fields based on type:
  - **Daily**: Interval (every N days)
  - **Weekly**: Day of week + interval (every N weeks on Tuesday)
  - **Monthly**: Day of month + interval (every N months on 5th)
  - **Yearly**: Month + day (every year on Jan 15)
  - **Custom**: Date list (explicit dates for irregular schedules)
- Preview next 3 occurrences (validates pattern before saving)
- Edge case warnings:
  - "Day 31 will adjust to last day of month in Feb (Feb 28/29)"
  - "Feb 29 only occurs in leap years (next: 2028)"
- Validation rules:
  - Prevent invalid dates (e.g., Feb 30)
  - Interval must be > 0
  - Custom dates must be chronological
- Keyboard navigation (Tab through fields, Enter to save, Esc to cancel)

**Visual (Monthly Pattern)**:
```
┌──────────────────────────────────────────────────┐
│ Configure Recurrence                             │
├──────────────────────────────────────────────────┤
│ Frequency Type                                   │
│ [ Daily ] [ Weekly ] [Monthly*] [Yearly] [Custom]│
│                                                  │
│ Day of Month: [5]                                │
│ Every: [1] month(s)                              │
│                                                  │
│ Preview Next 3 Occurrences:                      │
│ · Dec 5, 2024                                    │
│ · Jan 5, 2025                                    │
│ · Feb 5, 2025                                    │
│                                                  │
│           [Cancel]         [Save]                │
└──────────────────────────────────────────────────┘
```

**Visual (Edge Case Warning)**:
```
┌──────────────────────────────────────────────────┐
│ Configure Recurrence                             │
├──────────────────────────────────────────────────┤
│ Day of Month: [31]                               │
│                                                  │
│ ⚠️ Note: Day 31 will adjust to the last day of  │
│ months with fewer days (e.g., Feb 28/29, Apr 30) │
│                                                  │
│ Preview Next 3 Occurrences:                      │
│ · Dec 31, 2024                                   │
│ · Jan 31, 2025                                   │
│ · Feb 28, 2025 (adjusted)                        │
└──────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Configure payment schedules, income patterns
- Healthcare: Configure medication refill schedules, therapy sessions
- Legal: Configure recurring court dates, retainer billing
- Research: Configure grant disbursement schedules, lab maintenance
- Manufacturing: Configure equipment maintenance schedules, inventory audits
- Media: Configure content publication schedules, license renewals

---

### Vertical 3.4 (Tax Categorization)

#### 20. TaxCategorySelector ✅
**Full spec**: [TaxCategorySelector.md](TaxCategorySelector.md)

**Purpose**: Dropdown for selecting tax category with hierarchical display and auto-suggestions

**Key Features**:
- Hierarchical category display (parent → children indentation)
- Full-text search across category names and descriptions
- Jurisdiction filter (USA Federal, Mexico Federal, etc.)
- Deduction rate badges (100%, 50%, 0%)
- Auto-suggestions with confidence scores (95% = very high, 70-79% = medium)
- Group by taxonomy (Schedule C, SAT, etc.)
- Keyboard navigation (↑↓ select, Enter confirm, Esc close)
- Category path display ("Schedule C > Office Expenses > Software")
- Size variants (small, medium, large)

**Visual (Expanded with Auto-Suggestions)**:
```
┌──────────────────────────────────────────┐
│ 🔍 Search categories...                  │
├──────────────────────────────────────────┤
│ 💡 SUGGESTED (based on "Uber")           │
│ ● Travel (95% confidence)      100% 📗   │
│   Car and Truck (65%)          100% 📗   │
├──────────────────────────────────────────┤
│ SCHEDULE C (USA FEDERAL)                 │
│   Advertising                  100% 📗   │
│   Office Expenses              100% 📗   │
│   Meals (Business)              50% 📒   │
│   Travel                       100% 📗   │
│   Personal (Non-Deductible)      0% 📕   │
├──────────────────────────────────────────┤
│ SAT (MEXICO FEDERAL)                     │
│   Gastos de Publicidad         100% 📗   │
│   Gastos de Gestión            100% 📗   │
└──────────────────────────────────────────┘
```

**Reusability**:
- Finance: Select tax category (Schedule C, SAT)
- Healthcare: Select CPT code, ICD-10 diagnosis code
- Legal: Select case category, filing type
- Research: Select grant expense category, compliance classification
- Manufacturing: Select safety category, environmental classification
- Media: Select content classification, licensing category

---

#### 21. TaxCategoryManager ✅
**Full spec**: [TaxCategoryManager.md](TaxCategoryManager.md)

**Purpose**: Full UI for managing custom tax categories with tree view

**Key Features**:
- Tree view of categories (hierarchical display with expand/collapse)
- Create/edit custom categories modal (within jurisdiction)
- System vs user category distinction (🔒 system, ✏️ custom)
- Deduction rate editor with visual slider
- Category search and filter by jurisdiction
- Usage statistics (transaction count per category)
- Bulk operations (merge, archive)
- Parent category selector (must be in same jurisdiction)
- SAT code field for Mexico categories
- Empty states ("No custom categories yet - create your first!")
- Keyboard shortcuts (N = new category, ESC = close)

**Visual (Tree View)**:
```
┌──────────────────────────────────────────────────────────┐
│ 🔍 Search categories...           [+ New Category]       │
├──────────────────────────────────────────────────────────┤
│ Jurisdiction: [USA Federal ▼]     Show: [All ▼]          │
├──────────────────────────────────────────────────────────┤
│ 📁 SCHEDULE C (USA FEDERAL)                              │
│   ├─ 🔒 Advertising (100%)                    42 txns    │
│   ├─ 🔒 Office Expenses (100%)                28 txns    │
│   │   └─ ✏️ Software Subscriptions (100%)     15 txns    │
│   ├─ 🔒 Meals (Business) (50%)                18 txns    │
│   ├─ 🔒 Travel (100%)                         35 txns    │
│   │   ├─ ✏️ Client Meetings (100%)            20 txns    │
│   │   └─ ✏️ Conferences (100%)                 8 txns    │
│   └─ 🔒 Personal (Non-Deductible) (0%)        12 txns    │
└──────────────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Manage tax categories (Schedule C, SAT)
- Healthcare: Manage CPT/ICD-10 codes, custom billing categories
- Legal: Manage case types, jurisdiction-specific filings
- Research: Manage grant categories, custom compliance classifications
- Manufacturing: Manage safety categories, environmental permits
- Media: Manage content categories, licensing classifications

---

#### 22. FacturaUploadDialog ✅
**Full spec**: [FacturaUploadDialog.md](FacturaUploadDialog.md)

**Purpose**: Upload and link regulatory documents (Factura XML) to transactions

**Key Features**:
- Drag-drop XML file upload (CFDI 3.3 and 4.0)
- Real-time XML parsing and validation
- Display Factura details (RFC, UUID, amount, date, emisor)
- Validation feedback with specific error messages:
  - RFC format validation (12-13 alphanumeric)
  - UUID format validation (UUID v4)
  - Amount tolerance check (±5% variance allowed)
  - Date tolerance check (±7 days from transaction)
  - Digital signature validation (optional)
- Transaction linking (auto or manual)
- Force link option for manual override (amount/date mismatch)
- Orphan Factura support (upload before transaction exists)
- Download original XML and PDF
- Loading states, error states, success states
- Keyboard navigation (Tab, Enter, Esc)

**Visual (Upload Success with Validation)**:
```
┌──────────────────────────────────────────────────┐
│ Upload Factura (CFDI)                      [×]   │
├──────────────────────────────────────────────────┤
│ ✅ Factura validated successfully                │
│                                                  │
│ RFC: ABC123456XYZ             ✓ Valid format    │
│ UUID: 12345678-1234...        ✓ Valid format    │
│ Amount: $500.00 MXN           ✓ Matches ($500)  │
│ Date: 2024-11-01              ✓ Within ±7 days  │
│ Emisor: Acme Corp SA de CV                       │
│ CFDI Version: 4.0                                │
│                                                  │
│ Linked Transaction:                              │
│ 2024-11-01 · Acme Corp · -$500.00 MXN            │
│                                                  │
│ [Download XML] [Download PDF]    [Done]         │
└──────────────────────────────────────────────────┘
```

**Visual (Validation Error - Amount Mismatch)**:
```
┌──────────────────────────────────────────────────┐
│ Upload Factura (CFDI)                      [×]   │
├──────────────────────────────────────────────────┤
│ ⚠️ Validation warnings detected                  │
│                                                  │
│ RFC: DEF987654ABC             ✓ Valid format    │
│ UUID: 87654321-4321...        ✓ Valid format    │
│ Amount: $520.00 MXN           ⚠️ Variance: 4.0% │
│   Transaction shows: $500.00 MXN                 │
│ Date: 2024-11-01              ✓ Within ±7 days  │
│                                                  │
│ ☐ Force link despite amount mismatch            │
│                                                  │
│ [Cancel]             [Force Link Anyway]         │
└──────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Upload Factura (Mexico CFDI), tax receipts
- Healthcare: Upload CMS-1500 claim forms, EOB (Explanation of Benefits)
- Legal: Upload court filings (XML), legal briefs
- Research: Upload budget justifications, grant agreements
- Manufacturing: Upload safety inspection reports, environmental permits
- Media: Upload licensing agreements, content rights documentation

---

### Vertical 3.5 (Relationships)

#### 23. RelationshipPanel ✅
**Full spec**: [RelationshipPanel.md](RelationshipPanel.md)

**Props**: `transactionId`, `relationships[]`, `onUnlink`, `onNavigate`
**States**: loading | empty | single | multiple | unlinking
**Reusable across**: Finance transfers, Healthcare claim-payment, Legal case relationships, Research citations, E-commerce order-return

**Purpose**: Display linked transactions in transaction detail view with relationship types (🔗 transfer, 💱 fx_conversion, 💰 reimbursement, ✂️ split, 🔄 correction).

**Visual (Single Transfer Link)**:
```
┌─────────────────────────────────────────────────────┐
│ 🔗 Linked Transactions                              │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Transfer (95% confidence)                           │
│ ┌───────────────────────────────────────────────┐  │
│ │ → Wise USD                                    │  │
│ │   +$1,000.00 · 2025-10-15                     │  │
│ │   Auto-detected                               │  │
│ │                                    [Unlink]   │  │
│ └───────────────────────────────────────────────┘  │
│                                                     │
│ Click transaction to view details                   │
└─────────────────────────────────────────────────────┘
```

**Visual (FX Conversion with Gain)**:
```
┌─────────────────────────────────────────────────────┐
│ 🔗 Linked Transactions                              │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 💱 FX Conversion (98% confidence)                   │
│ ┌───────────────────────────────────────────────┐  │
│ │ → Wise MXN                                    │  │
│ │   +$18,500.00 MXN · 2025-10-16                │  │
│ │                                               │  │
│ │   Rate: 18.5000 MXN/USD                       │  │
│ │   Market: 18.3000 MXN/USD                     │  │
│ │   ✅ Gain: +$200 MXN (↑1.09%)                 │  │
│ │   Auto-detected                               │  │
│ │                                    [Unlink]   │  │
│ └───────────────────────────────────────────────┘  │
│                                                     │
│ [View FX Report]                                    │
└─────────────────────────────────────────────────────┘
```

**Visual (Transfer Chain - Multiple Links)**:
```
┌─────────────────────────────────────────────────────┐
│ 🔗 Linked Transactions                              │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Transfer Chain (3 links)                            │
│ ┌───────────────────────────────────────────────┐  │
│ │ 1. BofA Checking → Wise USD                   │  │
│ │    Transfer · -$1,000 → +$1,000 · 95%         │  │
│ ├───────────────────────────────────────────────┤  │
│ │ 2. Wise USD → Wise MXN                        │  │
│ │    FX Conv · -$1,000 → +$18,500 · 98%         │  │
│ ├───────────────────────────────────────────────┤  │
│ │ 3. Wise MXN → Scotia MXN                      │  │
│ │    Transfer · -$18,500 → +$18,500 · 95%       │  │
│ └───────────────────────────────────────────────┘  │
│                                                     │
│ [Export Chain]                                      │
└─────────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Transfer visualization, FX tracking, reimbursement linking
- Healthcare: Insurance claim → payment relationship, referral → visit tracking
- Legal: Case → subcase relationships, document → amendment linking
- Research: Paper → citation network, dataset → derived dataset lineage
- E-commerce: Order → return → refund chain visualization
- Manufacturing: Work order → subassembly → final product BOM tracking

---

#### 24. TransferLinkDialog ✅
**Full spec**: [TransferLinkDialog.md](TransferLinkDialog.md)

**Props**: `isOpen`, `onClose`, `currentTransaction`, `onCreate`
**States**: search | searching | results | preview | creating | error
**Reusable across**: Finance manual links, Healthcare claim linking, Legal document versions, Research citation management

**Purpose**: Manual link creation dialog with transaction search, relationship type selection (6 types), and preview before creation.

**Visual (Search Step)**:
```
┌─────────────────────────────────────────────────────┐
│ Link Transaction                              [×]   │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Current Transaction:                                │
│ Personal Card · -$47.32 · 2025-10-10                │
│ "Client dinner - expense report"                    │
│                                                     │
│ ─────────────────────────────────────────────────   │
│                                                     │
│ Search for transaction to link:                     │
│ ┌─────────────────────────────────────────────┐    │
│ │ Search transactions... 🔍                   │    │
│ └─────────────────────────────────────────────┘    │
│                                                     │
│ Advanced Filters ▼                                  │
│ Date Range: [2025-10-01] to [2025-10-31]           │
│ Amount Range: [$0] to [$1000]                       │
│ Account: [All accounts ▼]                           │
│                                                     │
│ Results (3):                                        │
│ ┌─────────────────────────────────────────────┐    │
│ │ ✓ Business Checking · +$50.00 · 2025-10-20  │    │
│ │   "Expense reimbursement Oct"               │    │
│ ├─────────────────────────────────────────────┤    │
│ │   Work Visa · +$250.00 · 2025-10-18         │    │
│ │   "Travel expense reimbursement"            │    │
│ ├─────────────────────────────────────────────┤    │
│ │   Personal Checking · +$47.32 · 2025-10-22  │    │
│ │   "Reimbursement from employer"             │    │
│ └─────────────────────────────────────────────┘    │
│                                                     │
│                            [Cancel] [Next]          │
└─────────────────────────────────────────────────────┘
```

**Visual (Preview Step - Reimbursement)**:
```
┌─────────────────────────────────────────────────────┐
│ Link Transaction                              [×]   │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Link Preview:                                       │
│ ┌─────────────────────────────────────────────┐    │
│ │ From: Personal Card · -$47.32 · 2025-10-10  │    │
│ │                                              │    │
│ │         ↓ 💰 Reimbursement ↓                 │    │
│ │                                              │    │
│ │ To:   Business Checking · +$50.00           │    │
│ │       · 2025-10-20                           │    │
│ └─────────────────────────────────────────────┘    │
│                                                     │
│ Relationship Type:                                  │
│ ● Reimbursement                                     │
│ ○ Transfer                                          │
│ ○ FX Conversion                                     │
│ ○ Split                                             │
│ ○ Correction                                        │
│ ○ Other                                             │
│                                                     │
│ Notes (optional):                                   │
│ ┌─────────────────────────────────────────────┐    │
│ │ Employer rounds to nearest $5               │    │
│ └─────────────────────────────────────────────┘    │
│                                                     │
│ ⚠️ Amount differs by $2.68 (5.7%)                   │
│                                                     │
│                      [Back] [Cancel] [Create Link]  │
└─────────────────────────────────────────────────────┘
```

**Visual (Mobile - Stacked Layout)**:
```
┌───────────────────────────┐
│ Link Transaction    [×]   │
├───────────────────────────┤
│ Current:                  │
│ Personal Card             │
│ -$47.32 · 2025-10-10      │
│                           │
│ ───────────────────       │
│                           │
│ Search:                   │
│ ┌───────────────────┐    │
│ │ Search... 🔍      │    │
│ └───────────────────┘    │
│                           │
│ Filters ▼                 │
│                           │
│ Results (3):              │
│ ┌───────────────────┐    │
│ │ ✓ Business Check  │    │
│ │ +$50 · 2025-10-20 │    │
│ └───────────────────┘    │
│                           │
│ [Cancel] [Next]           │
└───────────────────────────┘
```

**Reusability**:
- Finance: Link reimbursements, split expenses, corrections, FX conversions
- Healthcare: Link insurance claims to payments, referrals to specialist visits
- Legal: Link document versions (original → amended), case relationships
- Research: Link papers to citations, datasets to derived works
- E-commerce: Link orders to returns, refunds to original payments
- HR: Link expense reports to approvals and reimbursements

---

#### 25. FXConversionCard ✅
**Full spec**: [FXConversionCard.md](FXConversionCard.md)

**Props**: `relationship`, `fxDetails`, `showMarketRateComparison`
**States**: basic | comparison_gain | comparison_loss | loading | error | compact
**Reusable across**: Finance FX tracking, Healthcare international billing, Research grant conversions, Manufacturing material costs

**Purpose**: Display FX conversion details with exchange rate, market rate comparison, and gain/loss calculation with color coding.

**Visual (Basic - No Market Rate)**:
```
┌──────────────────────────────────────────────────┐
│ 💱 FX Conversion                                 │
├──────────────────────────────────────────────────┤
│                                                  │
│ From: $1,000.00 USD                              │
│  ↓   18.5000 MXN/USD (calculated)                │
│ To:   $18,500.00 MXN                             │
│                                                  │
│ Rate Source: Calculated from transaction amounts │
│                                                  │
│ [View FX Report]                                 │
└──────────────────────────────────────────────────┘
```

**Visual (Comparison - Gain)**:
```
┌──────────────────────────────────────────────────┐
│ 💱 FX Conversion                                 │
├──────────────────────────────────────────────────┤
│                                                  │
│ From: $1,000.00 USD                              │
│  ↓   18.5000 MXN/USD (calculated)                │
│ To:   $18,500.00 MXN                             │
│                                                  │
│ Market Rate: 18.3000 MXN/USD                     │
│ (exchangerate.host · 2025-10-16)                 │
│                                                  │
│ ✅ FX Gain: +$200.00 MXN (↑ 1.09%)               │
│                                                  │
│ Expected: $18,300 MXN (at market rate)           │
│ Actual:   $18,500 MXN                            │
│ Gain:     +$200 MXN                              │
│                                                  │
│ [View FX Report] [View Rate History]            │
└──────────────────────────────────────────────────┘
```

**Visual (Comparison - Loss)**:
```
┌──────────────────────────────────────────────────┐
│ 💱 FX Conversion                                 │
├──────────────────────────────────────────────────┤
│                                                  │
│ From: $5,000.00 USD                              │
│  ↓   0.9500 EUR/USD (calculated)                 │
│ To:   €4,750.00 EUR                              │
│                                                  │
│ Market Rate: 0.9200 EUR/USD                      │
│ (fixer.io · 2025-10-10)                          │
│                                                  │
│ ❌ FX Loss: -€150.00 EUR (↓ 3.26%)               │
│                                                  │
│ Expected: €4,600 EUR (at market rate)            │
│ Actual:   €4,750 EUR                             │
│ Loss:     -€150 EUR                              │
│                                                  │
│ [View FX Report]                                 │
└──────────────────────────────────────────────────┘
```

**Visual (Mobile - Compact)**:
```
┌───────────────────────────┐
│ 💱 FX Conversion          │
├───────────────────────────┤
│ $1,000 USD                │
│ ↓ 18.5000 MXN/USD         │
│ $18,500 MXN               │
│                           │
│ Market: 18.3000           │
│ ✅ +$200 (↑1.09%)         │
│                           │
│ [View Report]             │
└───────────────────────────┘
```

**Visual (Warning - High Variance)**:
```
┌──────────────────────────────────────────────────┐
│ 💱 FX Conversion                                 │
├──────────────────────────────────────────────────┤
│                                                  │
│ From: $1,000.00 USD                              │
│  ↓   1.1000 EUR/USD (calculated)                 │
│ To:   €1,100.00 EUR                              │
│                                                  │
│ Market Rate: 0.9500 EUR/USD                      │
│ (fixer.io · 2025-10-10)                          │
│                                                  │
│ ⚠️ VARIANCE WARNING: Rate differs by 15.79%      │
│                                                  │
│ This rate is significantly different from market │
│ rate. Verify transaction data before linking.    │
│                                                  │
│ [View FX Report] [Report Issue]                 │
└──────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Currency trading FX analysis, multi-currency account tracking
- Healthcare: International patient billing (USD → EUR), medical tourism pricing
- Research: Grant conversions (GBP → USD), international collaboration budgets
- Manufacturing: Raw material costs (CNY → USD), import/export pricing
- E-commerce: International sales (customer currency → base currency)
- Real Estate: Overseas property transactions (local → home currency)
- Energy: Commodity price conversions (oil barrels, natural gas)
- Travel: Expense tracking across multiple currencies

---

### Vertical 3.6 (Unit - Currency & Date Normalization)

#### 26. CurrencySelectorDialog ✅
**Full spec**: [CurrencySelectorDialog.md](CurrencySelectorDialog.md)

**Props**: `isOpen`, `onClose`, `onSelect`, `currentCurrency`, `includeCrypto`
**States**: idle | loading | loaded | error
**Reusable across**: Finance settings, Healthcare billing, E-commerce pricing, Travel expenses, Research grants

**Purpose**: Modal dialog for selecting base currency from 150+ ISO 4217 currencies with search and popular currencies first (USD, EUR, MXN, GBP, JPY).

**Visual (Default View)**:
```
┌─────────────────────────────────────────────────┐
│  Select Base Currency                        × │
├─────────────────────────────────────────────────┤
│                                                 │
│  🔍 [Search currencies...                   ]  │
│                                                 │
│  ✨ Popular Currencies                          │
│  ┌─────────────────────────────────────────┐   │
│  │ 🇺🇸 USD - US Dollar              [✓]   │   │
│  │ 🇪🇺 EUR - Euro                          │   │
│  │ 🇲🇽 MXN - Mexican Peso                  │   │
│  │ 🇬🇧 GBP - British Pound                 │   │
│  │ 🇯🇵 JPY - Japanese Yen                  │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  All Currencies (A-Z)                           │
│  ┌─────────────────────────────────────────┐   │
│  │ 🇦🇺 AUD - Australian Dollar             │   │
│  │ 🇨🇦 CAD - Canadian Dollar               │   │
│  │ 🇨🇭 CHF - Swiss Franc                   │   │
│  │ ... (scrollable)                         │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│                    [Cancel]  [Select]           │
└─────────────────────────────────────────────────┘
```

**Visual (Search Active)**:
```
┌─────────────────────────────────────────────────┐
│  Select Base Currency                        × │
├─────────────────────────────────────────────────┤
│                                                 │
│  🔍 [mex                                     ]  │
│                                                 │
│  Search Results (1):                            │
│  ┌─────────────────────────────────────────┐   │
│  │ 🇲🇽 MXN - Mexican Peso                  │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│                    [Cancel]  [Select]           │
└─────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: User settings (change base currency), transaction override
- Healthcare: International patient billing (select patient currency)
- E-commerce: Product pricing (set display currency for customer)
- Travel: Expense tracking (select local currency for trip)
- Research: Grant management (select funding currency)
- Manufacturing: Material costs (select supplier currency)
- Real Estate: Property transactions (select local market currency)

---

#### 27. AmountDisplayCard ✅
**Full spec**: [AmountDisplayCard.md](AmountDisplayCard.md)

**Props**: `amount`, `originalCurrency`, `baseCurrency`, `exchangeRate`, `showOriginal`, `onToggle`
**States**: normalized_only | dual_currency | loading | error | compact
**Reusable across**: Finance transactions, Healthcare claims, E-commerce orders, Travel expenses, Research budgets

**Purpose**: Display transaction amount in both original currency and user's base currency with side-by-side comparison and toggle to show/hide original.

**Visual (Dual Currency)**:
```
┌────────────────────────────────────────────┐
│  Amount                                    │
├────────────────────────────────────────────┤
│                                            │
│  Original:       $1,000.00 USD             │
│  ↓ 18.5000 MXN/USD                         │
│  Your Currency:  $18,500.00 MXN            │
│                                            │
│  [Hide Original]                           │
└────────────────────────────────────────────┘
```

**Visual (Normalized Only)**:
```
┌────────────────────────────────────────────┐
│  Amount                                    │
├────────────────────────────────────────────┤
│                                            │
│  $18,500.00 MXN                            │
│                                            │
│  [Show Original]                           │
└────────────────────────────────────────────┘
```

**Visual (Compact - Mobile)**:
```
┌─────────────────────────┐
│  Amount                 │
├─────────────────────────┤
│  $1,000 USD             │
│  ↓ 18.5000              │
│  $18,500 MXN            │
│                         │
│  [Hide Original]        │
└─────────────────────────┘
```

**Reusability**:
- Finance: Transaction detail view, account balances (multi-currency accounts)
- Healthcare: International insurance claims (claim currency → patient currency)
- E-commerce: Order totals (customer currency → merchant base currency)
- Travel: Expense reports (local currency → home currency)
- Research: Grant budgets (grant currency → institutional currency)
- Manufacturing: Material costs (supplier currency → company currency)
- Payroll: International contractor payments (contractor currency → company currency)

---

#### 28. ExchangeRateWidget ✅
**Full spec**: [ExchangeRateWidget.md](ExchangeRateWidget.md)

**Props**: `fromCurrency`, `toCurrency`, `rate`, `rateDate`, `rateSource`, `onRefresh`, `isStale`
**States**: current | stale | refreshing | error | compact
**Reusable across**: Finance dashboards, Healthcare billing, E-commerce admin, Travel planning, Research budgets

**Purpose**: Display current exchange rate with last update time, rate source badge (ECB, Federal Reserve, manual), staleness indicator, and refresh button.

**Visual (Current Rate)**:
```
┌────────────────────────────────────────────┐
│  💱 Exchange Rate                          │
├────────────────────────────────────────────┤
│                                            │
│  USD → MXN                                 │
│  18.5000 MXN/USD                           │
│                                            │
│  Source: ECB                               │
│  Last updated: 2025-10-24 10:30 AM         │
│                                            │
│  [🔄 Refresh]                              │
└────────────────────────────────────────────┘
```

**Visual (Stale Rate Warning)**:
```
┌────────────────────────────────────────────┐
│  💱 Exchange Rate                     ⚠️   │
├────────────────────────────────────────────┤
│                                            │
│  USD → EUR                                 │
│  0.9200 EUR/USD                            │
│                                            │
│  Source: Federal Reserve                   │
│  Last updated: 2025-10-22 (2 days ago)     │
│                                            │
│  ⚠️ Rate is stale (>24h old)               │
│  [🔄 Refresh Now]                          │
└────────────────────────────────────────────┘
```

**Visual (Manual Override)**:
```
┌────────────────────────────────────────────┐
│  💱 Exchange Rate                     ✏️   │
├────────────────────────────────────────────┤
│                                            │
│  GBP → USD                                 │
│  1.2500 USD/GBP                            │
│                                            │
│  Source: Manual Override                   │
│  Set by: user_darwin                       │
│  Set on: 2025-10-24 09:00 AM               │
│                                            │
│  [Edit Rate] [Use Market Rate]             │
└────────────────────────────────────────────┘
```

**Visual (Compact - Dashboard Widget)**:
```
┌─────────────────────────┐
│  💱 USD → MXN           │
├─────────────────────────┤
│  18.5000                │
│  ECB · 10:30 AM         │
│  [🔄]                   │
└─────────────────────────┘
```

**Reusability**:
- Finance: Multi-currency dashboard, FX trading view, account summary
- Healthcare: International billing dashboard (claims in multiple currencies)
- E-commerce: Admin panel (product pricing across regions)
- Travel: Expense tracker dashboard (real-time rate display)
- Research: Grant management (budget tracking across currencies)
- Manufacturing: Material cost dashboard (supplier currencies → base currency)
- Treasury: Corporate treasury dashboard (FX exposure monitoring)
- Banking: Customer-facing rate display (transfers, wire services)

---

### Vertical 3.7 (Parser Registry)

#### 29. ParserSelectorDialog ✅
**Full spec**: [ParserSelectorDialog.md](ParserSelectorDialog.md)

**Props**: `isOpen`, `file`, `availableParsers[]`, `onSelect`, `onCancel`
**States**: loading | loaded | selecting | error
**Reusable across**: Finance PDF/CSV parsers, Healthcare HL7/FHIR parsers, Legal document parsers, Research citation parsers, E-commerce EDI parsers

**Purpose**: Modal dialog for manual parser selection when auto-detection has low confidence or user wants to override. Shows parser list with capability badges and confidence scores.

**Visual (Parser List)**:
```
┌─────────────────────────────────────────────────┐
│  Select Parser                               × │
├─────────────────────────────────────────────────┤
│                                                 │
│  File: BofA_Statement_Nov2024.pdf               │
│  Type: PDF                                      │
│                                                 │
│  Available Parsers:                             │
│  ┌─────────────────────────────────────────┐   │
│  │ ● Bank of America PDF Parser        98% │   │
│  │   v2.1.0 · Latest                       │   │
│  │   ✓ date, amount, merchant, account     │   │
│  ├─────────────────────────────────────────┤   │
│  │   Chase PDF Parser                  45% │   │
│  │   v1.5.0 · May not match format         │   │
│  │   ✓ date, amount, merchant              │   │
│  ├─────────────────────────────────────────┤   │
│  │   Generic Bank PDF Parser           30% │   │
│  │   v3.0.0 · Fallback option              │   │
│  │   ✓ date, amount                        │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ℹ️ Auto-selected: Bank of America PDF Parser  │
│                                                 │
│                      [Cancel] [Use Selected]    │
└─────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: PDF parsers (BofA, Chase, Scotia), CSV parsers (Apple Card, Mint)
- Healthcare: HL7 v2.5 ADT parser, FHIR R4 parser, DICOM parser
- Legal: Court filing PDF parser, contract DOCX parser
- Research: BibTeX citation parser, LaTeX parser
- E-commerce: EDI X12 4010 invoice parser, XML invoice parser
- Any domain: Service/parser selection UI pattern

---

#### 30. ParserCapabilitiesCard ✅
**Full spec**: [ParserCapabilitiesCard.md](ParserCapabilitiesCard.md)

**Props**: `parser`, `capabilities[]`, `showConfidence`
**States**: collapsed | expanded | loading
**Reusable across**: Finance parsers, Healthcare protocol handlers, Legal document parsers, Research format parsers

**Purpose**: Display parser capabilities card showing extractable fields, supported file types, and extraction confidence scores.

**Visual (Expanded)**:
```
┌────────────────────────────────────────────┐
│  Parser Capabilities                       │
├────────────────────────────────────────────┤
│                                            │
│  BofA PDF Parser v2.1.0                    │
│                                            │
│  Extractable Fields:                       │
│  ✓ date          98% confidence            │
│  ✓ amount        99% confidence            │
│  ✓ merchant      95% confidence            │
│  ✓ account       97% confidence            │
│  ✓ balance       90% confidence            │
│                                            │
│  Supported Files:                          │
│  • PDF (.pdf)                              │
│                                            │
│  Filename Patterns:                        │
│  • *BofA*.pdf                              │
│  • *Bank of America*.pdf                   │
│                                            │
│  [View Version History]                    │
└────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Display parser capabilities before upload
- Healthcare: Show protocol handler capabilities (HL7 message types)
- Legal: Display document parser supported fields
- Research: Show citation parser extractable metadata
- E-commerce: Display EDI parser transaction sets
- Microservices: Show API endpoint capabilities

---

#### 31. ParserVersionDropdown ✅
**Full spec**: [ParserVersionDropdown.md](ParserVersionDropdown.md)

**Props**: `parserId`, `versions[]`, `currentVersion`, `onVersionChange`, `showDeprecated`
**States**: idle | loading | deprecated_warning | error
**Reusable across**: Finance parsers, Healthcare protocol versions, Legal document formats, Research citation formats

**Purpose**: Dropdown for selecting parser version with deprecation warnings and breaking change indicators.

**Visual (Version List)**:
```
┌────────────────────────────────────────────┐
│  Parser Version          ▼                 │
├────────────────────────────────────────────┤
│  ● v2.1.0  (Latest) ✨                     │
│    v2.0.0                                  │
│    v1.5.0  (Deprecated) ⚠️                 │
│    v1.0.0  (Breaking changes) 🔴           │
└────────────────────────────────────────────┘
```

**Visual (Deprecated Warning)**:
```
┌────────────────────────────────────────────┐
│  ⚠️ Version Deprecated                     │
├────────────────────────────────────────────┤
│                                            │
│  v1.5.0 will be sunset on 2025-12-31      │
│                                            │
│  Please upgrade to v2.1.0                  │
│                                            │
│  Breaking changes:                         │
│  • New date format required                │
│  • Account field now mandatory             │
│                                            │
│  [View Migration Guide] [Upgrade Now]      │
└────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Select parser version for upload
- Healthcare: Select HL7 version (v2.3, v2.5, v3.0)
- Legal: Select document schema version
- Research: Select citation format version (APA 6th vs 7th)
- E-commerce: Select EDI version (X12 4010 vs 5010)
- Any domain: Version selection with deprecation warnings

---


### Vertical 3.8 (Cluster Rules)

#### 32. MerchantRulesManager ✅
**Full spec**: [MerchantRulesManager.md](MerchantRulesManager.md)

**Props**: `rules[]`, `onRuleCreate`, `onRuleEdit`, `onRuleDelete`, `onRuleToggle`
**States**: idle | loading | editing | testing
**Reusable across**: Finance merchant normalization, Healthcare provider normalization, Legal case name normalization, Research institution normalization

**Purpose**: Full CRUD UI for managing normalization rules with priority ordering, pattern testing, and bulk operations.

**Visual (Rules List)**:
```
┌────────────────────────────────────────────┐
│  Merchant Normalization Rules              │
│  [+ Create Rule]                [Test All] │
├────────────────────────────────────────────┤
│                                            │
│  🔹 Priority 95 - ENABLED                  │
│  Pattern: UBER EATS.*                      │
│  Replacement: Uber Eats                    │
│  Type: regex                               │
│  [Edit] [Delete] [Toggle]                  │
│                                            │
│  🔹 Priority 90 - ENABLED                  │
│  Pattern: AMZN MKTP US                     │
│  Replacement: Amazon Marketplace           │
│  Type: exact                               │
│  [Edit] [Delete] [Toggle]                  │
│                                            │
│  🔸 Priority 50 - DISABLED                 │
│  Pattern: STARBUCKS.*                      │
│  Replacement: Starbucks                    │
│  Type: regex                               │
│  [Edit] [Delete] [Toggle]                  │
└────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Normalize merchant names ("UBER EATS PENDING" → "Uber Eats")
- Healthcare: Normalize provider names ("ST MARY'S HOSP" → "St. Mary's Hospital")
- Legal: Normalize case names ("DOE V. SMITH" → "Doe v. Smith")
- Research: Normalize institution names ("MIT CSAIL" → "MIT CSAIL")
- E-commerce: Normalize vendor names
- Any domain: Text normalization rule management

---

#### 33. ClusterViewer ✅
**Full spec**: [ClusterViewer.md](ClusterViewer.md)

**Props**: `clusters[]`, `onClusterExpand`, `onClusterSplit`, `onClusterMerge`, `onTransactionExclude`
**States**: collapsed | expanded | loading | editing
**Reusable across**: Finance transaction clustering, Healthcare claim clustering, Legal case clustering, Research paper clustering

**Purpose**: Display transaction clusters with expand/collapse, manual split/merge operations, and transaction exclusion.

**Visual (Cluster List)**:
```
┌────────────────────────────────────────────┐
│  Transaction Clusters                      │
│  [Refresh] [Settings]                      │
├────────────────────────────────────────────┤
│                                            │
│  ▸ Uber Eats (15 transactions)             │
│    Confidence: 95% | Last: 2024-11-01      │
│                                            │
│  ▾ Amazon Marketplace (8 transactions)     │
│    Confidence: 92% | Last: 2024-10-28      │
│    ├─ AMZN MKTP US       $45.00  Oct 28    │
│    ├─ AMZN.COM/BILL      $32.50  Oct 25    │
│    ├─ AMAZON MKTPLC      $67.00  Oct 22    │
│    └─ ...5 more                            │
│    [Split Cluster] [Exclude Transaction]   │
│                                            │
│  ▸ Starbucks (22 transactions)             │
│    Confidence: 98% | Last: 2024-11-02      │
└────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Group similar merchant transactions
- Healthcare: Cluster similar claims by provider
- Legal: Group related case filings
- Research: Cluster papers by institution/author
- E-commerce: Group orders by vendor
- Any domain: Similarity-based grouping with manual adjustments

---

#### 34. RuleEditorDialog ✅
**Full spec**: [RuleEditorDialog.md](RuleEditorDialog.md)

**Props**: `rule`, `mode`, `onSave`, `onCancel`, `testData[]`
**States**: idle | testing | saving | error
**Reusable across**: Finance normalization rules, Healthcare mapping rules, Legal redaction rules, Research parsing rules

**Purpose**: Create/edit normalization rule dialog with pattern testing, preview, and validation.

**Visual (Create Rule Dialog)**:
```
┌────────────────────────────────────────────┐
│  Create Normalization Rule            [X]  │
├────────────────────────────────────────────┤
│                                            │
│  Pattern *                                 │
│  [UBER EATS.*                        ]     │
│                                            │
│  Replacement *                             │
│  [Uber Eats                          ]     │
│                                            │
│  Rule Type *                               │
│  [ regex ▼ ]                               │
│    • exact - Exact string match            │
│    • regex - Regular expression            │
│    • fuzzy - Fuzzy matching (≥80%)         │
│    • soundex - Phonetic matching           │
│                                            │
│  Priority *                                │
│  [90                                 ]     │
│  (0-100, higher = applied first)           │
│                                            │
│  [Test Pattern]                            │
│                                            │
│  ✓ Matches: "UBER EATS PENDING"            │
│  ✓ Matches: "UBER EATS 123"                │
│  ✗ No match: "UBER"                        │
│                                            │
│  [Cancel]              [Create Rule]       │
└────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Create merchant normalization rules
- Healthcare: Create provider name mapping rules
- Legal: Create document redaction rules
- Research: Create citation parsing rules
- E-commerce: Create product category mapping rules
- Any domain: Pattern-based rule creation with validation

---

### Vertical 3.9 (Reconciliation Strategies)

#### 35. ReconciliationDashboard ✅
**Full spec**: [ReconciliationDashboard.md](ReconciliationDashboard.md)

**Props**: `unmatchedItems[]`, `suggestedMatches[]`, `matchedItems[]`, `config`, `onAcceptMatch`, `onRejectMatch`, `onManualMatch`, `onConfigChange`
**States**: idle | loading | reconciling | reviewing | error
**Reusable across**: Finance bank reconciliation, Healthcare claim matching, Legal case reconciliation, Research citation deduplication

**Purpose**: Comprehensive reconciliation UI with three-column layout showing unmatched items, suggested matches, and confirmed matches with progress tracking and bulk operations.

**Visual (Three-Column Layout)**:
```
┌─────────────────────────────────────────────────────────────────────┐
│  Reconciliation Dashboard                     [Settings] [Export]   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                  │
│  │ Unmatched   │ │ Suggested   │ │ Matched     │                  │
│  │ Items       │ │ Matches     │ │ Items       │                  │
│  │ (45)        │ │ (12)        │ │ (128)       │                  │
│  ├─────────────┤ ├─────────────┤ ├─────────────┤                  │
│  │             │ │             │ │             │                  │
│  │ 📄 Bank     │ │ ✨ 95%      │ │ ✅ Conf.    │                  │
│  │ $125.00     │ │ Bank↔Inv    │ │ $1,250.00   │                  │
│  │ 2024-11-01  │ │ $125.00     │ │ 2024-10-28  │                  │
│  │             │ │ [Review]    │ │             │                  │
│  │             │ │             │ │             │                  │
│  │ 📄 Invoice  │ │ ⚡ 82%      │ │ ✅ Conf.    │                  │
│  │ $450.00     │ │ Inv↔Bank    │ │ $89.50      │                  │
│  │ 2024-10-30  │ │ $450.00     │ │ 2024-10-25  │                  │
│  │             │ │ [Review]    │ │             │                  │
│  └─────────────┘ └─────────────┘ └─────────────┘                  │
│                                                                     │
│  Progress: 73% complete (128/173 items matched)                    │
│  [Refresh] [Accept All High Confidence] [Manual Match]             │
└─────────────────────────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Bank-Invoice reconciliation, credit card vs bank matching
- Healthcare: Claim-Payment matching across insurance systems
- Legal: Court filing reconciliation (PACER vs state courts)
- Research: Citation deduplication (DOI, arXiv, PubMed)
- E-commerce: Order-Shipment-Payment matching
- Logistics: Shipment-Customs-Delivery tracking

---

#### 36. MatchReviewDialog ✅
**Full spec**: [MatchReviewDialog.md](MatchReviewDialog.md)

**Props**: `matchCandidate`, `onAccept`, `onReject`, `onCancel`, `showFeatureScores`
**States**: idle | comparing | accepting | rejecting | error
**Reusable across**: Finance invoice matching, Healthcare claim review, Legal case matching, Research citation review

**Purpose**: Modal dialog for reviewing suggested matches with side-by-side comparison, confidence breakdown by feature, and accept/reject actions.

**Visual (Side-by-Side Comparison)**:
```
┌────────────────────────────────────────────────────────────────────┐
│  Review Match Suggestion                                      [X]  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Overall Confidence: 92% ⚡                                        │
│  Decision: Auto-suggest (review recommended)                      │
│                                                                    │
│  ┌──────────────────────┐  ┌──────────────────────┐              │
│  │ Bank Transaction     │  │ Invoice              │              │
│  ├──────────────────────┤  ├──────────────────────┤              │
│  │ Amount: $1,250.00    │  │ Amount: $1,250.00    │ ✓ 100%      │
│  │ Date: 2024-10-28     │  │ Date: 2024-10-30     │ ⚠ 93%       │
│  │ From: Apple Inc      │  │ To: APPLE COM BILL   │ ⚡ 88%       │
│  │ Desc: Invoice #4567  │  │ Desc: Monthly subs   │ △ 75%       │
│  └──────────────────────┘  └──────────────────────┘              │
│                                                                    │
│  Feature Breakdown:                                                │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━ Amount (40%): 100% ━━━━━━━━━━━━━━━━━━ │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━ Date (30%): 93%   ━━━━━━━━━━━━━━━━━━ │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━ Party (20%): 88%  ━━━━━━━━━━━━━━━━━━ │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━ Desc (10%): 75%   ━━━━━━━━━━━━━━━━━━ │
│                                                                    │
│  Notes:                                                            │
│  [                                                              ]  │
│                                                                    │
│  [Reject]                    [Accept Match]                       │
└────────────────────────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Review invoice-bank transaction matches
- Healthcare: Review claim-payment matches
- Legal: Review case filing matches
- Research: Review citation duplicate candidates
- E-commerce: Review order-shipment matches
- Any domain: Confidence-based match review pattern

---

#### 37. ManualMatchDialog ✅
**Full spec**: [ManualMatchDialog.md](ManualMatchDialog.md)

**Props**: `sourceItem`, `availableTargets[]`, `onMatch`, `onCancel`, `allowMultiSelect`
**States**: idle | searching | selecting | confirming | error
**Reusable across**: Finance payment allocation, E-commerce order matching, Healthcare claim linking, Legal case association

**Purpose**: Modal dialog for manually creating matches when auto-detection fails, supporting one-to-one and one-to-many cardinality with search and validation.

**Visual (Manual Match Creation)**:
```
┌────────────────────────────────────────────────────────────────────┐
│  Create Manual Match                                          [X]  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Source Item:                                                      │
│  Bank Transaction: $450.00 | 2024-10-30 | "OpenAI Subscription"   │
│                                                                    │
│  Match To:                                                         │
│  [Search invoices...                                        ] 🔍  │
│                                                                    │
│  Available Items (23):                                             │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ □ Invoice #4567 | $250.00 | 2024-10-28 | OpenAI API usage   │ │
│  │ □ Invoice #4568 | $200.00 | 2024-10-29 | OpenAI Plus plan   │ │
│  │ □ Invoice #4570 | $125.00 | 2024-10-15 | OpenAI tokens      │ │
│  │ □ Invoice #4571 | $89.00  | 2024-10-20 | Other vendor       │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  Selected: 2 items | Total: $450.00 ✓                             │
│                                                                    │
│  Cardinality: ● One-to-Many (split payment)                       │
│               ○ One-to-One                                         │
│                                                                    │
│  Notes:                                                            │
│  [Split payment across two OpenAI invoices                     ]  │
│                                                                    │
│  [Cancel]                              [Create Match]             │
└────────────────────────────────────────────────────────────────────┘
```

**Reusability**:
- Finance: Manually allocate payment to multiple invoices
- Healthcare: Link claim to multiple payment records
- Legal: Associate case filing with multiple dockets
- Research: Manually deduplicate citation records
- E-commerce: Match order to multiple shipments
- Any domain: Manual association with cardinality support

---

---

### Vertical 4.1 (Reminders)

#### 38. ReminderBell ✅
**Full spec**: [ReminderBell.md](ReminderBell.md)

**Props**: `unreadCount`, `notifications[]`, `onOpen`, `onMarkRead`, `onMarkAllRead`
**States**: idle | open | loading | error
**Reusable across**: Finance alerts, Healthcare appointment reminders, Legal deadline notifications, Research grant alerts, E-commerce inventory warnings

**Purpose**: Header bell icon with unread badge counter and dropdown panel for recent notifications. Provides real-time visual indicator of pending alerts with quick access to notification center.

**Visual (Bell Icon States)**:
```
No notifications:
🔔

1 unread:
🔔 (1)

5+ unread:
🔔 (5+)

Dropdown open:
┌──────────────────────────────────────┐
│ Notifications              [Mark All]│
├──────────────────────────────────────┤
│ ⚠️ Low balance: $450        2h ago  │
│ 📅 Payment due: Apple Card  1d ago  │
│ 💡 Large expense: $5000     Just now│
└──────────────────────────────────────┘
   [View All →]
```

**Reusability Examples:**
- **Finance**: Payment alerts, low balance warnings, large expense notifications
- **Healthcare**: Medication refill reminders, appointment alerts, lab results ready
- **Legal**: Filing deadline warnings, court date reminders, contract expiration
- **Research**: Grant deadline alerts, ethics review expiration, publication milestones
- **E-commerce**: Inventory low stock, price drop alerts, order shipped notifications

---

#### 39. NotificationPanel ✅
**Full spec**: [NotificationPanel.md](NotificationPanel.md)

**Props**: `notifications[]`, `onSnooze`, `onDismiss`, `onAction`, `tabs[]`, `filters`
**States**: idle | loading | refreshing | empty | error
**Reusable across**: Finance notification center, Healthcare patient portal, Legal case alerts, Research collaboration notifications, E-commerce merchant dashboard

**Purpose**: Full-featured notification center with tabs (All, Snoozed, Read), filtering, infinite scroll, snooze/dismiss actions, and bulk operations. Provides comprehensive notification management interface.

**Visual (Panel Layout)**:
```
┌────────────────────────────────────────────┐
│ Notifications                          [X] │
├────────────────────────────────────────────┤
│ [All (3)] [Snoozed (1)] [Read (12)]       │
├────────────────────────────────────────────┤
│ ┌────────────────────────────────────────┐ │
│ │ 🔴 Missing payment: Rent  2 hours ago │ │
│ │ Expected $1200 on Oct 1 (2 days late) │ │
│ │ [View Series] [Snooze ▼] [Dismiss]   │ │
│ └────────────────────────────────────────┘ │
│                                            │
│ ┌────────────────────────────────────────┐ │
│ │ ⚠️ Low balance: $450     4 hours ago  │ │
│ │ Chase Checking below $500 threshold    │ │
│ │ [View Account] [Snooze ▼] [Dismiss]   │ │
│ └────────────────────────────────────────┘ │
│                                            │
│ ↓ Scroll for more (12 total)              │
│ [Mark All Read]                            │
└────────────────────────────────────────────┘
```

**Reusability Examples:**
- **Finance**: Transaction alerts, balance warnings, payment reminders, budget notifications
- **Healthcare**: Appointment reminders, medication alerts, test results, insurance updates
- **Legal**: Deadline alerts, case updates, document filing confirmations, retainer warnings
- **Research**: Grant deadlines, publication milestones, collaboration invites, dataset updates
- **E-commerce**: Order updates, inventory alerts, price changes, customer reviews

---

#### 40. ReminderConfigDialog ✅
**Full spec**: [ReminderConfigDialog.md](ReminderConfigDialog.md)

**Props**: `mode`, `initialConfig`, `onSave`, `onCancel`, `accounts[]`, `series[]`
**States**: idle | validating | saving | error
**Reusable across**: Finance alert setup, Healthcare appointment scheduling, Legal deadline tracking, Research milestone reminders, E-commerce inventory management

**Purpose**: Modal dialog for creating and editing alert rules with condition builder, threshold configuration, channel selection (in-app, email, SMS, push), and scheduling options. Enables users to define custom notification triggers.

**Visual (Create/Edit Dialog)**:
```
┌──────────────────────────────────────────────────┐
│ Create Reminder                              [X] │
├──────────────────────────────────────────────────┤
│                                                  │
│ Name (optional)                                  │
│ ┌──────────────────────────────────────────────┐ │
│ │ Low balance alert - Checking                 │ │
│ └──────────────────────────────────────────────┘ │
│                                                  │
│ Alert Type *                                     │
│ ┌──────────────────────────────────────────────┐ │
│ │ Low Balance                              [v] │ │
│ └──────────────────────────────────────────────┘ │
│   Options: Low Balance, Missing Payment,         │
│            Large Expense, Payment Due,           │
│            Budget Exceeded                       │
│                                                  │
│ Threshold Amount *                               │
│ ┌──────────────────────────────────────────────┐ │
│ │ $  500.00                                    │ │
│ └──────────────────────────────────────────────┘ │
│                                                  │
│ Notify me via                                    │
│ ☑ In-app notification                            │
│ ☑ Email                                          │
│ ☐ SMS (optional)                                 │
│ ☐ Push notification (optional)                   │
│                                                  │
│ [ Cancel ]               [ Create Reminder ]    │
└──────────────────────────────────────────────────┘
```

**Reusability Examples:**
- **Finance**: Configure balance alerts ($500 threshold), missing payment detection (rent, subscriptions), large expense warnings (>3× average), payment due reminders (credit cards)
- **Healthcare**: Set medication refill reminders (30-day prescriptions), appointment alerts (24h notice), insurance expiration warnings (60 days), lab result notifications
- **Legal**: Create filing deadline alerts (court dates, motions), contract expiration reminders (30/60/90 days), retainer balance warnings (<$500), billable hours tracking
- **Research**: Configure grant deadline reminders (NSF, NIH), ethics review expiration (IRB renewal), publication milestone alerts (manuscript accepted), funding balance warnings (<10%)
- **E-commerce**: Set inventory low stock alerts (<10 units), price drop notifications (wishlist items), abandoned cart reminders (24h, 48h, 72h), subscription renewal warnings (7 days)

---

### Vertical 4.2 (Forecast)

#### 41. ForecastChart ✅
**Full spec**: [ForecastChart.md](ForecastChart.md)

**Props**: `data`, `algorithm`, `confidenceLevel`, `chartType`, `onHover`, `onZoom`
**States**: loading | rendering | empty | error
**Reusable across**: Finance projections, Healthcare budget forecasts, Legal revenue projections, Research grant runway, E-commerce revenue forecasting

**Purpose**: Interactive line/area/bar chart for displaying historical data + forecasted projections with confidence intervals. Supports multiple algorithms and comparison mode.

**Reusability Examples:**
- **Finance**: Cash balance projection, income/expense forecasts, investment returns
- **Healthcare**: Patient volume forecasting, department budget projections, equipment costs
- **Legal**: Case volume projections, billable hours trends, retainer runway
- **Research**: Grant funding trajectory, publication rate forecasting, student enrollment
- **E-commerce**: Revenue projections, inventory forecasting, customer acquisition trends

---

#### 42. GoalProgressCard ✅
**Full spec**: [GoalProgressCard.md](GoalProgressCard.md)

**Props**: `goal`, `currentValue`, `onEdit`, `onDelete`, `showDetails`
**States**: active | completed | overdue | archived
**Reusable across**: Finance savings goals, Healthcare budget targets, Legal billable hours goals, Research grant milestones, E-commerce revenue targets

**Purpose**: Display individual goal with progress bar, status indicators (on-track/behind/overdue), time/amount remaining, and milestone tracking.

**Reusability Examples:**
- **Finance**: Emergency fund ($10k goal), debt payoff ($25k credit card), retirement savings
- **Healthcare**: Annual budget compliance ($500k), patient satisfaction score (95% target)
- **Legal**: Billable hours target (2,000 hours/year), new client acquisition (50 clients)
- **Research**: Publication goals (12 papers/year), grant spending milestones ($100k/quarter)
- **E-commerce**: Monthly revenue target ($1M/month), customer retention (95% rate)

---

#### 43. GoalConfigDialog ✅
**Full spec**: [GoalConfigDialog.md](GoalConfigDialog.md)

**Props**: `mode`, `initialGoal`, `templates`, `onSave`, `onCancel`
**States**: idle | validating | saving | error
**Reusable across**: Finance goal creation, Healthcare budget planning, Legal target setting, Research milestone tracking, E-commerce KPI management

**Purpose**: Modal dialog for creating/editing goals with preset templates, tabbed interface (Basic/Advanced/Preview), real-time validation, smart suggestions, and live preview.

**Reusability Examples:**
- **Finance**: Create savings goal (emergency fund template), debt payoff (credit card template), income target
- **Healthcare**: Set department budget goal, equipment replacement fund, staffing cost target
- **Legal**: Configure billable hours goal, new client acquisition target, case closure rate
- **Research**: Set publication milestone, grant spending pacing, student graduation rate
- **E-commerce**: Create revenue goal, inventory turnover target, customer lifetime value objective

---

**Component Count**: 43 components across 17 verticals (1.1, 2.1-2.3, 3.1-3.9, 4.1-4.2)

**Last Updated**: 2025-10-24
