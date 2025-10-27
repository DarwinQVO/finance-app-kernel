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

### Research Domain (RSRCH - Utilitario)
```python
strategy = IndexStrategy(table="canonical_facts")

strategy.record_query(filters={"discovered_at", "subject_entity"}, sort="discovered_at")
strategy.record_query(filters={"fact_type"}, sort="discovered_at")

suggestions = strategy.suggest_indexes()
# → idx_facts_date_entity
# → idx_facts_type_date
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

## Simplicity Profiles

### Personal Profile (0 LOC)

**Contexto del Usuario:**
Aplicación personal con 871 transacciones (<1K). SQLite table-scan toma 8ms para todo el dataset. No necesita indexes adicionales más allá del primary key.

**Implementation:**
```python
# No implementation needed (0 LOC)
# SQLite automatically uses primary key (rowid)
# Table-scan 871 rows: ~8ms (acceptable performance)

# Example query - no indexes needed:
conn.execute("""
    SELECT * FROM canonical_transactions
    WHERE transaction_date >= '2025-04-01'
    ORDER BY transaction_date DESC
""")
# Execution time: 8ms (full table scan)
```

**Características Incluidas:**
- ✅ Primary key index (automático en SQLite)

**Características NO Incluidas:**
- ❌ IndexStrategy (YAGNI: table-scan 8ms es suficiente para <1K rows)
- ❌ Composite indexes (YAGNI: 1 query pattern simple)
- ❌ Index analysis (YAGNI: performance aceptable)

**Configuración:**
```python
# No config needed
```

**Performance:**
- Query time: 8ms (table-scan 871 rows)
- Insert time: 2ms (no index overhead)

**Upgrade Triggers:**
- Si queries >100ms → Small Business (agregar indexes)
- Si data >10K rows → Small Business (index strategy)
- Si >3 query patterns → Small Business (composite indexes)

---

### Small Business Profile (100 LOC)

**Contexto del Usuario:**
Firma con 45K transacciones (crece 3K/mes). Queries toman 200-500ms (lag notorio). Necesitan análisis sistemático de indexes basado en query logs.

**Implementation:**
```python
# scripts/analyze_indexes.py (Small Business - 100 LOC)
import sqlite3, re
from collections import Counter

class IndexStrategy:
    def __init__(self, db_path: str, log_path: str):
        self.db_path = db_path
        self.log_path = log_path

    def analyze_query_log(self):
        """Parse query log, extract WHERE/ORDER BY patterns."""
        patterns = []
        with open(self.log_path, "r") as f:
            for line in f:
                # Extract WHERE columns
                where_cols = re.findall(r'(\w+)\s*[=<>]', line)
                # Extract ORDER BY column
                order_match = re.search(r'ORDER BY (\w+)', line)
                patterns.append({
                    "filters": set(where_cols),
                    "sort": order_match.group(1) if order_match else None
                })
        return patterns

    def suggest_indexes(self, patterns):
        """Suggest top 3 indexes by frequency."""
        pattern_counts = Counter()
        for p in patterns:
            key = (frozenset(p["filters"]), p["sort"])
            pattern_counts[key] += 1

        suggestions = []
        for (filters, sort_col), count in pattern_counts.most_common(3):
            cols = list(filters)
            if sort_col and sort_col not in cols:
                cols.append(sort_col)

            suggestions.append({
                "name": f"idx_{'_'.join(cols)}",
                "sql": f"CREATE INDEX idx_{'_'.join(cols)} ON transactions ({', '.join(cols)});"
            })
        return suggestions

# Weekly workflow:
strategy = IndexStrategy("data.db", "logs/queries.log")
patterns = strategy.analyze_query_log()
suggestions = strategy.suggest_indexes(patterns)
# DBA reviews, runs SQL manually
```

**Características Incluidas:**
- ✅ Query log analysis (regex extraction)
- ✅ Pattern frequency ranking
- ✅ Top 3 index suggestions
- ✅ Manual review workflow

**Características NO Incluidas:**
- ❌ Automated index creation (manual approval required)
- ❌ PostgreSQL pg_stat_statements (SQLite query logs)
- ❌ CI/CD integration (weekly manual process)

**Configuración:**
```yaml
index_strategy:
  db_path: "data/rsrch.db"
  log_path: "logs/queries.log"
  review_frequency: "weekly"
```

**Performance:**
- Before indexes: 450ms queries
- After indexes: 35ms (12x improvement)
- Insert overhead: 2ms → 3ms

**Upgrade Triggers:**
- Si >100K rows → Enterprise (pg_stat_statements)
- Si >10 query patterns → Enterprise (automated analysis)
- Si necesita CI/CD → Enterprise (automated PRs)

---

### Enterprise Profile (700 LOC)

**Contexto del Usuario:**
FinTech con 8.5M transacciones (crece 200K/mes). Cientos de query patterns, SLA p95 <500ms. Análisis automático con pg_stat_statements, CI/CD integration para crear PRs semanales con index suggestions.

**Implementation:**
```python
# lib/index_strategy.py (Enterprise - 700 LOC)
import psycopg2
from dataclasses import dataclass

class IndexStrategy:
    def __init__(self, conn, table: str):
        self.conn = conn
        self.table = table

    def analyze_pg_stat_statements(self, hours: int = 168):
        """Query pg_stat_statements for patterns (past week)."""
        cursor = self.conn.cursor()
        cursor.execute(f"""
            SELECT query, calls, mean_exec_time
            FROM pg_stat_statements
            WHERE query LIKE '%{self.table}%'
              AND last_exec > NOW() - INTERVAL '{hours} hours'
            ORDER BY calls DESC LIMIT 500
        """)

        patterns = []
        for query_text, calls, avg_time in cursor.fetchall():
            filters = self._extract_where_columns(query_text)
            sort = self._extract_order_by_column(query_text)
            patterns.append({"filters": filters, "sort": sort, "count": calls, "avg_time_ms": avg_time})
        return patterns

    def suggest_indexes(self, patterns, min_query_count=100):
        """
        Rank by total_time (count × avg_time), eliminate redundant.
        Returns IndexSuggestion list.
        """
        # Group by (filters, sort), rank by total time
        pattern_stats = {}
        for p in patterns:
            key = (frozenset(p["filters"]), p["sort"])
            if key not in pattern_stats:
                pattern_stats[key] = {"count": 0, "total_time": 0}
            pattern_stats[key]["count"] += p["count"]
            pattern_stats[key]["total_time"] += p["count"] * p["avg_time_ms"]

        # Top suggestions (high impact: >100s total time)
        suggestions = []
        for (filters, sort_col), stats in sorted(pattern_stats.items(), key=lambda x: x[1]["total_time"], reverse=True):
            if stats["count"] < min_query_count:
                continue
            columns = list(filters) + ([sort_col] if sort_col and sort_col not in filters else [])
            suggestions.append({
                "name": f"idx_{self.table}_{'_'.join(columns)}",
                "columns": columns,
                "impact": "high" if stats["total_time"] > 100000 else "medium",
                "reason": f"{stats['count']} queries/week, {stats['total_time']/1000:.1f}s total"
            })

        return suggestions

    def detect_unused_indexes(self, min_days=90):
        """Find indexes with 0 scans in N days."""
        cursor = self.conn.cursor()
        cursor.execute(f"""
            SELECT indexname FROM pg_stat_user_indexes
            WHERE tablename = '{self.table}' AND idx_scan = 0
        """)
        return [row[0] for row in cursor.fetchall()]

    def generate_migration_sql(self, suggestions):
        """CREATE INDEX CONCURRENTLY (no table lock)."""
        sql = f"-- Migration: Add {len(suggestions)} indexes\n\n"
        for s in suggestions:
            sql += f"CREATE INDEX CONCURRENTLY {s['name']} ON {self.table} ({', '.join(s['columns'])});\n"
        return sql

# CI/CD: Weekly GitHub Actions cron job
# Runs analyze_pg_stat_statements(), creates PR with migration SQL
```

**Características Incluidas:**
- ✅ pg_stat_statements analysis (past week queries)
- ✅ Composite index suggestions (filters + sort)
- ✅ Impact estimation (high/medium by total time)
- ✅ Unused index detection (0 scans in 90d)
- ✅ CI/CD integration (weekly PRs)
- ✅ CREATE INDEX CONCURRENTLY (no locks)

**Características NO Incluidas:**
- ❌ Real-time analysis (weekly cron suficiente)
- ❌ Automatic index creation (DBA review required)

**Configuración:**
```yaml
index_strategy:
  database:
    url: "postgresql://host:5432/db"
  pg_stat_statements:
    analysis_hours: 168  # Past week
    min_query_count: 100
  ci_cd:
    schedule: "0 2 * * 1"  # Mondays 2 AM
    create_pr: true
```

**Performance:**
- Before: p95=1850ms (seq scan 8.5M rows)
- After: p95=120ms (index scan 12K rows)
- Impact: 18x faster (850ms → 45ms)
- Unused index cleanup: INSERT 8ms → 5ms

**No Further Tiers:**
- Scale horizontally (read replicas)

---

### Comparison Table

| Feature | Personal (Darwin) | Small Business | Enterprise |
|---------|------------------|----------------|------------|
| **LOC** | 0 (Not Needed) | ~100 | ~700 |
| **Data Volume** | 871 rows | 45K rows | 8.5M rows |
| **Query Time** | 8ms (acceptable) | 200-500ms (noticeable) | 850ms → 45ms |
| **Index Count** | 1 (primary key) | 3 (manual) | 12 (automated) |
| **Analysis** | None (not needed) | Script parses logs | pg_stat_statements |
| **Suggestions** | None | Weekly manual review | Automated PR |
| **Unused Detection** | N/A | Manual EXPLAIN | pg_stat_user_indexes |
| **CI/CD Integration** | No | No | Yes (GitHub Actions) |
| **Index Creation** | N/A | Manual SQL | CREATE INDEX CONCURRENTLY |

**Key Insight:**
- **Darwin (0 LOC)**: Table-scan 871 rows in 8ms - indexes would slow writes for no benefit
- **Small Business (100 LOC)**: Systematic analysis prevents ad-hoc indexing mistakes
- **Enterprise (700 LOC)**: Automated analysis required due to hundreds of query patterns

---

## Version History

- **v1.0** (2025-05-23): Initial release
  - Pattern analysis
  - Composite index suggestions
  - SQL DDL generation
