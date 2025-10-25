# ADR-0007: Provenance Traceability Architecture

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Data lineage and audit trail

---

## Context

Users need to trace data lineage from canonical transaction back to original source:

```
Canonical Transaction → Observation → Upload Event → Original PDF Artifact
```

**Key requirements:**

1. **Auditability**: Full trace for regulatory compliance (SOX, GDPR)
2. **Debugging**: Developers can verify normalization decisions
3. **Trust**: Users can inspect original source for disputed transactions
4. **Security**: Artifact access must be controlled and time-limited

**The problem:**
- Canonical transactions are normalized/deduplicated
- Users see "Amount: $50.00, Merchant: Amazon" but need to verify:
  - Which PDF page showed this?
  - Was amount rounded/normalized?
  - What exact text did parser extract?

---

## Decision

**We implement full provenance trace with signed URLs for artifact access.**

Architecture:
1. **ProvenanceTracer** traverses: `canonical_id → observation_id → upload_id → artifact_ref`
2. **ArtifactRetriever** generates signed URLs (S3 presigned, 1h expiry)
3. **DrillDownPanel** displays complete lineage with clickable artifact links

---

## Rationale

### 1. Regulatory Compliance
- SOX requires audit trail for financial data
- GDPR requires data lineage for user privacy
- Full trace proves data transformations are correct

### 2. Debugging Capabilities
- Developers can inspect original PDF when parser fails
- Compare normalized vs raw observation data
- Identify systematic parser errors

### 3. User Trust
- Users verify "Amazon" normalization came from "AMZN MKTPLACE"
- Inspect original receipt for disputed charges
- Understand why duplicate transactions were merged

### 4. Security
- Signed URLs prevent unauthorized artifact access
- Time-limited (1h) prevents long-term URL sharing
- Audit log tracks who accessed which artifacts

---

## Consequences

### ✅ Positive

- **Full audit trail**: Complete lineage for compliance
- **Enhanced debugging**: Developers trace issues to source
- **User trust**: Transparent normalization decisions
- **Secure access**: Signed URLs with expiration

### ⚠️ Trade-offs

- **More storage**: Provenance entries ~100 bytes per transaction
- **More latency**: 3 queries for full trace (canonical → observation → upload → artifact)
- **Index overhead**: Requires indexes on observation_id, upload_id
- **Complexity**: ProvenanceTracer logic adds code complexity

### 🔴 Risks (Mitigated)

- **Risk**: Artifact storage costs grow unbounded
  - **Mitigation**: S3 lifecycle policy archives artifacts >1 year to Glacier
  - Cost: $0.004/GB/month (S3) → $0.001/GB/month (Glacier)

- **Risk**: Signed URL expiration breaks saved links
  - **Mitigation**: UI refreshes signed URL on every click
  - Backend regenerates URL with new 1h expiration

---

## Alternatives Considered

### Alternative A: Observation-Only Trace (Rejected)

**Pros:**
- Simpler: Only track `canonical_id → observation_id`
- No artifact retrieval needed

**Cons:**
- **No original source**: Cannot verify parser correctness
- **Debugging limited**: Cannot inspect PDF when parser fails
- **Trust issues**: Users cannot verify normalization decisions

### Alternative B: Embed Artifacts in Database (Rejected)

**Pros:**
- No signed URLs needed
- Faster retrieval (no S3 round-trip)

**Cons:**
- **Database bloat**: PDFs are large (1-10MB each)
- **Duplicate storage**: Multiple observations → multiple PDF copies
- **Expensive**: Database storage ~10x more expensive than S3

### Alternative C: No Provenance Trace (Rejected)

**Pros:**
- Simplest implementation
- Zero storage overhead
- Fastest queries

**Cons:**
- **No compliance**: Fails SOX/GDPR requirements
- **No debugging**: Cannot trace parser errors
- **No trust**: Users cannot verify data transformations

---

## Implementation Notes

### Provenance Chain Structure

```
CanonicalTransaction
  ├─ canonical_id: "TX_abc123"
  └─ observation_ids: ["OBS_001", "OBS_002"]  ← Multiple observations merged

Observation
  ├─ observation_id: "OBS_001"
  ├─ upload_id: "UL_xyz789"
  └─ raw_data: { amount: "$50.00", merchant: "AMZN MKTPLACE" }

UploadRecord
  ├─ upload_id: "UL_xyz789"
  └─ artifact_ref: "s3://bucket/uploads/xyz789.pdf"

Artifact (S3)
  └─ Signed URL: "https://s3.../xyz789.pdf?signature=..."
```

### Query Flow

```sql
-- 1. Get observation IDs from canonical transaction
SELECT observation_ids FROM canonical_transactions
WHERE canonical_id = 'TX_abc123';

-- 2. Get upload ID from observation
SELECT upload_id FROM observations
WHERE observation_id = 'OBS_001';

-- 3. Get artifact reference from upload
SELECT artifact_ref FROM uploads
WHERE upload_id = 'UL_xyz789';

-- 4. Generate signed URL (application logic)
signed_url = s3_client.generate_presigned_url(artifact_ref, expires_in=3600)
```

### Required Indexes

```sql
CREATE INDEX idx_canonical_observations
ON canonical_transactions USING GIN (observation_ids);

CREATE INDEX idx_observation_upload
ON observations (upload_id);

CREATE INDEX idx_upload_artifact
ON uploads (artifact_ref);
```

---

## Retention Strategy

**Problem**: Without retention policies, provenance ledger grows unbounded:
```
1M transactions × 5 fields × 3 edits = 15M provenance records
At 2KB each = 30 GB/year (grows forever)
```

**Solution**: Implement tiered retention with archival workflows.

### Retention Classes

| Class | Duration | Use Case | Storage Cost |
|-------|----------|----------|--------------|
| `regulatory_7yr` | 7 years hot | SOX, HIPAA compliance | $0.023/GB/month (PostgreSQL) |
| `operational_2yr` | 2 years hot + archive | Business analytics | $0.023/GB → $0.004/GB (S3) |
| `temporary_90d` | 90 days then delete | Non-critical changes | $0.023/GB → $0 (deleted) |

### Archival Workflow

```
active (PostgreSQL)
  ↓ (after retention_class period)
archived_s3 (S3 cold storage, 74% compression)
  ↓ (after legal retention met)
deleted (removed from both PostgreSQL + S3)
```

### Implementation

**Schema fields** (added to provenance-record.schema.json):
```json
{
  "retention_class": {
    "enum": ["regulatory_7yr", "operational_2yr", "temporary_90d"]
  },
  "archival_status": {
    "enum": ["active", "archived_s3", "deleted"]
  },
  "previous_signature": {
    "description": "SHA-256 of previous record for hash chain validation"
  }
}
```

**Archival job** (runs nightly):
```sql
-- Move records past retention to S3
UPDATE provenance_ledger
SET archival_status = 'archived_s3'
WHERE archival_status = 'active'
  AND retention_class = 'operational_2yr'
  AND transaction_time < NOW() - INTERVAL '2 years';

-- Export to S3 (compressed JSONL)
COPY (SELECT * FROM provenance_ledger WHERE archival_status = 'archived_s3')
TO 's3://bucket/provenance/2023-10.jsonl.gz' WITH (FORMAT json, COMPRESSION gzip);

-- Delete after legal retention met
DELETE FROM provenance_ledger
WHERE retention_class = 'temporary_90d'
  AND transaction_time < NOW() - INTERVAL '90 days';
```

### Hash Chain Validation

**Purpose**: Detect tampering or missing records in provenance chain.

**Algorithm**:
```python
def validate_provenance_chain(entity_id: str, field_name: str):
    """
    Verify cryptographic integrity of provenance chain.
    Returns True if chain valid, False if tampered.
    """
    records = db.query(ProvenanceLedger).filter_by(
        entity_id=entity_id,
        field_name=field_name
    ).order_by(ProvenanceLedger.transaction_time.asc())

    for i, record in enumerate(records):
        if i == 0:
            # First record should have null previous_signature
            if record.previous_signature is not None:
                return False
        else:
            # Verify hash chain
            expected_hash = records[i-1].signature
            if record.previous_signature != expected_hash:
                return False  # Chain broken - tampering detected

        # Verify current record signature
        computed_hash = hashlib.sha256(
            f"{record.provenance_id}:{record.entity_id}:"
            f"{record.transaction_time}:{record.value}".encode()
        ).hexdigest()

        if record.signature != computed_hash:
            return False  # Record tampered

    return True  # Chain valid
```

### Storage Growth Mitigation

**Before retention**:
- Year 1: 30 GB (all in PostgreSQL)
- Year 2: 60 GB
- Year 5: 150 GB
- **Problem**: Database storage expensive (~$0.023/GB/month)

**After retention**:
- Year 1: 30 GB hot (PostgreSQL)
- Year 2: 30 GB hot + 7.8 GB cold (S3, 74% compression)
- Year 5: 30 GB hot + 31.2 GB cold
- **Savings**: 80% reduction in hot storage costs

---

## Related Decisions

- **ADR-0001**: Uses `upload_id` as stable identifier in provenance chain
- **ADR-0002**: State machine tracks upload lifecycle for provenance
- **Future**: May add blockchain-anchored hashes for immutable audit trail

---

**References:**
- Vertical 2.2: [docs/verticals/2.2-ol-exploration.md](../verticals/2.2-ol-exploration.md)
- ProvenanceTracer: [docs/primitives/ol/ProvenanceTracer.md](../primitives/ol/ProvenanceTracer.md)
- ArtifactRetriever: [docs/primitives/ol/ArtifactRetriever.md](../primitives/ol/ArtifactRetriever.md)
