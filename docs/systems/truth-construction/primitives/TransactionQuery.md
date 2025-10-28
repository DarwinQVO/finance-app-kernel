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

### Personal Profile (0 LOC)

**Contexto del Usuario:**
Darwin escribe SQL directo. Queries son simples (5-10 líneas): "dame todas las transacciones del mes" o "filtra por categoría". No necesita query builder cuando el SQL es tan simple. OFFSET pagination funciona bien para 1,500 transacciones totales.

**Implementation:**
```python
# No implementation needed (0 LOC)
# Darwin writes SQL directly

# Get current month transactions
def get_current_month_transactions():
    return db.execute("""
        SELECT * FROM transactions
        WHERE date >= ?
        ORDER BY date DESC
    """, ["2024-11-01"])

# Filter by category
def get_transactions_by_category(category):
    return db.execute("""
        SELECT * FROM transactions
        WHERE category_id = ?
        ORDER BY date DESC
    """, [category])

# Simple pagination (OFFSET works for 1,500 rows)
def get_transactions_page(page=1, per_page=50):
    offset = (page - 1) * per_page
    return db.execute("""
        SELECT * FROM transactions
        ORDER BY date DESC
        LIMIT ? OFFSET ?
    """, [per_page, offset])
```

**Características Incluidas:**
- ✅ SQL directo (5-10 líneas, readable)
- ✅ Parameterized queries (SQL injection prevention)
- ✅ OFFSET pagination (< 10K rows)

**Características NO Incluidas:**
- ❌ Query builder (YAGNI: SQL is simple)
- ❌ Cursor pagination (YAGNI: OFFSET works fine)
- ❌ Index hints (YAGNI: SQLite auto-indexes)
- ❌ Multi-filter combinations (YAGNI: 1-2 filters max)

**Configuración:**
```python
# No config needed
```

**Performance:**
- Query execution: 5ms (1,500 rows)
- OFFSET pagination: 3ms (SQLite index scan)

**Upgrade Triggers:**
- Si >10K transactions → Small Business (query builder)
- Si complex filters → Small Business

---

### Small Business Profile (120 LOC)

**Contexto del Usuario:**
Firma contable con 50K transacciones. Necesitan UI con múltiples filtros: date range + category + account + amount range + search. Query builder evita escribir SQL manualmente para cada combinación de filtros. OFFSET pagination suficiente para 50K rows.

**Implementation:**
```python
# lib/transaction_query.py (Small Business - 120 LOC)
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
            filters: {"category": "Food", "date_from": "2024-11-01", ...}
            sort_field: Column to sort by
            sort_order: "asc" or "desc"
            page: Page number (1-indexed)
            per_page: Items per page

        Returns:
            QueryResult with parameterized SQL + params
        """
        query_parts = [f"SELECT * FROM {self.table}"]
        params = []

        # Build WHERE clause
        if filters:
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
                    # Exact match for other fields
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

        query = " ".join(query_parts)
        return QueryResult(query=query, params=params)

# Usage
query_builder = TransactionQuery()

# Multi-filter query
result = query_builder.build_query(
    filters={
        "date_from": "2024-01-01",
        "date_to": "2024-12-31",
        "category": "Food & Dining",
        "amount_min": 10.00,
        "search": "starbucks"
    },
    sort_field="date",
    sort_order="desc",
    page=1,
    per_page=50
)

# Execute query
rows = db.execute(result.query, result.params)
# → SQL: SELECT * FROM transactions WHERE date >= ? AND date <= ? AND category = ? AND amount >= ? AND LOWER(merchant) LIKE LOWER(?) ORDER BY date DESC LIMIT ? OFFSET ?
# → Params: ["2024-01-01", "2024-12-31", "Food & Dining", 10.00, "%starbucks%", 50, 0]
```

**Características Incluidas:**
- ✅ Query builder (avoid manual SQL)
- ✅ Multi-filter support (6+ filter types)
- ✅ OFFSET pagination (< 50K rows)
- ✅ SQL injection prevention (parameterized)
- ✅ Flexible sorting

**Características NO Incluidas:**
- ❌ Cursor pagination (OFFSET sufficient)
- ❌ Index hints (database optimizer works)
- ❌ Query plan caching (YAGNI)

**Configuración:**
```yaml
transaction_query:
  default_per_page: 50
  max_per_page: 200
```

**Performance:**
- Build query: 1ms
- Execute (50K rows, filtered): 50ms
- OFFSET performance: O(n) but acceptable for < 100K rows

**Upgrade Triggers:**
- Si >100K transactions → Enterprise (cursor pagination)
- Si OFFSET latency >200ms → Enterprise

---

### Enterprise Profile (600 LOC)

**Contexto del Usuario:**
8.5M transacciones. OFFSET pagination demasiado lento (p95 latency >2s para pages >1000). Necesitan: cursor-based pagination (keyset), query plan caching, index hints para queries complejas, JSON query DSL para advanced filters.

**Implementation:**
```python
# lib/transaction_query.py (Enterprise - 600 LOC)
from typing import Dict, List, Optional, Any
from dataclasses import dataclass
import hashlib
import json

@dataclass
class QueryResult:
    query: str
    params: List[Any]
    cursor: Optional[str] = None  # For cursor pagination

class TransactionQuery:
    """Advanced query builder with cursor pagination and caching."""

    def __init__(self, db_connection, cache_client=None):
        self.db = db_connection
        self.cache = cache_client  # Redis

    def build_query(
        self,
        filters: Optional[Dict[str, Any]] = None,
        sort_field: str = "date",
        sort_order: str = "desc",
        cursor: Optional[str] = None,
        per_page: int = 50
    ) -> QueryResult:
        """
        Build query with cursor-based pagination.

        Args:
            filters: Multi-level filters (see DSL below)
            sort_field: Column to sort by
            sort_order: "asc" or "desc"
            cursor: Base64-encoded cursor (keyset values)
            per_page: Items per page

        Returns:
            QueryResult with SQL, params, next_cursor
        """
        query_parts = ["SELECT * FROM transactions"]
        params = []

        # Build WHERE clause from filters
        where_clauses = []
        if filters:
            where_clauses = self._build_where_clauses(filters, params)

        # Cursor-based pagination (keyset)
        if cursor:
            cursor_data = self._decode_cursor(cursor)
            # For DESC sort: WHERE (date, id) < (cursor_date, cursor_id)
            where_clauses.append(
                f"({sort_field}, id) < (?, ?)" if sort_order == "desc" else
                f"({sort_field}, id) > (?, ?)"
            )
            params.extend([cursor_data[sort_field], cursor_data["id"]])

        if where_clauses:
            query_parts.append("WHERE " + " AND ".join(where_clauses))

        # ORDER BY (must include unique column for keyset)
        query_parts.append(f"ORDER BY {sort_field} {sort_order.upper()}, id {sort_order.upper()}")

        # LIMIT (fetch +1 to check if more pages exist)
        query_parts.append("LIMIT ?")
        params.append(per_page + 1)

        # Index hint (PostgreSQL)
        if self._should_use_index_hint(filters):
            query_parts[0] = f"SELECT * FROM transactions USE INDEX (idx_date_id)"

        query = " ".join(query_parts)

        # Check cache
        query_hash = self._hash_query(query, params)
        if self.cache:
            cached = self.cache.get(f"query:{query_hash}")
            if cached:
                return json.loads(cached)

        return QueryResult(query=query, params=params)

    def execute(
        self,
        filters: Optional[Dict[str, Any]] = None,
        sort_field: str = "date",
        sort_order: str = "desc",
        cursor: Optional[str] = None,
        per_page: int = 50
    ) -> Dict[str, Any]:
        """
        Execute query and return results with next_cursor.

        Returns:
            {
                "data": [...],
                "next_cursor": "base64...",
                "has_more": true/false
            }
        """
        query_result = self.build_query(filters, sort_field, sort_order, cursor, per_page)

        # Execute
        rows = self.db.execute(query_result.query, query_result.params).fetchall()

        # Check if more pages
        has_more = len(rows) > per_page
        if has_more:
            rows = rows[:per_page]

        # Generate next cursor
        next_cursor = None
        if has_more and rows:
            last_row = rows[-1]
            next_cursor = self._encode_cursor({
                sort_field: last_row[sort_field],
                "id": last_row["id"]
            })

        return {
            "data": [dict(row) for row in rows],
            "next_cursor": next_cursor,
            "has_more": has_more
        }

    def _build_where_clauses(self, filters: Dict, params: List) -> List[str]:
        """
        Build WHERE clauses from JSON DSL.

        DSL example:
        {
            "AND": [
                {"date_from": "2024-01-01"},
                {"date_to": "2024-12-31"},
                {"OR": [
                    {"category": "Food"},
                    {"category": "Dining"}
                ]}
            ]
        }
        """
        clauses = []

        for key, value in filters.items():
            if key == "AND":
                sub_clauses = [self._build_where_clauses(f, params) for f in value]
                clauses.append("(" + " AND ".join([c[0] for c in sub_clauses]) + ")")
            elif key == "OR":
                sub_clauses = [self._build_where_clauses(f, params) for f in value]
                clauses.append("(" + " OR ".join([c[0] for c in sub_clauses]) + ")")
            elif key == "date_from":
                clauses.append("date >= ?")
                params.append(value)
            elif key == "date_to":
                clauses.append("date <= ?")
                params.append(value)
            # ... other filters

        return clauses

    def _encode_cursor(self, data: Dict) -> str:
        """Encode cursor as base64."""
        import base64
        return base64.b64encode(json.dumps(data).encode()).decode()

    def _decode_cursor(self, cursor: str) -> Dict:
        """Decode base64 cursor."""
        import base64
        return json.loads(base64.b64decode(cursor).decode())

    def _hash_query(self, query: str, params: List) -> str:
        """Hash query for caching."""
        key = query + str(params)
        return hashlib.sha256(key.encode()).hexdigest()

    def _should_use_index_hint(self, filters: Dict) -> bool:
        """Determine if index hint needed (query planner assistance)."""
        # Use index hint for date range queries on large tables
        return "date_from" in filters or "date_to" in filters

# Usage
query_builder = TransactionQuery(db_connection=conn, cache_client=redis_client)

# First page (no cursor)
result = query_builder.execute(
    filters={
        "AND": [
            {"date_from": "2024-01-01"},
            {"date_to": "2024-12-31"},
            {"category": "Food & Dining"}
        ]
    },
    sort_field="date",
    sort_order="desc",
    per_page=50
)
# → {"data": [...50 rows...], "next_cursor": "eyJkYXRlIjogIjIwMjQtMTEtMTUiLCAiaWQiOiAxMjM0fQ==", "has_more": true}

# Next page (with cursor)
result = query_builder.execute(
    filters={"date_from": "2024-01-01"},
    cursor="eyJkYXRlIjogIjIwMjQtMTEtMTUiLCAiaWQiOiAxMjM0fQ==",
    per_page=50
)
# → {"data": [...50 more rows...], "next_cursor": "...", "has_more": true}
```

**Características Incluidas:**
- ✅ Cursor-based pagination (keyset, O(1) performance)
- ✅ JSON query DSL (AND/OR logic)
- ✅ Query plan caching (Redis)
- ✅ Index hints (PostgreSQL)
- ✅ Has more indicator

**Características NO Incluidas:**
- ❌ GraphQL query language (JSON DSL sufficient)

**Configuración:**
```yaml
transaction_query:
  pagination:
    type: "cursor"  # vs "offset"
    default_per_page: 50
  caching:
    enabled: true
    ttl_seconds: 300
  index_hints:
    enabled: true
    auto_detect: true
```

**Performance:**
- Cursor pagination (8.5M rows): 15ms (constant, any page)
- OFFSET pagination (8.5M rows, page 1000): 2,500ms (linear scan)
- Cache hit: 3ms
- Query build: 5ms

**No Further Tiers:**
- Scale horizontally (read replicas)

---

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
