# ADR-0004: Raw-First Extraction (Store AS-IS, Validate Later)

**Status**: Accepted
**Date**: 2025-10-23
**Context**: Vertical 1.2 (Extraction) design
**Deciders**: System architects
**Related**: Vertical 1.2, Vertical 1.3 (Normalization)

---

## Context

When parsing documents (PDFs, CSVs, etc.), we must decide:
- **Do we validate/normalize during extraction?**
- **Or extract AS-IS and validate later?**

Example scenarios:
- Date appears as `"01/02/2024"` → Is it Jan 2 or Feb 1?
- Amount appears as `"(50.00)"` → Is it -50.00?
- Description has trailing spaces → Trim or keep?

---

## Decision

**We extract and store observations AS-IS (raw), without validation or transformation.**

Normalization happens in a separate stage (Vertical 1.3).

---

## Rationale

### 1. Separation of Concerns

**Extract (1.2):** "What did the parser see?"
**Normalize (1.3):** "What does it mean?"

**Benefits:**
- Parser failures isolated from normalization failures
- Re-normalize without re-parsing (if rules change)
- Audit trail: always know original raw data

### 2. Preserve Original Data

**Once normalized, raw data may be lost.**

Example:
- Raw: `"01/02/2024"`
- Normalized: `"2024-01-02"` (assumes MM/DD/YYYY)
- **If assumption wrong, can't recover original**

Storing AS-IS allows corrections without re-upload.

### 3. Idempotent Re-processing

**If normalization rules change, re-run normalizer on existing observations.**

Example:
- v1: Assume all dates are MM/DD/YYYY
- v2: Detect locale from `source_type`, apply DD/MM/YYYY for non-US sources

With raw data preserved:
```python
# Re-normalize all observations with new rules
for obs in observation_store.get_all():
    canonical = normalizer_v2.normalize(obs)
    canonical_store.upsert(canonical)
```

Without raw data: **must re-upload all files** (expensive, may not be possible).

### 4. Parser Simplicity

**Parsers only extract text, don't interpret.**

```python
# Simple parser (AS-IS)
def parse(pdf):
    return [
        {"date": "01/02/2024", "amount": "(50.00)"}  # Raw strings
    ]

# Complex parser (validate + normalize)
def parse(pdf):
    raw_date = "01/02/2024"
    # Which locale? MM/DD or DD/MM?
    # Parser doesn't know → guess wrong → data corruption
    date = parse_date(raw_date, locale="en_US")  # ASSUMPTION
    return [{"date": date.isoformat(), "amount": -50.00}]
```

Complexity moved to normalizer where domain context is available.

---

## Consequences

### Positive

✅ **Lossless extraction:** Original data always available
✅ **Re-normalization:** Can fix interpretation errors without re-upload
✅ **Parser simplicity:** Extract text, don't interpret
✅ **Audit trail:** See exactly what parser saw
✅ **Debugging:** Compare raw vs normalized to find normalization bugs

### Negative

⚠️ **Storage overhead:** Store both raw (ObservationStore) and normalized (CanonicalStore)
⚠️ **Two-stage pipeline:** Extraction + Normalization (more complexity)
⚠️ **Delayed validation:** Bad data not caught until normalization

### Mitigations

**Storage overhead:**
- Acceptable trade-off for data integrity
- Raw observations can be archived after N days (configurable)

**Pipeline complexity:**
- Well-defined stages with clear contracts
- State machine makes flow explicit

**Delayed validation:**
- Parser can emit warnings (not errors) for suspicious data
- Normalization runs immediately after extraction (minimal delay)

---

## Alternatives Considered

### Alternative 1: Validate During Extraction

**Approach:** Parser validates and normalizes immediately.

```python
def parse(pdf):
    raw_date = "01/02/2024"
    date = validate_and_normalize_date(raw_date)  # May fail
    return [{"date": date.isoformat(), ...}]
```

**Rejected because:**
- Parser doesn't have domain context (locale, business rules)
- Validation failure stops extraction (lose partial data)
- Can't re-normalize without re-parsing

---

### Alternative 2: Store Both Raw and Normalized in Same Record

**Approach:** Single table with `raw_date` and `normalized_date`.

```sql
CREATE TABLE transactions (
    raw_date TEXT,
    normalized_date DATE,
    ...
);
```

**Rejected because:**
- Tight coupling (extraction + normalization in single step)
- Can't re-normalize independently
- Schema changes affect both raw and canonical

**Current approach:** Separate stores with clear boundaries.

---

## Implementation Notes

### ObservationStore Schema

```typescript
interface ObservationTransaction {
  date: string         // AS-IS from parser
  amount: string       // AS-IS from parser
  description: string  // AS-IS from parser
  ...
}
```

**All fields are strings** (or raw types from parser), no validation.

### CanonicalStore Schema (Vertical 1.3)

```typescript
interface CanonicalTransaction {
  date: ISO8601Date    // Normalized
  amount: Decimal      // Normalized (signed)
  description: string  // Trimmed, cleaned
  ...
}
```

**All fields validated and normalized.**

---

## Related Decisions

- **ADR-0003:** Runner/Coordinator split (applies to both extraction and normalization)
- **Vertical 1.3:** Normalization rules and canonical schema

---

## Review & Updates

- **2025-10-23:** Initial decision (Vertical 1.2 specification)
- Next review: After implementing 10+ parsers, assess if pattern holds

---

**Decision**: Extract AS-IS (raw-first), normalize in separate stage.
**Status**: Accepted and implemented in Vertical 1.2 specification.
