# ADR-0001: Canonical ID Decision (upload_id vs file_id)

**Status**: ✅ Accepted
**Date**: 2025-10-22
**Scope**: Global (affects all verticals)

---

## Context

When a user uploads a file, we need a stable, public identifier to track it through the pipeline:

```
Upload → Parse → Normalize → View/Query
```

**Two options considered:**

1. **`file_id`** — Internal storage identifier (content-addressable hash or DB primary key)
2. **`upload_id`** — Semantic identifier representing "an upload event"

**The problem:**
- Same file uploaded twice → same `file_id` (dedupe by hash)
- But these are **two separate upload events** with potentially different:
  - Timestamps
  - Uploaders
  - Processing results (parser version might change)
  - Provenance chains

---

## Decision

**We use `upload_id` as the canonical public identifier.**

- Format: `UL_{random_alphanumeric}` (e.g., `UL_abc123`)
- **Never expose** `file_id` or internal storage paths in public APIs
- `file_id` remains internal (used by `StorageEngine` only)

---

## Rationale

### 1. Semantic Correctness
- An **upload** is an event in time (who, when, why)
- A **file** is just content (what)
- Two uploads of the same file = two events = two `upload_id`s
- Provenance must track the **event**, not just the content

### 2. Idempotency at the Right Level
- Dedupe by `file_hash` prevents **storage duplication**
- But we still create an `UploadRecord` with new `upload_id`
- This allows tracking: "User X uploaded same file on 2025-10-15 and 2025-10-22"

### 3. Versioning & Reprocessing
- If parser logic changes, we may want to **reprocess** the same file
- Each reprocessing gets a new `upload_id` → separate provenance chain
- Allows A/B testing parsers on same content

### 4. Multi-user Context
- Same file uploaded by User A and User B → two separate audit trails
- `upload_id` ties to `uploader_id` in `ProvenanceLedger`

### 5. API Stability
- Public API uses `upload_id` everywhere:
  - `POST /uploads → { upload_id }`
  - `GET /uploads/:upload_id`
  - `GET /transactions?upload_id=UL_abc123`
- Internal refactoring (storage backend, deduplication strategy) doesn't affect API

---

## Consequences

### ✅ Positive

- **Clear semantics**: "upload" is first-class citizen
- **Provenance is accurate**: Every event tracked independently
- **API is stable**: No internal IDs leak
- **Flexible deduplication**: Can change storage strategy without API changes

### ⚠️ Trade-offs

- **More records**: Same file uploaded twice = 2 `UploadRecord`s (but 1 storage entry)
- **Slightly more complex**: Need to map `upload_id → file_hash → storage_ref` internally

### 🔴 Risks (Mitigated)

- **Risk**: User confusion ("I uploaded this file before, why is it uploading again?")
  - **Mitigation**: `409 Conflict` response includes existing `upload_id` and log reference
  - UI can show: "This file already exists. [View existing upload]"

---

## Alternatives Considered

### Alternative A: Expose `file_id` (Rejected)

**Pros:**
- Simpler deduplication: Same file = same ID everywhere
- Less storage (one `UploadRecord` per unique file)

**Cons:**
- **Provenance breaks**: Can't track multiple upload events for same file
- **API couples to storage**: Changing storage backend requires API migration
- **Versioning impossible**: Can't reprocess same file with new parser
- **Multi-user conflict**: Who "owns" the `file_id`?

### Alternative B: Use both `upload_id` + `file_id` in API (Rejected)

**Pros:**
- Flexibility: Client can choose which to use

**Cons:**
- **Confusing**: Two IDs for same concept
- **Inconsistent queries**: `GET /uploads/UL_abc123` vs `GET /files/FILE_xyz789`?
- **Doubles API surface**: More endpoints to maintain

---

## Implementation Notes

### Internal Mapping (Not Exposed)

```
UploadRecord
  ├─ upload_id: "UL_abc123"  ← PUBLIC
  ├─ file_hash: "sha256:..."
  └─ storage_ref: "internal" ← PRIVATE

FileArtifact (internal)
  ├─ file_hash: "sha256:..."  ← Dedupe key
  ├─ storage_ref: "/storage/abc/123"
  └─ size, mime_type, etc.
```

### Dedupe Flow

1. User uploads file
2. Calculate `file_hash` in stream
3. Check if `file_hash` exists in `FileArtifact` table
4. If exists → `409 Conflict` with existing `upload_id`
5. If new → persist content, create `FileArtifact`, create `UploadRecord` with new `upload_id`

---

## Related Decisions

- **ADR-0002**: State machine uses `UploadRecord.status` (keyed by `upload_id`)
- **ADR-0003**: Provenance ledger uses `upload_id` as primary key
- **Future**: Multi-tenancy will add `tenant_id` prefix (e.g., `TENANT1_UL_abc123`)

---

**References:**
- Vertical 1.1 Upload Flow: [docs/verticals/1.1-upload-flow.md](../verticals/1.1-upload-flow.md)
- Schema: [docs/schemas/upload-record.schema.json](../schemas/upload-record.schema.json)
