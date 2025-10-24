# IL Components Summary

**Status**: Specification complete
**Last Updated**: 2025-10-23
**Verticals covered**: 1.1, 2.1, 2.2, 2.3

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

**Maturity**: All 11 components specified, ready for implementation

---

## Component Count by Vertical

| Vertical | Components | Status |
|----------|-----------|--------|
| 1.1 Upload Flow | 5 components | ✅ Specified |
| 2.1 Transaction List | 1 component | ✅ Specified |
| 2.2 OL Exploration | 2 components | ✅ Specified |
| 2.3 Finance Dashboard | 3 components | ✅ Specified |
| **TOTAL** | **11 components** | |

---

**Next vertical**: 3.1 Account Registry (expected: 1-2 new IL components for registry management)
