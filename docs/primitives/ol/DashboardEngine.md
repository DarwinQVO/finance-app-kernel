# DashboardEngine (OL Primitive)

## Definition

**DashboardEngine** calculates aggregate metrics from canonical data (sum, count, group by, averages). It transforms raw transaction data into dashboard-ready metrics (income, expenses, category breakdowns, top merchants, etc.).

**Problem it solves:**
- Manual aggregation queries are repetitive and error-prone
- Different dashboards need similar aggregations (sum by category, count by merchant)
- Performance degrades without proper query optimization (N+1 problems)
- No standard way to handle multi-currency aggregations

**Solution:**
- Single entry point for all dashboard metric calculations
- Optimized SQL queries with proper indexes
- Parallel metric calculation (multiple metrics simultaneously)
- Timeout protection (kill slow queries after 5s)
- Multi-currency support (aggregate in base currency)

---

## Interface Contract

```python
from typing import Dict, List, Optional
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class MetricResult:
    """Result of a single metric calculation"""
    metric_type: str  # "income_summary", "expense_by_category", etc.
    value: Dict  # Metric-specific data structure
    execution_time_ms: int
    transaction_count: int

class DashboardEngine:
    """
    Calculate aggregate metrics from canonical data.

    Domain-agnostic - works on ANY canonical schema.
    """

    def calculate_metrics(
        self,
        view_config: dict,
        user_id: str,
        timeout_ms: int = 5000
    ) -> Dict[str, MetricResult]:
        """
        Calculate all metrics for a dashboard view.

        Args:
            view_config: Dashboard configuration (date_range, filters, widgets)
            user_id: Current user (for access control)
            timeout_ms: Kill queries after this timeout (default: 5s)

        Returns:
            Dict of {metric_type: MetricResult}

        Raises:
            MetricTimeoutError: If calculation exceeds timeout
            InvalidConfigError: If view_config is malformed
        """

    def calculate_income_summary(
        self,
        filters: dict,
        user_id: str
    ) -> MetricResult:
        """
        Calculate total income and breakdown by source.

        SQL: SELECT SUM(amount) WHERE amount > 0 GROUP BY merchant
        """

    def calculate_expense_by_category(
        self,
        filters: dict,
        user_id: str
    ) -> MetricResult:
        """
        Calculate total expenses and breakdown by category.

        SQL: SELECT category, SUM(amount), COUNT(*) WHERE amount < 0 GROUP BY category
        """

    def calculate_top_merchants(
        self,
        filters: dict,
        user_id: str,
        limit: int = 10
    ) -> MetricResult:
        """
        Calculate top N merchants by total spend.

        SQL: SELECT merchant, SUM(amount), COUNT(*) GROUP BY merchant ORDER BY SUM DESC LIMIT N
        """

    def calculate_deductible_summary(
        self,
        filters: dict,
        user_id: str
    ) -> MetricResult:
        """
        Calculate total deductible expenses (USA + Mexico).

        SQL: SELECT SUM(amount) WHERE deductible_usa=true OR deductible_mexico=true
        """

    def calculate_trend(
        self,
        filters: dict,
        user_id: str,
        granularity: str = "month"  # "day", "week", "month", "quarter"
    ) -> MetricResult:
        """
        Calculate income/expense trend over time.

        SQL: SELECT DATE_TRUNC(granularity, date), SUM(amount) GROUP BY truncated_date
        """
```

---

## Multi-Domain Applicability

### Finance Domain
```python
# Income summary
result = engine.calculate_income_summary(
    filters={"date_from": "2025-05-01", "date_to": "2025-05-31"},
    user_id="usr_eugenio"
)
# Returns: {"total": 9000.00, "breakdown": [{"source": "HubSpot", "amount": 5000}, ...]}

# Expense by category
result = engine.calculate_expense_by_category(filters={...}, user_id="...")
# Returns: {"total": 8247.00, "breakdown": [{"category": "Software", "amount": 2500}, ...]}
```

### Healthcare Domain
```python
# Average vitals
result = engine.calculate_metric(
    metric_type="avg_vitals",
    filters={"patient_id": "pat_123", "date_range": "last_30_days"},
    user_id="doc_smith"
)
# Returns: {"avg_glucose": 120, "avg_blood_pressure": "125/80"}

# Abnormal flag rate
result = engine.calculate_metric(
    metric_type="abnormal_rate",
    filters={"diagnosis": "diabetes"},
    user_id="doc_smith"
)
# Returns: {"total_tests": 450, "abnormal_count": 87, "abnormal_rate": 0.19}
```

### Legal Domain
```python
# Cases won/lost
result = engine.calculate_metric(
    metric_type="case_outcomes",
    filters={"attorney": "atty_jones", "year": 2025},
    user_id="atty_jones"
)
# Returns: {"won": 45, "lost": 12, "settled": 8, "win_rate": 0.75}

# Hours billed by client
result = engine.calculate_metric(
    metric_type="hours_by_client",
    filters={"quarter": "Q1_2025"},
    user_id="atty_jones"
)
# Returns: {"total_hours": 320, "breakdown": [{"client": "ACME Corp", "hours": 120}, ...]}
```

### Research Domain
```python
# Citation count
result = engine.calculate_metric(
    metric_type="citation_stats",
    filters={"author": "researcher_123", "year": 2025},
    user_id="researcher_123"
)
# Returns: {"total_citations": 450, "h_index": 18, "top_papers": [...]}

# Papers by topic
result = engine.calculate_metric(
    metric_type="papers_by_topic",
    filters={"year": 2025},
    user_id="researcher_123"
)
# Returns: {"total_papers": 12, "breakdown": [{"topic": "AI", "count": 7}, ...]}
```

### Manufacturing Domain
```python
# Defect rate
result = engine.calculate_metric(
    metric_type="defect_rate",
    filters={"product_line": "widgets", "month": "May 2025"},
    user_id="qc_manager"
)
# Returns: {"units_produced": 10000, "defects": 87, "defect_rate": 0.0087}

# Production by shift
result = engine.calculate_metric(
    metric_type="production_by_shift",
    filters={"week": "2025-W20"},
    user_id="qc_manager"
)
# Returns: {"total": 50000, "breakdown": [{"shift": "morning", "units": 18000}, ...]}
```

### Media Domain
```python
# Engagement metrics
result = engine.calculate_metric(
    metric_type="engagement_summary",
    filters={"content_type": "video", "month": "May 2025"},
    user_id="creator_123"
)
# Returns: {"total_views": 450000, "avg_watch_time": 180, "engagement_rate": 0.15}

# Top content
result = engine.calculate_metric(
    metric_type="top_content",
    filters={"quarter": "Q1_2025"},
    user_id="creator_123",
    limit=10
)
# Returns: {"top_clips": [{"title": "How to...", "views": 120000}, ...]}
```

---

## Responsibilities

**DashboardEngine is responsible for:**
- ✅ Calculating aggregate metrics (sum, count, group by, avg)
- ✅ Handling multi-currency aggregations (convert to base currency)
- ✅ Query optimization (use indexes, avoid N+1)
- ✅ Timeout protection (kill slow queries after threshold)
- ✅ Parallel metric calculation (when metrics are independent)
- ✅ Error handling (graceful degradation on timeout)

---

## NOT Responsibilities

**DashboardEngine is NOT responsible for:**
- ❌ Storing saved views (SavedViewStore's job)
- ❌ Rendering dashboard UI (DashboardGrid's job)
- ❌ Querying individual transactions (TransactionQuery's job)
- ❌ Exporting to PDF (SnapshotExporter's job)
- ❌ Access control beyond user_id filter (CanonicalStore handles row-level security)

---

## Implementation Notes

**Query Optimization:**
```sql
-- Income summary (single query, uses index on transaction_date)
SELECT
    SUM(CASE WHEN amount > 0 THEN amount_usd ELSE 0 END) as income,
    SUM(CASE WHEN amount < 0 THEN ABS(amount_usd) ELSE 0 END) as expenses,
    SUM(amount_usd) as net
FROM canonical_transactions
WHERE user_id = $1
  AND transaction_date >= $2
  AND transaction_date <= $3;

-- Expense by category (uses composite index on transaction_date + category)
SELECT
    category,
    SUM(ABS(amount_usd)) as total,
    COUNT(*) as count
FROM canonical_transactions
WHERE user_id = $1
  AND transaction_date >= $2
  AND transaction_date <= $3
  AND amount < 0
GROUP BY category
ORDER BY total DESC;

-- Top merchants (uses composite index on transaction_date + merchant)
SELECT
    merchant,
    SUM(ABS(amount_usd)) as total,
    COUNT(*) as count
FROM canonical_transactions
WHERE user_id = $1
  AND transaction_date >= $2
  AND transaction_date <= $3
  AND amount < 0
GROUP BY merchant
ORDER BY total DESC
LIMIT 10;
```

**Parallel Metric Calculation:**
```python
def calculate_metrics(self, view_config, user_id, timeout_ms=5000):
    """Calculate multiple metrics in parallel using thread pool"""
    from concurrent.futures import ThreadPoolExecutor, TimeoutError

    widget_types = [w["type"] for w in view_config["widgets"]]
    filters = self._build_filters(view_config)

    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = {
            executor.submit(self._calculate_single_metric, wtype, filters, user_id): wtype
            for wtype in widget_types
        }

        results = {}
        for future in futures:
            try:
                results[futures[future]] = future.result(timeout=timeout_ms/1000)
            except TimeoutError:
                results[futures[future]] = {"error": "timeout", "partial": True}

    return results
```

**Multi-Currency Handling:**
```python
# All aggregations use amount_usd (converted amount from canonical record)
# Canonical record stores both original amount + amount_usd
# Exchange rate stored in canonical record for traceability

SELECT SUM(amount_usd) FROM ...  # Always aggregate in USD
```

**Timeout Protection:**
```python
# PostgreSQL query timeout
SET statement_timeout = '5s';

# Python-level timeout (kill thread if DB timeout doesn't trigger)
result = future.result(timeout=5.0)
```

---

## Related Primitives

**Dependencies:**
- CanonicalStore (source of transaction data)
- TransactionQuery (reuses filter building logic)

**Used by:**
- API endpoint GET /api/dashboard/metrics
- SnapshotExporter (includes metrics in PDF export)

**Reuses:**
- None (primitive-level aggregation)
