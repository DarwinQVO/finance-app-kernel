# FacturaUploadDialog (IL Component)

## Definition

**FacturaUploadDialog** is a modal dialog component for uploading and linking regulatory documents (Facturas in Mexico, claim forms in healthcare, court filings in legal) to transactions. It provides XML parsing, validation feedback, transaction linking with tolerance-based matching, and document download capabilities.

**Problem it solves:**
- No standardized UI for attaching regulatory documents to transactions
- Users manually validate document authenticity (RFC, UUID, amounts)
- Amount/date mismatches prevent automatic linking
- No visual feedback during XML parsing and validation
- Cannot handle orphan documents (uploaded before transaction exists)
- Difficult to download original documents for audit
- No clear error messages when validation fails

**Solution:**
- Drag-and-drop XML file upload with visual feedback
- Automatic XML parsing and field extraction (RFC, UUID, amount, date)
- Real-time validation with color-coded status indicators
- Tolerance-based transaction matching (±5% amount variance allowed)
- Force link option for manual override when auto-match fails
- Orphan document support (upload before transaction exists)
- Download original XML and generated PDF
- Clear error messages with actionable guidance
- Loading states for async operations
- Keyboard navigation (Esc to close, Tab to navigate)

---

## Interface Contract

```typescript
interface FacturaUploadDialogProps {
  // Transaction context
  transactionId?: string;  // If linking to specific transaction (null for orphan)
  transaction?: Transaction;  // Transaction details for validation
  jurisdiction: string;  // "mexico_federal", "hipaa", etc.

  // Data
  existingFactura?: FacturaRecord;  // If editing/viewing existing factura

  // Callbacks
  onUpload: (file: File, transactionId: string | null, forceLink?: boolean) => Promise<FacturaRecord>;
  onClose: () => void;
  onDownloadXML?: (facturaId: string) => void;
  onDownloadPDF?: (facturaId: string) => void;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Display options
  allowOrphan?: boolean;  // Allow upload without transaction (default: true)
  showForceLink?: boolean;  // Show force link option (default: true)
  maxFileSize?: number;  // Max file size in MB (default: 5)

  // Customization
  theme?: "light" | "dark";
}

interface Transaction {
  transaction_id: string;
  amount: number;  // Negative for expenses
  currency: string;  // "MXN", "USD", etc.
  date: string;  // ISO date
  merchant_name?: string;
}

interface FacturaRecord {
  factura_id: string;
  transaction_id: string | null;
  rfc: string;  // Mexican tax ID (12-13 chars)
  uuid: string;  // UUIDv4
  amount: number;  // Positive (Factura amounts are always positive)
  currency: string;
  issued_date: string;  // ISO date
  xml_file_id: string;
  pdf_file_id: string | null;
  status: "valid" | "invalid" | "pending";
  validation_errors: string[];
  created_at: string;
}

interface ValidationResult {
  is_valid: boolean;
  errors: string[];
  warnings: string[];
  parsed_data?: {
    rfc: string;
    uuid: string;
    amount: number;
    currency: string;
    issued_date: string;
  };
}
```

---

## Component Structure

```tsx
import React, { useState, useRef } from 'react';

export const FacturaUploadDialog: React.FC<FacturaUploadDialogProps> = ({
  transactionId,
  transaction,
  jurisdiction,
  existingFactura,
  onUpload,
  onClose,
  onDownloadXML,
  onDownloadPDF,
  loading = false,
  error = null,
  allowOrphan = true,
  showForceLink = true,
  maxFileSize = 5,
  theme = "light"
}) => {
  const [dragActive, setDragActive] = useState(false);
  const [selectedFile, setSelectedFile] = useState<File | null>(null);
  const [parsing, setParsing] = useState(false);
  const [validationResult, setValidationResult] = useState<ValidationResult | null>(null);
  const [uploading, setUploading] = useState(false);
  const [uploadError, setUploadError] = useState<string | null>(null);
  const [forceLink, setForceLink] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  // Parse and validate XML
  const parseAndValidate = async (file: File): Promise<ValidationResult> => {
    setParsing(true);
    setValidationResult(null);
    setUploadError(null);

    try {
      // File size validation
      const fileSizeMB = file.size / (1024 * 1024);
      if (fileSizeMB > maxFileSize) {
        throw new Error(`File too large. Maximum size is ${maxFileSize}MB`);
      }

      // Read file content
      const xmlContent = await file.text();

      // Parse XML (simplified - real implementation would use DOMParser)
      const parser = new DOMParser();
      const xmlDoc = parser.parseFromString(xmlContent, "text/xml");

      // Check for parsing errors
      const parseError = xmlDoc.querySelector("parsererror");
      if (parseError) {
        throw new Error("Invalid XML format");
      }

      // Extract CFDI fields (Mexico-specific, adapt for other domains)
      const rfc = xmlDoc.querySelector("[RFC]")?.getAttribute("RFC") || "";
      const uuid = xmlDoc.querySelector("[UUID]")?.getAttribute("UUID") || "";
      const amountStr = xmlDoc.querySelector("[Total]")?.getAttribute("Total") || "0";
      const amount = parseFloat(amountStr);
      const currency = xmlDoc.querySelector("[Moneda]")?.getAttribute("Moneda") || "MXN";
      const issued_date = xmlDoc.querySelector("[Fecha]")?.getAttribute("Fecha") || "";

      // Validation rules
      const errors: string[] = [];
      const warnings: string[] = [];

      // RFC format validation (12-13 alphanumeric)
      if (!rfc || !/^[A-Z0-9]{12,13}$/.test(rfc)) {
        errors.push("RFC format invalid: must be 12-13 alphanumeric characters");
      }

      // UUID format validation (UUIDv4)
      const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
      if (!uuid || !uuidRegex.test(uuid)) {
        errors.push("UUID format invalid");
      }

      // Amount validation
      if (amount <= 0) {
        errors.push("Amount must be greater than 0");
      }

      // Date validation
      const dateObj = new Date(issued_date);
      if (isNaN(dateObj.getTime())) {
        errors.push("Issued date is invalid");
      } else if (dateObj > new Date()) {
        errors.push("Issued date cannot be in the future");
      }

      // Transaction-specific validation (if linking to transaction)
      if (transaction) {
        // Amount matching (±5% tolerance)
        const transactionAmount = Math.abs(transaction.amount);
        const variance = Math.abs(amount - transactionAmount) / transactionAmount;

        if (variance > 0.05) {
          warnings.push(
            `Amount mismatch: Factura shows ${formatCurrency(amount, currency)}, ` +
            `transaction shows ${formatCurrency(transactionAmount, transaction.currency)} ` +
            `(${Math.round(variance * 100)}% difference)`
          );
        }

        // Date matching (±7 days tolerance)
        const transactionDate = new Date(transaction.date);
        const daysDiff = Math.abs(
          (dateObj.getTime() - transactionDate.getTime()) / (1000 * 60 * 60 * 24)
        );

        if (daysDiff > 7) {
          warnings.push(
            `Date mismatch: Factura issued on ${issued_date}, ` +
            `transaction on ${transaction.date} (${Math.round(daysDiff)} days apart)`
          );
        }

        // Currency matching
        if (currency !== transaction.currency) {
          warnings.push(
            `Currency mismatch: Factura in ${currency}, transaction in ${transaction.currency}`
          );
        }
      }

      const result: ValidationResult = {
        is_valid: errors.length === 0,
        errors,
        warnings,
        parsed_data: {
          rfc,
          uuid,
          amount,
          currency,
          issued_date
        }
      };

      setValidationResult(result);
      return result;

    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : "Failed to parse XML";
      const result: ValidationResult = {
        is_valid: false,
        errors: [errorMessage],
        warnings: []
      };
      setValidationResult(result);
      return result;
    } finally {
      setParsing(false);
    }
  };

  // Handle file selection
  const handleFileSelect = async (file: File) => {
    // Validate file type
    if (!file.name.endsWith('.xml')) {
      setUploadError("Please select an XML file");
      return;
    }

    setSelectedFile(file);
    await parseAndValidate(file);
  };

  // Handle drag and drop
  const handleDrag = (e: React.DragEvent) => {
    e.preventDefault();
    e.stopPropagation();

    if (e.type === "dragenter" || e.type === "dragover") {
      setDragActive(true);
    } else if (e.type === "dragleave") {
      setDragActive(false);
    }
  };

  const handleDrop = async (e: React.DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setDragActive(false);

    if (e.dataTransfer.files && e.dataTransfer.files[0]) {
      await handleFileSelect(e.dataTransfer.files[0]);
    }
  };

  // Handle file input change
  const handleFileInputChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files && e.target.files[0]) {
      await handleFileSelect(e.target.files[0]);
    }
  };

  // Handle upload
  const handleUpload = async () => {
    if (!selectedFile) return;
    if (!validationResult?.is_valid && !forceLink) return;

    setUploading(true);
    setUploadError(null);

    try {
      await onUpload(selectedFile, transactionId || null, forceLink);
      onClose();
    } catch (err) {
      setUploadError(err instanceof Error ? err.message : "Upload failed");
    } finally {
      setUploading(false);
    }
  };

  // Determine if can proceed
  const canProceed = selectedFile && (
    validationResult?.is_valid ||
    (showForceLink && forceLink && validationResult?.warnings.length === 0)
  );

  // Keyboard shortcuts
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [onClose]);

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div
        className="modal-content factura-upload-dialog"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="modal-header">
          <div className="header-title">
            <h3>
              {existingFactura ? "Factura Details" : "Upload Factura"}
            </h3>
            {transactionId && transaction && (
              <div className="transaction-context">
                {transaction.merchant_name} • {formatCurrency(Math.abs(transaction.amount), transaction.currency)} • {transaction.date}
              </div>
            )}
            {!transactionId && allowOrphan && (
              <div className="orphan-notice">
                💡 Uploading without transaction link (will be matched automatically)
              </div>
            )}
          </div>
          <button className="close-button" onClick={onClose} aria-label="Close">
            ×
          </button>
        </div>

        {/* Body */}
        <div className="modal-body">
          {/* Existing Factura Display */}
          {existingFactura ? (
            <div className="factura-details">
              <FacturaDisplay
                factura={existingFactura}
                onDownloadXML={onDownloadXML}
                onDownloadPDF={onDownloadPDF}
              />
            </div>
          ) : (
            <>
              {/* File Upload Area */}
              {!selectedFile && (
                <div
                  className={`file-drop-zone ${dragActive ? 'drag-active' : ''}`}
                  onDragEnter={handleDrag}
                  onDragLeave={handleDrag}
                  onDragOver={handleDrag}
                  onDrop={handleDrop}
                  onClick={() => fileInputRef.current?.click()}
                >
                  <input
                    ref={fileInputRef}
                    type="file"
                    accept=".xml"
                    onChange={handleFileInputChange}
                    style={{ display: 'none' }}
                  />

                  <div className="drop-zone-content">
                    <div className="drop-zone-icon">📄</div>
                    <div className="drop-zone-text">
                      Drag and drop Factura XML here
                      <br />
                      or click to browse
                    </div>
                    <div className="drop-zone-hint">
                      CFDI 3.3 or 4.0 format • Max {maxFileSize}MB
                    </div>
                  </div>
                </div>
              )}

              {/* Parsing Loader */}
              {parsing && (
                <div className="parsing-loader">
                  <div className="loader-spinner"></div>
                  <div className="loader-text">Parsing XML...</div>
                </div>
              )}

              {/* Validation Results */}
              {selectedFile && !parsing && validationResult && (
                <div className="validation-results">
                  {/* File Info */}
                  <div className="file-info">
                    <div className="file-icon">📄</div>
                    <div className="file-details">
                      <div className="file-name">{selectedFile.name}</div>
                      <div className="file-meta">
                        {(selectedFile.size / 1024).toFixed(2)} KB
                      </div>
                    </div>
                    <button
                      className="remove-file-button"
                      onClick={() => {
                        setSelectedFile(null);
                        setValidationResult(null);
                        setForceLink(false);
                      }}
                      aria-label="Remove file"
                    >
                      ×
                    </button>
                  </div>

                  {/* Parsed Data Display */}
                  {validationResult.parsed_data && (
                    <div className="parsed-data">
                      <div className="data-row">
                        <span className="data-label">RFC</span>
                        <span className="data-value">{validationResult.parsed_data.rfc}</span>
                      </div>
                      <div className="data-row">
                        <span className="data-label">UUID</span>
                        <span className="data-value monospace">
                          {validationResult.parsed_data.uuid}
                        </span>
                      </div>
                      <div className="data-row">
                        <span className="data-label">Amount</span>
                        <span className="data-value">
                          {formatCurrency(
                            validationResult.parsed_data.amount,
                            validationResult.parsed_data.currency
                          )}
                        </span>
                      </div>
                      <div className="data-row">
                        <span className="data-label">Issued Date</span>
                        <span className="data-value">
                          {validationResult.parsed_data.issued_date}
                        </span>
                      </div>
                    </div>
                  )}

                  {/* Validation Status */}
                  <div className={`validation-status ${validationResult.is_valid ? 'valid' : 'invalid'}`}>
                    <div className="status-icon">
                      {validationResult.is_valid ? "✅" : "❌"}
                    </div>
                    <div className="status-text">
                      {validationResult.is_valid
                        ? "Factura validated successfully"
                        : "Validation failed"}
                    </div>
                  </div>

                  {/* Errors */}
                  {validationResult.errors.length > 0 && (
                    <div className="validation-errors">
                      <div className="errors-header">Errors:</div>
                      <ul className="errors-list">
                        {validationResult.errors.map((error, index) => (
                          <li key={index} className="error-item">
                            {error}
                          </li>
                        ))}
                      </ul>
                    </div>
                  )}

                  {/* Warnings */}
                  {validationResult.warnings.length > 0 && (
                    <div className="validation-warnings">
                      <div className="warnings-header">Warnings:</div>
                      <ul className="warnings-list">
                        {validationResult.warnings.map((warning, index) => (
                          <li key={index} className="warning-item">
                            {warning}
                          </li>
                        ))}
                      </ul>

                      {/* Force Link Option */}
                      {showForceLink && validationResult.errors.length === 0 && (
                        <label className="force-link-option">
                          <input
                            type="checkbox"
                            checked={forceLink}
                            onChange={(e) => setForceLink(e.target.checked)}
                          />
                          Force link despite warnings
                        </label>
                      )}
                    </div>
                  )}
                </div>
              )}

              {/* Upload Error */}
              {uploadError && (
                <div className="upload-error">
                  <div className="error-icon">⚠️</div>
                  <div className="error-message">{uploadError}</div>
                </div>
              )}
            </>
          )}
        </div>

        {/* Footer Actions */}
        {!existingFactura && (
          <div className="modal-footer">
            <button
              className="cancel-button"
              onClick={onClose}
              disabled={uploading}
            >
              Cancel
            </button>
            <button
              className="upload-button"
              onClick={handleUpload}
              disabled={!canProceed || uploading}
            >
              {uploading ? "Uploading..." : "Upload Factura"}
            </button>
          </div>
        )}
      </div>
    </div>
  );
};

// Factura Display Component (for existing facturas)
const FacturaDisplay: React.FC<{
  factura: FacturaRecord;
  onDownloadXML?: (facturaId: string) => void;
  onDownloadPDF?: (facturaId: string) => void;
}> = ({ factura, onDownloadXML, onDownloadPDF }) => {
  return (
    <div className="factura-display">
      <div className={`status-badge status-${factura.status}`}>
        {factura.status === "valid" ? "✅ Valid" :
         factura.status === "invalid" ? "❌ Invalid" :
         "⏳ Pending"}
      </div>

      <div className="factura-data">
        <div className="data-row">
          <span className="data-label">RFC</span>
          <span className="data-value">{factura.rfc}</span>
        </div>
        <div className="data-row">
          <span className="data-label">UUID</span>
          <span className="data-value monospace">{factura.uuid}</span>
        </div>
        <div className="data-row">
          <span className="data-label">Amount</span>
          <span className="data-value">
            {formatCurrency(factura.amount, factura.currency)}
          </span>
        </div>
        <div className="data-row">
          <span className="data-label">Issued Date</span>
          <span className="data-value">{factura.issued_date}</span>
        </div>
      </div>

      {factura.validation_errors.length > 0 && (
        <div className="validation-errors">
          <div className="errors-header">Validation Errors:</div>
          <ul className="errors-list">
            {factura.validation_errors.map((error, index) => (
              <li key={index} className="error-item">
                {error}
              </li>
            ))}
          </ul>
        </div>
      )}

      <div className="download-actions">
        {onDownloadXML && (
          <button
            className="download-button"
            onClick={() => onDownloadXML(factura.factura_id)}
          >
            📄 Download XML
          </button>
        )}
        {onDownloadPDF && factura.pdf_file_id && (
          <button
            className="download-button"
            onClick={() => onDownloadPDF(factura.factura_id)}
          >
            📑 Download PDF
          </button>
        )}
      </div>
    </div>
  );
};

// Helper function
function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
    minimumFractionDigits: 2
  }).format(amount);
}
```

---

## Validation Rules

### RFC Validation (Mexico)
```typescript
// Format: 12-13 alphanumeric characters
const rfcRegex = /^[A-Z0-9]{12,13}$/;

// Valid: "ABC123456XYZ" (12 chars) or "ABC1234567XYZ" (13 chars)
// Invalid: "abc123456xyz" (lowercase), "ABC12345" (too short)
```

### UUID Validation
```typescript
// Format: UUIDv4 (8-4-4-4-12 hexadecimal)
const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

// Valid: "12345678-1234-4123-8123-123456789012"
// Invalid: "12345678-1234-1234-1234-123456789012" (not v4)
```

### Amount Matching (Tolerance-Based)
```typescript
// Allow ±5% variance between Factura and transaction amount
const transactionAmount = Math.abs(transaction.amount);  // 500.00
const facturaAmount = 520.00;
const variance = Math.abs(facturaAmount - transactionAmount) / transactionAmount;
// variance = 20 / 500 = 0.04 = 4% ✅ Within tolerance

// variance = 0.04 < 0.05 → Auto-match ✅
// variance = 0.06 > 0.05 → Warning + Force Link option ⚠️
```

### Date Matching (Tolerance-Based)
```typescript
// Allow ±7 days difference between Factura issue date and transaction date
const transactionDate = new Date("2024-11-01");
const facturaDate = new Date("2024-11-03");
const daysDiff = Math.abs((facturaDate - transactionDate) / (1000 * 60 * 60 * 24));
// daysDiff = 2 days ✅ Within tolerance

// daysDiff <= 7 → Auto-match ✅
// daysDiff > 7 → Warning + Force Link option ⚠️
```

### Currency Matching
```typescript
// Exact match required
transaction.currency === factura.currency
// "MXN" === "MXN" ✅
// "MXN" === "USD" ❌ Warning
```

---

## User Interactions

### 1. Upload Factura (Valid, Auto-Match)
```
User: Drags factura.xml into drop zone
System:
  1. Accepts file
  2. Parses XML (200ms)
  3. Extracts: RFC=ABC123456XYZ, UUID=..., Amount=500 MXN, Date=2024-11-01
  4. Validates:
     - RFC format ✅
     - UUID format ✅
     - Amount matches transaction (±5%) ✅
     - Date within ±7 days ✅
  5. Shows ✅ "Factura validated successfully"
  6. Enables "Upload Factura" button

User: Clicks "Upload Factura"
System:
  1. Uploads XML to server
  2. Creates FacturaRecord
  3. Links to transaction
  4. Closes dialog
  5. Shows success notification
```

### 2. Upload with Warnings (Force Link)
```
User: Uploads factura.xml
System:
  1. Parses XML
  2. Validates:
     - Amount: Factura 520 MXN vs Transaction 500 MXN (4% diff) ⚠️
     - Date: Factura 2024-11-05 vs Transaction 2024-11-01 (4 days) ✅
  3. Shows warning: "Amount mismatch: 4% difference"
  4. Shows checkbox: "Force link despite warnings"

User: Checks "Force link" and clicks "Upload Factura"
System:
  1. Uploads with forceLink=true
  2. Creates FacturaRecord with warning flag
  3. Links to transaction
  4. Success notification with ⚠️ icon
```

### 3. Upload with Errors (Cannot Proceed)
```
User: Uploads factura.xml
System:
  1. Parses XML
  2. Validates:
     - RFC format invalid (only 10 chars) ❌
     - UUID format invalid ❌
     - Amount valid ✅
  3. Shows ❌ "Validation failed"
  4. Lists errors:
     - "RFC format invalid: must be 12-13 characters"
     - "UUID format invalid"
  5. "Upload Factura" button remains disabled

User: Must fix XML and re-upload
```

### 4. Orphan Upload (No Transaction)
```
User: Opens dialog from "Upload Factura" menu (no transaction context)
System:
  1. Shows notice: "💡 Uploading without transaction link"
  2. User uploads factura.xml
  3. Validates Factura fields only (no amount/date matching)
  4. Creates FacturaRecord with transaction_id=null
  5. Background job will auto-match later (±7 days, ±5% amount)
```

### 5. View Existing Factura
```
User: Clicks "📄 View Factura" on transaction with existing factura
System:
  1. Opens dialog in view-only mode
  2. Displays Factura details:
     - RFC, UUID, Amount, Date
     - Status badge (✅ Valid)
  3. Shows download buttons:
     - "📄 Download XML"
     - "📑 Download PDF"
  4. No upload area (view-only)
```

### 6. Remove and Re-Upload
```
User: Clicks × button to remove selected file
System:
  1. Clears selected file
  2. Resets validation results
  3. Resets forceLink checkbox
  4. Shows drop zone again

User: Can select new file
```

---

## Wireframes (ASCII Art)

### Initial State (Empty Drop Zone)
```
┌─────────────────────────────────────────────┐
│ Upload Factura                           × │
│ Uber • $500.00 MXN • 2024-11-01            │
├─────────────────────────────────────────────┤
│                                             │
│  ╔═════════════════════════════════════╗   │
│  ║          📄                         ║   │
│  ║                                     ║   │
│  ║  Drag and drop Factura XML here     ║   │
│  ║       or click to browse            ║   │
│  ║                                     ║   │
│  ║  CFDI 3.3 or 4.0 format • Max 5MB   ║   │
│  ╚═════════════════════════════════════╝   │
│                                             │
└─────────────────────────────────────────────┘
```

### Parsing State
```
┌─────────────────────────────────────────────┐
│ Upload Factura                           × │
│ Uber • $500.00 MXN • 2024-11-01            │
├─────────────────────────────────────────────┤
│                                             │
│          ⏳                                  │
│       Parsing XML...                        │
│                                             │
└─────────────────────────────────────────────┘
```

### Valid Factura (Auto-Match)
```
┌─────────────────────────────────────────────┐
│ Upload Factura                           × │
│ Uber • $500.00 MXN • 2024-11-01            │
├─────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────┐ │
│ │ 📄 factura_uber_nov01.xml          × │ │
│ │ 12.45 KB                               │ │
│ └─────────────────────────────────────────┘ │
│                                             │
│ RFC:         ABC123456XYZ                   │
│ UUID:        12345678-1234-4123-8...        │
│ Amount:      $500.00 MXN                    │
│ Issued Date: 2024-11-01                     │
│                                             │
│ ✅ Factura validated successfully           │
│                                             │
│                    [Cancel] [Upload Factura]│
└─────────────────────────────────────────────┘
```

### Warnings (Force Link Option)
```
┌─────────────────────────────────────────────┐
│ Upload Factura                           × │
│ Uber • $500.00 MXN • 2024-11-01            │
├─────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────┐ │
│ │ 📄 factura_uber_nov01.xml          × │ │
│ └─────────────────────────────────────────┘ │
│                                             │
│ RFC:         ABC123456XYZ                   │
│ UUID:        12345678-1234-4123-8...        │
│ Amount:      $520.00 MXN                    │
│ Issued Date: 2024-11-01                     │
│                                             │
│ ⚠️ Warnings:                                 │
│ • Amount mismatch: Factura shows $520.00    │
│   MXN, transaction shows $500.00 MXN (4%    │
│   difference)                               │
│                                             │
│ ☐ Force link despite warnings               │
│                                             │
│                    [Cancel] [Upload Factura]│
└─────────────────────────────────────────────┘
```

### Errors (Cannot Upload)
```
┌─────────────────────────────────────────────┐
│ Upload Factura                           × │
│ Uber • $500.00 MXN • 2024-11-01            │
├─────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────┐ │
│ │ 📄 factura_invalid.xml             × │ │
│ └─────────────────────────────────────────┘ │
│                                             │
│ ❌ Validation failed                         │
│                                             │
│ Errors:                                     │
│ • RFC format invalid: must be 12-13         │
│   alphanumeric characters                   │
│ • UUID format invalid                       │
│                                             │
│                             [Cancel] [Upload]│
│                                    (disabled)│
└─────────────────────────────────────────────┘
```

### Orphan Upload (No Transaction)
```
┌─────────────────────────────────────────────┐
│ Upload Factura                           × │
│ 💡 Uploading without transaction link       │
│    (will be matched automatically)          │
├─────────────────────────────────────────────┤
│  ╔═════════════════════════════════════╗   │
│  ║          📄                         ║   │
│  ║  Drag and drop Factura XML here     ║   │
│  ║       or click to browse            ║   │
│  ╚═════════════════════════════════════╝   │
└─────────────────────────────────────────────┘
```

### View Existing Factura
```
┌─────────────────────────────────────────────┐
│ Factura Details                          × │
│ Uber • $500.00 MXN • 2024-11-01            │
├─────────────────────────────────────────────┤
│                                             │
│ ✅ Valid                                     │
│                                             │
│ RFC:         ABC123456XYZ                   │
│ UUID:        12345678-1234-4123-8123-...    │
│ Amount:      $500.00 MXN                    │
│ Issued Date: 2024-11-01                     │
│                                             │
│ [📄 Download XML]  [📑 Download PDF]        │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Styling

```css
.factura-upload-dialog {
  max-width: 600px;
  width: 90%;
}

.modal-header .transaction-context {
  font-size: 13px;
  color: var(--text-muted);
  margin-top: 4px;
}

.orphan-notice {
  font-size: 13px;
  color: var(--info-color);
  background: var(--info-bg);
  padding: 8px 12px;
  border-radius: 6px;
  margin-top: 8px;
}

/* File Drop Zone */
.file-drop-zone {
  border: 2px dashed var(--border-color);
  border-radius: 12px;
  padding: 48px 24px;
  text-align: center;
  cursor: pointer;
  transition: all 0.2s;
  background: var(--drop-zone-bg);
}

.file-drop-zone:hover {
  border-color: var(--primary-color);
  background: var(--drop-zone-hover-bg);
}

.file-drop-zone.drag-active {
  border-color: var(--primary-color);
  background: var(--primary-bg-light);
  border-style: solid;
}

.drop-zone-icon {
  font-size: 64px;
  margin-bottom: 16px;
  opacity: 0.5;
}

.drop-zone-text {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 8px;
  line-height: 1.5;
}

.drop-zone-hint {
  font-size: 13px;
  color: var(--text-muted);
}

/* Parsing Loader */
.parsing-loader {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  padding: 48px 24px;
}

.loader-spinner {
  width: 48px;
  height: 48px;
  border: 4px solid var(--border-color);
  border-top-color: var(--primary-color);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.loader-text {
  font-size: 14px;
  color: var(--text-muted);
}

/* Validation Results */
.validation-results {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: var(--file-info-bg);
  border: 1px solid var(--border-color);
  border-radius: 8px;
}

.file-icon {
  font-size: 32px;
}

.file-details {
  flex: 1;
}

.file-name {
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 2px;
}

.file-meta {
  font-size: 12px;
  color: var(--text-muted);
}

.remove-file-button {
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  font-size: 24px;
  color: var(--text-muted);
  cursor: pointer;
  border-radius: 6px;
  transition: all 0.2s;
}

.remove-file-button:hover {
  background: var(--hover-bg);
  color: var(--text-color);
}

/* Parsed Data */
.parsed-data {
  background: var(--data-bg);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.data-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  border-bottom: 1px solid var(--divider-color);
}

.data-row:last-child {
  border-bottom: none;
}

.data-label {
  font-weight: 600;
  color: var(--text-muted);
  font-size: 13px;
}

.data-value {
  font-weight: 600;
  color: var(--text-color);
  font-size: 14px;
}

.data-value.monospace {
  font-family: monospace;
  font-size: 12px;
}

/* Validation Status */
.validation-status {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  border-radius: 8px;
  font-weight: 600;
}

.validation-status.valid {
  background: var(--success-bg);
  color: var(--success-color);
  border: 1px solid var(--success-border);
}

.validation-status.invalid {
  background: var(--error-bg);
  color: var(--error-color);
  border: 1px solid var(--error-border);
}

.status-icon {
  font-size: 24px;
}

.status-text {
  font-size: 15px;
}

/* Errors */
.validation-errors {
  background: var(--error-bg);
  border: 1px solid var(--error-border);
  border-radius: 8px;
  padding: 16px;
}

.errors-header {
  font-weight: 600;
  color: var(--error-color);
  margin-bottom: 12px;
  font-size: 14px;
}

.errors-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.error-item {
  font-size: 13px;
  color: var(--error-color);
  padding-left: 20px;
  position: relative;
}

.error-item::before {
  content: "•";
  position: absolute;
  left: 0;
  font-weight: bold;
}

/* Warnings */
.validation-warnings {
  background: var(--warning-bg);
  border: 1px solid var(--warning-border);
  border-radius: 8px;
  padding: 16px;
}

.warnings-header {
  font-weight: 600;
  color: var(--warning-color);
  margin-bottom: 12px;
  font-size: 14px;
}

.warnings-list {
  list-style: none;
  padding: 0;
  margin: 0 0 16px 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.warning-item {
  font-size: 13px;
  color: var(--warning-color);
  padding-left: 20px;
  position: relative;
  line-height: 1.5;
}

.warning-item::before {
  content: "⚠";
  position: absolute;
  left: 0;
}

/* Force Link Option */
.force-link-option {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px;
  background: var(--option-bg);
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  font-size: 14px;
  color: var(--warning-color);
}

.force-link-option input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

/* Upload Error */
.upload-error {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  background: var(--error-bg);
  border: 1px solid var(--error-border);
  border-radius: 8px;
}

.upload-error .error-icon {
  font-size: 24px;
}

.upload-error .error-message {
  font-size: 14px;
  color: var(--error-color);
}

/* Factura Display (View Mode) */
.factura-display {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.status-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  border-radius: 6px;
  font-weight: 600;
  font-size: 14px;
  width: fit-content;
}

.status-badge.status-valid {
  background: var(--success-bg);
  color: var(--success-color);
}

.status-badge.status-invalid {
  background: var(--error-bg);
  color: var(--error-color);
}

.status-badge.status-pending {
  background: var(--warning-bg);
  color: var(--warning-color);
}

.factura-data {
  background: var(--data-bg);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  padding: 16px;
}

.download-actions {
  display: flex;
  gap: 12px;
}

.download-button {
  flex: 1;
  padding: 12px 16px;
  background: var(--button-bg);
  border: 1px solid var(--border-color);
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.download-button:hover {
  background: var(--button-hover-bg);
  border-color: var(--primary-color);
}

/* Modal Footer */
.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding: 20px 24px;
  border-top: 1px solid var(--divider-color);
}

.cancel-button {
  padding: 10px 20px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.upload-button {
  padding: 10px 24px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.upload-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Theme Variables */
.factura-upload-dialog.theme-light {
  --drop-zone-bg: #f9fafb;
  --drop-zone-hover-bg: #f3f4f6;
  --file-info-bg: #ffffff;
  --data-bg: #f9fafb;
  --button-bg: #ffffff;
  --button-hover-bg: #f9fafb;
  --option-bg: rgba(251, 191, 36, 0.1);
  --success-bg: #d1fae5;
  --success-color: #065f46;
  --success-border: #a7f3d0;
  --warning-bg: #fef3c7;
  --warning-color: #92400e;
  --warning-border: #fde68a;
  --error-bg: #fee2e2;
  --error-color: #991b1b;
  --error-border: #fecaca;
  --info-bg: #dbeafe;
  --info-color: #1e40af;
}

.factura-upload-dialog.theme-dark {
  --drop-zone-bg: #111827;
  --drop-zone-hover-bg: #1f2937;
  --file-info-bg: #1f2937;
  --data-bg: #111827;
  --button-bg: #1f2937;
  --button-hover-bg: #374151;
  --option-bg: rgba(251, 191, 36, 0.2);
  --success-bg: #064e3b;
  --success-color: #d1fae5;
  --success-border: #065f46;
  --warning-bg: #78350f;
  --warning-color: #fef3c7;
  --warning-border: #92400e;
  --error-bg: #7f1d1d;
  --error-color: #fee2e2;
  --error-border: #991b1b;
  --info-bg: #1e3a8a;
  --info-color: #dbeafe;
}

/* Responsive Design */
@media (max-width: 768px) {
  .factura-upload-dialog {
    max-width: 100%;
    border-radius: 16px 16px 0 0;
  }

  .file-drop-zone {
    padding: 32px 16px;
  }

  .download-actions {
    flex-direction: column;
  }
}
```

---

## Accessibility

```tsx
// ARIA attributes
<div
  role="dialog"
  aria-labelledby="factura-upload-title"
  aria-modal="true"
>
  <h3 id="factura-upload-title">Upload Factura</h3>
</div>

// Keyboard navigation
// Esc: Close dialog
// Tab: Navigate form fields
// Enter: Submit (when valid)

// Screen reader announcements
aria-live="polite"
// "Factura validated successfully"
// "Validation failed: RFC format invalid, UUID format invalid"
// "Warning: Amount mismatch detected"
// "Factura uploaded and linked to transaction"
```

---

## Multi-Domain Examples

### Finance Domain (Mexico Factura - CFDI)
```tsx
<FacturaUploadDialog
  transactionId={transaction.id}
  transaction={transaction}
  jurisdiction="mexico_federal"
  onUpload={uploadFactura}
  onClose={closeDialog}
  onDownloadXML={downloadFacturaXML}
  onDownloadPDF={downloadFacturaPDF}
  allowOrphan={true}
  showForceLink={true}
  maxFileSize={5}
/>
// Validates: RFC, UUID, Amount, Currency (MXN), Issued Date
```

### Healthcare Domain (CMS-1500 Claim Form)
```tsx
<FacturaUploadDialog
  transactionId={claim.id}
  transaction={claim}
  jurisdiction="hipaa"
  onUpload={uploadClaimForm}
  onClose={closeDialog}
  onDownloadXML={downloadClaimXML}
  allowOrphan={false}  // Claims must link to transaction
  showForceLink={true}
  maxFileSize={10}
/>
// Validates: NPI (National Provider ID), Patient ID, Procedure Codes, Amounts
```

### Legal Domain (Court Filing XML)
```tsx
<FacturaUploadDialog
  transactionId={filing.id}
  transaction={filing}
  jurisdiction="ca_state"
  onUpload={uploadCourtFiling}
  onClose={closeDialog}
  onDownloadXML={downloadFilingXML}
  onDownloadPDF={downloadFilingPDF}
  allowOrphan={true}
  showForceLink={true}
  maxFileSize={20}
/>
// Validates: Case Number, Filing Type, Court Fee Amount, Filing Date
```

### Research Domain (Grant Expense Receipt)
```tsx
<FacturaUploadDialog
  transactionId={expense.id}
  transaction={expense}
  jurisdiction="nsf"
  onUpload={uploadGrantReceipt}
  onClose={closeDialog}
  onDownloadXML={downloadReceiptXML}
  allowOrphan={true}
  showForceLink={true}
  maxFileSize={5}
/>
// Validates: Grant Number, Expense Category, Amount, Allowability
```

---

## Related Components

**Uses:**
- None (primitive UI component)

**Used by:**
- TransactionList (inline factura link)
- DrillDownPanel (transaction detail drawer)
- FacturaManager (bulk upload interface)

**Similar patterns:**
- FileUpload (generic file upload)
- ReceiptScanner (OCR-based receipt capture)
- DocumentViewer (view/download documents)

**OL Dependencies:**
- FacturaStore (upload, parse, create)
- FacturaValidator (XML validation)
- TransactionStore (auto-matching)
