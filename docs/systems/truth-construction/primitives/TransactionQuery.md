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

## Version History

- **v1.0** (2025-05-23): Initial release
  - Cursor-based pagination
  - Multi-filter support
  - Index hints
  - SQL injection prevention
