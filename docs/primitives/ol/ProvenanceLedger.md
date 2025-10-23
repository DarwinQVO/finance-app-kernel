# OL Primitive: ProvenanceLedger

**Type**: Audit / Governance
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification

---

## Purpose

Append-only, immutable audit trail that records every event, decision, and transformation in the system. Provides complete lineage from raw input to final output, enabling compliance, debugging, and trust.

---

## Interface Contract

### Core Methods

```typescript
interface ProvenanceLedger {
  // Append new entry (returns entry ID)
  append(entry: ProvenanceEntry): EntryID

  // Query entries by upload_id
  getByUploadId(upload_id: string): ProvenanceEntry[]

  // Query entries by timestamp range
  getByTimeRange(start: ISO8601, end: ISO8601): ProvenanceEntry[]

  // Query entries by event type
  getByEventType(event_type: string): ProvenanceEntry[]

  // Get full chain for a canonical fact
  getChain(canonical_id: string): ProvenanceChain

  // Verify integrity (check no tampering)
  verifyIntegrity(): IntegrityReport
}
```

### Types

```typescript
type EntryID = string  // Format: "PE_{timestamp}_{sequence}"

interface ProvenanceEntry {
  entry_id: EntryID
  timestamp: ISO8601Timestamp
  upload_id: string
  event_type: string  // e.g., "upload.created", "parse.completed"
  actor: Actor  // Who/what caused this event
  data: Record<string, any>  // Event-specific payload
  previous_entry_id: EntryID | null  // For chain validation
  signature?: string  // Cryptographic signature (optional)
}

interface Actor {
  type: "user" | "system" | "runner"
  id: string  // user_id, system_component_name, or runner_instance_id
}

interface ProvenanceChain {
  canonical_id: string
  entries: ProvenanceEntry[]  // Ordered by timestamp
  lineage_summary: string  // Human-readable description
}

interface IntegrityReport {
  total_entries: number
  verified_entries: number
  broken_chains: number
  tampered_entries: EntryID[]
}
```

---

## Behavior Specifications

### 1. Append-Only Guarantee

**Property**: Once written, entries NEVER change or delete

**Enforcement:**
```python
class ProvenanceLedger:
    def append(self, entry: ProvenanceEntry) -> EntryID:
        # No update or delete methods exist
        # Writes are atomic (single transaction)
        return self._atomic_append(entry)

    # ❌ NO update() method
    # ❌ NO delete() method
```

**Implications:**
- Corrections create new entries (don't modify old)
- Soft delete: Add `deleted=true` entry (original stays)
- Full history always available

---

### 2. Chain Validation

**Property**: Each entry references previous entry (blockchain-like)

**Structure:**
```
Entry 1: upload.created
  ↓ previous_entry_id = null
Entry 2: parse.started
  ↓ previous_entry_id = Entry 1 ID
Entry 3: parse.completed
  ↓ previous_entry_id = Entry 2 ID
Entry 4: normalize.completed
    previous_entry_id = Entry 3 ID
```

**Validation:**
```python
def verify_chain(self, upload_id: str) -> bool:
    entries = self.get_by_upload_id(upload_id)

    for i, entry in enumerate(entries):
        if i == 0:
            assert entry.previous_entry_id is None
        else:
            expected_prev = entries[i - 1].entry_id
            if entry.previous_entry_id != expected_prev:
                raise ChainBrokenError(f"Entry {entry.entry_id} points to wrong previous")

    return True
```

**Benefit**: Detect tampering (if entry removed, chain breaks)

---

### 3. Event Types (Standard Taxonomy)

**Upload Lifecycle:**
- `upload.created`
- `upload.dedupe_hit`

**Parse Lifecycle:**
- `parse.started`
- `parse.completed`
- `parse.failed`

**Normalize Lifecycle:**
- `normalize.started`
- `normalize.completed`
- `normalize.failed`

**State Transitions:**
- `state.transition` (from → to, reason)

**Corrections:**
- `correction.proposed`
- `correction.applied`
- `correction.rejected`

**User Actions:**
- `user.flag_review`
- `user.add_note`

---

### 4. Data Payload Structure

**upload.created:**
```json
{
  "entry_id": "PE_20251022143201_001",
  "timestamp": "2025-10-22T14:32:01Z",
  "upload_id": "UL_abc123",
  "event_type": "upload.created",
  "actor": {"type": "user", "id": "user_001"},
  "data": {
    "source_type": "bofa_pdf",
    "file_name": "statement.pdf",
    "file_hash": "sha256:abc123...",
    "file_size_bytes": 2457600
  },
  "previous_entry_id": null
}
```

**parse.completed:**
```json
{
  "entry_id": "PE_20251022143545_002",
  "timestamp": "2025-10-22T14:35:45Z",
  "upload_id": "UL_abc123",
  "event_type": "parse.completed",
  "actor": {"type": "runner", "id": "parser-worker-3"},
  "data": {
    "parser_id": "bofa_pdf_parser",
    "parser_version": "1.2.0",
    "observations_count": 42,
    "parse_log_ref": "/logs/parse/UL_abc123.log.json",
    "duration_ms": 3421
  },
  "previous_entry_id": "PE_20251022143201_001"
}
```

**state.transition:**
```json
{
  "entry_id": "PE_20251022143546_003",
  "timestamp": "2025-10-22T14:35:46Z",
  "upload_id": "UL_abc123",
  "event_type": "state.transition",
  "actor": {"type": "system", "id": "coordinator"},
  "data": {
    "from_state": "parsing",
    "to_state": "parsed",
    "trigger": "parse.completed"
  },
  "previous_entry_id": "PE_20251022143545_002"
}
```

---

## Query Patterns

### 1. Full Upload Timeline

```python
def get_upload_timeline(upload_id: str) -> List[ProvenanceEntry]:
    return ledger.get_by_upload_id(upload_id)

# Returns:
# [
#   upload.created,
#   parse.started,
#   parse.completed,
#   state.transition (parsing → parsed),
#   normalize.started,
#   normalize.completed,
#   state.transition (normalizing → normalized)
# ]
```

### 2. Canonical Fact Lineage

```python
def get_canonical_lineage(canonical_id: str) -> ProvenanceChain:
    # Find which upload created this canonical
    upload_id = canonical_store.get_upload_id(canonical_id)

    # Get full chain
    entries = ledger.get_by_upload_id(upload_id)

    return ProvenanceChain(
        canonical_id=canonical_id,
        entries=entries,
        lineage_summary=f"""
        Canonical {canonical_id} was:
        1. Uploaded from {entries[0].data['file_name']}
        2. Parsed by {entries[2].data['parser_id']} v{entries[2].data['parser_version']}
        3. Normalized on {entries[4].timestamp}
        """
    )
```

### 3. Error Forensics

```python
def find_failed_uploads(since: ISO8601) -> List[str]:
    # Find all parse.failed or normalize.failed events
    failed_parses = ledger.get_by_event_type("parse.failed")
    failed_normalizes = ledger.get_by_event_type("normalize.failed")

    return [e.upload_id for e in (failed_parses + failed_normalizes) if e.timestamp > since]
```

### 4. User Actions Audit

```python
def get_user_actions(user_id: str, start: ISO8601, end: ISO8601) -> List[ProvenanceEntry]:
    entries = ledger.get_by_time_range(start, end)
    return [e for e in entries if e.actor.type == "user" and e.actor.id == user_id]
```

---

## Storage Backend

### Option 1: Append-Only Log File

```python
class FileProvenanceLedger(ProvenanceLedger):
    def __init__(self, log_path: str):
        self.log_path = log_path

    def append(self, entry: ProvenanceEntry) -> EntryID:
        # Generate entry_id
        entry.entry_id = self._generate_entry_id()

        # Serialize
        line = json.dumps(entry.to_dict()) + "\n"

        # Atomic append (O_APPEND flag ensures atomicity)
        with open(self.log_path, 'a') as f:
            f.write(line)

        return entry.entry_id

    def get_by_upload_id(self, upload_id: str) -> List[ProvenanceEntry]:
        # Sequential scan (optimize with index in production)
        entries = []
        with open(self.log_path, 'r') as f:
            for line in f:
                entry = ProvenanceEntry.from_json(line)
                if entry.upload_id == upload_id:
                    entries.append(entry)
        return entries
```

**Pros:**
- Simple, crash-resistant (append-only)
- Easy to backup (just copy file)

**Cons:**
- Slow queries (need to scan entire file)

**Optimization:** Build index on upload_id, event_type, timestamp

---

### Option 2: Database (PostgreSQL)

```sql
CREATE TABLE provenance_ledger (
    entry_id TEXT PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL,
    upload_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    actor_type TEXT NOT NULL,
    actor_id TEXT NOT NULL,
    data JSONB NOT NULL,
    previous_entry_id TEXT REFERENCES provenance_ledger(entry_id),
    signature TEXT
);

CREATE INDEX idx_upload_id ON provenance_ledger(upload_id);
CREATE INDEX idx_event_type ON provenance_ledger(event_type);
CREATE INDEX idx_timestamp ON provenance_ledger(timestamp);
```

**Insert:**
```python
def append(self, entry: ProvenanceEntry) -> EntryID:
    entry.entry_id = self._generate_entry_id()

    self.db.execute("""
        INSERT INTO provenance_ledger (entry_id, timestamp, upload_id, event_type, actor_type, actor_id, data, previous_entry_id)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
    """, (entry.entry_id, entry.timestamp, entry.upload_id, entry.event_type, entry.actor.type, entry.actor.id, json.dumps(entry.data), entry.previous_entry_id))

    return entry.entry_id
```

**Query:**
```python
def get_by_upload_id(self, upload_id: str) -> List[ProvenanceEntry]:
    rows = self.db.execute("""
        SELECT * FROM provenance_ledger WHERE upload_id = %s ORDER BY timestamp
    """, (upload_id,))

    return [ProvenanceEntry.from_row(r) for r in rows]
```

---

## Cryptographic Integrity (Optional)

### Signed Entries

```python
def append_signed(self, entry: ProvenanceEntry, private_key: Key) -> EntryID:
    # Serialize entry data
    payload = json.dumps({
        "timestamp": entry.timestamp,
        "upload_id": entry.upload_id,
        "event_type": entry.event_type,
        "data": entry.data
    }, sort_keys=True)

    # Sign with private key
    entry.signature = sign(payload, private_key)

    return self.append(entry)

def verify_signature(self, entry: ProvenanceEntry, public_key: Key) -> bool:
    payload = json.dumps({
        "timestamp": entry.timestamp,
        "upload_id": entry.upload_id,
        "event_type": entry.event_type,
        "data": entry.data
    }, sort_keys=True)

    return verify_signature(payload, entry.signature, public_key)
```

**Use case:** Regulatory compliance (prove entries not tampered)

---

## Reusability Across Domains

### Finance
- Track transaction corrections, categorization changes
- Audit who flagged transactions for review
- Prove compliance (GDPR: "show me all changes to my data")

### Health
- Medical record updates (HIPAA compliance)
- Lab result corrections (who, when, why)
- Prescription history (dosage changes)

### Legal
- Document chain of custody
- Contract version history
- Evidence handling audit trail

### Generic Data Processing
- ETL pipeline lineage
- ML model training provenance (data → features → model)
- Report generation audit (who ran report, with what filters)

---

## Extension Points

### 1. Bitemporal Model (Future)

```python
interface BitemporalProvenanceEntry(ProvenanceEntry):
    valid_from: ISO8601  # When fact became true in real world
    valid_to: ISO8601    # When fact ceased to be true
    system_time: ISO8601  # When we learned about it
```

**Use case:** "What did we believe was true on 2024-01-15, based on data available at that time?"

---

### 2. Event Sourcing Integration

```python
class EventSourcedProvenanceLedger(ProvenanceLedger):
    def append(self, entry: ProvenanceEntry) -> EntryID:
        # Publish to event stream (Kafka, Redis Streams)
        self.event_stream.publish(entry.event_type, entry.to_dict())

        # Also persist
        return super().append(entry)
```

**Use case:** Real-time monitoring, webhooks, downstream consumers

---

### 3. Redaction for PII

```python
def append_with_redaction(self, entry: ProvenanceEntry, pii_fields: List[str]) -> EntryID:
    # Redact sensitive fields
    redacted_entry = entry.copy()
    for field in pii_fields:
        if field in redacted_entry.data:
            redacted_entry.data[field] = "[REDACTED]"

    return self.append(redacted_entry)

def get_unredacted(self, entry_id: EntryID, requester: User) -> ProvenanceEntry:
    # Only authorized users see PII
    if not requester.has_permission("view_pii"):
        return self.get(entry_id)  # Returns redacted version

    return self.get_raw(entry_id)  # Returns original
```

---

## Performance Characteristics

| Operation | Time Complexity | Notes |
|-----------|-----------------|-------|
| `append()` | O(1) | Single write |
| `get_by_upload_id()` | O(n) | Scan entire ledger (optimize with index) |
| `get_by_time_range()` | O(n) | Scan entire ledger |
| `verify_chain()` | O(k) | k = entries for upload_id |

**Optimization:**
- Index on `upload_id` → O(1) lookup
- Index on `timestamp` → Range queries in O(log n)

---

## Testing Strategy

### Unit Tests

```python
def test_append_only():
    ledger = ProvenanceLedger()

    entry_id = ledger.append(ProvenanceEntry(...))

    # Ensure no update/delete methods exist
    assert not hasattr(ledger, 'update')
    assert not hasattr(ledger, 'delete')

def test_chain_validation():
    ledger = ProvenanceLedger()

    entry1 = ledger.append(ProvenanceEntry(upload_id="UL_123", previous_entry_id=None))
    entry2 = ledger.append(ProvenanceEntry(upload_id="UL_123", previous_entry_id=entry1))

    assert ledger.verify_chain("UL_123") == True

def test_broken_chain_detection():
    ledger = ProvenanceLedger()

    entry1 = ledger.append(ProvenanceEntry(upload_id="UL_123", previous_entry_id=None))
    # Simulate tampering: entry2 points to non-existent previous
    entry2 = ledger.append(ProvenanceEntry(upload_id="UL_123", previous_entry_id="FAKE_ID"))

    with pytest.raises(ChainBrokenError):
        ledger.verify_chain("UL_123")
```

---

## Security Considerations

### 1. Immutability Enforcement

```python
# DB-level: Use INSERT-only permissions
GRANT INSERT ON provenance_ledger TO app_user;
REVOKE UPDATE, DELETE ON provenance_ledger FROM app_user;

# Application-level: No update/delete methods in interface
```

### 2. Access Control

```python
def get_by_upload_id(self, upload_id: str, requester: User) -> List[ProvenanceEntry]:
    # Check if user can view this upload
    if not acl.can_view(upload_id, requester):
        raise PermissionDeniedError()

    return self._get_by_upload_id(upload_id)
```

### 3. Audit Logging

```python
def append(self, entry: ProvenanceEntry) -> EntryID:
    entry_id = self._append(entry)

    # Log who appended what
    audit_log.write({
        "action": "provenance.append",
        "entry_id": entry_id,
        "caller": get_current_user()
    })

    return entry_id
```

---

## Observability

### Metrics

```python
provenance_entries_total{event_type="upload.created"} 1234
provenance_append_latency_seconds{quantile="0.95"} 0.002
provenance_chain_verifications_total{status="success"} 500
provenance_chain_verifications_total{status="broken"} 0
```

### Logs

```json
{
  "timestamp": "2025-10-22T14:35:00Z",
  "level": "INFO",
  "operation": "append",
  "entry_id": "PE_20251022143500_001",
  "event_type": "upload.created",
  "upload_id": "UL_abc123"
}
```

---

## Related Primitives

- `UploadRecord`: Reads ProvenanceLedger to show timeline
- `StorageEngine`: ProvenanceLedger records which StorageRef was used
- `CanonicalStore`: Queries ProvenanceLedger for lineage info

---

**Last Updated**: 2025-10-22
**Maturity**: Spec complete, ready for implementation
