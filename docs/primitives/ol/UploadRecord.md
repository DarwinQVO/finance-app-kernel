# OL Primitive: UploadRecord

**Type**: State Machine / Orchestrator
**Domain**: Universal pattern (multi-stage async pipelines)
**Version**: 1.0
**Status**: Specification

---

## Purpose

Universal state machine pattern for async multi-stage pipelines. Tracks an entity from creation through processing stages. Single source of truth for current status.

**This pattern applies to ANY domain requiring:**
- Multi-stage async processing (upload → parse → normalize)
- Progressive disclosure of results (show what's available at each stage)
- Clear status tracking (queued → processing → complete)
- Separation of data writing (runners) from orchestration (coordinator)

---

## Schema

See [upload-record.schema.json](../../schemas/upload-record.schema.json) for complete JSON Schema.

```typescript
interface UploadRecord {
  // Identity (PUBLIC)
  upload_id: string  // Format: "UL_{random}"

  // State (single source of truth)
  status: UploadStatus

  // Error tracking
  error_message: string | null
  error_code: string | null

  // Logs (progressive disclosure)
  upload_log_ref: string  // Always present
  parse_log_ref?: string  // Present if status ≥ parsing
  normalizer_log_ref?: string  // Present if status ≥ normalizing

  // Counts (progressive disclosure)
  observations_count?: number  // Present if status ≥ parsed
  canonicals_count?: number  // Present if status = normalized

  // Internal (not in public API)
  file_id: string  // FK to FileArtifact
  created_at: ISO8601Timestamp
  updated_at: ISO8601Timestamp
}

type UploadStatus =
  | "queued_for_parse"
  | "parsing"
  | "parsed"
  | "normalizing"
  | "normalized"
  | "error"
```

---

## State Machine

```
[Initial]
    ↓
queued_for_parse  ← Entry point (created by upload endpoint)
    ↓
  parsing         ← Coordinator transitions when parser starts
    ↓
  parsed          ← Coordinator transitions when parser succeeds
    ↓
normalizing       ← Coordinator transitions when normalizer starts
    ↓
normalized        ← Coordinator transitions when normalizer succeeds

Note: "error" can occur at any stage
```

### Transition Rules

| From | To | Trigger | Actor |
|------|----|---------| ------|
| `null` | `queued_for_parse` | Upload created | API endpoint |
| `queued_for_parse` | `parsing` | Parser started | Coordinator |
| `parsing` | `parsed` | Parser succeeded | Coordinator |
| `parsing` | `error` | Parser failed | Coordinator |
| `parsed` | `normalizing` | Normalizer started | Coordinator |
| `normalizing` | `normalized` | Normalizer succeeded | Coordinator |
| `normalizing` | `error` | Normalizer failed | Coordinator |
| `error` | `queued_for_parse` | Manual retry | User/Admin |
| `error` | `parsed` | Retry from normalize | User/Admin |

---

## Behavior

### 1. Status Authority (See ADR-0003)

**Rule**: ONLY Coordinator can update `status`

```python
class UploadRecord:
    def transition(self, to_state: str, coordinator: Coordinator):
        # Enforce: Only coordinator can call this
        if not isinstance(coordinator, Coordinator):
            raise PermissionError("Only Coordinator can update status")

        self.status = to_state
        self.updated_at = now()
        self.save()
```

**Runners CANNOT update status:**
```python
# ❌ FORBIDDEN in runner code
upload_record.status = "parsed"

# ✅ ALLOWED: Runner emits event, Coordinator updates status
events.emit("parse.completed", {"upload_id": upload_id})
```

---

### 2. Progressive Disclosure (Conditional Fields)

**Principle**: Fields appear only when relevant to current status

**queued_for_parse:**
```json
{
  "upload_id": "UL_abc123",
  "status": "queued_for_parse",
  "error_message": null,
  "upload_log_ref": "/logs/uploads/UL_abc123.log"
}
```

**parsed:**
```json
{
  "upload_id": "UL_abc123",
  "status": "parsed",
  "error_message": null,
  "upload_log_ref": "/logs/uploads/UL_abc123.log",
  "parse_log_ref": "/logs/parse/UL_abc123.log.json",  ← NEW
  "observations_count": 42  ← NEW
}
```

**normalized:**
```json
{
  "upload_id": "UL_abc123",
  "status": "normalized",
  "error_message": null,
  "upload_log_ref": "/logs/uploads/UL_abc123.log",
  "parse_log_ref": "/logs/parse/UL_abc123.log.json",
  "observations_count": 42,
  "normalizer_log_ref": "/logs/normalize/UL_abc123.log.json",  ← NEW
  "canonicals_count": 42  ← NEW
}
```

---

### 3. Idempotent Creation

**Dedupe by file_hash (handled at FileArtifact level):**

```python
def create_upload_record(file: UploadedFile, source_type: str) -> Tuple[UploadRecord, bool]:
    """
    Returns (record, is_duplicate)
    """
    # Store artifact (dedupe happens here)
    artifact, is_new = FileArtifact.get_or_create(file)

    if not is_new:
        # Find existing UploadRecord for this artifact
        existing = UploadRecord.query.filter_by(file_id=artifact.file_id).first()
        if existing:
            return (existing, True)  # Duplicate

    # Create new UploadRecord
    record = UploadRecord(
        upload_id=generate_upload_id(),
        file_id=artifact.file_id,
        status="queued_for_parse",
        upload_log_ref=f"/logs/uploads/{upload_id}.log"
    )
    record.save()

    return (record, False)  # New upload
```

---

## Database Schema

```sql
CREATE TABLE upload_records (
    upload_id TEXT PRIMARY KEY,
    file_id TEXT NOT NULL REFERENCES file_artifacts(file_id),
    status TEXT NOT NULL CHECK (status IN ('queued_for_parse', 'parsing', 'parsed', 'normalizing', 'normalized', 'error')),
    error_message TEXT,
    error_code TEXT,
    upload_log_ref TEXT NOT NULL,
    parse_log_ref TEXT,
    normalizer_log_ref TEXT,
    observations_count INTEGER,
    canonicals_count INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_status ON upload_records(status);
CREATE INDEX idx_file_id ON upload_records(file_id);
CREATE INDEX idx_created_at ON upload_records(created_at DESC);
```

---

## Queries

### Workers: Find Work

```sql
-- Parser: Find uploads queued for parsing
SELECT upload_id, file_id
FROM upload_records
WHERE status = 'queued_for_parse'
ORDER BY created_at
LIMIT 10;

-- Normalizer: Find uploads ready for normalization
SELECT upload_id
FROM upload_records
WHERE status = 'parsed'
ORDER BY created_at
LIMIT 10;
```

### Monitoring: Count by Status

```sql
SELECT status, COUNT(*) as count
FROM upload_records
GROUP BY status;

-- Result:
-- queued_for_parse | 5
-- parsing          | 2
-- parsed           | 3
-- normalizing      | 1
-- normalized       | 1234
-- error            | 7
```

### Forensics: Recent Errors

```sql
SELECT upload_id, error_message, created_at
FROM upload_records
WHERE status = 'error'
ORDER BY created_at DESC
LIMIT 10;
```

---

## Multi-Domain Applicability

This primitive constructs verifiable truth about **processing lifecycle state** - a universal pattern for ANY multi-stage async pipeline:

**Finance Domain:**
```typescript
interface TransactionUploadRecord {
  upload_id: string
  status: "queued_for_parse" | "parsing" | "parsed" | "normalizing" | "normalized" | "error"
  parse_log_ref?: string
  observations_count?: number
}
```

**Healthcare Domain:**
```typescript
interface LabResultUploadRecord {
  upload_id: string
  status: "uploaded" | "validating" | "validated" | "analyzing" | "analyzed" | "approved" | "error"
  validation_log_ref?: string
  test_results_count?: number
  approval_timestamp?: ISO8601
}
```

**Image Processing:**
```typescript
interface ImageUploadRecord {
  upload_id: string
  status: "queued" | "resizing" | "resized" | "thumbnailing" | "complete" | "error"
  resize_log_ref?: string
  thumbnail_ref?: string
}
```

**Document OCR:**
```typescript
interface DocumentUploadRecord {
  upload_id: string
  status: "queued" | "ocr_pending" | "ocr_complete" | "indexing" | "indexed" | "error"
  ocr_log_ref?: string
  page_count?: number
}
```

**Manufacturing Quality Control:**
```typescript
interface InspectionRecord {
  inspection_id: string
  status: "pending" | "scanning" | "scanned" | "analyzing" | "passed" | "failed" | "error"
  scan_log_ref?: string
  defects_count?: number
}
```

**Research Data Pipeline:**
```typescript
interface ExperimentRecord {
  experiment_id: string
  status: "submitted" | "validating" | "validated" | "processing" | "analyzed" | "published" | "error"
  validation_log_ref?: string
  data_points_count?: number
}
```

---

## Extension Points

### 1. Retry Metadata

```typescript
interface UploadRecordWithRetry extends UploadRecord {
  retry_count: number
  last_retry_at: ISO8601Timestamp
  max_retries: number  // Configurable per source_type
}
```

### 2. Priority Queue

```typescript
interface PrioritizedUploadRecord extends UploadRecord {
  priority: number  // 1 (low) to 10 (high)
  priority_reason: string  // "user_request", "premium_user", "admin"
}

// Workers query by priority
SELECT upload_id WHERE status = 'queued_for_parse' ORDER BY priority DESC, created_at
```

### 3. Timeouts

```typescript
interface UploadRecordWithTimeout extends UploadRecord {
  parsing_started_at: ISO8601Timestamp
  parsing_timeout_seconds: number  // Default: 300

  normalizing_started_at: ISO8601Timestamp
  normalizing_timeout_seconds: number  // Default: 60
}

// Monitor: Find stuck uploads
SELECT upload_id FROM upload_records
WHERE status = 'parsing' AND parsing_started_at < NOW() - INTERVAL '5 minutes'
```

---

## Testing

```python
def test_state_transition():
    record = UploadRecord.create(upload_id="UL_test", status="queued_for_parse")

    coordinator.transition(record, "parsing")
    assert record.status == "parsing"

    coordinator.transition(record, "parsed")
    assert record.status == "parsed"

def test_invalid_transition():
    record = UploadRecord.create(upload_id="UL_test", status="queued_for_parse")

    with pytest.raises(InvalidTransitionError):
        # Can't skip "parsing" and go directly to "parsed"
        coordinator.transition(record, "parsed")

def test_progressive_disclosure():
    record = UploadRecord.create(upload_id="UL_test", status="queued_for_parse")

    # Initially no parse_log_ref
    assert record.parse_log_ref is None

    coordinator.transition(record, "parsing")
    coordinator.transition(record, "parsed", observations_count=42, parse_log_ref="/logs/...")

    # Now present
    assert record.parse_log_ref == "/logs/..."
    assert record.observations_count == 42
```

---

## Related Primitives

- `FileArtifact`: UploadRecord references via `file_id`
- `ProvenanceLedger`: Logs all status transitions
- `Coordinator`: Only entity that updates `status`

---

**Last Updated**: 2025-10-22
**Maturity**: Spec complete, ready for implementation
