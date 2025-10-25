# ADR-0002: Global State Machine (Single Source of Truth)

**Status**: ✅ Accepted
**Date**: 2025-10-22
**Scope**: Global (affects all verticals)

---

## Context

Our pipeline has multiple stages:

```
Upload → Parse → Normalize → (Future: Categorize, Reconcile, etc.)
```

Each stage is asynchronous (workers process files independently). We need to track:
- Where is each upload in the pipeline?
- Has processing succeeded or failed?
- Can we retry from a specific stage?

**Options considered:**

1. **Per-stage status fields**: `upload_status`, `parse_status`, `normalize_status`, etc.
2. **Global state machine**: Single `status` field representing the entire pipeline
3. **Event log only**: No status field, reconstruct state from events

---

## Decision

**We use a single, global state machine with one `status` field in `UploadRecord`.**

### State Diagram

```
[Initial]
    ↓
queued_for_parse
    ↓
  parsing
    ↓
  parsed
    ↓
normalizing
    ↓
normalized
    ↓
(future: categorized, reconciled, indexed, etc.)

Note: "error" can occur at ANY stage
```

### Valid States (v1)

| State | Meaning | Next State(s) |
|-------|---------|---------------|
| `queued_for_parse` | Upload complete, waiting for parser | `parsing`, `error` |
| `parsing` | Parser is running | `parsed`, `error` |
| `parsed` | Parser succeeded, observations stored | `normalizing`, `error` |
| `normalizing` | Normalizer is running | `normalized`, `error` |
| `normalized` | Normalizer succeeded, canonicals stored | (future: `categorizing`) |
| `error` | Pipeline failed at some stage | (manual retry resets to previous state) |

---

## Rationale

### 1. Single Source of Truth

**Problem with per-stage status:**
```json
{
  "upload_status": "complete",
  "parse_status": "in_progress",
  "normalize_status": "pending"
}
```
- **Ambiguous**: What's the "current" state? Need complex logic to determine.
- **Inconsistencies**: What if `parse_status="complete"` but `normalize_status="in_progress"`? Invalid state.

**Global state machine:**
```json
{
  "status": "parsing"
}
```
- **Unambiguous**: Always know where we are
- **No invalid states**: State transitions are strictly controlled

### 2. Simplified API Contract

**One field, multiple consumers:**
- Frontend: `if (status === 'error') { showRetryButton() }`
- Workers: `SELECT * FROM uploads WHERE status = 'queued_for_parse'`
- Monitoring: `COUNT(*) WHERE status = 'error' GROUP BY error_message`

**Alternative (per-stage) would require:**
- Complex queries: `WHERE parse_status != 'error' AND normalize_status = 'pending'`
- Multiple polling endpoints: `/uploads/:id/parse_status`, `/uploads/:id/normalize_status`

### 3. Easy to Extend

Adding new stages:
```diff
  normalized
+     ↓
+ categorizing
+     ↓
+ categorized
```

Just extend the enum. No new columns, no schema migration (if using string enum).

### 4. Auditability

Provenance ledger tracks state transitions:
```json
[
  { "timestamp": "...", "old_status": null, "new_status": "queued_for_parse" },
  { "timestamp": "...", "old_status": "queued_for_parse", "new_status": "parsing" },
  { "timestamp": "...", "old_status": "parsing", "new_status": "parsed" }
]
```

Clear timeline of pipeline progression.

### 5. Idempotency & Retries

If a stage fails:
- `status` moves to `error`
- `error_message` explains why
- Retry can reset to previous valid state:
  - `error` (failed at parse) → retry → `queued_for_parse`
  - `error` (failed at normalize) → retry → `parsed`

---

## Consequences

### ✅ Positive

- **Simple queries**: `WHERE status = 'X'` (no joins or complex conditions)
- **Clear semantics**: Status always reflects "what's happening now"
- **Easy monitoring**: Count uploads per state, track error rate
- **Extensible**: Add new states without breaking existing API
- **Atomic transitions**: Single field update = atomic state change

### ⚠️ Trade-offs

- **Granularity**: Can't represent "parse succeeded AND normalize in progress" explicitly
  - **Mitigation**: Use conditional fields (`parse_log_ref` appears when parse done)
- **History loss**: Single field doesn't show "was parsing, now normalizing"
  - **Mitigation**: `ProvenanceLedger` tracks all transitions

### 🔴 Risks (Mitigated)

- **Risk**: Race condition if multiple coordinators update `status` concurrently
  - **Mitigation #1**: Only **Coordinator** can update `status` (see ADR-0003)
  - **Mitigation #2**: **Optimistic locking** using `version` field (see below)
  - Workers write data/logs, emit events, but never touch `status`

- **Risk**: "Stuck" states (status never advances)
  - **Mitigation**: Timeout monitoring (if `status=parsing` for >60min → alert)

---

## Optimistic Locking (Concurrency Control)

**Problem**: In distributed deployments with multiple coordinator instances, two coordinators might try to update the same upload's status simultaneously:

```
Time  Coordinator A              Coordinator B
----  ------------------------  ------------------------
T0    Read: status=parsed,       Read: status=parsed,
      version=1                  version=1
T1    Transition to normalizing  Transition to normalizing
T2    Write: status=normalizing  Write: status=normalizing
      (overwrites B's update!)   (lost update!)
```

**Solution**: Add `version` field (integer, auto-incremented) to `UploadRecord`:

```sql
CREATE TABLE upload_records (
  upload_id TEXT PRIMARY KEY,
  status TEXT NOT NULL,
  version INTEGER NOT NULL DEFAULT 0,  -- Optimistic lock
  -- ... other fields
);
```

### Implementation

**Coordinator transition logic:**

```python
def transition_status(upload_id: str, from_state: str, to_state: str):
    """
    Transition upload status with optimistic locking.

    Raises:
        ConcurrentModificationError: If another coordinator modified the record
    """
    # 1. Read current record with version
    upload = db.query(UploadRecord).filter_by(upload_id=upload_id).first()
    current_version = upload.version
    current_status = upload.status

    # 2. Validate state transition
    if current_status != from_state:
        raise InvalidStateTransition(
            f"Expected {from_state}, found {current_status}"
        )

    # 3. Attempt atomic update with version check
    result = db.execute(
        """
        UPDATE upload_records
        SET status = :to_state,
            version = :new_version,
            updated_at = NOW()
        WHERE upload_id = :upload_id
          AND version = :current_version  -- Optimistic lock check
        """,
        {
            "upload_id": upload_id,
            "to_state": to_state,
            "current_version": current_version,
            "new_version": current_version + 1
        }
    )

    # 4. Check if update succeeded
    if result.rowcount == 0:
        # Version mismatch → concurrent modification detected
        raise ConcurrentModificationError(
            f"Upload {upload_id} was modified by another coordinator. Retry."
        )

    # 5. Log state transition to ProvenanceLedger
    log_state_transition(upload_id, from_state, to_state)
```

### Behavior

**Concurrent update scenario:**

```
Time  Coordinator A                    Coordinator B
----  ------------------------------  ------------------------------
T0    Read: version=1, status=parsed   Read: version=1, status=parsed
T1    UPDATE ... WHERE version=1       (processing...)
      ✅ Success (version → 2)
T2    (done)                           UPDATE ... WHERE version=1
                                       ❌ rowcount=0 (version is now 2)
                                       → ConcurrentModificationError
T3                                     Retry: Read version=2
                                       UPDATE ... WHERE version=2
                                       ✅ Success
```

### Error Handling

```python
MAX_RETRIES = 3

for attempt in range(MAX_RETRIES):
    try:
        transition_status(upload_id, from_state, to_state)
        break  # Success
    except ConcurrentModificationError:
        if attempt == MAX_RETRIES - 1:
            raise  # Give up after max retries
        time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
```

### Benefits

✅ **No locks needed**: Database-level atomic update (no `SELECT FOR UPDATE`)
✅ **Performance**: No row-level locks, high concurrency
✅ **Simple**: Single integer field, standard pattern
✅ **Detects lost updates**: Guaranteed to catch concurrent modifications

### Trade-offs

⚠️ **Retry required**: Coordinators must handle `ConcurrentModificationError` and retry
⚠️ **Version field overhead**: +4 bytes per record (minimal)

---

## Alternatives Considered

### Alternative A: Per-Stage Status Fields (Rejected)

```json
{
  "upload_status": "complete",
  "parse_status": "in_progress",
  "normalize_status": "pending",
  "categorize_status": null
}
```

**Cons:**
- Complex state space (4 fields × 5 states = potentially invalid combinations)
- No clear "current" state
- Harder to query ("give me all uploads currently processing")
- Schema grows with every new stage

### Alternative B: Event Log Only (Rejected)

No `status` field. Reconstruct state by querying event log:

```sql
SELECT event_type FROM events WHERE upload_id = 'UL_abc123' ORDER BY timestamp DESC LIMIT 1
```

**Cons:**
- **Performance**: Every status check requires log query + aggregation
- **Complexity**: Client must understand event → state mapping
- **Monitoring difficulty**: Can't index/query by current state efficiently

### Alternative C: Status + Substatus (Rejected)

```json
{
  "status": "processing",
  "substatus": "parsing"
}
```

**Cons:**
- **Redundant**: `substatus` IS the real status
- **Two sources of truth**: Which field to trust?
- **No benefit**: Doesn't solve any problem the global state machine doesn't already solve

---

## Implementation Details

### State Authority (see ADR-0003)

**Only Coordinator updates `status`:**
```python
# ❌ NEVER in runner code
upload_record.status = "parsing"

# ✅ ONLY in coordinator
coordinator.transition(upload_id, from_state="queued_for_parse", to_state="parsing")
```

### Conditional Fields in API

`GET /uploads/:upload_id` returns different fields based on `status`:

```json
// status = "queued_for_parse"
{
  "upload_id": "UL_abc123",
  "status": "queued_for_parse",
  "upload_log_ref": "/logs/uploads/UL_abc123.log"
}

// status = "parsed"
{
  "upload_id": "UL_abc123",
  "status": "parsed",
  "upload_log_ref": "/logs/uploads/UL_abc123.log",
  "parse_log_ref": "/logs/parse/UL_abc123.log.json",  ← NEW
  "observations_count": 42                             ← NEW
}
```

This gives stage-specific details **without separate status fields**.

---

## Validation Rules

### Allowed Transitions (v1)

```
queued_for_parse → parsing | error
parsing          → parsed | error
parsed           → normalizing | error
normalizing      → normalized | error
error            → queued_for_parse | parsed  (retry logic)
normalized       → (future states)
```

### Forbidden Transitions

```
parsing → normalized  ❌ (must go through parsed)
normalized → parsing  ❌ (no backward moves except retry from error)
```

Coordinator enforces these rules.

---

## Future Extensions

### Additional States (v2+)

```
normalized
    ↓
categorizing
    ↓
categorized
    ↓
reconciling
    ↓
reconciled
    ↓
indexed
```

### State Metadata (Optional)

```json
{
  "status": "parsing",
  "status_metadata": {
    "started_at": "2025-10-22T14:35:00Z",
    "worker_id": "parser-worker-3",
    "retry_count": 0
  }
}
```

Could be added without breaking existing API.

---

## Related Decisions

- **ADR-0001**: Uses `upload_id` as key for `UploadRecord`
- **ADR-0003**: Runner/Coordinator split (only Coordinator updates `status`)
- **Future ADR**: Timeout policies per state

---

**References:**
- Vertical 1.1: [docs/verticals/1.1-upload-flow.md](../verticals/1.1-upload-flow.md)
- Schema: [docs/schemas/upload-record.schema.json](../schemas/upload-record.schema.json)
- `.claude.md` section: [Non-Negotiable Decisions → Identity & State](../.claude.md#identity--state)
