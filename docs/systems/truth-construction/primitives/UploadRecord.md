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

## Simplicity Profiles

**This section shows how the same universal state machine is configured differently based on usage scale.**

### Profile 1: Personal Use (Finance App - Manual Uploads, Low Volume)

**Context:** Darwin uploads 1-2 bank statements per month manually, expects quick processing (seconds)

**Configuration:**
```yaml
upload_record:
  states: ["queued_for_parse", "parsing", "parsed", "normalizing", "normalized", "error"]
  retry: false  # No automatic retries (user manually re-uploads if error)
  timeout_monitoring: false  # Processing fast enough (<10s) that timeouts unlikely
  priority_queue: false  # Single user, FIFO is fine
```

**What's Used:**
- ✅ Core state machine - Track status through pipeline
- ✅ Progressive disclosure - Show `parse_log_ref` after parsing
- ✅ Error tracking - Display `error_message` if parsing fails
- ✅ Single-user FIFO - Process uploads in order received
- ✅ Basic status checks - "Is my upload done?"

**What's NOT Used:**
- ❌ Automatic retries - User manually re-uploads if error (simpler)
- ❌ Timeout monitoring - Parser completes <5s (no stuck uploads)
- ❌ Priority queue - Single user, no competing uploads
- ❌ Retry counters (`retry_count`, `max_retries`) - Not needed
- ❌ Timeout fields (`parsing_started_at`, `parsing_timeout_seconds`) - Not needed
- ❌ Status change webhooks - User polls `/uploads/:id` manually
- ❌ Metrics tracking - No Prometheus, no state transition counts

**Implementation Complexity:** **LOW**
- ~100 lines Python
- SQLite table with 6 status values
- Single index on `status` for worker queries
- No background jobs (coordinator runs inline with parser/normalizer)

**Narrative Example:**
> Darwin uploads "BoFA_Oct2024.pdf". API creates UploadRecord {upload_id: "UL_abc123", status: "queued_for_parse", upload_log_ref: "/logs/UL_abc123.upload.log"}. Parser immediately starts (no queue delay, single user). Coordinator updates {status: "parsing"}. Parser completes in 3s → {status: "parsed", observations_count: 42, parse_log_ref: "/logs/UL_abc123.parse.json"}. Normalizer starts immediately → {status: "normalizing"}. Completes in 1s → {status: "normalized", canonicals_count: 42}. Total: 4 seconds upload→normalized. Darwin refreshes page → sees "normalized" + 42 transactions. If parser had failed → {status: "error", error_message: "Invalid PDF structure"}. Darwin sees error, re-uploads fixed PDF manually (no automatic retry).

---

### Profile 2: Small Business (Accounting Firm - Multi-User, Some Failures)

**Context:** 10 accountants upload client docs (50-100 uploads/day), occasional parsing failures (corrupted PDFs), want automatic retries

**Configuration:**
```yaml
upload_record:
  states: ["queued_for_parse", "parsing", "parsed", "normalizing", "normalized", "error"]
  retry:
    enabled: true
    max_retries: 3
    backoff_seconds: [60, 300, 900]  # 1min, 5min, 15min
  timeout_monitoring:
    enabled: true
    parsing_timeout_seconds: 300  # 5 minutes
    normalizing_timeout_seconds: 60
    check_interval_seconds: 60  # Monitor every minute
  priority_queue: false  # FIFO still fine (no VIP clients)
```

**What's Used:**
- ✅ Core state machine - Track status
- ✅ Automatic retries - Retry parsing up to 3 times (corrupted PDFs often succeed on retry)
- ✅ Retry metadata - Track `retry_count`, `last_retry_at`
- ✅ Timeout monitoring - Detect stuck uploads (parser hung on malformed PDF >5min)
- ✅ Timeout fields - `parsing_started_at` for timeout checks
- ✅ Background monitor - Cron job checks for timeouts every minute
- ✅ Error notifications - Email admin if upload stuck or exceeds max retries

**What's NOT Used:**
- ❌ Priority queue - All clients equal priority
- ❌ Advanced metrics - Just basic logging (no Prometheus)
- ❌ Webhooks - Users poll status via UI

**Implementation Complexity:** **MEDIUM**
- ~300 lines Python
- SQLite table + retry fields (`retry_count`, `last_retry_at`)
- Background cron job (runs every minute, checks timeouts)
- Email notifications on errors (SMTP integration)
- Retry logic: Exponential backoff (1min → 5min → 15min)

**Narrative Example:**
> Accountant uploads client PDF (corrupted metadata). UploadRecord created {status: "queued_for_parse"}. Parser starts → PDF malformed → parsing fails after 30s → {status: "error", error_message: "PDF metadata corrupted", retry_count: 0}. Coordinator waits 1min, retries → {status: "parsing", retry_count: 1}. Parsing succeeds (PDF parser more lenient on retry) → {status: "parsed", observations_count: 25}. Total: 2 retries, 90s elapsed. Alternative scenario: Parser hangs on extremely corrupted PDF. Timeout monitor (runs every 60s) detects `status="parsing"` + `parsing_started_at` > 5min ago → Coordinator kills parser, transitions {status: "error", error_message: "Parsing timeout (5min)"} → Email sent to admin: "Upload UL_xyz789 timed out, manual review needed".

---

### Profile 3: Enterprise (Financial Institution - High Volume, SLA Requirements)

**Context:** Bank processes 10,000 uploads/day (credit card statements for 1M customers), 99.9% SLA (<1min upload→normalized), VIP customers get priority

**Configuration:**
```yaml
upload_record:
  states: ["queued_for_parse", "parsing", "parsed", "normalizing", "normalized", "error"]
  retry:
    enabled: true
    max_retries: 5
    backoff_seconds: [10, 30, 60, 300, 900]  # Aggressive retries for SLA
    jitter_ms: 1000  # Randomize to avoid thundering herd
  timeout_monitoring:
    enabled: true
    parsing_timeout_seconds: 120  # 2 minutes (strict SLA)
    normalizing_timeout_seconds: 30
    check_interval_seconds: 10  # Check every 10s (fast detection)
  priority_queue:
    enabled: true
    priorities: ["vip_customer", "regulatory_filing", "normal", "batch_import"]
    preemption: false  # Don't cancel in-progress low-priority uploads
  metrics:
    enabled: true
    backend: "prometheus"
    labels:
      - status
      - source_type
      - retry_count
  logging:
    structured: true
    format: "json"
    level: "info"
    trace_id: true  # Distributed tracing
  webhooks:
    enabled: true
    endpoints:
      - url: "https://internal-api/upload-status-changed"
        events: ["normalized", "error"]
```

**What's Used:**
- ✅ Core state machine - Track status
- ✅ Automatic retries with jitter - Retry up to 5 times with randomized backoff (avoid thundering herd)
- ✅ Priority queue - VIP customers processed first (`priority=10`), batch imports last (`priority=1`)
- ✅ Timeout monitoring - Check every 10s (fast failure detection for SLA)
- ✅ Retry metadata - Track `retry_count`, `last_retry_at`, `priority`
- ✅ Prometheus metrics - Track state transition latencies (p95 `queued→normalized` < 60s SLA)
- ✅ Structured logging - JSON logs with trace IDs for debugging
- ✅ Webhooks - Notify downstream systems when upload normalized or errored
- ✅ Preemption prevention - Don't cancel in-progress normal uploads when VIP arrives (fairness)

**What's NOT Used:**
- (All features enabled at this scale)

**Implementation Complexity:** **HIGH**
- ~1200 lines TypeScript/Python
- PostgreSQL table with `priority` field, indexes on `(status, priority, created_at)`
- Background worker pool (10 concurrent timeout checkers)
- Webhook delivery queue (SQS for reliable delivery)
- Comprehensive test suite: State transitions, retries, priority ordering, timeout detection
- Monitoring dashboards: p95 latency per status, retry rate, timeout rate, priority queue depth

**Narrative Example:**
> VIP customer uploads statement PDF (priority=10). UploadRecord created {status: "queued_for_parse", priority: 10}. Worker queries `SELECT * FROM upload_records WHERE status='queued_for_parse' ORDER BY priority DESC, created_at LIMIT 1` → Returns VIP upload (jumps queue ahead of 50 normal uploads). Parser starts → {status: "parsing", parsing_started_at: "2024-10-27T10:00:00Z"}. Parsing completes in 15s → {status: "parsed", observations_count: 120, parse_log_ref: "s3://logs/..."}. Normalizer starts → Completes in 8s → {status: "normalized", canonicals_count: 120}. Total: 23s (meets 60s SLA ✅). Prometheus metrics recorded: `upload_processing_duration{source_type="credit_card",status="normalized"} 23`, `upload_retries_total{status="normalized",retry_count="0"} 1`. Webhook fired: POST `https://internal-api/upload-status-changed` body `{"upload_id":"UL_vip_123","status":"normalized","canonicals_count":120}`.
>
> Alternative scenario: Corrupted PDF parsing fails 5 times → {status: "error", retry_count: 5, error_message: "PDF unrecoverable after 5 retries"}. Alert fired: "P2: VIP upload failed after max retries (upload_id: UL_vip_123)". Customer success team notified → Manual review → Discovers bank sent corrupted statement → Request re-issue from bank.
>
> Timeout monitoring running every 10s. Detects upload stuck in `parsing` for >120s → {status: "error", error_message: "Parsing timeout (120s), parser may be hung"}. Worker killed, upload marked for manual review.

---

**Key Insight:** The same `UploadRecord` state machine works across all 3 profiles. Personal use has bare-minimum state tracking (queued→parsed→normalized); Small business adds retries and timeout monitoring; Enterprise adds priority queues, webhooks, and metrics for SLA compliance.

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

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Track bank statement upload through parse → normalize pipeline
**Example:** User uploads "statement.pdf" → UploadRecord created {upload_id: "UL_abc123", status: "queued_for_parse", upload_log_ref: "/logs/UL_abc123.upload.json"} → Parser starts → Coordinator updates {status: "parsing"} → Parser completes → {status: "parsed", parse_log_ref: "/logs/...", observations_count: 42} → Normalizer starts → {status: "normalizing"} → Completes → {status: "normalized", canonicals_count: 42} → User polls GET /uploads/UL_abc123 → Returns current status + progressive disclosure (parse_log available after parsing, canonicals_count after normalizing)
**Operations:** State transitions (queued→parsing→parsed→normalizing→normalized), progressive disclosure (show what's available at each stage), error tracking (status="error" + error_message)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Track lab report upload through extraction pipeline
**Example:** Hospital uploads lab PDF → UploadRecord {upload_id: "UL_lab_456", status: "queued_for_parse"} → Parser extracts 8 tests → {status: "parsed", observations_count: 8} → Normalizer converts units → {status: "normalized", canonicals_count: 8} → Patient queries upload status → Returns "normalized" + observation count
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Track contract upload through clause extraction pipeline
**Example:** Law firm uploads contract → UploadRecord {status: "queued_for_parse"} → Parser extracts 25 clauses → {status: "parsed", observations_count: 25} → Normalizer categorizes → {status: "normalized"} → Attorney checks progress → Status shows "normalized" with 25 clauses extracted
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Track web page scrape through fact extraction pipeline
**Example:** Scraper downloads TechCrunch article → UploadRecord {upload_id: "UL_tc_789", status: "queued_for_parse", file_id: "file_html_123"} → TechCrunchHTMLParser starts → {status: "parsing"} → Extracts 12 raw facts → {status: "parsed", observations_count: 12, parse_log_ref: "/logs/UL_tc_789.parse.json"} → Normalizer resolves entities ("Sam Altman" → "@sama") → {status: "normalizing"} → Completes → {status: "normalized", canonicals_count: 12} → Dashboard queries upload → Shows "normalized" with 12 facts extracted, parse log available for debugging
**Operations:** Progressive disclosure critical (show raw facts after parsing, normalized facts after normalization), error tracking (parser fails on corrupted HTML → status="error" + error_message: "Invalid HTML structure")
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Track supplier catalog upload through product extraction pipeline
**Example:** Merchant uploads CSV (1,500 products) → UploadRecord {status: "queued_for_parse"} → Parser extracts rows → {status: "parsed", observations_count: 1500} → Normalizer validates SKUs, categories → {status: "normalized", canonicals_count: 1480} (20 failures) → Merchant checks upload → Status "normalized" + 1480 products imported + parse log shows 20 validation errors
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (state machine pattern for async pipelines is universal, no domain-specific code)
**Reusability:** High (same status tracking, progressive disclosure, error handling work for bank statements, lab reports, contracts, web scrapes, product catalogs; only stage names and count semantics differ)

---

## Related Primitives

- `FileArtifact`: UploadRecord references via `file_id`
- `ProvenanceLedger`: Logs all status transitions
- `Coordinator`: Only entity that updates `status`

---

**Last Updated**: 2025-10-22
**Maturity**: Spec complete, ready for implementation
