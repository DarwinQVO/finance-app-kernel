# ADR-0003: Runner/Coordinator Separation of Concerns

**Status**: ✅ Accepted
**Date**: 2025-10-22
**Scope**: Global (affects all async processing)

---

## Context

Our pipeline has multiple async workers:

```
[Coordinator] → [Parse Runner] → [Normalize Runner] → [Future Runners...]
```

When a runner completes work, **someone needs to update the state machine**. The question is: **who?**

**Two options:**

1. **Runners update state directly**: Parser sets `status="parsed"` after extracting observations
2. **Coordinator pattern**: Runners emit events/write data, Coordinator updates state

---

## Decision

**We use the Coordinator pattern:**

- **Runners**: Execute domain logic, write data/logs, emit completion events
- **Coordinator**: Orchestrates state transitions, sole authority on `UploadRecord.status`

### Responsibilities

| Actor | Can Do | Cannot Do |
|-------|--------|-----------|
| **Runner** | • Execute parser/normalizer logic<br>• Write to `ObservationStore` / `CanonicalStore`<br>• Write structured logs (`ParseLog`, `NormalizerLog`)<br>• Emit events (`parse.completed`, `normalize.completed`) | ❌ Update `UploadRecord.status`<br>❌ Transition state machine<br>❌ Determine "what's next" |
| **Coordinator** | • Listen to events<br>• Validate state transitions<br>• Update `UploadRecord.status`<br>• Trigger next stage (e.g., queue for normalization)<br>• Handle retries and errors | ❌ Execute domain logic (parsing/normalizing)<br>❌ Write to domain stores directly |

---

## Rationale

### 1. Single Source of Truth for State

**Problem with runners updating state:**

```python
# Parse Runner
def parse(upload_id):
    observations = parser.extract(file)
    store.save(observations)

    # 🚨 PROBLEM: What if save() succeeded but this fails?
    upload_record.status = "parsed"
```

**Failure scenarios:**
- Network error after data write but before status update → stuck in `parsing` forever
- Race condition: Two runners processing same upload → conflicting state updates
- No single place to validate transitions → invalid states possible

**With Coordinator:**

```python
# Parse Runner
def parse(upload_id):
    observations = parser.extract(file)
    store.save(observations)
    log.write(parse_log)

    # Just emit event, don't touch status
    events.emit("parse.completed", {
        "upload_id": upload_id,
        "observations_count": len(observations)
    })

# Coordinator (separate process)
def on_parse_completed(event):
    upload_id = event["upload_id"]

    # Atomic state transition
    coordinator.transition(
        upload_id,
        from_state="parsing",
        to_state="parsed"
    )

    # Trigger next stage
    queue.enqueue("normalize", upload_id)
```

**Benefits:**
- **Idempotent events**: If runner retries, coordinator deduplicates
- **Clear failure boundary**: If coordinator crashes, state stays consistent (event reprocessed)
- **Centralized validation**: All transition rules in one place

### 2. Separation of Concerns

**Runners know domain logic:**
- How to parse BoFA PDFs
- How to normalize dates/amounts
- Business rules (validation, enrichment)

**Runners DON'T know:**
- What state comes next
- Whether to retry on failure
- How to handle concurrent requests

**Coordinator knows orchestration:**
- State machine transitions
- Retry policies
- Queue management
- Error handling strategies

**Coordinator DOESN'T know:**
- How to parse PDFs
- Domain validation rules

**Clean separation** → easier to:
- Test runners in isolation (no state dependencies)
- Change orchestration logic without touching domain code
- Add new runners without modifying coordinator (just register event handlers)

### 3. Auditability & Observability

**With runners updating state:**
```
# Logs scattered across runner instances
[parser-worker-1] Status: parsing → parsed
[parser-worker-2] Status: parsing → error
[normalizer-worker-3] Status: normalizing → normalized
```

**With Coordinator:**
```
# All transitions in one place
[coordinator] Upload UL_abc123: queued_for_parse → parsing (triggered by upload.created)
[coordinator] Upload UL_abc123: parsing → parsed (triggered by parse.completed, 42 observations)
[coordinator] Upload UL_abc123: parsed → normalizing (triggered by coordinator)
[coordinator] Upload UL_abc123: normalizing → normalized (triggered by normalize.completed, 42 canonicals)
```

**Benefits:**
- **Single audit trail** for state transitions
- **Provenance ledger** only needs to listen to coordinator events
- **Easier debugging**: "Why is upload stuck?" → check coordinator logs

### 4. Retry & Error Handling

**Runners can fail and retry without affecting state:**

```python
# Parse Runner (retries internally)
@retry(max_attempts=3, backoff=exponential)
def parse(upload_id):
    try:
        observations = parser.extract(file)
        store.save(observations)
        log.write(parse_log)
        events.emit("parse.completed", {...})
    except Exception as e:
        log.write({"error": str(e)})
        events.emit("parse.failed", {"upload_id": upload_id, "error": str(e)})
        raise  # Let retry decorator handle it
```

**Coordinator handles final outcome:**

```python
def on_parse_completed(event):
    coordinator.transition(upload_id, "parsing" → "parsed")

def on_parse_failed(event):
    coordinator.transition(upload_id, "parsing" → "error")
    coordinator.record_error(upload_id, event["error"])
```

**Benefits:**
- Runners retry transient errors (I/O, network) without coordinator involvement
- Coordinator only sees final result (success/failure)
- State machine never enters invalid intermediate states

---

## Consequences

### ✅ Positive

- **Consistency**: State always reflects actual system state (no partial updates)
- **Testability**: Runners are pure functions (input → output, no side effects on state)
- **Scalability**: Runners can scale horizontally (stateless), coordinator is single instance (or leader-elected)
- **Clear contracts**: Event schemas define runner/coordinator interface
- **Easier debugging**: One place to check "why did this transition happen?"

### ⚠️ Trade-offs

- **Latency**: Extra hop (runner → event → coordinator → state update) adds ~100ms
  - **Mitigation**: Acceptable for async pipeline (not user-facing)
- **Complexity**: Two components instead of one
  - **Mitigation**: Clear separation makes each component simpler

### 🔴 Risks (Mitigated)

- **Risk**: Coordinator becomes bottleneck
  - **Mitigation**: Coordinator is lightweight (just state updates), can handle 1000s/sec
  - **Future**: Shard by `upload_id` prefix if needed

- **Risk**: Event delivery failure (runner completes but event lost)
  - **Mitigation**: Use reliable queue (e.g., Redis Streams, Kafka)
  - **Timeout monitoring**: If `status=parsing` for >60min → coordinator re-checks store

- **Risk**: Coordinator crashes before updating state
  - **Mitigation**: Event reprocessing on coordinator restart (idempotent handlers)

---

## Alternatives Considered

### Alternative A: Runners Update State Directly (Rejected)

```python
# In runner code
upload_record.status = "parsed"
upload_record.observations_count = len(observations)
upload_record.save()
```

**Cons:**
- **Distributed state management**: Every runner needs DB write access to `UploadRecord`
- **Race conditions**: Two runners processing same upload → last write wins (data loss)
- **No validation**: Runners could set invalid states (`parsing` → `normalized` skipping `parsed`)
- **Tight coupling**: Runners need to know state machine logic
- **Hard to audit**: State changes happen in 10+ runner instances

### Alternative B: Runners Call Coordinator API (Rejected)

```python
# In runner code
coordinator.update_status(upload_id, "parsed")
```

**Pros:**
- Coordinator still controls state

**Cons:**
- **Synchronous coupling**: Runner blocks waiting for coordinator response
- **Failure complexity**: What if coordinator API is down? Runner must retry status update
- **No clear separation**: Runner still knows about state transitions

### Alternative C: Database Triggers (Rejected)

```sql
-- When ObservationStore gets new rows, auto-update UploadRecord
CREATE TRIGGER on_observations_inserted
AFTER INSERT ON observations
FOR EACH ROW
BEGIN
  UPDATE upload_records
  SET status = 'parsed'
  WHERE upload_id = NEW.upload_id AND status = 'parsing';
END;
```

**Cons:**
- **Hidden logic**: State transitions buried in DB, not visible in code
- **Hard to test**: Requires DB setup for tests
- **Inflexible**: Can't add complex logic (e.g., "only transition if observations_count > 0")
- **Poor observability**: No logs/events for state changes

---

## Implementation Details

### Event Schema

**parse.completed**
```json
{
  "event_type": "parse.completed",
  "upload_id": "UL_abc123",
  "observations_count": 42,
  "parse_log_ref": "/logs/parse/UL_abc123.log.json",
  "timestamp": "2025-10-22T14:35:42Z"
}
```

**parse.failed**
```json
{
  "event_type": "parse.failed",
  "upload_id": "UL_abc123",
  "error_code": "PDF_CORRUPT",
  "error_message": "Unable to extract text from pages 3-5",
  "parse_log_ref": "/logs/parse/UL_abc123.log.json",
  "timestamp": "2025-10-22T14:35:42Z"
}
```

### Coordinator Pseudocode

```python
class Coordinator:
    def on_event(self, event):
        handler = self.handlers[event["event_type"]]
        handler(event)

    def on_parse_completed(self, event):
        upload_id = event["upload_id"]

        # Validate current state
        record = db.get_upload_record(upload_id)
        if record.status != "parsing":
            logger.warning(f"Ignoring parse.completed for {upload_id} (status={record.status})")
            return  # Idempotent: event already processed

        # Update state atomically
        db.update_upload_record(upload_id, {
            "status": "parsed",
            "parse_log_ref": event["parse_log_ref"],
            "observations_count": event["observations_count"]
        })

        # Log transition
        provenance.append({
            "upload_id": upload_id,
            "event": "state_transition",
            "from": "parsing",
            "to": "parsed",
            "trigger": "parse.completed"
        })

        # Trigger next stage
        if event["observations_count"] > 0:
            queue.enqueue("normalize", upload_id)
        else:
            logger.warning(f"{upload_id}: 0 observations, skipping normalization")
```

---

## Testing Strategy

### Runner Tests (Isolated)

```python
def test_parse_runner():
    file = load_fixture("bofa_statement.pdf")

    # No database, no coordinator needed
    observations = parse_runner.run(file)

    assert len(observations) == 42
    assert observations[0].date == "2024-01-15"
    # ... domain logic tests
```

**No state machine dependencies** → fast, pure unit tests

### Coordinator Tests (State Logic)

```python
def test_coordinator_parse_completed():
    # Setup: Upload in "parsing" state
    db.create_upload_record("UL_test", status="parsing")

    # Trigger event
    coordinator.on_parse_completed({
        "upload_id": "UL_test",
        "observations_count": 42
    })

    # Assert state transition
    record = db.get_upload_record("UL_test")
    assert record.status == "parsed"
    assert record.observations_count == 42

    # Assert next stage triggered
    assert queue.peek() == ("normalize", "UL_test")
```

**No domain logic** → tests focus on orchestration

### Integration Tests

```python
def test_end_to_end_pipeline():
    # Upload file
    response = api.post("/uploads", file="bofa.pdf")
    upload_id = response.json()["upload_id"]

    # Wait for pipeline to complete
    wait_for(lambda: api.get(f"/uploads/{upload_id}").json()["status"] == "normalized")

    # Verify final state
    record = api.get(f"/uploads/{upload_id}").json()
    assert record["status"] == "normalized"
    assert record["canonicals_count"] == 42
```

---

## Related Decisions

- **ADR-0001**: `upload_id` is the key coordinator uses to track state
- **ADR-0002**: Global state machine that coordinator enforces
- **Future ADR**: Event queue technology choice (Redis Streams vs Kafka)

---

**References:**
- Vertical 1.2: [docs/verticals/1.2-extraction.md](../verticals/1.2-extraction.md) (TBD)
- Vertical 1.3: [docs/verticals/1.3-normalization.md](../verticals/1.3-normalization.md) (TBD)
- `.claude.md` section: [Non-Negotiable Decisions → Identity & State](../.claude.md#identity--state)
