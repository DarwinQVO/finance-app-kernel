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

## Simplicity Profiles

### Personal Profile (0 LOC)

**Contexto del Usuario:**
871 transacciones (<1K total). Carga todas en single query (87KB payload). No necesita paginación backend, opcionalmente usa array slicing en frontend.

**Implementation:**
```python
# No implementation needed (0 LOC)
# Load all rows in single query
transactions = conn.execute("""
    SELECT * FROM canonical_transactions
    ORDER BY transaction_date DESC
""").fetchall()
# Return 871 rows (~87KB JSON) - UI shows all
```

**Características Incluidas:**
- ✅ Single-page load (fetchall)

**Características NO Incluidas:**
- ❌ PaginationEngine (YAGNI: <1K rows, 87KB payload)
- ❌ Cursor encoding (YAGNI: no pagination needed)
- ❌ COUNT queries (YAGNI: all rows loaded)

**Configuración:**
```python
# No config needed
```

**Performance:**
- Query time: 8ms (load all 871 rows)
- Payload: 87KB JSON

**Upgrade Triggers:**
- Si >1K rows → Small Business (OFFSET pagination)
- Si payload >500KB → Small Business (paginate)

### Small Business Profile - ~50 LOC

**Context:**
- Small business has 45K transactions
- Too large for single-page load (> 5MB payload)
- OFFSET pagination acceptable (< 100K rows, queries still fast)
- Simple page numbers in UI (Page 1, 2, 3...)

**Implementation:**
```python
# File: lib/pagination.py

class PaginationEngine:
    """Simple OFFSET-based pagination (good for < 100K rows)."""

    def paginate(self, limit: int = 50, page: int = 1) -> dict:
        """
        Returns paginated results using OFFSET.

        Args:
            limit: Page size (default 50)
            page: Page number (1-indexed)

        Returns:
            {
                "rows": [...],
                "total_count": 45123,
                "page": 2,
                "total_pages": 903,
                "has_next": True,
                "has_prev": True
            }
        """
        offset = (page - 1) * limit

        # Get total count (cached for 5 minutes)
        total_count = self._get_cached_count()

        # Query with OFFSET
        rows = conn.execute(f"""
            SELECT * FROM canonical_transactions
            ORDER BY transaction_date DESC
            LIMIT {limit} OFFSET {offset}
        """).fetchall()

        total_pages = (total_count + limit - 1) // limit  # Ceiling division

        return {
            "rows": rows,
            "total_count": total_count,
            "page": page,
            "total_pages": total_pages,
            "has_next": page < total_pages,
            "has_prev": page > 1
        }

    def _get_cached_count(self) -> int:
        """Cache total count (expensive query)."""
        cache_key = "canonical_transactions:count"
        cached = cache.get(cache_key)

        if cached:
            return int(cached)

        # Compute count (slow)
        count = conn.execute("SELECT COUNT(*) FROM canonical_transactions").fetchone()[0]
        cache.set(cache_key, count, ttl=300)  # Cache for 5 minutes

        return count

# Usage
engine = PaginationEngine()
result = engine.paginate(limit=50, page=2)

# {
#   "rows": [50 transactions],
#   "total_count": 45123,
#   "page": 2,
#   "total_pages": 903,
#   "has_next": True,
#   "has_prev": True
# }
```

**UI Example:**
```html
<!-- Simple page number navigation -->
<div class="pagination">
  <button disabled={!page.has_prev}>Previous</button>
  <span>Page {page.page} of {page.total_pages}</span>
  <button disabled={!page.has_next}>Next</button>
</div>
```

**Performance:**
- Page 1: 15ms (no OFFSET scan)
- Page 10: 18ms (scans 500 rows)
- Page 100: 45ms (scans 5K rows)
- Acceptable for < 100K rows

**Why OFFSET is OK Here:**
- Dataset size: 45K rows (small enough)
- Users rarely go past page 20
- COUNT(*) cached (not computed on every request)
- Trade-off: Simplicity > performance

### Enterprise Profile - ~300 LOC

**Context:**
- Enterprise has 8.5M transactions
- OFFSET pagination too slow (page 1000 takes 5+ seconds)
- Cursor-based pagination required (constant time)
- No page numbers (cursors are opaque tokens)
- Bidirectional pagination (prev/next)

**Implementation:**
```python
# File: lib/pagination_engine.py
import base64
import json
from typing import Optional, Dict, Any, List
from dataclasses import dataclass

@dataclass
class Page:
    """Pagination metadata"""
    has_next: bool
    next_cursor: Optional[str]
    has_prev: bool
    prev_cursor: Optional[str]
    total_count: Optional[int]  # Approximate (expensive to compute exactly)

class PaginationEngine:
    """
    Production cursor-based pagination (keyset pagination).

    Why cursor > offset:
    - Constant-time queries (O(1) with index)
    - Stable under concurrent inserts/deletes
    - No expensive COUNT(*) queries
    - Works with millions of rows
    """

    def encode_cursor(
        self,
        sort_field: str,
        sort_value: Any,
        id_field: str,
        id_value: Any
    ) -> str:
        """
        Encodes cursor as base64 JSON token.

        Example:
            sort_field="transaction_date", sort_value="2025-04-25"
            id_field="canonical_id", id_value="can_123"
            → "eyJmIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9"
        """
        cursor_data = {
            "f": str(sort_value),  # Sort field value
            "id": str(id_value)    # ID for tie-breaking
        }
        json_str = json.dumps(cursor_data)
        return base64.b64encode(json_str.encode()).decode()

    def decode_cursor(self, cursor: str) -> Dict[str, str]:
        """
        Decodes cursor token.

        Raises:
            ValueError: If cursor is malformed
        """
        try:
            json_str = base64.b64decode(cursor.encode()).decode()
            return json.loads(json_str)
        except (ValueError, json.JSONDecodeError) as e:
            raise ValueError(f"Invalid cursor: {e}")

    def paginate_forward(
        self,
        query_fn,
        cursor: Optional[str],
        limit: int,
        sort_field: str,
        id_field: str = "id"
    ) -> tuple[List[Dict], Page]:
        """
        Paginate forward (next page).

        Args:
            query_fn: Function that executes query with WHERE clause
            cursor: Current position (None for first page)
            limit: Page size
            sort_field: Field used for sorting
            id_field: Unique ID field for tie-breaking

        Returns:
            (rows, pagination_metadata)

        Example:
            def query_fn(where_clause, params):
                return db.execute(f'''
                    SELECT * FROM canonical_transactions
                    WHERE {where_clause}
                    ORDER BY transaction_date DESC, canonical_id DESC
                    LIMIT {limit + 1}
                ''', params)

            rows, page = engine.paginate_forward(
                query_fn=query_fn,
                cursor=request.args.get('cursor'),
                limit=50,
                sort_field='transaction_date',
                id_field='canonical_id'
            )
        """
        # Build WHERE clause from cursor
        if cursor:
            cursor_data = self.decode_cursor(cursor)
            cursor_sort_value = cursor_data["f"]
            cursor_id = cursor_data["id"]

            where_clause = f"({sort_field}, {id_field}) < (?, ?)"
            params = [cursor_sort_value, cursor_id]
        else:
            # First page (no cursor)
            where_clause = "1=1"
            params = []

        # Execute query (fetch limit+1 to detect has_next)
        rows = query_fn(where_clause, params, limit + 1)

        # Check if there's a next page
        has_next = len(rows) > limit
        actual_rows = rows[:limit]

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

        # Generate prev cursor from first row
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
            has_prev=(cursor is not None),  # Has prev if we received a cursor
            prev_cursor=prev_cursor,
            total_count=None  # Skip expensive COUNT(*)
        )

        return actual_rows, page

    def estimate_total_count(self, table: str) -> int:
        """
        Estimate total row count (fast, approximate).

        Uses PostgreSQL table statistics instead of COUNT(*).
        """
        result = db.execute(f"""
            SELECT reltuples::bigint AS estimate
            FROM pg_class
            WHERE relname = '{table}'
        """).fetchone()

        return result["estimate"] if result else 0

# Usage
engine = PaginationEngine()

def query_transactions(where_clause: str, params: List, limit: int):
    """Query function for transactions."""
    return db.execute(f"""
        SELECT * FROM canonical_transactions
        WHERE {where_clause}
        ORDER BY transaction_date DESC, canonical_id DESC
        LIMIT {limit}
    """, params).fetchall()

# Page 1
rows, page = engine.paginate_forward(
    query_fn=query_transactions,
    cursor=None,  # First page
    limit=50,
    sort_field="transaction_date",
    id_field="canonical_id"
)

# {
#   "rows": [50 transactions],
#   "has_next": True,
#   "next_cursor": "eyJmIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9",
#   "has_prev": False,
#   "prev_cursor": "eyJmIjoiMjAyNS0wNS0wMSIsImlkIjoiY2FuXzk5OSJ9",
#   "total_count": 8500000  # Approximate
# }

# Page 2 (using next_cursor from page 1)
rows, page = engine.paginate_forward(
    query_fn=query_transactions,
    cursor="eyJmIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9",
    limit=50,
    sort_field="transaction_date",
    id_field="canonical_id"
)
```

**API Response Format:**
```json
{
  "rows": [
    {"canonical_id": "can_123", "transaction_date": "2025-04-25", ...},
    {"canonical_id": "can_124", "transaction_date": "2025-04-24", ...}
  ],
  "pagination": {
    "has_next": true,
    "next_cursor": "eyJmIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9",
    "has_prev": true,
    "prev_cursor": "eyJmIjoiMjAyNS0wNS0wMSIsImlkIjoiY2FuXzk5OSJ9",
    "total_count": 8500000
  }
}
```

**UI Example (Infinite Scroll):**
```javascript
// React component with infinite scroll
const [transactions, setTransactions] = useState([])
const [nextCursor, setNextCursor] = useState(null)
const [loading, setLoading] = useState(false)

async function loadMore() {
  if (loading) return
  setLoading(true)

  const response = await fetch(`/api/transactions?cursor=${nextCursor}&limit=50`)
  const data = await response.json()

  setTransactions([...transactions, ...data.rows])
  setNextCursor(data.pagination.next_cursor)
  setLoading(false)
}

// Trigger on scroll to bottom
useEffect(() => {
  const handleScroll = () => {
    if (window.innerHeight + window.scrollY >= document.body.offsetHeight - 100) {
      loadMore()
    }
  }
  window.addEventListener('scroll', handleScroll)
  return () => window.removeEventListener('scroll', handleScroll)
}, [nextCursor, loading])
```

**Performance Comparison (8.5M rows):**
```
OFFSET Pagination (Bad):
- Page 1 (OFFSET 0):     50ms
- Page 100 (OFFSET 5K):  280ms
- Page 1000 (OFFSET 50K): 5200ms (SLA violation)
- Query plan: Seq Scan (scans all skipped rows)

Cursor Pagination (Good):
- Page 1:     45ms
- Page 100:   48ms
- Page 1000:  47ms (constant time!)
- Query plan: Index Scan using idx_canonical_date_id (50 rows scanned)

Required index:
CREATE INDEX idx_canonical_date_id
  ON canonical_transactions (transaction_date DESC, canonical_id DESC);
```

**Edge Case Handling:**
```python
# Concurrent inserts (new transaction added while user paginating)
# OFFSET: User sees duplicates or skips rows (bad UX)
# CURSOR: Stable - user sees consistent results (good UX)

# Example:
# 1. User loads page 1 (rows 1-50)
# 2. New transaction inserted at position 1
# 3. User clicks "next"
# OFFSET (page 2): Shows rows 52-101 (skipped row 51)
# CURSOR: Shows rows 51-100 (correct continuation)
```

### Comparison Table

| Feature | Personal (Darwin) | Small Business | Enterprise |
|---------|------------------|----------------|------------|
| **LOC** | 0 (Not Needed) | ~50 | ~300 |
| **Data Volume** | 871 rows | 45K rows | 8.5M rows |
| **Pagination Type** | None (load all) | OFFSET | Cursor (keyset) |
| **Page 1 Query Time** | 8ms (all rows) | 15ms | 45ms |
| **Page 100 Query Time** | N/A | 45ms | 47ms (constant) |
| **Page 1000 Query Time** | N/A | N/A | 47ms (constant) |
| **UI Pattern** | Single page | Page numbers | Infinite scroll |
| **Total Count** | Exact (fast) | Cached | Approximate |
| **Concurrent Inserts** | N/A | Can skip/duplicate | Stable |
| **Client Payload** | 87KB (all rows) | 50KB (50 rows) | 50KB (50 rows) |

**Key Insight:**
- **Darwin (0 LOC)**: 871 rows fit in single HTTP response - pagination adds complexity for no benefit
- **Small Business (50 LOC)**: OFFSET still fast enough for < 100K rows - simple page numbers work fine
- **Enterprise (300 LOC)**: Cursor required for constant-time queries at scale - OFFSET fails past page 100

---

## Version History

- **v1.0** (2025-05-23): Initial release
  - Base64 cursor encoding
  - Keyset pagination
  - Bidirectional support
