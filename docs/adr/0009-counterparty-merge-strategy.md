# ADR-0009: Counterparty Merge Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Duplicate counterparty deduplication

---

## Context

Users create duplicate counterparties over time:

```
User creates: "Apple Inc"     (from invoice)
            → "APPLE COM BILL" (from bank statement)
            → "Apple"          (manual entry)
```

**The problem:**
- Same entity appears as 3 separate counterparties
- Splits transaction history across duplicates
- Breaks analytics (total spend with Apple fragmented)
- Confuses autocomplete (3 Apple suggestions)

**Key requirements:**

1. **Data integrity**: Consolidate transactions under single canonical counterparty
2. **Reversibility**: Users can undo merge if mistake
3. **Alias preservation**: Search still works for old names
4. **Transaction history**: No data loss during merge

---

## Decision

**We use cascade update strategy: migrate all transactions to canonical counterparty, preserve aliases.**

Merge process:
1. User selects duplicates in **MergeCounterpartiesDialog**
2. Choose canonical counterparty (primary to keep)
3. **Migrate transactions**: `UPDATE transactions SET counterparty_id = canonical_id`
4. **Add aliases**: Copy names from merged counterparties to canonical
5. **Soft delete**: Archive merged counterparties with `merge_target_id`

---

## Rationale

### 1. Data Integrity
- Single source of truth for counterparty
- All transactions consolidated under canonical entity
- Analytics accurate (total Apple spend = all 3 merged)

### 2. Reversible Operation
- Soft delete allows undo
- Merged counterparties archived with `merge_target_id = canonical_id`
- Can reverse: migrate transactions back, restore counterparty

### 3. Alias Preservation
- Old names added as aliases to canonical counterparty
- Search for "APPLE COM BILL" still finds transactions
- Autocomplete shows canonical name + aliases

### 4. Transaction History Preserved
- Zero data loss: All transactions migrated
- Provenance maintained: Transaction metadata unchanged
- Audit trail: Merge operation logged

---

## Consequences

### ✅ Positive

- **Clean consolidated data**: Single counterparty for each entity
- **Reversible**: Soft delete allows undo
- **Complete history**: All transactions preserved
- **Search works**: Aliases maintain discoverability

### ⚠️ Trade-offs

- **Requires confirmation**: Irreversible cascade needs user confirmation
- **Complex rollback**: Reverting merge requires transaction migration + restore
- **Alias bloat**: Canonical counterparty accumulates many aliases over time
- **Performance**: Large merges (10K+ transactions) may take seconds

### 🔴 Risks (Mitigated)

- **Risk**: User merges wrong counterparties by mistake
  - **Mitigation**: Confirmation dialog shows transaction counts + preview
  - UI: "Merge Apple Inc (50 transactions) ← APPLE COM BILL (30 transactions)?"

- **Risk**: Cannot auto-detect all duplicates (fuzzy matching limitations)
  - **Mitigation**: Provide manual merge + "Suggest Duplicates" tool
  - Future: ML-based duplicate detection

---

## Alternatives Considered

### Alternative A: Hard Delete Merged Counterparties (Rejected)

**Pros:**
- Clean database: No archived counterparties
- Simple implementation: Just DELETE

**Cons:**
- **Irreversible**: Cannot undo merge
- **Breaks audit trail**: Provenance lost
- **Data loss risk**: If merge was mistake, cannot recover

### Alternative B: Soft Delete Without Transaction Migration (Rejected)

**Pros:**
- Preserves original counterparty IDs
- Easier to reverse

**Cons:**
- **Orphaned transactions**: Transactions point to deleted counterparty
- **Broken analytics**: Queries must follow merge chain
- **Confusing**: User sees deleted counterparty in transaction history

### Alternative C: Anonymize Merged Counterparties (Rejected)

**Pros:**
- No hard delete
- Transactions remain linked

**Cons:**
- **Loses semantic meaning**: "Apple Inc" becomes "Merged_123"
- **Breaks search**: Cannot find original name
- **Confusing audit trail**: Why does transaction show "Merged_123"?

### Alternative D: Prevent Merge If Transactions Exist (Rejected)

**Pros:**
- Zero risk of data migration issues
- Simple implementation

**Cons:**
- **Too restrictive**: Core use case is merging counterparties WITH transactions
- **Defeats purpose**: Users create duplicates specifically to merge transactions

---

## Implementation Notes

### Merge Flow

```
1. User selects duplicates:
   - Canonical: "Apple Inc" (counterparty_id: "CP_001")
   - Duplicates: ["APPLE COM BILL" (CP_002), "Apple" (CP_003)]

2. Preview shows:
   - Apple Inc: 50 transactions
   - APPLE COM BILL: 30 transactions
   - Apple: 10 transactions
   - TOTAL after merge: 90 transactions

3. User confirms → Execute merge:

   a. Migrate transactions:
      UPDATE transactions
      SET counterparty_id = 'CP_001'
      WHERE counterparty_id IN ('CP_002', 'CP_003');

   b. Add aliases to canonical:
      UPDATE counterparties
      SET aliases = array_append(aliases, 'APPLE COM BILL', 'Apple')
      WHERE counterparty_id = 'CP_001';

   c. Soft delete merged counterparties:
      UPDATE counterparties
      SET deleted_at = NOW(),
          merge_target_id = 'CP_001'
      WHERE counterparty_id IN ('CP_002', 'CP_003');

4. Success: User sees 90 transactions under "Apple Inc"
```

### Database Schema

```sql
CREATE TABLE counterparties (
  counterparty_id VARCHAR(50) PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  aliases TEXT[] DEFAULT '{}',
  deleted_at TIMESTAMP,
  merge_target_id VARCHAR(50) REFERENCES counterparties(counterparty_id),
  created_at TIMESTAMP NOT NULL,
  modified_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_counterparty_aliases
ON counterparties USING GIN (aliases);

CREATE INDEX idx_counterparty_merge_target
ON counterparties (merge_target_id)
WHERE merge_target_id IS NOT NULL;
```

### Rollback Flow (Undo Merge)

```sql
-- 1. Restore merged counterparties
UPDATE counterparties
SET deleted_at = NULL, merge_target_id = NULL
WHERE merge_target_id = 'CP_001';

-- 2. Reverse transaction migrations (requires merge history)
-- (This is complex - need to store original counterparty_id)
UPDATE transactions
SET counterparty_id = original_counterparty_id
FROM merge_history
WHERE transactions.transaction_id = merge_history.transaction_id;

-- 3. Remove aliases from canonical
UPDATE counterparties
SET aliases = '{}'
WHERE counterparty_id = 'CP_001';
```

**Note**: Rollback requires `merge_history` table to track original `counterparty_id` per transaction.

---

## Related Decisions

- **ADR-0005**: Normalization engine creates duplicate counterparties (triggers need for merge)
- **Future**: ML-based duplicate detection to suggest merges
- **Future**: Bulk merge operations for batch deduplication

---

**References:**
- Vertical 3.2: [docs/verticals/3.2-counterparty-registry.md](../verticals/3.2-counterparty-registry.md)
- AliasMerger: [docs/primitives/ol/AliasMerger.md](../primitives/ol/AliasMerger.md)
- Schema: [docs/schemas/counterparty.schema.json](../schemas/counterparty.schema.json)
