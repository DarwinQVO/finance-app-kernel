# PaginationEngine (OL Primitive)

## Definition

**PaginationEngine** implements cursor-based pagination (keyset pagination) for large datasets. It provides stable, performant pagination that doesn't break when rows are inserted/deleted during user navigation.

**Problem it solves:**
- OFFSET pagination is slow on large tables (scans all skipped rows)
- OFFSET pagination breaks when rows inserted (skips/duplicates items)
- Page numbers don't work for real-time data (page 5 changes as data updates)
- Complex to implement cursor logic correctly (encoding, decoding, edge cases)

**Solution:**
- Cursor encodes `(last_sort_value, last_id)` - not row offset
- Queries use `WHERE (sort_field, id) > (cursor_value, cursor_id)`
- Constant-time performance (doesn't scan skipped rows)
- Stable ordering even with concurrent inserts

---

## Interface Contract

```python
from typing import Optional, Dict, Any, List
from dataclasses import dataclass
import base64
import json

@dataclass
class Page:
    """Pagination metadata"""
    has_next: bool
    next_cursor: Optional[str]
    has_prev: bool
    prev_cursor: Optional[str]
    total_count: Optional[int]  # Approximate (expensive to compute)

class PaginationEngine:
    """
    Cursor-based pagination engine.

    Domain-agnostic - works with any ordered dataset.
    """

    def encode_cursor(
        self,
        sort_field: str,
        sort_value: Any,
        id_field: str,
        id_value: Any
    ) -> str:
        """
        Encodes cursor as opaque token (base64 JSON)

        Example:
            sort_field="transaction_date", sort_value="2025-04-25"
            id_field="canonical_id", id_value="can_123"

            → encodes to: "eyJmIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9"

        Returns:
            Base64-encoded JSON string
        """
        cursor_data = {
            "f": str(sort_value),  # Sort field value
            "id": str(id_value)    # ID field value (for tie-breaking)
        }
        json_str = json.dumps(cursor_data)
        return base64.b64encode(json_str.encode()).decode()

    def decode_cursor(self, cursor: str) -> Dict[str, str]:
        """
        Decodes cursor token to (sort_value, id_value)

        Example:
            "eyJmIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9"
            → {"f": "2025-04-25", "id": "can_123"}

        Raises:
            ValueError: If cursor is malformed or expired
        """
        try:
            json_str = base64.b64decode(cursor.encode()).decode()
            return json.loads(json_str)
        except (ValueError, json.JSONDecodeError) as e:
            raise ValueError(f"Invalid cursor: {e}")

    def paginate(
        self,
        rows: List[Dict],
        limit: int,
        sort_field: str,
        id_field: str = "id"
    ) -> tuple[List[Dict], Page]:
        """
        Paginates result set and generates pagination metadata

        Args:
            rows: Query results (MUST fetch limit+1 rows)
            limit: Page size (e.g., 50)
            sort_field: Field used for sorting (e.g., "transaction_date")
            id_field: Unique ID field for tie-breaking (default: "id")

        Returns:
            (paginated_rows, pagination_metadata)

        Example:
            # Query fetched 51 rows (limit=50 + 1)
            rows = query_db("SELECT * ... LIMIT 51")
            paginated, page = engine.paginate(rows, limit=50, sort_field="date")

            if page.has_next:
                print(f"Next page cursor: {page.next_cursor}")
        """
        has_next = len(rows) > limit
        actual_rows = rows[:limit]  # Discard extra row

        # Generate next cursor from last row
        next_cursor = None
        if has_next and actual_rows:
            last_row = actual_rows[-1]
            next_cursor = self.encode_cursor(
                sort_field=sort_field,
                sort_value=last_row[sort_field],
                id_field=id_field,
                id_value=last_row[id_field]
            )

        # Generate prev cursor from first row (for bidirectional pagination)
        prev_cursor = None
        if actual_rows:
            first_row = actual_rows[0]
            prev_cursor = self.encode_cursor(
                sort_field=sort_field,
                sort_value=first_row[sort_field],
                id_field=id_field,
                id_value=first_row[id_field]
            )

        page = Page(
            has_next=has_next,
            next_cursor=next_cursor,
            has_prev=False,  # Determined by caller (did they provide cursor?)
            prev_cursor=prev_cursor,
            total_count=None  # Expensive to compute, defer to caller
        )

        return actual_rows, page
```

---

## Multi-Domain Applicability

**Universal Pattern:** Paginate any ordered dataset

### Finance Domain
```python
engine = PaginationEngine()

# Query transactions (fetch limit+1)
rows = db.execute("""
    SELECT * FROM canonical_transactions
    WHERE (transaction_date, canonical_id) > ($1, $2)
    ORDER BY transaction_date DESC, canonical_id DESC
    LIMIT 51
""", cursor_date, cursor_id)

transactions, page = engine.paginate(
    rows=rows,
    limit=50,
    sort_field="transaction_date",
    id_field="canonical_id"
)

# UI: Show page.next_cursor for "Next" button
```

### Healthcare Domain
```python
# Paginate lab results
rows = db.execute("""
    SELECT * FROM canonical_lab_results
    WHERE (test_date, result_id) < ($1, $2)
    ORDER BY test_date DESC, result_id DESC
    LIMIT 101
""", cursor_date, cursor_id)

results, page = engine.paginate(
    rows=rows,
    limit=100,
    sort_field="test_date",
    id_field="result_id"
)
```

### Legal Domain
```python
# Paginate contract clauses
rows = db.execute("""
    SELECT * FROM canonical_contract_clauses
    WHERE (effective_date, clause_id) > ($1, $2)
    ORDER BY effective_date ASC, clause_id ASC
    LIMIT 201
""", cursor_date, cursor_id)

clauses, page = engine.paginate(
    rows=rows,
    limit=200,
    sort_field="effective_date",
    id_field="clause_id"
)
```

### Research Domain (RSRCH - Utilitario)
```python
# Paginate founder facts
rows = db.execute("""
    SELECT * FROM canonical_facts
    WHERE (discovered_at, fact_id) < ($1, $2)
    ORDER BY discovered_at DESC, fact_id DESC
    LIMIT 51
""", cursor_date, cursor_id)

facts, page = engine.paginate(
    rows=rows,
    limit=50,
    sort_field="discovered_at",
    id_field="fact_id"
)
```

### Manufacturing Domain
```python
# Paginate QC measurements
rows = db.execute("""
    SELECT * FROM canonical_qc_measurements
    WHERE (measurement_date, measurement_id) > ($1, $2)
    ORDER BY measurement_date DESC, measurement_id DESC
    LIMIT 101
""", cursor_date, cursor_id)

measurements, page = engine.paginate(
    rows=rows,
    limit=100,
    sort_field="measurement_date",
    id_field="measurement_id"
)
```

### Media Domain
```python
# Paginate transcript snippets
rows = db.execute("""
    SELECT * FROM canonical_transcript_snippets
    WHERE (timestamp, snippet_id) > ($1, $2)
    ORDER BY timestamp ASC, snippet_id ASC
    LIMIT 51
""", cursor_timestamp, cursor_id)

snippets, page = engine.paginate(
    rows=rows,
    limit=50,
    sort_field="timestamp",
    id_field="snippet_id"
)
```

---

## Responsibilities

✅ **DOES:**
- Encode cursor tokens (base64 JSON)
- Decode cursor tokens (with validation)
- Generate pagination metadata (has_next, has_prev, cursors)
- Handle tie-breaking (sort_field + id_field for stability)
- Detect last page (has_next = False)

---

## NOT Responsibilities

❌ **DOES NOT:**
- Build SQL queries (caller does this)
- Execute queries
- Calculate total_count (expensive, caller can estimate)
- Store cursors (cursors are stateless, stored client-side)
- Handle page numbers (cursor pagination doesn't use page numbers)
- Cache results

---

## Implementation Notes

### Why Cursor > Offset

**OFFSET pagination (old way):**
```sql
-- Page 10 (slow - scans 500 rows)
SELECT * FROM transactions
ORDER BY date DESC
OFFSET 500 LIMIT 50;
```

**Problems:**
- Scans all skipped rows (O(n) where n = offset)
- Breaks when rows inserted (page 10 shifts)
- Can't jump to arbitrary pages (page numbers meaningless)

---

**Cursor pagination (this primitive):**
```sql
-- Next page (fast - scans 51 rows)
SELECT * FROM transactions
WHERE (date, id) > ($cursor_date, $cursor_id)
ORDER BY date DESC, id DESC
LIMIT 51;
```

**Benefits:**
- Constant time (O(1) with index)
- Stable under inserts/deletes
- Cursors encode position, not page number

---

### Cursor Format

**Structure:**
```json
{
  "f": "2025-04-25",  // Sort field value (date, amount, etc.)
  "id": "can_123"     // ID for tie-breaking
}
```

**Encoded (base64):**
```
eyJmIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9
```

**Why base64:**
- Opaque (user can't easily manipulate)
- URL-safe
- Short (compact cursors)

**Optional: Add HMAC signature to prevent tampering**
```json
{
  "f": "2025-04-25",
  "id": "can_123",
  "sig": "sha256_hmac_signature"  // Verify cursor wasn't modified
}
```

---

### Bidirectional Pagination

**Forward (next page):**
```sql
WHERE (date, id) > ($cursor_date, $cursor_id)
ORDER BY date DESC, id DESC
```

**Backward (prev page):**
```sql
WHERE (date, id) < ($cursor_date, $cursor_id)
ORDER BY date ASC, id ASC  -- Reversed sort
LIMIT 51
```

Then reverse result set in application code.

---

### Edge Cases

**Empty result set:**
```python
paginated, page = engine.paginate(rows=[], limit=50, sort_field="date")
# page.has_next = False
# page.next_cursor = None
```

**Last page (fewer than limit rows):**
```python
paginated, page = engine.paginate(rows=[...30 rows...], limit=50, sort_field="date")
# page.has_next = False (len(rows) < limit)
# page.next_cursor = None
```

**Total count estimation:**
```sql
-- Approximate count (fast)
SELECT reltuples::bigint AS estimate
FROM pg_class
WHERE relname = 'canonical_transactions';
```

---

## Related Primitives

**Used by:**
- **TransactionQuery** - Calls PaginationEngine.encode_cursor / decode_cursor
- **ExportEngine** - Uses for large exports (paginate internally, stream to file)

**Replaces:**
- OFFSET/LIMIT pagination
- Manual cursor logic in application code

---

## Version History

- **v1.0** (2025-05-23): Initial release
  - Base64 cursor encoding
  - Keyset pagination
  - Bidirectional support
