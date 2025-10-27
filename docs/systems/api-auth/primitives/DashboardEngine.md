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

### Research Domain (RSRCH - Utilitario)
```python
# Fact count by founder
result = engine.calculate_metric(
    metric_type="fact_stats",
    filters={"subject_entity": "Sam Altman", "year": 2025},
    user_id="rsrch_analyst_001"
)
# Returns: {"total_facts": 127, "fact_types": {"investment": 45, "founding": 12, "employment": 70}, "top_sources": ["TechCrunch", "Lex Fridman Podcast"]}

# Facts by topic
result = engine.calculate_metric(
    metric_type="facts_by_topic",
    filters={"fact_type": "investment"},
    user_id="rsrch_analyst_001"
)
# Returns: {"total_facts": 89, "breakdown": [{"topic": "AI", "count": 34}, {"topic": "Energy", "count": 21}, ...]}
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

## Simplicity Profiles

### Personal Profile (30 LOC)

**Contexto del Usuario:**
El usuario tiene una aplicación personal de finanzas. El dashboard muestra 4 métricas fijas: ingreso total, gastos por categoría (5 categorías), top 5 comerciantes, balance actual. Las métricas están hardcodeadas en el código (no se necesita configuración). El usuario siempre ve el mismo dashboard (no hay personalización).

**Implementation:**
```python
# dashboard.py (Personal - 30 LOC)
from decimal import Decimal

DASHBOARD_METRICS = ["income", "expense_by_category", "top_merchants", "balance"]

def get_dashboard_data(db, date_range):
    """
    Calcular métricas hardcodeadas para dashboard personal.

    Métricas fijas: income, expense_by_category, top_merchants, balance
    """
    # Metric 1: Income total
    income = db.execute("""
        SELECT SUM(amount) FROM canonical_transactions
        WHERE amount > 0 AND transaction_date BETWEEN ? AND ?
    """, date_range).fetchone()[0] or Decimal(0)

    # Metric 2: Expenses by category (hardcoded 5 categories)
    expense_by_cat = db.execute("""
        SELECT category, SUM(amount)
        FROM canonical_transactions
        WHERE amount < 0 AND transaction_date BETWEEN ? AND ?
        GROUP BY category
    """, date_range).fetchall()

    # Metric 3: Top 5 merchants
    top_merchants = db.execute("""
        SELECT merchant, SUM(amount), COUNT(*)
        FROM canonical_transactions
        WHERE transaction_date BETWEEN ? AND ?
        GROUP BY merchant
        ORDER BY SUM(amount) ASC
        LIMIT 5
    """, date_range).fetchall()

    return {
        "income": float(income),
        "expense_by_category": dict(expense_by_cat),
        "top_merchants": top_merchants
    }
```

**Características Incluidas:**
- ✅ 4 métricas hardcodeadas (income, expense_by_category, top_merchants, balance)
- ✅ Agregaciones SQL simples (SUM, GROUP BY)
- ✅ Top N queries (LIMIT 5)

**Características NO Incluidas:**
- ❌ Configuración de métricas (YAGNI: siempre las mismas 4 métricas)
- ❌ Personalización por usuario (YAGNI: 1 usuario, 1 dashboard)
- ❌ A/B testing (YAGNI: no hay experimentos)
- ❌ Cache de métricas (YAGNI: query toma <50ms para 871 rows)
- ❌ Timeout protection (YAGNI: queries nunca exceden 100ms)
- ❌ Parallel execution (YAGNI: 4 queries secuenciales toman <200ms total)

**Configuración:**
```python
# No se necesita configuración
# Métricas hardcodeadas en código
```

**Performance:**
- Latency: <50ms por métrica, <200ms total (4 métricas secuenciales)
- Memory: ~1MB (resultados en memoria)
- Dependencies: 0 (solo SQLite built-in)

**Upgrade Triggers:**
- Si necesita >10 métricas diferentes → Small Business (YAML config)
- Si necesita dashboards personalizados por usuario → Small Business (role-based config)
- Si queries exceden 500ms → Small Business (caching layer)

---

### Small Business Profile (120 LOC)

**Contexto del Usuario:**
Firma de consultoría pequeña (10 empleados). Cada empleado ve un dashboard diferente según su rol: Admin ve todas las métricas (20 widgets), Analyst ve solo analytics (8 widgets), Viewer ve summary (4 widgets). Las métricas están configuradas en archivo YAML (no hardcoded). Algunas métricas tardan 200-300ms (necesitan cache).

**Implementation:**
```python
# dashboard.py (Small Business - 120 LOC)
import yaml
from typing import Dict, List
from decimal import Decimal

class DashboardEngine:
    def __init__(self, config_path: str, db):
        self.db = db
        # Load metrics config from YAML
        with open(config_path, 'r') as f:
            self.config = yaml.safe_load(f)
        self.metric_registry = self._build_registry()

    def _build_registry(self):
        """Build metric calculator registry from config."""
        return {
            "income": self.calc_income,
            "expense_by_category": self.calc_expense_by_category,
            "top_merchants": self.calc_top_merchants,
            "balance": self.calc_balance,
            "monthly_trend": self.calc_monthly_trend,
            "tax_deductible": self.calc_tax_deductible
        }

    def get_dashboard_for_role(self, role: str, date_range: tuple) -> Dict:
        """
        Calcular métricas según rol del usuario.

        Role config defines which metrics to show.
        """
        role_config = self.config["roles"][role]
        metrics_to_calc = role_config["metrics"]

        results = {}
        for metric_name in metrics_to_calc:
            calculator = self.metric_registry[metric_name]
            results[metric_name] = calculator(date_range)

        return results

    def calc_income(self, date_range):
        """Calculate total income."""
        result = self.db.execute("""
            SELECT SUM(amount) FROM canonical_transactions
            WHERE amount > 0 AND transaction_date BETWEEN ? AND ?
        """, date_range).fetchone()[0]
        return float(result or Decimal(0))

    def calc_expense_by_category(self, date_range):
        """Calculate expenses by category."""
        rows = self.db.execute("""
            SELECT category, SUM(amount), COUNT(*)
            FROM canonical_transactions
            WHERE amount < 0 AND transaction_date BETWEEN ? AND ?
            GROUP BY category
        """, date_range).fetchall()
        return {cat: {"total": float(total), "count": count} for cat, total, count in rows}

    # ... similar for other metrics
```

**Características Incluidas:**
- ✅ YAML configuration (6+ metric types)
- ✅ Role-based dashboards (Admin, Analyst, Viewer)
- ✅ Metric registry (pluggable calculators)
- ✅ Dynamic metric selection (basado en rol)

**Características NO Incluidas:**
- ❌ Database-backed config (YAML suficiente para 10 usuarios, 6 metric types)
- ❌ A/B testing (no hay experimentos todavía)
- ❌ Personalization per user (solo por rol, no individual)
- ❌ Parallel metric execution (sequential OK para <10 métricas)
- ❌ Redis cache (en-memory cache suficiente)

**Configuración:**
```yaml
dashboard_engine:
  roles:
    admin:
      metrics:
        - income
        - expense_by_category
        - top_merchants
        - balance
        - monthly_trend
        - tax_deductible
    analyst:
      metrics:
        - expense_by_category
        - monthly_trend
        - top_merchants
    viewer:
      metrics:
        - income
        - balance
```

**Performance:**
- Latency: <100ms por métrica (sin cache), <20ms (con in-memory cache)
- Memory: ~5MB (metric results cached)
- Dependencies: yaml (config loading)

**Upgrade Triggers:**
- Si necesita >20 metric types → Enterprise (database config)
- Si necesita A/B testing → Enterprise (experiment framework)
- Si queries exceden 1s → Enterprise (Redis cache, parallel execution)
- Si >50 usuarios → Enterprise (personalization per user)

---

### Enterprise Profile (600 LOC)

**Contexto del Usuario:**
Startup FinTech (1000 empleados, 10K clientes API). Dashboard altamente personalizado: cada usuario tiene configuración custom (widgets, layout, filters). A/B testing de nuevas métricas (feature flags). Cache Redis para métricas expensive (p95 <500ms SLA). Parallel execution de 20+ métricas simultáneas.

**Implementation:**
```python
# dashboard_engine.py (Enterprise - 600 LOC)
import asyncio
import redis
from typing import Dict, List, Optional
from dataclasses import dataclass

@dataclass
class MetricConfig:
    name: str
    calculator: str
    cache_ttl_seconds: int
    timeout_ms: int
    enabled: bool  # Feature flag for A/B testing

class DashboardEngine:
    def __init__(self, db, cache: redis.Redis):
        self.db = db
        self.cache = cache
        self.metric_registry = self._build_registry()

    async def calculate_metrics(
        self,
        user_id: str,
        date_range: tuple,
        timeout_ms: int = 5000
    ) -> Dict:
        """
        Calcular métricas con parallel execution, caching, timeout protection.

        Steps:
        1. Load user-specific dashboard config from database
        2. Check Redis cache for each metric
        3. For cache misses: Calculate in parallel (asyncio.gather)
        4. Apply timeout protection (kill slow queries after 5s)
        5. Cache results in Redis (TTL según config)
        6. Return aggregated results
        """
        # 1. Load user config from database
        user_config = await self._load_user_config(user_id)

        # 2. Filter metrics by feature flags (A/B testing)
        enabled_metrics = [
            m for m in user_config["metrics"]
            if self._is_metric_enabled(m, user_id)
        ]

        # 3. Check cache for each metric
        cache_keys = [f"metric:{user_id}:{m['name']}:{date_range}" for m in enabled_metrics]
        cached_results = self.cache.mget(cache_keys)

        # 4. Calculate cache misses in parallel
        tasks = []
        for i, metric_config in enumerate(enabled_metrics):
            if cached_results[i] is None:
                # Cache miss: Schedule calculation
                calculator = self.metric_registry[metric_config["calculator"]]
                task = asyncio.create_task(
                    asyncio.wait_for(
                        calculator(date_range, user_id),
                        timeout=metric_config["timeout_ms"] / 1000
                    )
                )
                tasks.append((metric_config, task))

        # 5. Await all calculations (parallel execution)
        results = {}
        for metric_config, task in tasks:
            try:
                result = await task
                results[metric_config["name"]] = result

                # 6. Cache result
                cache_key = f"metric:{user_id}:{metric_config['name']}:{date_range}"
                self.cache.setex(
                    cache_key,
                    metric_config["cache_ttl_seconds"],
                    json.dumps(result)
                )
            except asyncio.TimeoutError:
                # Metric timed out, log and skip
                logger.warning(f"Metric {metric_config['name']} timed out")

        return results

    async def _load_user_config(self, user_id: str) -> Dict:
        """
        Load user-specific dashboard config from database.

        Config includes:
        - Which metrics to show
        - Metric order/layout
        - Custom filters
        - Feature flag overrides (A/B testing)
        """
        row = await self.db.fetch_one("""
            SELECT config_json FROM dashboard_configs
            WHERE user_id = ?
        """, (user_id,))
        return json.loads(row["config_json"])

    def _is_metric_enabled(self, metric_config: Dict, user_id: str) -> bool:
        """
        Check if metric is enabled for user (A/B testing).

        Uses feature flag system to enable/disable metrics.
        Example: 50% of users see "investment_summary" metric (experiment).
        """
        metric_name = metric_config["name"]

        # Check global feature flag
        feature_flag = self.db.fetch_one("""
            SELECT enabled, rollout_percentage FROM feature_flags
            WHERE flag_name = ?
        """, (f"metric:{metric_name}",))

        if not feature_flag or not feature_flag["enabled"]:
            return False

        # Check if user is in rollout percentage
        user_hash = hash(user_id) % 100
        return user_hash < feature_flag["rollout_percentage"]
```

**Características Incluidas:**
- ✅ Database-backed configuration (user-specific dashboards)
- ✅ A/B testing (feature flags con rollout percentage)
- ✅ Redis caching (TTL configurable por métrica)
- ✅ Parallel metric execution (asyncio.gather)
- ✅ Timeout protection (kill slow queries después de 5s)
- ✅ Personalization per user (cada usuario tiene custom config)
- ✅ Metrics collection (Prometheus: metric_calculation_latency_ms)

**Características NO Incluidas:**
- ❌ Real-time updates (polling es suficiente, no WebSockets)
- ❌ ML-powered recommendations (separado en RecommendationEngine)

**Configuración:**
```yaml
dashboard_engine:
  database:
    url: "postgresql://user:pass@host:5432/db"
    pool_size: 20
  cache:
    redis_url: "redis://localhost:6379"
    default_ttl_seconds: 300  # 5 minutes
  timeouts:
    default_metric_timeout_ms: 5000
    max_parallel_metrics: 50
  feature_flags:
    enabled: true
    default_rollout_percentage: 100
```

**Performance:**
- Latency: <50ms p95 (cache hit), <500ms p95 (cache miss, parallel execution)
- Memory: ~200MB Redis cache (10K users × 20 metrics)
- Throughput: 10K dashboard requests/sec (single instance)
- Dependencies: redis, asyncio, postgresql

**No Further Tiers:**
- Scaling beyond Enterprise es horizontal (más instancias, load balancer)

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
