# TransactionQuery (OL Primitive)

## Definition

**TransactionQuery** is a universal query builder that translates filter/sort parameters into optimized SQL queries for canonical data stores. It provides a domain-agnostic interface for querying structured, validated records.

**Problem it solves:**
- Manual SQL generation is error-prone (SQL injection, syntax errors)
- Filter logic scattered across codebase (inconsistent behavior)
- Query optimization requires deep SQL knowledge
- Pagination strategies vary (offset vs cursor)

**Solution:**
- Declarative filter specification (JSON/dict)
- Automatic SQL generation with parameterization
- Built-in index hints for performance
- Cursor-based pagination (keyset pagination)

---

## Interface Contract

```python
from typing import Optional, List, Dict, Any
from dataclasses import dataclass

@dataclass
class FilterParams:
    """Filter specification (domain-specific fields)"""
    filters: Dict[str, Any]  # e.g., {"date_from": "2025-04-01", "category": "Business"}

@dataclass
class SortParams:
    """Sort specification"""
    field: str   # Column name
    order: str   # "asc" | "desc"

@dataclass
class SQLQuery:
    """Generated SQL query"""
    query: str          # Parameterized SQL
    params: List[Any]   # Query parameters (for SQL injection prevention)
    index_hints: List[str]  # Suggested indexes to use

class TransactionQuery:
    """
    Universal query builder for canonical stores.

    Domain-agnostic - works on ANY schema with proper field mapping.
    """

    def __init__(self, schema: str, field_mapping: Dict[str, str]):
        """
        Args:
            schema: Table name (e.g., "canonical_transactions")
            field_mapping: Maps filter keys to DB columns
                           e.g., {"date_from": "transaction_date", "merchant": "merchant"}
        """
        self.schema = schema
        self.field_mapping = field_mapping

    def build_query(
        self,
        filters: FilterParams,
        sort: SortParams,
        cursor: Optional[str] = None,
        limit: int = 50
    ) -> SQLQuery:
        """
        Builds optimized SQL query with:
        - WHERE clause from filters
        - ORDER BY from sort params
        - Cursor-based pagination (keyset pagination)
        - Index hints for performance

        Example:
            filters = {"date_from": "2025-04-01", "category": "Business"}
            sort = {"field": "date", "order": "desc"}
            cursor = None  # First page
            limit = 50

            → SELECT * FROM canonical_transactions
              WHERE transaction_date >= $1 AND category = $2
              ORDER BY transaction_date DESC, canonical_id DESC
              LIMIT 51  -- +1 to check if has_next
              -- Index hint: idx_canonical_date_category

        Returns:
            SQLQuery with parameterized query + params + index hints
        """

    def parse_cursor(self, cursor: str) -> Dict[str, Any]:
        """
        Decodes opaque cursor token to (last_sort_value, last_id)

        Example:
            "eyJkYXRlIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9" (base64)
            → {"date": "2025-04-25", "id": "can_123"}

        Raises:
            ValueError: If cursor is invalid or expired
        """

    def encode_cursor(self, last_row: Dict[str, Any], sort_field: str) -> str:
        """
        Encodes last row into opaque cursor token

        Example:
            last_row = {"transaction_date": "2025-04-25", "canonical_id": "can_123"}
            sort_field = "transaction_date"
            → "eyJkYXRlIjoiMjAyNS0wNC0yNSIsImlkIjoiY2FuXzEyMyJ9"
        """

    def validate_filters(self, filters: FilterParams) -> None:
        """
        Validates filter params against schema

        Raises:
            ValueError: If filter field unknown or value type invalid
        """
```

---

## Multi-Domain Applicability

**Universal Pattern:** Query canonical records with filters/sort/pagination

### Finance Domain
```python
query = TransactionQuery(
    schema="canonical_transactions",
    field_mapping={
        "date_from": "transaction_date",
        "date_to": "transaction_date",
        "account": "account_id",
        "merchant": "merchant",
        "category": "category",
        "amount_min": "amount",
        "amount_max": "amount"
    }
)

sql = query.build_query(
    filters={"date_from": "2025-04-01", "category": "Business > Software"},
    sort={"field": "date", "order": "desc"},
    limit=50
)
# → SELECT * FROM canonical_transactions
#   WHERE transaction_date >= '2025-04-01' AND category = 'Business > Software'
#   ORDER BY transaction_date DESC, canonical_id DESC
#   LIMIT 51
```

### Healthcare Domain
```python
query = TransactionQuery(
    schema="canonical_lab_results",
    field_mapping={
        "date_from": "test_date",
        "test_type": "test_type",
        "normal": "is_normal",
        "doctor": "ordering_physician"
    }
)

sql = query.build_query(
    filters={"date_from": "2025-01-01", "normal": False},
    sort={"field": "test_date", "order": "desc"},
    limit=100
)
# → Query abnormal lab results from 2025
```

### Legal Domain
```python
query = TransactionQuery(
    schema="canonical_contract_clauses",
    field_mapping={
        "contract_type": "contract_type",
        "party": "counterparty",
        "risk_level": "risk_level",
        "effective_date": "effective_date"
    }
)

sql = query.build_query(
    filters={"risk_level": "high", "party": "Acme Corp"},
    sort={"field": "effective_date", "order": "asc"},
    limit=200
)
# → Query high-risk clauses for specific counterparty
```

### Research Domain
```python
query = TransactionQuery(
    schema="canonical_citations",
    field_mapping={
        "author": "author",
        "year": "publication_year",
        "topic": "topic",
        "publication": "journal"
    }
)

sql = query.build_query(
    filters={"author": "Smith", "year_from": 2020},
    sort={"field": "year", "order": "desc"},
    limit=50
)
# → Query recent papers by author
```

### Manufacturing Domain
```python
query = TransactionQuery(
    schema="canonical_qc_measurements",
    field_mapping={
        "product": "product_id",
        "batch": "batch_number",
        "pass": "passed_qc",
        "date": "measurement_date"
    }
)

sql = query.build_query(
    filters={"product": "SKU-12345", "pass": False},
    sort={"field": "date", "order": "desc"},
    limit=100
)
# → Query failed QC measurements for product
```

### Media Domain
```python
query = TransactionQuery(
    schema="canonical_transcript_snippets",
    field_mapping={
        "speaker": "speaker_id",
        "topic": "topic",
        "timestamp": "timestamp",
        "sentiment": "sentiment_score"
    }
)

sql = query.build_query(
    filters={"speaker": "John Doe", "sentiment_min": 0.7},
    sort={"field": "timestamp", "order": "asc"},
    limit=50
)
# → Query positive statements by speaker
```

---

## Responsibilities

✅ **DOES:**
- Translate filter params to SQL WHERE clauses
- Generate ORDER BY from sort params
- Implement cursor-based pagination (keyset pagination)
- Sanitize inputs (SQL injection prevention)
- Suggest index hints for performance
- Validate filter params against schema
- Handle multi-field sorts (e.g., date DESC, id DESC for stability)
- Encode/decode cursor tokens (base64 JSON)

---

## NOT Responsibilities

❌ **DOES NOT:**
- Execute queries (returns SQLQuery, doesn't run it)
- Manage database connections
- Cache query results
- Transform result rows (returns raw DB rows)
- Handle authentication/authorization
- Create/manage indexes (only suggests them)
- Perform aggregations (SUM, AVG, etc.) - caller handles summary metrics

---

## Implementation Notes

### Storage

**No persistent storage** - TransactionQuery is stateless

**Cursor storage:**
- Cursors are ephemeral (encoded in API response, stored client-side)
- Optional: Cache cursor → query mapping in Redis (TTL 1 hour) for validation

---

### Performance

**Keyset Pagination (Why not OFFSET):**
```sql
-- OFFSET pagination (BAD - slow on large tables)
SELECT * FROM transactions
ORDER BY date DESC
OFFSET 500 LIMIT 50;  -- Scans first 500 rows

-- Keyset pagination (GOOD - constant time)
SELECT * FROM transactions
WHERE (date, id) < ($cursor_date, $cursor_id)
ORDER BY date DESC, id DESC
LIMIT 51;  -- Only scans 51 rows
```

**Index Hints:**
```python
# TransactionQuery suggests indexes based on filter/sort combo
filters = {"date_from": "2025-04-01", "category": "Business"}
sort = {"field": "date", "order": "desc"}

→ suggests: "idx_canonical_date_category"
```

**Query Limit:**
- Always fetch `limit + 1` rows
- If result count == limit + 1 → `has_next = True`, discard last row
- Efficient check without COUNT(*)

---

### Edge Cases

**Multi-currency filtering:**
```python
# Filter by amount in USD, but table has MXN rows
filters = {"amount_min": 100, "currency": "USD"}

# TransactionQuery generates:
WHERE (amount >= 100 AND currency = 'USD')
   OR (amount_usd >= 100 AND currency != 'USD')
```

**Case-insensitive search:**
```python
filters = {"search": "Uber"}

# Generates:
WHERE LOWER(merchant) LIKE LOWER('%Uber%')
```

**Date range inclusive:**
```python
filters = {"date_from": "2025-04-01", "date_to": "2025-04-30"}

# Generates:
WHERE transaction_date >= '2025-04-01'
  AND transaction_date <= '2025-04-30 23:59:59'
```

---

## Related Primitives

**Uses:**
- **PaginationEngine** - Cursor encoding/decoding logic
- **IndexStrategy** - Suggests which indexes to use

**Used by:**
- **CanonicalStore** - Queries canonical records
- **ExportEngine** - Queries data for export (no pagination)

**Replaces:**
- Manual SQL generation in application code
- ORM query builders (for complex queries with performance requirements)

---

## Simplicity Profiles

The same TransactionQuery interface supports three implementation scales: Personal (hardcoded SQL), Small Business (simple query builder), Enterprise (optimized cursor pagination + index hints).

### Personal Profile (0 LOC - Not Needed)

**Darwin's Approach:** No query builder - write SQL directly

**Why Darwin doesn't need this:**
- Darwin has simple queries ("get all transactions" or "filter by date")
- SQL is 5-10 lines max, easy to read
- No cursor pagination needed (Darwin has < 1000 transactions total)
- No complex multi-filter combinations

**What Darwin DOESN'T need:**
- ❌ Query builder abstraction
- ❌ Cursor-based pagination (OFFSET sufficient for 1000 rows)
- ❌ Index hints (SQLite auto-indexes work fine at this scale)
- ❌ Multi-field filter combinations

**Implementation:**

Darwin skips this primitive and writes SQL directly.

**Example (Darwin's simple queries):**
```python
# Get all transactions for current month
def get_current_month_transactions():
    query = """
        SELECT * FROM transactions
        WHERE date >= ?
        ORDER BY date DESC
    """
    return db.execute(query, ["2024-11-01"])

# Filter by category
def get_transactions_by_category(category):
    query = """
        SELECT * FROM transactions
        WHERE category_id = ?
        ORDER BY date DESC
    """
    return db.execute(query, [category])

# Simple pagination (OFFSET is fine for 1000 rows)
def get_transactions_page(page=1, per_page=50):
    offset = (page - 1) * per_page
    query = """
        SELECT * FROM transactions
        ORDER BY date DESC
        LIMIT ? OFFSET ?
    """
    return db.execute(query, [per_page, offset])
```

**Decision Context:**
- Darwin has ~500 transactions/year → 3 years = 1500 transactions total
- OFFSET pagination is fast for < 10K rows
- No need for query builder when SQL is 5 lines
- YAGNI: Skip TransactionQuery when you write SQL directly

---

### Small Business Profile (120 LOC)

**Use Case:** Accounting firm with multiple filters, basic pagination

**What they need (vs Personal):**
- ✅ **Simple query builder** (avoid manual SQL for common queries)
- ✅ **Multi-filter support** (date range + category + account)
- ✅ **OFFSET pagination** (< 50K transactions, OFFSET is fine)
- ✅ **SQL injection prevention** (parameterized queries)
- ❌ No cursor pagination (not enough volume)
- ❌ No index hints (database handles this)

**Implementation:**

**Code Example (120 LOC):**
```python
# transaction_query.py (Small Business)

from typing import Dict, List, Optional, Any
from dataclasses import dataclass

@dataclass
class QueryResult:
    query: str
    params: List[Any]

class TransactionQuery:
    """Simple query builder with filters and OFFSET pagination."""

    def __init__(self, table: str = "transactions"):
        self.table = table

    def build_query(
        self,
        filters: Optional[Dict[str, Any]] = None,
        sort_field: str = "date",
        sort_order: str = "desc",
        page: int = 1,
        per_page: int = 50
    ) -> QueryResult:
        """
        Build SQL query with filters and pagination.

        Args:
            filters: Dict of field → value (e.g., {"category": "Food", "date_from": "2024-11-01"})
            sort_field: Column to sort by
            sort_order: "asc" or "desc"
            page: Page number (1-indexed)
            per_page: Items per page

        Returns:
            QueryResult with parameterized SQL + params
        """
        # Start with base query
        query_parts = [f"SELECT * FROM {self.table}"]
        params = []

        # Build WHERE clause
        if filters:
            where_clauses = []

            for field, value in filters.items():
                # Handle special filter types
                if field == "date_from":
                    where_clauses.append("date >= ?")
                    params.append(value)
                elif field == "date_to":
                    where_clauses.append("date <= ?")
                    params.append(value)
                elif field == "amount_min":
                    where_clauses.append("amount >= ?")
                    params.append(value)
                elif field == "amount_max":
                    where_clauses.append("amount <= ?")
                    params.append(value)
                elif field == "search":
                    # Case-insensitive merchant search
                    where_clauses.append("LOWER(merchant) LIKE LOWER(?)")
                    params.append(f"%{value}%")
                else:
                    # Exact match
                    where_clauses.append(f"{field} = ?")
                    params.append(value)

            if where_clauses:
                query_parts.append("WHERE " + " AND ".join(where_clauses))

        # ORDER BY
        query_parts.append(f"ORDER BY {sort_field} {sort_order.upper()}")

        # OFFSET pagination
        offset = (page - 1) * per_page
        query_parts.append(f"LIMIT ? OFFSET ?")
        params.extend([per_page, offset])

        return QueryResult(
            query=" ".join(query_parts),
            params=params
        )

    def count(self, filters: Optional[Dict[str, Any]] = None) -> QueryResult:
        """Build COUNT query for total rows."""
        query_parts = [f"SELECT COUNT(*) FROM {self.table}"]
        params = []

        if filters:
            # Same WHERE logic as build_query
            where_clauses = []

            for field, value in filters.items():
                if field == "date_from":
                    where_clauses.append("date >= ?")
                    params.append(value)
                elif field == "date_to":
                    where_clauses.append("date <= ?")
                    params.append(value)
                elif field == "amount_min":
                    where_clauses.append("amount >= ?")
                    params.append(value)
                elif field == "amount_max":
                    where_clauses.append("amount <= ?")
                    params.append(value)
                elif field == "search":
                    where_clauses.append("LOWER(merchant) LIKE LOWER(?)")
                    params.append(f"%{value}%")
                else:
                    where_clauses.append(f"{field} = ?")
                    params.append(value)

            if where_clauses:
                query_parts.append("WHERE " + " AND ".join(where_clauses))

        return QueryResult(
            query=" ".join(query_parts),
            params=params
        )
```

**Usage:**
```python
# Build query
query_builder = TransactionQuery("transactions")

result = query_builder.build_query(
    filters={
        "date_from": "2024-11-01",
        "date_to": "2024-11-30",
        "category": "Food & Dining"
    },
    sort_field="date",
    sort_order="desc",
    page=1,
    per_page=50
)

print(result.query)
# SELECT * FROM transactions
# WHERE date >= ? AND date <= ? AND category = ?
# ORDER BY date DESC
# LIMIT ? OFFSET ?

print(result.params)
# ['2024-11-01', '2024-11-30', 'Food & Dining', 50, 0]

# Execute query
rows = db.execute(result.query, result.params)

# Get total count
count_result = query_builder.count(filters={
    "date_from": "2024-11-01",
    "category": "Food & Dining"
})
total = db.execute(count_result.query, count_result.params)[0][0]
```

**Multi-filter example:**
```python
# Complex filter: Date range + category + merchant search + amount range
result = query_builder.build_query(
    filters={
        "date_from": "2024-10-01",
        "date_to": "2024-10-31",
        "category": "Business > Software",
        "search": "Adobe",  # Merchant contains "Adobe"
        "amount_min": 10.00,
        "amount_max": 100.00
    },
    sort_field="amount",
    sort_order="desc",
    page=1,
    per_page=100
)
```

**Testing:**
- ✅ Unit tests for each filter type
- ✅ Integration test: Execute query against test database
- ✅ SQL injection test: Try malicious inputs

**Decision Context:**
- Accounting firm has 20K transactions (50 clients × 400 transactions/year)
- OFFSET pagination fast for < 50K rows
- Simple query builder reduces code duplication
- Parameterized queries prevent SQL injection

---

### Enterprise Profile (600 LOC)

**Use Case:** Multi-tenant platform with millions of transactions, optimized cursor pagination

**What they need (vs Small Business):**
- ✅ All Small Business features (filters, SQL injection prevention)
- ✅ **Cursor-based pagination** (keyset pagination for millions of rows)
- ✅ **Index hints** (suggest optimal indexes based on filter/sort combo)
- ✅ **Multi-field sorts** (stable sorts with tie-breaking)
- ✅ **Query optimization** (analyze filter selectivity)
- ✅ **Opaque cursor tokens** (base64 encoded, not exposing internal IDs)

**Implementation:**

**Database Schema:**
```sql
-- Optimized indexes for common query patterns
CREATE INDEX idx_transactions_date_id ON transactions(date DESC, id DESC);
CREATE INDEX idx_transactions_date_category ON transactions(date DESC, category, id DESC);
CREATE INDEX idx_transactions_account_date ON transactions(account_id, date DESC, id DESC);
CREATE INDEX idx_transactions_merchant ON transactions(LOWER(merchant), id);
```

**Code Example (600 LOC - excerpt):**
```python
# transaction_query.py (Enterprise)

import json
import base64
from typing import Dict, List, Optional, Any, Tuple
from dataclasses import dataclass

@dataclass
class QueryResult:
    query: str
    params: List[Any]
    index_hints: List[str]

class TransactionQuery:
    """Enterprise query builder with cursor pagination + index hints."""

    def __init__(self, table: str = "transactions"):
        self.table = table
        # Define index mappings for common query patterns
        self.index_map = {
            ("date",): "idx_transactions_date_id",
            ("date", "category"): "idx_transactions_date_category",
            ("account_id", "date"): "idx_transactions_account_date",
            ("merchant",): "idx_transactions_merchant"
        }

    def build_query(
        self,
        filters: Optional[Dict[str, Any]] = None,
        sort_field: str = "date",
        sort_order: str = "desc",
        cursor: Optional[str] = None,
        limit: int = 50
    ) -> QueryResult:
        """
        Build optimized cursor-based query.

        Cursor pagination (keyset):
        - WHERE (sort_field, id) < (cursor_sort_value, cursor_id)
        - Constant-time performance (no OFFSET scan)
        """
        query_parts = [f"SELECT * FROM {self.table}"]
        params = []

        # Build WHERE clauses
        where_clauses = []

        # 1. User filters
        if filters:
            where_clauses.extend(self._build_filter_clauses(filters, params))

        # 2. Cursor filter (keyset pagination)
        if cursor:
            cursor_data = self._decode_cursor(cursor)
            cursor_sort_value = cursor_data.get(sort_field)
            cursor_id = cursor_data.get("id")

            if sort_order.lower() == "desc":
                where_clauses.append(f"({sort_field}, id) < (?, ?)")
            else:
                where_clauses.append(f"({sort_field}, id) > (?, ?)")

            params.extend([cursor_sort_value, cursor_id])

        if where_clauses:
            query_parts.append("WHERE " + " AND ".join(where_clauses))

        # ORDER BY (multi-field for stable sort)
        query_parts.append(f"ORDER BY {sort_field} {sort_order.upper()}, id {sort_order.upper()}")

        # LIMIT (fetch +1 to check has_next)
        query_parts.append(f"LIMIT ?")
        params.append(limit + 1)

        # Suggest index hint
        index_hints = self._suggest_indexes(filters, sort_field)

        return QueryResult(
            query=" ".join(query_parts),
            params=params,
            index_hints=index_hints
        )

    def _build_filter_clauses(self, filters: Dict[str, Any], params: List[Any]) -> List[str]:
        """Build WHERE clauses from filters."""
        clauses = []

        for field, value in filters.items():
            if field == "date_from":
                clauses.append("date >= ?")
                params.append(value)
            elif field == "date_to":
                clauses.append("date <= ?")
                params.append(value)
            elif field == "amount_min":
                clauses.append("amount >= ?")
                params.append(value)
            elif field == "amount_max":
                clauses.append("amount <= ?")
                params.append(value)
            elif field == "search":
                clauses.append("LOWER(merchant) LIKE LOWER(?)")
                params.append(f"%{value}%")
            elif field == "account_id":
                clauses.append("account_id = ?")
                params.append(value)
            elif field == "category":
                clauses.append("category = ?")
                params.append(value)
            elif field == "tags":
                # Array contains (PostgreSQL)
                clauses.append("? = ANY(tags)")
                params.append(value)
            else:
                # Generic exact match
                clauses.append(f"{field} = ?")
                params.append(value)

        return clauses

    def _suggest_indexes(self, filters: Optional[Dict[str, Any]], sort_field: str) -> List[str]:
        """Suggest optimal indexes based on filters + sort."""
        if not filters:
            # No filters → use sort-only index
            return [self.index_map.get((sort_field,), "")]

        # Build index key from filter fields + sort field
        filter_fields = tuple(sorted(filters.keys()))

        # Try exact match
        if (sort_field, *filter_fields) in self.index_map:
            return [self.index_map[(sort_field, *filter_fields)]]

        # Try filter + sort combo
        if ("account_id" in filters and sort_field == "date"):
            return ["idx_transactions_account_date"]

        if ("category" in filters and sort_field == "date"):
            return ["idx_transactions_date_category"]

        # Default to sort-only index
        return [self.index_map.get((sort_field,), "idx_transactions_date_id")]

    def _encode_cursor(self, row: Dict[str, Any], sort_field: str) -> str:
        """Encode cursor (base64 JSON)."""
        cursor_data = {
            sort_field: row[sort_field],
            "id": row["id"]
        }
        json_str = json.dumps(cursor_data)
        return base64.b64encode(json_str.encode()).decode()

    def _decode_cursor(self, cursor: str) -> Dict[str, Any]:
        """Decode cursor."""
        try:
            json_str = base64.b64decode(cursor.encode()).decode()
            return json.loads(json_str)
        except Exception as e:
            raise ValueError(f"Invalid cursor: {e}")

    def has_next_page(self, rows: List[Dict], limit: int) -> Tuple[List[Dict], bool]:
        """
        Check if there are more pages.

        Args:
            rows: Result rows (should have fetched limit + 1)
            limit: Requested limit

        Returns:
            (trimmed_rows, has_next)
        """
        if len(rows) > limit:
            return (rows[:limit], True)
        else:
            return (rows, False)
```

**Usage Example (Cursor Pagination):**
```python
query_builder = TransactionQuery("transactions")

# Page 1
result = query_builder.build_query(
    filters={"category": "Food & Dining"},
    sort_field="date",
    sort_order="desc",
    cursor=None,  # First page
    limit=50
)

rows = db.execute(result.query, result.params)
trimmed_rows, has_next = query_builder.has_next_page(rows, limit=50)

# Generate cursor for next page
if has_next:
    last_row = trimmed_rows[-1]
    next_cursor = query_builder._encode_cursor(last_row, "date")

# Page 2
result = query_builder.build_query(
    filters={"category": "Food & Dining"},
    sort_field="date",
    sort_order="desc",
    cursor=next_cursor,  # Use cursor from page 1
    limit=50
)
```

**Index Hint Usage:**
```python
result = query_builder.build_query(
    filters={"account_id": "ACC_123", "date_from": "2024-11-01"},
    sort_field="date",
    sort_order="desc",
    limit=50
)

print(result.index_hints)
# ['idx_transactions_account_date']

# Database can use this hint:
# SELECT * FROM transactions USE INDEX (idx_transactions_account_date)
# WHERE account_id = 'ACC_123' AND date >= '2024-11-01'
# ORDER BY date DESC, id DESC
# LIMIT 51
```

**Performance Comparison:**
```
Dataset: 10M transactions

OFFSET Pagination (Bad):
- Page 1: 50ms
- Page 100: 500ms
- Page 1000: 5000ms (scans 50K rows to skip)

Cursor Pagination (Good):
- Page 1: 50ms
- Page 100: 50ms
- Page 1000: 50ms (constant time)
```

**Testing:**
- ✅ Unit tests for cursor encoding/decoding
- ✅ Integration tests with large dataset (1M rows)
- ✅ Performance benchmarks (cursor vs OFFSET)
- ✅ Index hint validation

**Monitoring:**
- Query latency by filter type (P50/P95/P99)
- Index hit rate (% queries using suggested index)
- Slow query log (> 100ms)

**Decision Context:**
- Platform has 100M+ transactions across all tenants
- Cursor pagination required for consistent performance
- Index hints reduce slow queries by 80%
- Opaque cursors prevent direct ID manipulation

---

## Version History

- **v1.0** (2025-05-23): Initial release
  - Cursor-based pagination
  - Multi-filter support
  - Index hints
  - SQL injection prevention
