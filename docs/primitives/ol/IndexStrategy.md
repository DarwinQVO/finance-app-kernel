# IndexStrategy (OL Primitive)

## Definition

**IndexStrategy** analyzes query patterns and suggests optimal database indexes for performance. It provides a systematic approach to indexing based on actual usage patterns, not guesswork.

**Problem it solves:**
- Manual index creation is ad-hoc (miss important indexes, create unused ones)
- Query performance degrades as data grows (no indexes on filter columns)
- Complex queries need composite indexes (not obvious which columns to combine)
- Over-indexing slows writes (every index has cost)

**Solution:**
- Analyze filter/sort patterns from TransactionQuery
- Suggest composite indexes for common query combinations
- Prioritize by query frequency and performance impact
- Monitor index usage (drop unused indexes)

---

## Interface Contract

```python
from typing import List, Dict, Set
from dataclasses import dataclass

@dataclass
class IndexSuggestion:
    """Suggested index specification"""
    name: str                   # e.g., "idx_canonical_date_category"
    table: str                  # e.g., "canonical_transactions"
    columns: List[str]          # e.g., ["transaction_date", "category"]
    index_type: str             # "btree" | "gin" | "hash"
    where_clause: str | None    # Partial index condition (optional)
    estimated_impact: str       # "high" | "medium" | "low"
    reason: str                 # Why this index is needed

@dataclass
class QueryPattern:
    """Captured query pattern"""
    filters: Set[str]           # Columns used in WHERE
    sort: str                   # Column used in ORDER BY
    frequency: int              # How often this pattern is used

class IndexStrategy:
    """
    Index suggestion engine based on query patterns.

    Domain-agnostic - works with any schema.
    """

    def __init__(self, table: str):
        """
        Args:
            table: Table name to analyze (e.g., "canonical_transactions")
        """
        self.table = table
        self.query_patterns: List[QueryPattern] = []

    def record_query(self, filters: Set[str], sort: str) -> None:
        """
        Records a query pattern for analysis

        Example:
            strategy.record_query(
                filters={"transaction_date", "category"},
                sort="transaction_date"
            )

        Call this from TransactionQuery after each query.
        """

    def suggest_indexes(self) -> List[IndexSuggestion]:
        """
        Analyzes recorded patterns and suggests indexes

        Algorithm:
        1. Group queries by filter+sort combination
        2. Rank by frequency
        3. Suggest composite indexes for top patterns
        4. Suggest single-column indexes for common filters

        Returns:
            List of index suggestions, ordered by priority
        """

    def generate_sql(self, suggestion: IndexSuggestion) -> str:
        """
        Generates SQL DDL for creating index

        Example:
            suggestion = IndexSuggestion(
                name="idx_canonical_date_category",
                table="canonical_transactions",
                columns=["transaction_date", "category"],
                index_type="btree"
            )

            → "CREATE INDEX idx_canonical_date_category
               ON canonical_transactions (transaction_date DESC, category);"
        """
```

---

## Multi-Domain Applicability

**Universal Pattern:** Optimize queries with appropriate indexes

### Finance Domain
```python
strategy = IndexStrategy(table="canonical_transactions")

# Record typical queries
strategy.record_query(filters={"transaction_date"}, sort="transaction_date")
strategy.record_query(filters={"transaction_date", "category"}, sort="transaction_date")
strategy.record_query(filters={"account_id"}, sort="transaction_date")

suggestions = strategy.suggest_indexes()
# [
#   IndexSuggestion(
#     name="idx_canonical_date_category",
#     columns=["transaction_date", "category"],
#     estimated_impact="high",
#     reason="Used in 847 queries (42% of total)"
#   ),
#   IndexSuggestion(
#     name="idx_canonical_account_date",
#     columns=["account_id", "transaction_date"],
#     estimated_impact="medium",
#     reason="Used in 312 queries (15% of total)"
#   )
# ]
```

### Healthcare Domain
```python
strategy = IndexStrategy(table="canonical_lab_results")

strategy.record_query(filters={"test_date", "is_normal"}, sort="test_date")
strategy.record_query(filters={"patient_id"}, sort="test_date")

suggestions = strategy.suggest_indexes()
# → idx_lab_results_date_normal
# → idx_lab_results_patient_date
```

### Legal Domain
```python
strategy = IndexStrategy(table="canonical_contract_clauses")

strategy.record_query(filters={"effective_date", "risk_level"}, sort="effective_date")
strategy.record_query(filters={"counterparty"}, sort="effective_date")

suggestions = strategy.suggest_indexes()
# → idx_clauses_date_risk
# → idx_clauses_counterparty_date
```

### Research Domain
```python
strategy = IndexStrategy(table="canonical_citations")

strategy.record_query(filters={"publication_year", "author"}, sort="publication_year")
strategy.record_query(filters={"topic"}, sort="publication_year")

suggestions = strategy.suggest_indexes()
# → idx_citations_year_author
# → idx_citations_topic_year
```

### Manufacturing Domain
```python
strategy = IndexStrategy(table="canonical_qc_measurements")

strategy.record_query(filters={"measurement_date", "passed_qc"}, sort="measurement_date")
strategy.record_query(filters={"product_id"}, sort="measurement_date")

suggestions = strategy.suggest_indexes()
# → idx_qc_date_pass
# → idx_qc_product_date
```

### Media Domain
```python
strategy = IndexStrategy(table="canonical_transcript_snippets")

strategy.record_query(filters={"timestamp", "speaker_id"}, sort="timestamp")
strategy.record_query(filters={"topic"}, sort="timestamp")

suggestions = strategy.suggest_indexes()
# → idx_snippets_time_speaker
# → idx_snippets_topic_time
```

---

## Responsibilities

✅ **DOES:**
- Record query patterns (filters + sort)
- Analyze patterns to find common combinations
- Suggest composite indexes for multi-column queries
- Suggest single-column indexes for frequent filters
- Generate SQL DDL for creating indexes
- Estimate impact (high/medium/low) based on frequency
- Identify unused indexes (for cleanup)

---

## NOT Responsibilities

❌ **DOES NOT:**
- Execute DDL (only generates SQL, DBA/migration runs it)
- Monitor query performance (uses query logs, not EXPLAIN)
- Automatically create indexes (requires human review)
- Drop indexes (only suggests which are unused)
- Optimize queries (rewrites, not indexing)

---

## Implementation Notes

### Index Selection Algorithm

**Step 1: Capture patterns**
```python
# From query logs
patterns = [
    {"filters": {"date", "category"}, "sort": "date", "count": 847},
    {"filters": {"date"}, "sort": "date", "count": 623},
    {"filters": {"account"}, "sort": "date", "count": 312},
    {"filters": {"date", "category", "account"}, "sort": "date", "count": 156}
]
```

**Step 2: Rank by frequency**
```python
# Top patterns
1. date + category → 847 queries (42%)
2. date only → 623 queries (31%)
3. account → 312 queries (15%)
```

**Step 3: Suggest indexes**
```sql
-- Composite index (covers pattern 1)
CREATE INDEX idx_canonical_date_category
  ON canonical_transactions (transaction_date DESC, category);

-- Single-column index (covers pattern 2, part of pattern 1)
CREATE INDEX idx_canonical_date
  ON canonical_transactions (transaction_date DESC);

-- Account index (covers pattern 3)
CREATE INDEX idx_canonical_account_date
  ON canonical_transactions (account_id, transaction_date DESC);
```

**Step 4: Eliminate redundancy**
```
idx_canonical_date is redundant with idx_canonical_date_category
  → Remove idx_canonical_date (composite covers single-column)
```

---

### Composite Index Column Order

**Rule:** Most selective column first, sort column last

**Example:**
```sql
-- GOOD (category is selective, date for sorting)
CREATE INDEX idx_canonical_category_date
  ON canonical_transactions (category, transaction_date DESC);

-- BAD (date is not selective, poor filtering)
CREATE INDEX idx_canonical_date_category
  ON canonical_transactions (transaction_date DESC, category);
```

**Exception:** If sort is in WHERE clause, it goes first
```sql
-- Query: WHERE date >= '2025-04-01' ORDER BY date DESC
CREATE INDEX idx_canonical_date
  ON canonical_transactions (transaction_date DESC);
```

---

### Partial Indexes (Advanced)

**For queries with common filter values:**
```sql
-- Query: WHERE deductible = true (80% of queries filter this)
CREATE INDEX idx_canonical_deductible_date
  ON canonical_transactions (transaction_date DESC)
  WHERE deductible = true;

-- Smaller index, faster queries (only indexes 20% of rows)
```

---

### Index Usage Monitoring

**Detect unused indexes:**
```sql
-- PostgreSQL
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,  -- Number of scans
  idx_tup_read  -- Rows read
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Never used
  AND indexname NOT LIKE 'pk_%';  -- Exclude primary keys
```

**Flag for removal:**
```python
suggestion = IndexSuggestion(
    name="idx_canonical_old_unused",
    table="canonical_transactions",
    columns=[],
    index_type="btree",
    estimated_impact="negative",  # Slows writes
    reason="Unused for 90 days, 0 scans, recommend DROP"
)
```

---

## Related Primitives

**Uses:**
- Query logs (from TransactionQuery)
- Performance metrics (from observability)

**Used by:**
- **TransactionQuery** - Consults IndexStrategy for index hints
- Migration scripts - Use suggested indexes in DDL

---

## Version History

- **v1.0** (2025-05-23): Initial release
  - Pattern analysis
  - Composite index suggestions
  - SQL DDL generation
