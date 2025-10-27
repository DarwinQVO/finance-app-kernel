# IL Component: FileUpload

**Type**: Input / Form Control
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification

---

## Purpose

Standard file input component with drag-and-drop, validation, and progress feedback. Reusable across any file upload scenario.

---

## Props Contract

```typescript
interface FileUploadProps {
  // Validation
  accept?: string[]  // MIME types, e.g., ["application/pdf"]
  maxSizeBytes?: number  // Default: 50MB
  multiple?: boolean  // Allow multiple files

  // Behavior
  onFileSelect: (file: File) => void
  onError: (error: FileUploadError) => void

  // Styling
  disabled?: boolean
  className?: string
  dragActiveClassName?: string

  // Labels
  label?: string  // Default: "Choose file or drag & drop"
  buttonText?: string  // Default: "Browse"
}

interface FileUploadError {
  code: "FILE_TOO_LARGE" | "INVALID_TYPE" | "MULTIPLE_NOT_ALLOWED"
  message: string
  file?: File
}
```

---

## States

```typescript
type FileUploadState =
  | "idle"       // Waiting for user action
  | "dragover"   // User dragging file over component
  | "validating" // Checking file type/size
  | "error"      // Validation failed
```

---

## Events Emitted

```typescript
// User selected file (validation passed)
onFileSelect(file: File)

// Validation failed
onError({
  code: "FILE_TOO_LARGE",
  message: "File exceeds 50MB limit",
  file: selectedFile
})

// Lifecycle events (for analytics)
onDragEnter()
onDragLeave()
onDrop(file: File)
onClick()  // User clicked "Browse" button
```

---

## Behavior Specifications

### 1. Drag & Drop

```
┌─────────────────────────────────┐
│  Drag file here or click to    │
│  browse                         │  ← idle state
└─────────────────────────────────┘

[User drags PDF over component]

┌─────────────────────────────────┐
│  📄 Drop file to upload         │  ← dragover state (visual feedback)
└─────────────────────────────────┘

[User drops file]

[Component validates file]
- Check MIME type against `accept`
- Check size against `maxSizeBytes`

[If valid]
→ onFileSelect(file) called

[If invalid]
→ onError() called with specific code
```

---

### 2. Click to Browse

```
[User clicks "Browse" button]
→ Opens native file picker

[User selects file]
→ Validation runs
→ onFileSelect(file) if valid
→ onError() if invalid
```

---

### 3. Validation

```typescript
function validateFile(file: File, props: FileUploadProps): FileUploadError | null {
  // Check MIME type
  if (props.accept && !props.accept.includes(file.type)) {
    return {
      code: "INVALID_TYPE",
      message: `Expected ${props.accept.join(", ")}, got ${file.type}`,
      file
    }
  }

  // Check size
  if (props.maxSizeBytes && file.size > props.maxSizeBytes) {
    return {
      code: "FILE_TOO_LARGE",
      message: `File exceeds ${formatBytes(props.maxSizeBytes)} limit`,
      file
    }
  }

  // Check multiple
  if (!props.multiple && files.length > 1) {
    return {
      code: "MULTIPLE_NOT_ALLOWED",
      message: "Only one file allowed"
    }
  }

  return null  // Valid
}
```

---

## Visual States

### Idle
```
┌──────────────────────────────────────────┐
│  📄 Choose file or drag & drop           │
│                                          │
│  [Browse]                                │
│                                          │
│  Accepted: PDF, up to 50MB               │
└──────────────────────────────────────────┘
```

### Dragover (Highlighted)
```
┌══════════════════════════════════════════┐
║  📄 Drop file to upload                  ║  ← Border highlighted
║                                          ║
║                                          ║
║                                          ║
└══════════════════════════════════════════┘
```

### Error
```
┌──────────────────────────────────────────┐
│  ❌ File too large                       │  ← Error message
│                                          │
│  [Try again]                             │
│                                          │
│  Accepted: PDF, up to 50MB               │
└──────────────────────────────────────────┘
```

### Disabled
```
┌──────────────────────────────────────────┐
│  📄 Choose file or drag & drop           │  ← Greyed out
│                                          │
│  [Browse]                                │  ← Disabled button
│                                          │
│  Upload disabled                         │
└──────────────────────────────────────────┘
```

---

## Accessibility (a11y)

```html
<div
  role="button"
  tabindex="0"
  aria-label="Upload file"
  aria-describedby="file-upload-hint"
  onKeyDown={(e) => e.key === 'Enter' && openFilePicker()}
>
  <input
    type="file"
    accept=".pdf"
    onChange={handleFileSelect}
    aria-hidden="true"
    style={{ display: 'none' }}
  />

  <p id="file-upload-hint">
    Accepted: PDF, up to 50MB
  </p>
</div>
```

**Features:**
- Keyboard navigable (Tab, Enter)
- Screen reader friendly (ARIA labels)
- Error messages announced by screen readers

---

## Example Usage

### Finance App (Upload BoFA Statement)

```tsx
<FileUpload
  accept={["application/pdf"]}
  maxSizeBytes={50 * 1024 * 1024}  // 50MB
  onFileSelect={(file) => {
    uploadToBackend(file, "bofa_pdf")
  }}
  onError={(error) => {
    toast.error(error.message)
  }}
  label="Upload bank statement"
  buttonText="Choose PDF"
/>
```

### Image Upload (Profile Picture)

```tsx
<FileUpload
  accept={["image/jpeg", "image/png"]}
  maxSizeBytes={5 * 1024 * 1024}  // 5MB
  onFileSelect={(file) => {
    uploadAvatar(file)
  }}
  onError={(error) => {
    showError(error.message)
  }}
  label="Upload profile picture"
/>
```

### Document Upload (Legal Contracts)

```tsx
<FileUpload
  accept={["application/pdf", "application/msword"]}
  maxSizeBytes={100 * 1024 * 1024}  // 100MB
  multiple={true}  // Allow multiple files
  onFileSelect={(files) => {
    uploadDocuments(files)
  }}
  label="Upload contract documents"
/>
```

---

## Styling Contract

Component accepts `className` for custom styling. Default styles:

```css
.file-upload {
  border: 2px dashed #ccc;
  padding: 2rem;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.2s;
}

.file-upload--dragover {
  border-color: #4CAF50;
  background-color: #f0f8ff;
}

.file-upload--error {
  border-color: #f44336;
}

.file-upload--disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

---

## Testing

```typescript
describe("FileUpload", () => {
  it("accepts valid PDF file", () => {
    const onFileSelect = jest.fn()
    const { getByLabelText } = render(
      <FileUpload
        accept={["application/pdf"]}
        onFileSelect={onFileSelect}
      />
    )

    const file = new File(["content"], "test.pdf", { type: "application/pdf" })
    const input = getByLabelText("Upload file")

    fireEvent.change(input, { target: { files: [file] } })

    expect(onFileSelect).toHaveBeenCalledWith(file)
  })

  it("rejects file exceeding size limit", () => {
    const onError = jest.fn()
    const { getByLabelText } = render(
      <FileUpload
        maxSizeBytes={1024}
        onError={onError}
      />
    )

    const file = new File(["x".repeat(2000)], "large.pdf", { type: "application/pdf" })
    const input = getByLabelText("Upload file")

    fireEvent.change(input, { target: { files: [file] } })

    expect(onError).toHaveBeenCalledWith(
      expect.objectContaining({ code: "FILE_TOO_LARGE" })
    )
  })

  it("highlights on drag over", () => {
    const { container } = render(<FileUpload />)

    fireEvent.dragEnter(container.firstChild)

    expect(container.firstChild).toHaveClass("file-upload--dragover")
  })
})
```

---

## Reusability

- **Finance**: Upload statements, receipts, invoices
- **Health**: Upload lab results, prescriptions, medical images
- **Legal**: Upload contracts, court documents, evidence
- **Generic**: Any file upload UI

---

**Last Updated**: 2025-10-22
**Maturity**: Spec complete, ready for implementation
