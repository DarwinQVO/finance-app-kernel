# ADR-0006: Cursor-Based Pagination Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Global (affects all list views)

---

## Context

When displaying transaction lists with 10K+ items, we need a pagination strategy that handles:

```
User views page 1 → New transactions arrive → User navigates to page 2
```

**Key requirements:**

1. **Stability**: Pagination cursors remain valid under concurrent inserts
2. **Performance**: Efficient at any dataset size (10K+ transactions)
3. **Consistency**: No duplicate or missing items when paginating
4. **Scalability**: Works efficiently regardless of page depth

**The problem with offset pagination:**
- Offset 100 means "skip first 100 rows"
- If 5 new transactions inserted → offset 100 now points to different rows
- Result: Duplicate items across pages OR missing items
- Performance degrades at high offsets (must scan all skipped rows)

---

## Decision

**We use cursor-based (keyset) pagination for all transaction list views.**

- Cursor format: `base64(timestamp_id)` where `timestamp_id` is `{timestamp}_{canonical_id}`
- **Never use** offset-based pagination for unbounded lists
- Offset pagination allowed only for small, bounded lists (<1000 items)

---

## Rationale

### 1. Stability Under Concurrent Writes
- Cursor points to specific row by primary key
- New inserts don't shift cursor position
- User sees consistent data when paginating forward/backward

### 2. Performance at Any Scale
- Index-based seeks: O(log n) regardless of page depth
- Offset pagination degrades: page 1000 requires scanning 100K rows
- Cursor pagination: constant time for any page

### 3. Consistency Guarantees
- No duplicate items: cursor ensures strict ordering
- No missing items: cursor captures exact position
- Works correctly with streaming inserts

### 4. Scalability
- Efficient with millions of transactions
- Database index handles seek operations
- No full table scans required

---

## Consequences

### ✅ Positive

- **Stable pagination**: Cursors remain valid under concurrent inserts
- **Consistent performance**: O(log n) seeks at any page depth
- **No duplicates/missing**: Guaranteed consistent ordering
- **Scalable**: Works efficiently with unlimited dataset size

### ⚠️ Trade-offs

- **More complex implementation**: Cursor encoding/decoding logic required
- **Cannot jump to arbitrary pages**: Only next/prev navigation supported
- **Stateful cursors**: Client must maintain cursor between requests
- **Index dependency**: Requires compound index on (timestamp, canonical_id)

### 🔴 Risks (Mitigated)

- **Risk**: User expects "page 5 of 100" UI with numeric pages
  - **Mitigation**: Use infinite scroll or "Load More" pattern
  - Alternative: "Showing 50 of 10,234 transactions" with next/prev buttons

---

## Alternatives Considered

### Alternative A: Offset Pagination (Rejected)

**Pros:**
- Simple implementation
- Easy to understand ("page 5")
- Can jump to arbitrary page numbers

**Cons:**
- **Unstable**: Offset shifts under concurrent inserts
- **Poor performance**: O(n) scans at high offsets
- **Inconsistent**: Duplicate/missing items across pages
- **Not scalable**: Degrades with large datasets

### Alternative B: Seek Method with Direct Queries (Rejected)

**Pros:**
- Similar performance to cursor-based
- No cursor encoding needed

**Cons:**
- **Less standardized**: Every query needs custom WHERE clause
- **Harder to maintain**: No reusable pagination primitive
- **No direction reversal**: Difficult to paginate backward

### Alternative C: Opaque Page Tokens (Rejected)

**Pros:**
- Flexible: Can change pagination strategy without breaking API
- Secure: Client cannot decode/manipulate cursors

**Cons:**
- **Requires server-side state**: Must store token → query mapping
- **More infrastructure**: Need cache/database for token storage
- **Expiration issues**: Tokens must be cleaned up

---

## Known Limitations

### Race Condition with Backdated Transactions

**Problem**: Cursor pagination based on `timestamp` can skip records when backdated transactions are inserted:

```
Timeline:
T0: Client fetches page 1 (records with timestamp ≤ 10:00)
    Cursor = "2025-10-24T10:00:00Z_TX_xyz"

T1: Delayed upload arrives with backdated transaction
    timestamp = 2025-10-24T09:59:00Z (before cursor)

T2: Client fetches page 2 (WHERE timestamp > 10:00)
    Result: Record at 09:59 is SKIPPED forever
```

**When this occurs:**
1. **Backdated transactions**: Corrections, delayed bank uploads, manual adjustments
2. **Clock skew**: Distributed systems with unsynchronized clocks
3. **Batch imports**: Historical data loaded after current data

**Current behavior**: ⚠️ **Records inserted with timestamp < cursor will be skipped**

### Solution A: Sequence-Based Cursors (Recommended for v2)

**Strategy**: Use insertion order instead of business timestamp:

```sql
-- Add insertion sequence to table
ALTER TABLE transactions
ADD COLUMN insertion_sequence BIGSERIAL PRIMARY KEY;

-- Cursor based on insertion order (not timestamp)
Cursor = base64({insertion_sequence})

-- Query using insertion sequence (immune to backdating)
SELECT * FROM transactions
WHERE insertion_sequence > cursor_sequence
ORDER BY insertion_sequence ASC
LIMIT 50;
```

**Benefits:**
- ✅ Immune to backdated transactions (insertion order never changes)
- ✅ No clock skew issues (database-generated sequence)
- ✅ Simpler cursor format (single integer instead of compound key)

**Trade-offs:**
- ⚠️ UI displays transactions in insertion order, not chronological order
- ⚠️ Requires schema change (add insertion_sequence column)
- ⚠️ Must rebuild index for existing data

**Migration path:**
```sql
-- Step 1: Add column with backfill
ALTER TABLE transactions
ADD COLUMN insertion_sequence BIGSERIAL;

-- Step 2: Create index
CREATE INDEX idx_transactions_insertion_seq
ON transactions (insertion_sequence);

-- Step 3: Update API to use insertion_sequence cursor
-- Step 4: Drop old index on (timestamp, canonical_id) after migration
```

### Solution B: Snapshot Isolation (Alternative)

**Strategy**: Use database snapshot for entire pagination session:

```sql
-- Client starts pagination
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- All subsequent queries see same snapshot
SELECT * FROM transactions WHERE ...;  -- Page 1
SELECT * FROM transactions WHERE ...;  -- Page 2
...

-- Client finishes pagination
COMMIT;
```

**Benefits:**
- ✅ Perfect consistency (all pages see same data snapshot)
- ✅ No schema changes required
- ✅ Works with existing cursors

**Trade-offs:**
- ⚠️ Requires long-running transactions (minutes)
- ⚠️ Locks database resources during pagination
- ⚠️ Not practical for infinite scroll (session can last hours)

### Solution C: Explicit Snapshot ID (Hybrid)

**Strategy**: Capture snapshot timestamp, include in cursor:

```json
{
  "cursor": "base64(timestamp_id)",
  "snapshot_id": "uuid",  // References immutable query snapshot
  "snapshot_ts": "2025-10-24T10:00:00Z",
  "expires_at": "2025-10-24T12:00:00Z"  // 2-hour expiry
}
```

**Implementation:**
```sql
-- Page 1: Capture snapshot
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT NOW() AS snapshot_ts;  -- Store for subsequent pages

-- Page 2+: Query using snapshot timestamp
SELECT * FROM transactions
WHERE created_at < :snapshot_ts  -- Only see records before snapshot
  AND (timestamp, canonical_id) > (cursor_timestamp, cursor_id)
ORDER BY timestamp ASC, canonical_id ASC
LIMIT 50;
```

**Benefits:**
- ✅ Consistent pagination (no skipped records)
- ✅ No schema changes
- ✅ Reasonable session duration (expire after 2 hours)

**Trade-offs:**
- ⚠️ More complex cursor format
- ⚠️ Requires cursor expiration logic
- ⚠️ Client must include snapshot_ts in all requests

### Recommendation

**For current version (v1):**
- Document this as **known limitation**
- Acceptable for MVP (backdated transactions rare in normal operation)
- Add warning in UI: "Newly corrected transactions may not appear in current view. Refresh to see updates."

**For production (v2):**
- Implement **Solution A (Sequence-Based Cursors)**
- Cleanest solution, eliminates problem entirely
- One-time migration cost, permanent benefit

**For high-accuracy requirements (v3):**
- Optionally add **Solution C (Snapshot ID)** for audit/compliance views
- Use sequence cursors for normal views, snapshot cursors for "point-in-time" queries

---

## Implementation Notes

### Cursor Structure (Transparent Base64)

```
Cursor = base64({timestamp}_{canonical_id})

Example:
  timestamp: 2025-10-24T12:34:56Z
  canonical_id: TX_abc123
  → cursor: base64("2025-10-24T12:34:56Z_TX_abc123")
  → "MjAyNS0xMC0yNFQxMjozNDo1NlpfVFhfYWJjMTIz"
```

### Query Pattern

```sql
-- Forward pagination (next page)
SELECT * FROM transactions
WHERE (timestamp, canonical_id) > (cursor_timestamp, cursor_id)
ORDER BY timestamp ASC, canonical_id ASC
LIMIT 50;

-- Backward pagination (previous page)
SELECT * FROM transactions
WHERE (timestamp, canonical_id) < (cursor_timestamp, cursor_id)
ORDER BY timestamp DESC, canonical_id DESC
LIMIT 50;
```

### Required Index

```sql
CREATE INDEX idx_transactions_pagination
ON transactions (timestamp, canonical_id);
```

---

## Related Decisions

- **ADR-0001**: Uses `canonical_id` as stable identifier in cursor
- **Future**: Will extend to support multi-column sorting (e.g., amount + timestamp)
- **Future**: May add cursor expiration for security (24h TTL)

---

**References:**
- Vertical 2.1: [docs/verticals/2.1-transaction-list-view.md](../verticals/2.1-transaction-list-view.md)
- PaginationEngine: [docs/primitives/ol/PaginationEngine.md](../primitives/ol/PaginationEngine.md)
- IndexStrategy: [docs/primitives/ol/IndexStrategy.md](../primitives/ol/IndexStrategy.md)
