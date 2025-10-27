# OL Primitive: NormalizationLog

**Type**: Observability / Structured Log
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.3 (Normalization)

---

## Purpose

Structured execution log for normalization operations. Records validation results (successes, failures, warnings), performance metrics, and applied rules. Enables debugging, quality monitoring, and audit trails for normalization pipelines.

---

## Schema

See [`normalization-log.schema.json`](../../schemas/normalization-log.schema.json) for complete schema.

**Key fields:**
```typescript
interface NormalizationLog {
  upload_id: string
  normalizer_version: string
  rule_set_version: string
  observations_processed: number
  canonicals_created: number
  failures: number
  warnings: number
  failure_details: Array<{observation_id, field, error}>
  warning_details: Array<{canonical_id, flag, message}>
  duration_ms: number
  started_at: ISO8601DateTime
  completed_at: ISO8601DateTime
}
```

---

## Behavior

**Success case (all observations normalized):**
```json
{
  "upload_id": "UL_abc123",
  "normalizer_version": "1.0.0",
  "rule_set_version": "2024.10",
  "observations_processed": 100,
  "canonicals_created": 100,
  "failures": 0,
  "warnings": 8,
  "duration_ms": 5120,
  "warning_details": [
    {"canonical_id": "CT_xyz", "flag": "possible_duplicate", "message": "Similar transaction found"}
  ]
}
```

**Partial success (some observations failed):**
```json
{
  "upload_id": "UL_def456",
  "normalizer_version": "1.0.0",
  "rule_set_version": "2024.10",
  "observations_processed": 100,
  "canonicals_created": 95,
  "failures": 5,
  "warnings": 12,
  "failure_details": [
    {"observation_id": "UL_def456:42", "field": "date", "error": "Unparseable date format"}
  ],
  "duration_ms": 6200
}
```

---

## Storage

**Path:** `/logs/normalize/{upload_id}.log.json`
**Persistence:** Write-once, immutable

---

## Multi-Domain Applicability

**Finance:** Log transaction normalization (validation failures, duplicate rate)
**Healthcare:** Log lab result normalization (unit conversion errors, range violations)
**Legal:** Log clause normalization (categorization accuracy, parsing issues)
**Research (RSRCH - Utilitario):** Log fact normalization (entity name resolution errors, missing source credibility, '@sama' → 'Sam Altman' transformations)
**Manufacturing:** Log measurement normalization (calibration errors, out-of-range values)

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Log transaction normalization execution for quality monitoring
**Example:** Normalizer processes 100 raw transactions → NormalizationLog records {upload_id: "UL_abc123", observations_processed: 100, canonicals_created: 100, failures: 0, warnings: 8, duration_ms: 5120, warning_details: [{canonical_id: "CT_xyz", flag: "possible_duplicate"}]} → Quality dashboard queries logs → Calculates normalization success rate (100%), warning rate (8%), p95 latency (6.1s) → Alert if failures > 5%
**Operations:** Write immutable log, query for quality metrics (success rate, latency, failure types), track warning trends (duplicate rate increasing = potential data quality issue)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Log lab result normalization for error tracking and compliance
**Example:** Normalizer processes 50 lab observations → 45 succeed, 5 fail → NormalizationLog records {observations_processed: 50, canonicals_created: 45, failures: 5, failure_details: [{observation_id: "obs_42", field: "unit", error: "Unknown unit 'mmol' for test 'Glucose'"}]} → Admin checks log → Sees unit conversion errors → Adds mmol→mg/dL conversion rule
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Log clause normalization for audit trail (compliance requirement)
**Example:** Normalizer categorizes 25 contract clauses → NormalizationLog records {observations_processed: 25, canonicals_created: 25, rule_set_version: "2024.10", warnings: 3, warning_details: [{canonical_id: "clause_12", flag: "low_confidence", message: "Clause category unclear (0.65 confidence)"}]} → Audit requires proof of categorization → Log provides immutable record with rule version, timestamps
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Log fact normalization for entity resolution tracking and data quality
**Example:** Normalizer processes 50 raw founder facts from TechCrunch → Resolves entity names → NormalizationLog records {upload_id: "UL_tc_123", observations_processed: 50, canonicals_created: 48, failures: 2, warnings: 12, failure_details: [{observation_id: "obs_23", field: "subject_entity", error: "Cannot resolve 'Sammy A.' to canonical founder name"}], warning_details: [{canonical_id: "fact_5", flag: "low_confidence_entity_match", message: "'Sam Altman' → '@sama' confidence 0.72"}], duration_ms: 3200} → Dashboard tracks entity resolution accuracy (96% success rate), low confidence matches (24% of facts), normalization latency (p95: 4.5s)
**Operations:** Logs track entity resolution quality (success rate, confidence scores), failure details guide manual review (unresolved entities queued for human labeling), warning trends signal data quality issues (low confidence matches increasing = need better entity rules)
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Log product normalization for catalog quality monitoring
**Example:** Normalizer processes 1,500 product observations → 1,480 succeed, 20 fail → NormalizationLog records {observations_processed: 1500, canonicals_created: 1480, failures: 20, failure_details: [{observation_id: "prod_342", field: "category", error: "Invalid category 'Electronics > Phones > iPhone Pro Max' (too many levels)"}]} → Alert fires if failure rate > 2% → Admin reviews failed products, fixes category taxonomy
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (structured execution log for validation pipeline, no domain-specific code)
**Reusability:** High (same log schema works for transaction normalization, lab test normalization, clause categorization, fact entity resolution, product categorization; only failure/warning types differ)

---

## Simplicity Profiles

### Personal Profile (60 LOC)

**Contexto del Usuario:**
Darwin procesa 12 uploads al año (1 upload mensual ×12 meses). Necesita saber si la normalización funcionó, cuántas observaciones se procesaron, y qué errores ocurrieron para revisión manual. JSON simple en disco local es suficiente.

**Implementation:**
```python
# lib/normalization_log.py (Personal - 60 LOC)
import json
from datetime import datetime
from pathlib import Path

class NormalizationLog:
    """Simple JSON file logging for normalization execution."""

    def __init__(self, base_path="~/.finance-app/logs/normalize"):
        self.base_path = Path(base_path).expanduser()
        self.base_path.mkdir(parents=True, exist_ok=True)

    def write(self, upload_id: str, log_entry: dict):
        """Write normalization log to JSON file."""
        log_path = self.base_path / f"{upload_id}.log.json"

        # Add timestamps
        if "started_at" not in log_entry:
            log_entry["started_at"] = datetime.now().isoformat()
        if "completed_at" not in log_entry:
            log_entry["completed_at"] = datetime.now().isoformat()

        # Write (pretty-printed for readability)
        with open(log_path, "w") as f:
            json.dump(log_entry, f, indent=2)

    def read(self, upload_id: str) -> dict:
        """Read normalization log for specific upload."""
        log_path = self.base_path / f"{upload_id}.log.json"
        if not log_path.exists():
            return None
        with open(log_path, "r") as f:
            return json.load(f)

# Usage
log = NormalizationLog()

# Normalize observations
start = datetime.now()
results = process_all_observations(upload_id)
duration_ms = int((datetime.now() - start).total_seconds() * 1000)

# Write log
log.write(upload_id, {
    "upload_id": upload_id,
    "normalizer_version": "1.0.0",
    "observations_processed": results["total"],
    "canonicals_created": results["success"],
    "failures": results["failures"],
    "warnings": results["warnings"],
    "failure_details": results["error_list"],
    "duration_ms": duration_ms
})
```

**Log File Example:**
```json
{
  "upload_id": "UL_abc123",
  "normalizer_version": "1.0.0",
  "observations_processed": 42,
  "canonicals_created": 40,
  "failures": 2,
  "warnings": 5,
  "failure_details": [
    {"row_id": "OBS_row_15", "reason": "Invalid amount: 'N/A'"},
    {"row_id": "OBS_row_23", "reason": "Missing merchant name"}
  ],
  "warnings": [
    {"row_id": "OBS_row_7", "reason": "Low confidence merchant match: 0.72"}
  ],
  "duration_ms": 450,
  "started_at": "2025-10-27T10:15:00",
  "completed_at": "2025-10-27T10:15:01"
}
```

**Características Incluidas:**
- ✅ JSON file per upload (~12 files/year)
- ✅ Success/failure counts
- ✅ Error details for manual review
- ✅ Duration tracking
- ✅ Human-readable format (pretty-printed)

**Características NO Incluidas:**
- ❌ Elasticsearch (YAGNI: 12 files/year, grep works fine)
- ❌ Distributed tracing (YAGNI: single-process app)
- ❌ Real-time monitoring (YAGNI: batch processing, check after upload)
- ❌ Grafana dashboards (YAGNI: manual review sufficient)
- ❌ PagerDuty alerts (YAGNI: non-critical personal app)

**Configuración:**
```python
# Hardcoded (no config needed)
LOG_DIR = "~/.finance-app/logs/normalize"
```

**Performance:**
- Write latency: 5ms (local SSD)
- Read latency: 3ms
- Storage: ~2KB per log file
- Total storage: 24KB/year (12 files × 2KB)

**Upgrade Triggers:**
- Si >100 uploads/year → Small Business (agregación needed)
- Si multi-usuario → Small Business (shared log analysis)

---

### Small Business Profile (250 LOC)

**Contexto del Usuario:**
Empresa procesa 100 uploads/mes (1,200/año). Necesitan agregaciones mensuales: tasa de éxito, errores comunes, duración promedio. JSON files + análisis en memoria es suficiente. No necesitan Elasticsearch todavía.

**Implementation:**
```python
# lib/normalization_log.py (Small Business - 250 LOC)
import json, sqlite3
from datetime import datetime
from pathlib import Path
from collections import defaultdict

class NormalizationLog:
    """JSON logs + SQLite index for aggregations."""

    def __init__(self, base_path="./logs/normalize"):
        self.base_path = Path(base_path)
        self.base_path.mkdir(parents=True, exist_ok=True)
        self.db_path = self.base_path / "index.db"
        self._init_db()

    def _init_db(self):
        """Create SQLite index for fast queries."""
        conn = sqlite3.connect(self.db_path)
        conn.execute("""
            CREATE TABLE IF NOT EXISTS normalization_logs (
                upload_id TEXT PRIMARY KEY,
                started_at TEXT,
                completed_at TEXT,
                observations_processed INTEGER,
                canonicals_created INTEGER,
                failures INTEGER,
                warnings INTEGER,
                duration_ms INTEGER
            )
        """)
        conn.commit()
        conn.close()

    def write(self, upload_id: str, log_entry: dict):
        """Write log to JSON + SQLite index."""
        # Write JSON file
        log_path = self.base_path / f"{upload_id}.log.json"
        with open(log_path, "w") as f:
            json.dump(log_entry, f, indent=2)

        # Update SQLite index
        conn = sqlite3.connect(self.db_path)
        conn.execute("""
            INSERT OR REPLACE INTO normalization_logs VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            upload_id,
            log_entry["started_at"],
            log_entry["completed_at"],
            log_entry["observations_processed"],
            log_entry["canonicals_created"],
            log_entry["failures"],
            log_entry.get("warnings", 0),
            log_entry["duration_ms"]
        ))
        conn.commit()
        conn.close()

    def get_monthly_stats(self, year: int, month: int) -> dict:
        """Aggregate stats for a month."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.execute("""
            SELECT
                COUNT(*) as total_uploads,
                SUM(observations_processed) as total_observations,
                SUM(canonicals_created) as total_canonicals,
                SUM(failures) as total_failures,
                SUM(warnings) as total_warnings,
                AVG(duration_ms) as avg_duration_ms
            FROM normalization_logs
            WHERE started_at LIKE ?
        """, (f"{year}-{month:02d}%",))

        row = cursor.fetchone()
        conn.close()

        return {
            "total_uploads": row[0],
            "total_observations": row[1],
            "total_canonicals": row[2],
            "total_failures": row[3],
            "total_warnings": row[4],
            "avg_duration_ms": int(row[5]) if row[5] else 0,
            "success_rate": (row[2] / row[1] * 100) if row[1] > 0 else 0
        }

# Usage
log = NormalizationLog()

# Monthly report
stats = log.get_monthly_stats(2025, 10)
print(f"Success rate: {stats['success_rate']:.1f}%")
print(f"Avg duration: {stats['avg_duration_ms']}ms")
```

**Características Incluidas:**
- ✅ JSON files + SQLite index
- ✅ Monthly aggregation queries
- ✅ Success rate calculation
- ✅ Average duration tracking
- ✅ Fast lookups (SQLite indexed)

**Características NO Incluidas:**
- ❌ Elasticsearch (YAGNI: 1,200 logs/year, SQLite handles it)
- ❌ Distributed tracing (YAGNI: single server)
- ❌ Real-time dashboards (monthly reports sufficient)
- ❌ Daily index rotation (YAGNI: single DB file works)

**Configuración:**
```yaml
normalization_log:
  base_path: "./logs/normalize"
  retention_days: 365  # Keep 1 year
```

**Performance:**
- Write: 8ms (JSON + SQLite insert)
- Monthly aggregation: 50ms (1,200 rows scan)
- Storage: 2MB/year (1,200 × 2KB)

**Upgrade Triggers:**
- Si >10K uploads/mes → Enterprise (Elasticsearch)
- Si necesitan real-time monitoring → Enterprise

---

### Enterprise Profile (900 LOC)

**Contexto del Usuario:**
Procesa 5M transacciones/día = 5M normalization logs/día. Necesitan real-time monitoring: dashboards con p95 latency, alertas para failure rate >1%, distributed tracing para debug, histórico de 90 días. Elasticsearch con daily index rotation.

**Implementation:**
```python
# lib/normalization_log.py (Enterprise - 900 LOC)
from elasticsearch import Elasticsearch
from datetime import datetime
import uuid

class NormalizationLog:
    """Enterprise logging with Elasticsearch + distributed tracing."""

    def __init__(self, es_hosts=["localhost:9200"]):
        self.es = Elasticsearch(es_hosts)
        self._ensure_index_template()

    def _ensure_index_template(self):
        """Create daily index template."""
        self.es.indices.put_index_template(
            name="normalization-logs",
            body={
                "index_patterns": ["normalization-logs-*"],
                "template": {
                    "mappings": {
                        "properties": {
                            "upload_id": {"type": "keyword"},
                            "trace_id": {"type": "keyword"},
                            "span_id": {"type": "keyword"},
                            "normalizer_version": {"type": "keyword"},
                            "observations_processed": {"type": "integer"},
                            "canonicals_created": {"type": "integer"},
                            "failures": {"type": "integer"},
                            "warnings": {"type": "integer"},
                            "duration_ms": {"type": "integer"},
                            "started_at": {"type": "date"},
                            "completed_at": {"type": "date"},
                            "failure_details": {"type": "object", "enabled": False},
                            "warning_details": {"type": "object", "enabled": False}
                        }
                    }
                }
            }
        )

    def write(self, upload_id: str, log_entry: dict, trace_id: str = None):
        """Write to Elasticsearch with distributed tracing."""
        # Generate IDs
        if not trace_id:
            trace_id = str(uuid.uuid4())
        span_id = str(uuid.uuid4())

        # Daily index (rotation)
        today = datetime.now().strftime("%Y.%m.%d")
        index = f"normalization-logs-{today}"

        # Add trace context
        log_entry["trace_id"] = trace_id
        log_entry["span_id"] = span_id

        # Write to Elasticsearch
        self.es.index(index=index, id=upload_id, document=log_entry)

    def get_success_rate(self, last_hours: int = 24) -> float:
        """Real-time success rate from Elasticsearch."""
        resp = self.es.search(
            index="normalization-logs-*",
            body={
                "size": 0,
                "query": {
                    "range": {
                        "started_at": {
                            "gte": f"now-{last_hours}h"
                        }
                    }
                },
                "aggs": {
                    "total_observations": {"sum": {"field": "observations_processed"}},
                    "total_canonicals": {"sum": {"field": "canonicals_created"}}
                }
            }
        )

        total_obs = resp["aggregations"]["total_observations"]["value"]
        total_can = resp["aggregations"]["total_canonicals"]["value"]
        return (total_can / total_obs * 100) if total_obs > 0 else 0

    def get_p95_latency(self, last_minutes: int = 60) -> float:
        """P95 duration from Elasticsearch."""
        resp = self.es.search(
            index="normalization-logs-*",
            body={
                "size": 0,
                "query": {
                    "range": {
                        "started_at": {
                            "gte": f"now-{last_minutes}m"
                        }
                    }
                },
                "aggs": {
                    "duration_percentiles": {
                        "percentiles": {
                            "field": "duration_ms",
                            "percents": [95]
                        }
                    }
                }
            }
        )

        return resp["aggregations"]["duration_percentiles"]["values"]["95.0"]

# Usage with distributed tracing
from opentelemetry import trace

tracer = trace.get_tracer(__name__)
log = NormalizationLog(es_hosts=["es-cluster:9200"])

with tracer.start_as_current_span("normalize_observations") as span:
    trace_id = format(span.get_span_context().trace_id, '032x')

    # Normalize
    results = process_all_observations(upload_id)

    # Log with trace context
    log.write(upload_id, {
        "upload_id": upload_id,
        "normalizer_version": "3.5.0",
        "observations_processed": results["total"],
        "canonicals_created": results["success"],
        "failures": results["failures"],
        "duration_ms": 450
    }, trace_id=trace_id)

# Real-time monitoring
success_rate = log.get_success_rate(last_hours=1)
p95_latency = log.get_p95_latency(last_minutes=60)

if success_rate < 99.0:
    send_pagerduty_alert(f"Normalization success rate: {success_rate:.1f}%")
```

**Prometheus Metrics:**
```python
from prometheus_client import Counter, Histogram

normalization_observations_total = Counter(
    'normalization_observations_total',
    'Total observations processed',
    ['normalizer_version', 'status']
)

normalization_duration_seconds = Histogram(
    'normalization_duration_seconds',
    'Normalization duration',
    ['normalizer_version']
)

# Usage
normalization_observations_total.labels(
    normalizer_version="3.5.0",
    status="success"
).inc(results["success"])

normalization_duration_seconds.labels(
    normalizer_version="3.5.0"
).observe(results["duration_ms"] / 1000)
```

**Características Incluidas:**
- ✅ Elasticsearch with daily index rotation
- ✅ Distributed tracing (trace_id, span_id)
- ✅ Real-time aggregations (success rate, p95 latency)
- ✅ Prometheus metrics + Grafana dashboards
- ✅ PagerDuty alerts for failure rate >1%
- ✅ 90-day retention (archived to S3)

**Características NO Incluidas:**
- ❌ Custom ML anomaly detection (use standard alerts)

**Configuración:**
```yaml
normalization_log:
  elasticsearch:
    hosts:
      - "es-node-1:9200"
      - "es-node-2:9200"
      - "es-node-3:9200"
    index_prefix: "normalization-logs"
    retention_days: 90
  tracing:
    enabled: true
    service_name: "normalization-service"
  monitoring:
    prometheus_port: 9090
    pagerduty_integration_key: "xxx"
```

**Performance:**
- Write latency: 15ms (Elasticsearch async bulk)
- Query latency (24h aggregation): 200ms
- Throughput: 10K logs/minute
- Storage: 5M logs/day × 2KB = 10GB/day → 900GB/90 days

**No Further Tiers:**
- Scale horizontally (more Elasticsearch nodes)

---

- Elasticsearch retention (90 days, then archived to S3)

**Decision Context:**
- Bank processes 5M transactions/day → 5M normalization logs/day
- Need real-time monitoring to detect data quality regressions
- Distributed tracing required to correlate normalization errors with upstream parsing issues
- ML confidence tracking guides manual review prioritization

---

## Related Primitives

- **Normalizer**: Execution is logged by NormalizationLog
- **UploadRecord**: References NormalizationLog via `normalization_log_ref`
- **ProvenanceLedger**: Records normalize events using data from NormalizationLog

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
