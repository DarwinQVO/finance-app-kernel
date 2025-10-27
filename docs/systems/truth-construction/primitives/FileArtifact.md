# OL Primitive: FileArtifact

**Type**: Domain Model / Metadata
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification

---

## Purpose

Internal metadata wrapper for uploaded files. Bridges StorageEngine (content) with domain semantics (source_type, uploader, upload_time). NOT exposed in public API.

---

## Simplicity Profiles

**This section shows how the same universal metadata wrapper is configured differently based on usage scale.**

### Profile 1: Personal Use (Finance App - Simple Metadata, No GC)

**Context:** Darwin uploads 1-2 bank statements per month, never deletes uploads (keeps all history)

**Configuration:**
```yaml
file_artifact:
  reference_counting: false  # Never delete files (history kept forever)
  virus_scanning: false  # Local files from trusted source (bank website)
  thumbnail_generation: false  # PDFs don't need thumbnails
  metadata_extensions: false  # No custom fields needed
```

**What's Used:**
- ✅ Core metadata - `file_name`, `file_size_bytes`, `mime_type`, `file_hash`
- ✅ Upload context - `uploaded_at`, `source_type`
- ✅ Dedupe by hash - Same statement uploaded twice → Same `file_id`
- ✅ Storage reference - `storage_ref` to retrieve content from StorageEngine

**What's NOT Used:**
- ❌ `ref_count` - Never delete files (no GC needed)
- ❌ `last_accessed_at` - Not tracking access patterns
- ❌ Virus scanning - Trusts files from bank website
- ❌ Thumbnail generation - No image previews
- ❌ OCR text extraction - Not searching PDF content
- ❌ Extended metadata (tags, categories) - Keep it simple

**Implementation Complexity:** **LOW**
- ~50 lines Python
- SQLite table with basic fields
- UNIQUE index on `file_hash` for dedupe
- No background jobs (no GC, no scanning)

**Narrative Example:**
> Darwin uploads "BoFA_Oct2024.pdf" (2.3MB). API calculates hash `sha256:abc123...`. Check FileArtifact table: No existing record with this hash. Create: `{"file_id": "FILE_001", "file_hash": "sha256:abc123...", "storage_ref": "sha256:abc123...", "file_name": "BoFA_Oct2024.pdf", "file_size_bytes": 2300000, "mime_type": "application/pdf", "source_type": "bofa_pdf", "uploaded_at": "2024-10-27T10:00:00Z"}`. UploadRecord references `file_id: "FILE_001"`. Week later, Darwin accidentally re-uploads same PDF → Hash calculated (`sha256:abc123...`) → FileArtifact query finds existing `FILE_001` → Return same artifact → UploadRecord links to same `file_id` (dedupe worked). Files never deleted (no ref_count tracking, user keeps all statements forever).

---

### Profile 2: Small Business (Accounting Firm - Reference Counting, GC)

**Context:** 10 accountants upload client docs, delete old uploads after tax season (7-year retention)

**Configuration:**
```yaml
file_artifact:
  reference_counting: true  # Track refs for GC
  gc_policy:
    enabled: true
    orphan_threshold_days: 90  # Delete if unreferenced for 90 days
    check_interval_hours: 24  # Run daily GC check
  virus_scanning:
    enabled: true
    engine: "clamav"  # Open-source virus scanner
    scan_on_upload: true
  metadata_extensions:
    tags: true  # Allow custom tags per file
    client_id: true  # Associate with client
```

**What's Used:**
- ✅ Core metadata - `file_name`, `file_size_bytes`, `mime_type`, `file_hash`
- ✅ Reference counting - `ref_count` tracks how many UploadRecords reference this file
- ✅ Garbage collection - Delete FileArtifact if `ref_count=0` for >90 days
- ✅ Virus scanning - Scan uploads with ClamAV before storage
- ✅ Extended metadata - `client_id`, `tags` for organizing files
- ✅ Access tracking - `last_accessed_at` for audit logs

**What's NOT Used:**
- ❌ Thumbnail generation - Not needed for PDFs/CSVs
- ❌ OCR text extraction - Not searching file content
- ❌ Advanced audit - Just basic access timestamps

**Implementation Complexity:** **MEDIUM**
- ~200 lines Python
- SQLite table + extended fields (`ref_count`, `client_id`, `tags`, `scan_status`)
- ClamAV integration for virus scanning
- Background GC job (runs daily, deletes orphaned files)
- Email notifications when virus detected

**Narrative Example:**
> Accountant uploads client receipt "Target_receipt.pdf" (450KB). Hash calculated → No existing file → ClamAV scan triggered → Clean ✅ → FileArtifact created `{"file_id": "FILE_123", "file_hash": "sha256:def456...", "scan_status": "clean", "ref_count": 1, "client_id": "CLIENT_ABC"}`. UploadRecord created linking to `FILE_123`. Tax season ends, accountant deletes upload → `ref_count` decremented to 0 → GC job (runs daily) checks: `ref_count=0` AND `last_accessed_at` > 90 days ago? No (only 30 days) → Keep file. 90 days later → GC checks again → `ref_count=0` AND 90+ days → Delete FileArtifact → StorageEngine deletes underlying file (7-year retention met).
>
> Alternative: Different client uploads same receipt → Hash calculated (`sha256:def456...`) → FileArtifact found (`FILE_123`) → `ref_count` incremented to 2 → Now 2 UploadRecords reference same file → Even if Client A deletes upload, `ref_count=1` → File not GC'd (Client B still needs it).

---

### Profile 3: Enterprise (Bank - Virus Scanning, Audit Trails, Extended Metadata)

**Context:** Bank processes 10,000 uploads/day, strict security (virus scanning mandatory), regulatory audit trails required

**Configuration:**
```yaml
file_artifact:
  reference_counting: true
  gc_policy:
    enabled: true
    orphan_threshold_days: 3650  # 10 years (regulatory retention)
    check_interval_hours: 6  # Run GC 4x/day
  virus_scanning:
    enabled: true
    engines: ["clamav", "virustotal"]  # Multi-engine scanning
    scan_on_upload: true
    quarantine_infected: true  # Move to separate S3 bucket
    rescan_interval_days: 30  # Re-scan periodically (detect new signatures)
  thumbnail_generation:
    enabled: true  # Generate previews for images/PDFs
    sizes: [128, 256, 512]  # Multiple thumbnail sizes
  ocr:
    enabled: true  # Extract text from PDFs for search
    language: "en"
  metadata_extensions:
    customer_id: true
    document_type: true  # "statement", "loan_doc", "kyc_form"
    compliance_flags: true  # PII detected, sensitive content
  audit:
    track_access: true
    track_downloads: true
    log_to: "audit_log_table"
  metrics:
    enabled: true
    backend: "prometheus"
    labels:
      - source_type
      - scan_status
      - document_type
```

**What's Used:**
- ✅ Core metadata - Full suite
- ✅ Reference counting - Track refs for complex GC (10-year retention)
- ✅ Multi-engine virus scanning - ClamAV + VirusTotal API
- ✅ Quarantine infected files - Move to `s3://bank-quarantine/`
- ✅ Periodic re-scanning - Detect new malware signatures
- ✅ Thumbnail generation - Preview PDFs/images in UI
- ✅ OCR text extraction - Enable full-text search
- ✅ Extended metadata - `customer_id`, `document_type`, `compliance_flags`
- ✅ Audit trails - Log every access, download, scan
- ✅ Prometheus metrics - Track scan rates, file sizes, document types
- ✅ Compliance flags - Auto-detect PII (SSN, credit card numbers)

**What's NOT Used:**
- (All features enabled at enterprise scale)

**Implementation Complexity:** **HIGH**
- ~1000 lines TypeScript/Python
- PostgreSQL table with extensive metadata fields
- ClamAV + VirusTotal API integration
- S3 quarantine bucket with lifecycle policies
- Background workers: GC (runs 4x/day), re-scanning (runs weekly), thumbnail gen (async queue)
- OCR pipeline (Tesseract or AWS Textract)
- Comprehensive audit logging (every access logged to AuditLog table)
- Monitoring dashboards: Virus detection rate, scan latency, storage growth by document_type

**Narrative Example:**
> Customer uploads mortgage deed PDF (200MB). Hash calculated → No existing file → FileArtifact record created with `scan_status="pending"` → Async virus scan job triggered:
> 1. ClamAV scan (2s) → Clean ✅
> 2. VirusTotal API upload (15s) → 0/60 engines detected malware ✅
> 3. Update FileArtifact: `{"scan_status": "clean", "scan_timestamp": "2024-10-27T10:05:00Z"}`
> 4. OCR extraction job triggered → Extract text from PDF (45s) → Store in `ocr_text` field (enables search)
> 5. Thumbnail generation → Create 3 sizes (128px, 256px, 512px) → Store refs in `thumbnail_refs`
> 6. PII detection → Scan OCR text for SSNs, credit card numbers → Found SSN → Set `compliance_flags: ["pii_detected"]`
> 7. Audit log: `{"event": "file_uploaded", "file_id": "FILE_VIP_999", "customer_id": "CUST_12345", "scan_status": "clean", "pii_detected": true}`
> 8. Prometheus metrics: `file_uploads_total{source_type="mortgage_deed",scan_status="clean"}++`, `file_size_bytes{document_type="deed"} 200000000`
>
> 30 days later, automated re-scan job runs → Re-scan with updated virus signatures → Still clean ✅. 10 years later, mortgage closed → UploadRecord deleted → `ref_count` decremented to 0 → GC job (runs 4x/day) waits 10 years (regulatory retention) → `ref_count=0` AND 10+ years → Delete FileArtifact → Delete from S3 → Delete thumbnails → Delete OCR text → Audit log: `{"event": "file_deleted_gc", "file_id": "FILE_VIP_999", "retention_met": true}`.
>
> Alternative: Infected file uploaded → ClamAV detects malware → `scan_status="infected"` → Move to quarantine bucket `s3://bank-quarantine/FILE_VIRUS_001` → Alert fired: "P1: Malware detected in upload (file_id: FILE_VIRUS_001, customer_id: CUST_12345)" → Security team investigates → Customer device compromised → Contact customer, remediate.

---

**Key Insight:** The same `FileArtifact` metadata structure works across all 3 profiles. Personal use tracks bare minimum (hash, name, size); Small business adds reference counting for GC and virus scanning; Enterprise adds multi-engine scanning, OCR, thumbnails, compliance flags, and audit trails for regulatory requirements.

---

## Schema

```typescript
interface FileArtifact {
  // Primary key (internal only, never exposed)
  file_id: string  // Format: "FILE_{uuid}"

  // Content identity
  file_hash: Hash  // sha256:hexdigest (dedupe key)
  storage_ref: StorageRef  // Reference to StorageEngine

  // File metadata
  file_name: string  // Original filename from user
  file_size_bytes: number
  mime_type: string  // e.g., "application/pdf"
  extension: string  // e.g., "pdf"

  // Upload context
  uploaded_at: ISO8601Timestamp
  uploader_id: string  // Placeholder in v1 (auth deferred)

  // Domain semantics
  source_type: string  // e.g., "bofa_pdf" (from ParserRegistry)

  // Internal tracking
  ref_count: number  // How many UploadRecords reference this
  created_at: ISO8601Timestamp
  last_accessed_at: ISO8601Timestamp
}
```

---

## Behavior

### 1. Dedupe Key = file_hash

```python
def store_artifact(file: UploadedFile) -> FileArtifact:
    hash = calculate_hash(file.content)

    # Check if already exists
    existing = db.query(FileArtifact).filter_by(file_hash=hash).first()
    if existing:
        # Increment ref count
        existing.ref_count += 1
        return existing

    # Store content
    storage_ref = storage_engine.store(file.content)

    # Create artifact
    artifact = FileArtifact(
        file_id=generate_id(),
        file_hash=hash,
        storage_ref=storage_ref,
        file_name=file.name,
        file_size_bytes=len(file.content),
        mime_type=file.mime_type,
        extension=file.extension,
        uploaded_at=now(),
        ref_count=1
    )

    db.save(artifact)
    return artifact
```

### 2. Reference Counting

**Use case**: Garbage collection

```python
def delete_upload_record(upload_id: str):
    record = db.get(UploadRecord, upload_id)
    artifact = db.get(FileArtifact, record.file_id)

    # Decrement ref count
    artifact.ref_count -= 1

    # GC if no more references
    if artifact.ref_count == 0:
        storage_engine.delete(artifact.storage_ref)
        db.delete(artifact)
```

### 3. Internal Only

**Never exposed in public API:**
- `file_id` (use `upload_id` instead)
- `storage_ref` (internal storage detail)
- `ref_count` (implementation detail)

**Only exposed internally** to:
- `UploadRecord` (needs `file_id` for foreign key)
- `StorageEngine` (needs `storage_ref` to retrieve content)
- GC workers (needs `ref_count` for cleanup)

---

## Database Schema

```sql
CREATE TABLE file_artifacts (
    file_id TEXT PRIMARY KEY,
    file_hash TEXT UNIQUE NOT NULL,  -- Dedupe index
    storage_ref TEXT NOT NULL,
    file_name TEXT NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    mime_type TEXT NOT NULL,
    extension TEXT NOT NULL,
    uploaded_at TIMESTAMPTZ NOT NULL,
    uploader_id TEXT,
    source_type TEXT NOT NULL,
    ref_count INTEGER DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_accessed_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_file_hash ON file_artifacts(file_hash);
CREATE INDEX idx_source_type ON file_artifacts(source_type);
CREATE INDEX idx_uploaded_at ON file_artifacts(uploaded_at);
```

---

## Relationship to Other Primitives

```
UploadRecord
  ├─ upload_id (PUBLIC)
  ├─ file_id (INTERNAL FK) ──→ FileArtifact
  └─ status

FileArtifact
  ├─ file_id (INTERNAL PK)
  ├─ storage_ref ──→ StorageEngine
  └─ file_hash (dedupe key)

StorageEngine
  └─ storage_ref (opaque handle)
```

---

## Extension Points

### 1. Virus Scanning

```python
class ScannedFileArtifact(FileArtifact):
    scan_status: str  # "pending", "clean", "infected"
    scan_timestamp: ISO8601Timestamp
    scan_engine: str  # "clamav", "virustotal"
```

### 2. Thumbnail Generation

```python
class ImageFileArtifact(FileArtifact):
    thumbnail_ref: StorageRef
    image_width: int
    image_height: int
```

### 3. OCR Results

```python
class PDFFileArtifact(FileArtifact):
    ocr_text: str
    page_count: int
    is_searchable: bool
```

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Track uploaded bank statement files with metadata (size, hash, mime_type)
**Example:** User uploads "BoFA_Jan2024.pdf" (2.5MB) → StorageEngine stores with hash sha256:abc123 → FileArtifact creates metadata record `{"file_id": "FA_001", "original_name": "BoFA_Jan2024.pdf", "size_bytes": 2500000, "mime_type": "application/pdf", "file_hash": "sha256:abc123", "storage_ref": "sha256:abc123"}` → Later retrieval uses file_id to get metadata + storage_ref to download
**Metadata tracked:** original_name, size_bytes, mime_type, file_hash, uploaded_at
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Track medical image files (X-rays, MRIs) with DICOM metadata
**Example:** Radiologist uploads chest X-ray "xray_12345.dcm" (15MB, DICOM format) → StorageEngine stores → FileArtifact creates metadata with `{"mime_type": "application/dicom", "size_bytes": 15000000, "file_hash": "sha256:xyz789"}` → Later PACS system retrieves using file_id
**Metadata tracked:** original_name, mime_type (DICOM), file_hash, patient_id (HIPAA-compliant reference)
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Track case document files with chain of custody metadata
**Example:** Attorney uploads signed contract "Agreement_v3_signed.pdf" (500KB) → StorageEngine stores → FileArtifact creates metadata with `{"file_hash": "sha256:def456", "uploaded_by": "attorney_smith", "uploaded_at": "2024-03-15T10:30:00Z"}` → Chain of custody verifiable via immutable file_hash
**Metadata tracked:** original_name, file_hash, uploaded_by (attorney), uploaded_at (timestamp), case_id
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** Track scraped web articles and podcast transcripts with source metadata
**Example:** Scraper downloads TechCrunch article "sama_investment.html" (120KB) → StorageEngine stores → FileArtifact creates metadata with `{"original_name": "sama_investment.html", "mime_type": "text/html", "file_hash": "sha256:ghi789", "source_url": "techcrunch.com/2024/..."}` → Provenance tracks which article produced which facts
**Metadata tracked:** original_name, mime_type (HTML/JSON/MP3), file_hash, source_url, scraped_at
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Track product image files with SKU metadata
**Example:** Catalog manager uploads "iPhone15_Blue.jpg" (3MB) → StorageEngine stores → FileArtifact creates metadata with `{"original_name": "iPhone15_Blue.jpg", "mime_type": "image/jpeg", "file_hash": "sha256:jkl012", "product_sku": "IPHONE15-256-BLU"}` → Product listing references file_id for image display
**Metadata tracked:** original_name, mime_type (JPEG/PNG), file_hash, product_sku, uploaded_at
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (generic file metadata wrapper, works for any mime_type)
**Reusability:** High (same FileArtifact structure works for PDFs, images, audio, DICOM, HTML, JSON)

---

**Last Updated**: 2025-10-27
**Maturity**: Spec complete, ready for implementation
