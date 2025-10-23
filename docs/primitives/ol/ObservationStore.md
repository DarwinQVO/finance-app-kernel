# OL Primitive: ObservationStore

**Type**: Storage / Data Repository
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.2 (Extraction)

---

## Purpose

Persistent storage for raw extracted observations (unvalidated, AS-IS from parser). Provides idempotent upsert operations and efficient querying by `upload_id`. Observations are immutable once stored - they are inputs to normalization (vertical 1.3), never modified.

---

## Interface Contract

### Core Methods

```python
from typing import List, Optional

class ObservationStore:
    def upsert(self, observation: Observation) -> None:
        """
        Insert or update observation by composite key (upload_id, row_id).
        Idempotent: Re-running parser produces same observations.

        Args:
            observation: Raw observation from parser

        Raises:
            ValidationError: If observation schema invalid
        """
        pass

    def get_by_upload(self, upload_id: str) -> List[Observation]:
        """
        Retrieve all observations for an upload, ordered by row_id.

        Args:
            upload_id: Upload identifier (e.g., "UL_abc123")

        Returns:
            List of observations (empty list if none found)
        """
        pass

    def get(self, upload_id: str, row_id: int) -> Optional[Observation]:
        """
        Retrieve single observation by composite key.

        Returns:
            Observation if found, None otherwise
        """
        pass

    def delete_by_upload(self, upload_id: str) -> int:
        """
        Delete all observations for an upload.

        Returns:
            Number of observations deleted
        """
        pass

    def count_by_upload(self, upload_id: str) -> int:
        """
        Count observations for an upload.

        Returns:
            Number of observations
        """
        pass
```

### Types

```python
from dataclasses import dataclass
from typing import Any, Dict, List

@dataclass
class Observation:
    """
    Raw extracted data (domain-specific schema).
    Examples: ObservationTransaction, ObservationLabResult, etc.
    """
    upload_id: str          # Composite key part 1
    row_id: int             # Composite key part 2 (0-indexed)
    raw_data: Dict[str, Any]  # Domain-specific fields
    parser_id: str          # Which parser extracted this
    parser_version: str     # Parser version (e.g., "1.2.0")
    extracted_at: str       # ISO 8601 timestamp
    warnings: List[str]     # Row-level warnings
```

---

## Behavior Specifications

### 1. Idempotent Upserts

**Property**: Re-parsing same file → same observations → no duplicates.

**Implementation:**
```python
def upsert(self, observation: Observation) -> None:
    # Composite key: (upload_id, row_id)
    key = (observation.upload_id, observation.row_id)

    # Upsert (INSERT ON CONFLICT UPDATE)
    db.execute(
        """
        INSERT INTO observations (upload_id, row_id, raw_data, ...)
        VALUES (:upload_id, :row_id, :raw_data, ...)
        ON CONFLICT (upload_id, row_id)
        DO UPDATE SET raw_data = :raw_data, ...
        """,
        observation
    )
```

**Guarantee**: Calling `upsert()` multiple times with same `(upload_id, row_id)` → single row.

---

### 2. Immutability After Storage

**Property**: Once stored, observations are read-only (never modified).

**Rationale** (from ADR-0004):
- Preserve original data from parser
- Enable re-normalization without re-parsing
- Observations are inputs to normalization, not outputs

**Enforcement:**
- No `update()` method (only `upsert()` during parsing)
- Application-level: Only parser writes observations
- Database-level: Read-only access for normalization stage

---

### 3. Ordered Retrieval

**Property**: `get_by_upload()` returns observations in `row_id` order (stable).

```python
observations = store.get_by_upload("UL_abc123")

assert observations[0].row_id == 0
assert observations[1].row_id == 1
# ... ordered sequence
```

**SQL:**
```sql
SELECT * FROM observations
WHERE upload_id = ?
ORDER BY row_id ASC
```

---

### 4. Domain-Specific Schemas

**Property**: `raw_data` field is JSONB (flexible schema per domain).

**Finance Domain:**
```python
observation = Observation(
    upload_id="UL_abc123",
    row_id=0,
    raw_data={
        "date": "01/15/2024",
        "amount": "-5.75",
        "description": "STARBUCKS #1234",
        "currency": "USD",
        "account": "bofa_debit"
    },
    parser_id="bofa_pdf_parser",
    parser_version="1.2.0",
    extracted_at="2025-10-23T14:35:10Z",
    warnings=[]
)
```

**Healthcare Domain:**
```python
observation = Observation(
    upload_id="UL_lab456",
    row_id=0,
    raw_data={
        "test_name": "Glucose",
        "value": "95",
        "unit": "mg/dL",
        "normal_range": "70-100"
    },
    parser_id="lab_results_parser",
    parser_version="1.0.0",
    extracted_at="2025-10-23T14:40:20Z",
    warnings=[]
)
```

---

## Storage Schema

### SQL (PostgreSQL)

```sql
CREATE TABLE observations (
    upload_id TEXT NOT NULL,
    row_id INTEGER NOT NULL,
    raw_data JSONB NOT NULL,
    parser_id TEXT NOT NULL,
    parser_version TEXT NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL,
    warnings TEXT[] DEFAULT '{}',

    PRIMARY KEY (upload_id, row_id)
);

CREATE INDEX idx_observations_upload_id ON observations(upload_id);
CREATE INDEX idx_observations_extracted_at ON observations(extracted_at);
```

### NoSQL (MongoDB)

```javascript
db.observations.createIndex({ "upload_id": 1, "row_id": 1 }, { unique: true })
db.observations.createIndex({ "upload_id": 1 })
db.observations.createIndex({ "extracted_at": 1 })
```

---

## Error Handling

| Error | When | Response |
|-------|------|----------|
| `DuplicateKeyError` | Same (upload_id, row_id) inserted twice | Overwrite (upsert semantics) |
| `ValidationError` | observation schema invalid | Raise error, log details |
| `ForeignKeyError` | upload_id doesn't exist in upload_records | Raise error (referential integrity) |

---

## Configuration

```yaml
observation_store:
  backend: "postgresql"  # or "mongodb", "dynamodb"
  connection_string: "postgresql://localhost/finance_app"
  table_name: "observations"
  batch_size: 1000  # For bulk inserts
  retention_days: 365  # Archive observations after N days
```

---

## Multi-Domain Applicability

This primitive constructs verifiable truth about **raw extracted data storage** - a universal concept across ALL domains:

**Finance Domain:**
- Store: `ObservationTransaction[]` from bank statements
- Composite key: `(upload_id, row_id)`
- Example: 42 transaction observations from single PDF

**Healthcare Domain:**
- Store: `ObservationLabResult[]` from lab reports
- Composite key: `(upload_id, row_id)`
- Example: 15 test results from single lab report PDF

**Legal Domain:**
- Store: `ObservationClause[]` from contracts
- Composite key: `(upload_id, row_id)`
- Example: 50 clauses from contract PDF

**Research Domain:**
- Store: `ObservationCitation[]` from bibliographies
- Composite key: `(upload_id, row_id)`
- Example: 30 citations from research paper

**Manufacturing Domain:**
- Store: `ObservationMeasurement[]` from sensor logs
- Composite key: `(upload_id, row_id)`
- Example: 10,000 sensor readings from CSV

**Media Domain:**
- Store: `ObservationUtterance[]` from transcripts
- Composite key: `(upload_id, row_id)`
- Example: 500 utterances from video transcript

---

## Performance Characteristics

| Operation | Time Complexity | Notes |
|-----------|-----------------|-------|
| `upsert()` | O(1) | Single row insert/update |
| `get_by_upload()` | O(n) where n = rows | Index scan on upload_id |
| `get()` | O(1) | Primary key lookup |
| `count_by_upload()` | O(1) | Index-only scan |
| `delete_by_upload()` | O(n) where n = rows | Bulk delete |

**Benchmarks (PostgreSQL, 100 observations):**
- Upsert single observation: <5ms
- Bulk upsert 100 observations: <50ms
- Get by upload_id (100 rows): <20ms
- Count by upload_id: <10ms

---

## Testing Strategy

### Unit Tests

```python
def test_idempotent_upsert():
    store = ObservationStore()

    obs = Observation(
        upload_id="UL_test",
        row_id=0,
        raw_data={"amount": "-5.75"},
        parser_id="test_parser",
        parser_version="1.0.0",
        extracted_at="2025-10-23T14:00:00Z",
        warnings=[]
    )

    # Insert
    store.upsert(obs)
    assert store.count_by_upload("UL_test") == 1

    # Re-insert (upsert)
    store.upsert(obs)
    assert store.count_by_upload("UL_test") == 1  # Still 1

def test_ordered_retrieval():
    store = ObservationStore()

    # Insert out of order
    store.upsert(Observation(upload_id="UL_test", row_id=2, ...))
    store.upsert(Observation(upload_id="UL_test", row_id=0, ...))
    store.upsert(Observation(upload_id="UL_test", row_id=1, ...))

    # Retrieve in order
    observations = store.get_by_upload("UL_test")
    assert [obs.row_id for obs in observations] == [0, 1, 2]
```

---

## Security Considerations

### 1. Referential Integrity

```python
def upsert(self, observation: Observation) -> None:
    # Verify upload_id exists
    if not upload_record_exists(observation.upload_id):
        raise ValidationError(f"upload_id {observation.upload_id} not found")

    # Proceed with upsert
    super().upsert(observation)
```

### 2. Query Limits

```python
def get_by_upload(self, upload_id: str, limit: int = 10000) -> List[Observation]:
    # Prevent unbounded queries
    observations = db.query(
        "SELECT * FROM observations WHERE upload_id = ? ORDER BY row_id LIMIT ?",
        upload_id, limit
    )

    if len(observations) == limit:
        logger.warn(f"Hit query limit: {limit} rows for {upload_id}")

    return observations
```

---

## Observability

### Metrics

```python
observation_store_operations_total{operation="upsert", status="success"} 5240
observation_store_operations_total{operation="get_by_upload", status="success"} 320
observation_store_rows_total 52420
observation_store_latency_seconds{operation="upsert", quantile="0.95"} 0.005
```

### Logs

```json
{
  "timestamp": "2025-10-23T14:35:10Z",
  "level": "INFO",
  "event": "observation.upserted",
  "upload_id": "UL_abc123",
  "row_id": 0,
  "parser_id": "bofa_pdf_parser"
}
```

---

## Related Primitives

- **Parser**: Produces observations that are stored here
- **UploadRecord**: References observations via `observations_count`
- **Normalizer** (Vertical 1.3): Reads observations, produces canonicals
- **ProvenanceLedger**: Tracks parse events that create observations

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
