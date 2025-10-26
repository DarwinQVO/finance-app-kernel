# ProvenanceTracer (OL Primitive)

## Definition

**ProvenanceTracer** traverses the complete data lineage from canonical record → observation → upload artifact. It assembles the full drill-down view by coordinating across multiple stores (CanonicalStore, ObservationStore, StorageEngine, ProvenanceLedger).

**Problem it solves:**
- Data lineage is scattered across multiple stores (canonical, observation, artifact, provenance)
- Manual assembly is error-prone (missing fields, incorrect joins)
- Access control must be enforced at each layer
- Performance degrades if queries are not optimized (N+1 problem)

**Solution:**
- Single entry point for complete lineage retrieval
- Optimized queries with JOIN strategies (not N+1)
- Built-in access control checks
- Graceful handling of missing data (deleted artifacts, corrupted observations)

---

## Interface Contract

```python
from typing import Optional
from dataclasses import dataclass

@dataclass
class DrillDownData:
    """Complete drill-down data for a canonical record"""
    canonical: dict  # Canonical record (all fields)
    observation: Optional[dict]  # Raw observation (may be None if missing)
    artifact: dict  # Artifact metadata (may have status="deleted")
    provenance: list[dict]  # Provenance entries (upload → parse → normalize)
    decisions: list[dict]  # Decision explanations (field-level)
    validation: dict  # Validation results (checks performed, warnings)
    logs: dict  # Log URLs (parse_log_url, normalization_log_url)

class ProvenanceTracer:
    """
    Trace data lineage from canonical → observation → artifact.

    Domain-agnostic - works on ANY canonical schema.
    """

    def __init__(
        self,
        canonical_store,
        observation_store,
        storage_engine,
        provenance_ledger,
        decision_explainer
    ):
        """Initialize with required stores"""

    def trace_lineage(
        self,
        canonical_id: str,
        user_id: str
    ) -> DrillDownData:
        """
        Retrieves complete lineage for a canonical record.

        Steps:
        1. Load canonical from CanonicalStore
        2. Verify user owns this canonical (access control)
        3. Load observation from ObservationStore (via upload_id, row_id)
        4. Load artifact metadata from StorageEngine (via upload_id)
        5. Load provenance entries from ProvenanceLedger (via upload_id)
        6. Explain normalization decisions (via DecisionExplainer)
        7. Assemble DrillDownData response

        Args:
            canonical_id: Canonical record ID (e.g., "can_bofa_20250426_001")
            user_id: Current user ID (for access control)

        Returns:
            DrillDownData with all layers populated

        Raises:
            CanonicalNotFoundError: If canonical_id doesn't exist
            UnauthorizedError: If user doesn't own this canonical
            ObservationNotFoundError: If observation missing (handled gracefully)
        """

    def get_artifact_download_url(
        self,
        upload_id: str,
        user_id: str,
        expires_in_seconds: int = 3600
    ) -> str:
        """
        Generates signed URL for artifact download.

        Args:
            upload_id: Upload ID
            user_id: Current user (for access control)
            expires_in_seconds: URL expiration (default: 1 hour)

        Returns:
            Signed URL for artifact download

        Raises:
            ArtifactNotFoundError: If artifact doesn't exist
            ArtifactDeletedError: If artifact was deleted (retention policy)
        """
```

---

## Multi-Domain Applicability

**Finance:**
```python
tracer = ProvenanceTracer(...)
data = tracer.trace_lineage("can_bofa_20250426_001", "usr_eugenio")
# Returns: Transaction → parser output → bank statement PDF
```

**Healthcare:**
```python
tracer = ProvenanceTracer(...)
data = tracer.trace_lineage("can_lab_20250415_123", "usr_patient_456")
# Returns: Lab result → device data → lab report PDF
```

**Legal:**
```python
tracer = ProvenanceTracer(...)
data = tracer.trace_lineage("can_contract_clause_789", "usr_lawyer_01")
# Returns: Contract clause → OCR output → contract PDF
```

**Research (RSRCH - Utilitario):**
```python
tracer = ProvenanceTracer(...)
data = tracer.trace_lineage("fact_sama_helion_001", "usr_rsrch_analyst_01")
# Returns: Canonical fact → Entity resolution → TechCrunch HTML page + Lex Fridman podcast transcript
```

**Manufacturing:**
```python
tracer = ProvenanceTracer(...)
data = tracer.trace_lineage("can_qc_measurement_567", "usr_inspector_12")
# Returns: QC measurement → sensor reading → inspection report
```

**Media:**
```python
tracer = ProvenanceTracer(...)
data = tracer.trace_lineage("can_transcript_snippet_999", "usr_editor_55")
# Returns: Transcript snippet → ASR output → video file
```

**Domain-agnostic nature:** Works identically across all domains. Only the field names and artifact types change.

---

## Responsibilities

**ProvenanceTracer IS responsible for:**
- ✅ Traversing lineage across multiple stores
- ✅ Enforcing access control (user owns canonical)
- ✅ Handling missing data gracefully (deleted artifacts, corrupted observations)
- ✅ Optimizing queries (JOIN strategies, not N+1)
- ✅ Assembling complete DrillDownData response

**ProvenanceTracer is NOT responsible for:**
- ❌ Explaining normalization decisions (delegates to DecisionExplainer)
- ❌ Rendering UI components (that's IL layer)
- ❌ Modifying canonical/observation data (read-only)
- ❌ Creating new provenance entries (that's ProvenanceLedger)
- ❌ Streaming artifact content (that's ArtifactRetriever)

---

## Implementation Notes

### Storage Pattern

**No persistent storage** - ProvenanceTracer is a coordinator, not a store.

It reads from:
- CanonicalStore (canonical records)
- ObservationStore (raw observations)
- StorageEngine (artifact metadata)
- ProvenanceLedger (audit trail)

### Performance Optimization

**Avoid N+1 queries:**
```python
# BAD (N+1 queries)
canonical = canonical_store.get(canonical_id)  # Query 1
observation = observation_store.get(canonical.upload_id, canonical.row_id)  # Query 2
artifact = storage_engine.get(canonical.upload_id)  # Query 3
provenance = provenance_ledger.get_entries(canonical.upload_id)  # Query 4

# GOOD (Single JOIN query)
SELECT
  c.*, o.*, a.*, p.*
FROM canonical_transactions c
LEFT JOIN observation_transactions o ON c.upload_id = o.upload_id AND c.row_id = o.row_id
LEFT JOIN upload_records a ON c.upload_id = a.upload_id
LEFT JOIN provenance_entries p ON c.upload_id = p.upload_id
WHERE c.canonical_id = 'can_123';
```

**Caching strategy:**
```python
# Cache drill-down data (canonical rarely changes)
@cache(ttl=300)  # 5 minutes
def trace_lineage(canonical_id, user_id):
    ...
```

### Access Control

```python
def trace_lineage(canonical_id, user_id):
    canonical = canonical_store.get(canonical_id)

    # Verify user owns this canonical
    if canonical.user_id != user_id:
        raise UnauthorizedError(f"User {user_id} cannot access {canonical_id}")

    # Continue with lineage traversal...
```

### Handling Missing Data

```python
def trace_lineage(canonical_id, user_id):
    canonical = canonical_store.get(canonical_id)

    # Observation may be missing (parser error)
    try:
        observation = observation_store.get(canonical.upload_id, canonical.row_id)
    except ObservationNotFoundError:
        observation = None  # Handle gracefully

    # Artifact may be deleted (retention policy)
    artifact = storage_engine.get_metadata(canonical.upload_id)
    if artifact.status == "deleted":
        artifact["download_url"] = None  # No download for deleted files

    return DrillDownData(
        canonical=canonical,
        observation=observation,  # May be None
        artifact=artifact,
        ...
    )
```

---

## Related Primitives

**Upstream dependencies:**
- **CanonicalStore** (1.3) - Source of canonical records
- **ObservationStore** (1.2) - Source of raw observations
- **StorageEngine** (1.1) - Source of artifact metadata
- **ProvenanceLedger** (1.1) - Source of audit trail
- **DecisionExplainer** (2.2) - Explains normalization decisions

**Downstream consumers:**
- **DrillDownPanel** (IL) - Renders DrillDownData in UI
- **ProvenanceTimeline** (IL) - Visualizes provenance chain
- **ArtifactRetriever** (OL) - Downloads artifact using signed URL

**Composition pattern:**
```python
# ProvenanceTracer composes multiple primitives
class ProvenanceTracer:
    def __init__(self, canonical_store, observation_store, storage_engine,
                 provenance_ledger, decision_explainer):
        self.canonical = canonical_store
        self.observation = observation_store
        self.storage = storage_engine
        self.provenance = provenance_ledger
        self.explainer = decision_explainer

    def trace_lineage(self, canonical_id, user_id):
        # Coordinate across all stores
        canonical = self.canonical.get(canonical_id)
        observation = self.observation.get(canonical.upload_id, canonical.row_id)
        artifact = self.storage.get_metadata(canonical.upload_id)
        provenance = self.provenance.get_entries(canonical.upload_id)
        decisions = self.explainer.explain_all(canonical_id)

        return DrillDownData(...)
```

---

## Example Usage

### Finance Domain

```python
from primitives.ol import ProvenanceTracer

# Initialize
tracer = ProvenanceTracer(
    canonical_store=canonical_store,
    observation_store=observation_store,
    storage_engine=storage_engine,
    provenance_ledger=provenance_ledger,
    decision_explainer=decision_explainer
)

# Trace lineage
data = tracer.trace_lineage(
    canonical_id="can_bofa_20250426_001",
    user_id="usr_eugenio"
)

# Access layers
print(data.canonical["merchant"])  # "Aeromexico"
print(data.observation["merchant_raw"])  # "AEROMEXICO MEX"
print(data.artifact["filename"])  # "apple_card_statement_2025-04.pdf"
print(len(data.provenance))  # 3 (uploaded, parsed, normalized)
print(data.decisions[0]["decision"])  # "Normalized 'AEROMEXICO MEX' → 'Aeromexico'"
```

### Healthcare Domain

```python
# Same tracer, different domain
data = tracer.trace_lineage(
    canonical_id="can_lab_result_456",
    user_id="usr_patient_789"
)

# Access layers
print(data.canonical["test_name"])  # "Blood Glucose"
print(data.observation["value_raw"])  # "180 mg/dL"
print(data.artifact["filename"])  # "lab_report_2025-04-15.pdf"
print(data.decisions[0]["decision"])  # "Flagged as abnormal (180 > 140 threshold)"
```

**Key insight:** Same code, different domain. ProvenanceTracer doesn't know it's tracing transactions vs lab results.
