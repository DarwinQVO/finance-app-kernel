# ADR-0010: Canonical ID Format (UUIDv5 Deterministic)

**Status**: ✅ Accepted
**Date**: 2025-10-25
**Scope**: Global (affects Vertical 1.3 Normalization and all downstream verticals)
**Supersedes**: Previous mixed-format approach (`can_{source}_{date}_{seq}` and `CT_{hash}`)

---

## Context

CanonicalTransaction records need a **globally unique, deterministic, and stable identifier** (`canonical_id`) for:

1. **Deduplication**: Prevent re-normalization from creating duplicate canonicals
2. **Idempotency**: Same observation + normalizer version → same canonical_id
3. **Cross-system compatibility**: Enable external systems to reference canonicals
4. **Auditing**: Track canonical creation and updates over time

**Previous approach (REJECTED):**
```json
"canonical_id": "can_bofa_20250426_001"  // Human-readable but inconsistent
"canonical_id": "CT_abc123xyz"           // Short but non-deterministic
```

**Problems with mixed formats:**
- ❌ Inconsistent: Two formats for same purpose
- ❌ Non-deterministic: `CT_{hash}` depends on hashing implementation
- ❌ Collision risk: `can_{source}_{date}_{seq}` requires sequence tracking
- ❌ Not RFC-compliant: Custom format, no industry standard

---

## Decision

**Use UUIDv5 (RFC 4122) for all canonical_id values.**

### Format Specification

```
Pattern: ^[0-9a-f]{8}-[0-9a-f]{4}-5[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$
Type: UUID version 5 (SHA-1 hash-based)
```

### Generation Algorithm

```python
import uuid

# Namespace UUID (fixed, project-specific)
NAMESPACE_CANONICAL = uuid.UUID('6ba7b810-9dad-11d1-80b4-00c04fd430c8')

def generate_canonical_id(upload_id: str, row_id: int, normalizer_version: str) -> str:
    """
    Generate deterministic UUIDv5 canonical ID.

    Args:
        upload_id: Source upload ID (e.g., "UL_abc123")
        row_id: Row index in source file (0-indexed)
        normalizer_version: Semantic version (e.g., "1.0.0")

    Returns:
        UUIDv5 string (e.g., "550e8400-e29b-51d4-a716-446655440000")
    """
    # Construct deterministic input string
    input_data = f"{upload_id}:{row_id}:{normalizer_version}"

    # Generate UUIDv5 using namespace + input
    canonical_id = uuid.uuid5(NAMESPACE_CANONICAL, input_data)

    return str(canonical_id)
```

### Example

```python
# Input
upload_id = "UL_abc123"
row_id = 0
normalizer_version = "1.0.0"

# Output (deterministic)
canonical_id = generate_canonical_id(upload_id, row_id, normalizer_version)
# → "550e8400-e29b-51d4-a716-446655440000"

# Re-normalization with same inputs → same ID (idempotent)
canonical_id_2 = generate_canonical_id("UL_abc123", 0, "1.0.0")
# → "550e8400-e29b-51d4-a716-446655440000" (identical)

# Different normalizer version → different ID (allows versioning)
canonical_id_3 = generate_canonical_id("UL_abc123", 0, "2.0.0")
# → "7c9e6679-7425-5a94-85d2-e56b7c3e5e21" (different)
```

---

## Rationale

### ✅ Advantages

1. **Deterministic**: Same inputs → same UUID (SHA-1 hash-based)
   - Prevents duplicate canonicals during re-normalization
   - Enables idempotent `UPSERT` operations: `INSERT ... ON CONFLICT (canonical_id) DO UPDATE`

2. **RFC 4122 Compliant**: Industry-standard UUID format
   - Compatible with all databases (PostgreSQL `uuid` type, MySQL `BINARY(16)`, etc.)
   - Native support in ORMs (SQLAlchemy, Django, etc.)
   - Sortable by creation order (if using UUIDv7 in future)

3. **Globally Unique**: Collision probability ≈ 0
   - 128-bit space (2^122 unique UUIDs for version 5)
   - No need for centralized ID generation service

4. **Versioning Support**: `normalizer_version` in input
   - Re-normalization with new rules → new canonical_id
   - Allows A/B testing of normalization logic
   - Preserves old canonicals for comparison

5. **Cross-System Compatibility**:
   - External APIs can reference canonical_id without custom parsing
   - UUID format understood by all systems (GraphQL, REST, gRPC)

### ⚠️ Trade-offs

1. **Not human-readable**: `550e8400-...` vs `can_bofa_20250426_001`
   - **Mitigation**: Use `observation_id` (human-readable: `UL_abc123:0`) for debugging
   - **Mitigation**: Add `metadata.display_label` field for UI display

2. **Longer storage**: 36 bytes (string) or 16 bytes (binary) vs 20 bytes (custom format)
   - **Impact**: Minimal (< 0.01% of total DB size)
   - **Mitigation**: Use PostgreSQL `uuid` type (16 bytes) instead of `TEXT`

3. **SHA-1 collision risk (theoretical)**: UUIDv5 uses SHA-1 (deprecated for crypto)
   - **Reality**: Collision probability ≈ 2^-63 (1 in 9 quintillion)
   - **Context**: Not used for security, only for unique ID generation
   - **Future**: Can migrate to UUIDv8 (SHA-256) if needed

### 🔴 Rejected Alternatives

#### Alternative A: UUIDv4 (Random)
```python
canonical_id = str(uuid.uuid4())  # Non-deterministic
```
**Pros:** Simple, no collision risk
**Cons:** ❌ Non-deterministic (re-normalization creates duplicates), requires database query to check existence

#### Alternative B: Auto-increment sequence
```sql
canonical_id = "CT_000001"  -- Database sequence
```
**Pros:** Human-readable, compact
**Cons:** ❌ Requires centralized sequence, race conditions in distributed systems, not globally unique

#### Alternative C: Content hash (SHA-256)
```python
canonical_id = hashlib.sha256(json.dumps(canonical_data)).hexdigest()
```
**Pros:** Deterministic, collision-resistant
**Cons:** ❌ Non-standard format, too long (64 chars), changes if any field changes (not stable across re-normalization)

---

## Implementation Notes

### Schema Update

**Before (Mixed formats):**
```json
{
  "canonical_id": {
    "type": "string",
    "pattern": "^(can_|CT_)[a-zA-Z0-9_]+$"
  }
}
```

**After (UUIDv5 only):**
```json
{
  "canonical_id": {
    "type": "string",
    "format": "uuid",
    "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-5[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$",
    "description": "UUIDv5 canonical transaction ID (RFC 4122)"
  }
}
```

### Database Schema

**PostgreSQL (recommended):**
```sql
CREATE TABLE canonical_transactions (
  canonical_id UUID PRIMARY KEY,  -- 16 bytes, indexed
  upload_id TEXT NOT NULL,
  observation_id TEXT NOT NULL,
  -- ... other fields
  UNIQUE (upload_id, row_id, normalizer_version)  -- Enforce determinism
);
```

**SQLite (for testing):**
```sql
CREATE TABLE canonical_transactions (
  canonical_id TEXT PRIMARY KEY CHECK(canonical_id GLOB '[0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f]-5[0-9a-f][0-9a-f][0-9a-f]-[89ab][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f]'),
  -- ... other fields
);
```

### Migration Strategy (if needed)

For systems using old `can_*` or `CT_*` formats:

```python
def migrate_canonical_ids():
    """One-time migration: old format → UUIDv5"""
    for old_canonical in db.query(CanonicalTransaction).filter(
        CanonicalTransaction.canonical_id.notlike('____-____-5___-____-____________')
    ):
        # Extract original inputs
        upload_id = old_canonical.upload_id
        row_id = old_canonical.row_id  # Must be stored
        normalizer_version = old_canonical.normalizer_version

        # Generate new UUIDv5
        new_canonical_id = generate_canonical_id(upload_id, row_id, normalizer_version)

        # Update canonical_id
        old_canonical.canonical_id = new_canonical_id
        db.commit()
```

---

## Consequences

### ✅ Positive

1. **Idempotent normalization**: Re-running normalization won't create duplicates
2. **Database-friendly**: Native UUID support in PostgreSQL, MySQL, etc.
3. **API-friendly**: Standard format, no custom parsing needed
4. **Versioning support**: Different normalizer versions → different canonical_ids
5. **Global uniqueness**: No centralized ID service needed

### ⚠️ Neutral

1. **Less human-readable**: Developers must use `observation_id` for debugging
2. **Slightly larger storage**: 16 bytes (binary UUID) vs ~12 bytes (custom format)

### 🔴 Risks (Mitigated)

1. **Risk**: SHA-1 collisions (UUIDv5 uses SHA-1)
   - **Probability**: ≈ 2^-63 (1 in 9 quintillion) for typical workload
   - **Mitigation**: Not used for security, only unique ID generation
   - **Future**: Can migrate to UUIDv8 (SHA-256) if needed

2. **Risk**: Re-normalization with different `normalizer_version` creates new canonical_id
   - **Expected behavior**: This is intentional (allows versioning)
   - **Mitigation**: Archive old canonicals with `superseded_by` field

---

## Related Decisions

- **ADR-0001**: `upload_id` format (this feeds into canonical_id generation)
- **ADR-0002**: State machine (canonical creation is final state)
- **Vertical 1.3**: Normalization pipeline (uses this canonical_id format)

---

## References

- **RFC 4122**: [UUID Specification](https://datatracker.ietf.org/doc/html/rfc4122)
- **Python uuid module**: [uuid.uuid5() docs](https://docs.python.org/3/library/uuid.html#uuid.uuid5)
- **PostgreSQL UUID type**: [UUID datatype](https://www.postgresql.org/docs/current/datatype-uuid.html)

---

**Example canonical_id values (for testing):**
```
550e8400-e29b-51d4-a716-446655440000  # UL_abc123:0:1.0.0
6ba7b810-9dad-51d1-80b4-00c04fd430c8  # UL_def456:15:1.0.0
7c9e6679-7425-5a94-85d2-e56b7c3e5e14  # UL_ghi789:42:1.0.0
```
