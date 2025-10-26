# OL Primitive: FileArtifact

**Type**: Domain Model / Metadata
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification

---

## Purpose

Internal metadata wrapper for uploaded files. Bridges StorageEngine (content) with domain semantics (source_type, uploader, upload_time). NOT exposed in public API.

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

## Multi-Domain Applicability

This primitive constructs verifiable truth about **uploaded content metadata** - a universal concept across ALL domains:

**Finance Domain:**
- BoFA PDFs, receipts, invoices
- Payment confirmations, tax documents

**Healthcare Domain:**
- Lab results, medical images (DICOM), prescriptions
- Patient records, insurance claims

**Legal Domain:**
- Contracts, court documents, evidence files
- Signed agreements, discovery materials

**Research Domain (RSRCH - Utilitario):**
- TechCrunch HTML pages (web scraping sources), podcast transcripts (Lex Fridman interviews)
- Tweet JSON dumps (Twitter API responses), interview audio files (founder interviews)

**Media/Publishing Domain:**
- Articles, images, videos for publication
- Source files, final outputs

**Generic Systems:**
- Any file upload system requiring metadata tracking
- Any system needing reference counting for garbage collection

---

**Last Updated**: 2025-10-22
**Maturity**: Spec complete, ready for implementation
