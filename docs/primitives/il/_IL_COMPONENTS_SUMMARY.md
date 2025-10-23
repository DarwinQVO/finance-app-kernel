# IL Components Summary (Vertical 1.1)

**Status**: Specification complete
**Last Updated**: 2025-10-22

---

## Component Catalog

### 1. FileUpload ✅
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

**Maturity**: All 5 components specified, ready for implementation
