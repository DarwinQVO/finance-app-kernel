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

The same NormalizationLog interface supports three implementation scales: Personal (simple JSON files), Small Business (JSON + metrics aggregation), Enterprise (Elasticsearch + distributed tracing + real-time monitoring).

### Personal Profile (60 LOC)

**Darwin's Approach:** Simple JSON file per upload with normalization execution details

**What Darwin needs:**
- Did normalization succeed or fail?
- How many observations were processed?
- Which fields had errors? (for manual review)
- How long did it take?

**What Darwin DOESN'T need:**
- ❌ Elasticsearch with daily index rotation
- ❌ Distributed tracing (trace_id/span_id)
- ❌ Real-time alerting (PagerDuty)
- ❌ Grafana dashboards with p95 latency
- ❌ Historical trend analysis

**Implementation:**

**Storage:**
```
~/.finance-app/logs/normalize/
  UL_abc123.log.json
  UL_def456.log.json
  UL_xyz789.log.json
```

**Code Example (60 LOC):**
```python
# normalize_service.py (Darwin's personal finance app)

import json
import os
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

        # Add timestamp if not present
        if "started_at" not in log_entry:
            log_entry["started_at"] = datetime.now().isoformat()
        if "completed_at" not in log_entry:
            log_entry["completed_at"] = datetime.now().isoformat()

        # Write to file (pretty-printed for human readability)
        with open(log_path, "w") as f:
            json.dump(log_entry, f, indent=2)

    def read(self, upload_id: str) -> dict:
        """Read normalization log for specific upload."""
        log_path = self.base_path / f"{upload_id}.log.json"

        if not log_path.exists():
            return None

        with open(log_path, "r") as f:
            return json.load(f)

# Usage in normalizer
def normalize_observations(upload_id: str):
    log = NormalizationLog()
    start = datetime.now()

    try:
        # Process observations
        results = process_all_observations(upload_id)

        # Calculate duration
        duration_ms = int((datetime.now() - start).total_seconds() * 1000)

        # Write log
        log.write(upload_id, {
            "upload_id": upload_id,
            "normalizer_version": "1.0.0",
            "rule_set_version": "2024.10",
            "observations_processed": results["total"],
            "canonicals_created": results["success"],
            "failures": results["failures"],
            "warnings": results["warnings"],
            "failure_details": results["error_list"],
            "warning_details": results["warning_list"],
            "duration_ms": duration_ms
        })

    except Exception as e:
        # Log error
        log.write(upload_id, {
            "upload_id": upload_id,
            "observations_processed": 0,
            "canonicals_created": 0,
            "failures": -1,
            "error": str(e)
        })
```

**Example Log File (UL_abc123.log.json):**
```json
{
  "upload_id": "UL_abc123",
  "normalizer_version": "1.0.0",
  "rule_set_version": "2024.10",
  "observations_processed": 42,
  "canonicals_created": 42,
  "failures": 0,
  "warnings": 3,
  "warning_details": [
    {
      "canonical_id": "CT_001",
      "flag": "possible_duplicate",
      "message": "Similar transaction on 10/14 for $87.42"
    },
    {
      "canonical_id": "CT_015",
      "flag": "merchant_fuzzy_match",
      "message": "Matched 'WHL FOODS' to 'Whole Foods Market' (85% confidence)"
    },
    {
      "canonical_id": "CT_028",
      "flag": "category_inference",
      "message": "Auto-categorized as 'Gas Stations' based on merchant"
    }
  ],
  "duration_ms": 3421,
  "started_at": "2024-10-27T09:15:32.123456",
  "completed_at": "2024-10-27T09:15:35.544456"
}
```

**Checking logs (Darwin's workflow):**
```bash
# Check if normalization succeeded
$ cat ~/.finance-app/logs/normalize/UL_abc123.log.json | jq '.failures'
0

# See which transactions had warnings
$ cat ~/.finance-app/logs/normalize/UL_abc123.log.json | jq '.warning_details'
[
  {
    "canonical_id": "CT_001",
    "flag": "possible_duplicate",
    "message": "Similar transaction on 10/14 for $87.42"
  }
]
```

**Testing:**
- ❌ No automated tests (simple file I/O, low risk)
- Manual verification: Run normalizer, check JSON file exists and has correct fields

**Monitoring:**
- ❌ No monitoring (Darwin manually checks logs when reviewing transactions)

**Decision Context:**
- Darwin processes 10 PDFs/month → 10 log files/month → 120 files/year
- Total storage: ~50 KB/year (negligible)
- When normalization fails, Darwin checks the log file to see which observations had errors
- YAGNI: No need for Elasticsearch when you have 120 JSON files

---

### Small Business Profile (250 LOC)

**Use Case:** Accounting firm processing 50 client statements/month with quality monitoring

**What they need (vs Personal):**
- ✅ All Personal features (JSON files, error details)
- ✅ **Metrics aggregation** (success rate, warning rate, failure trends)
- ✅ **Rule version tracking** (which rule set was used?)
- ✅ **Email alerts** (if failure rate > 5%)
- ❌ No Elasticsearch (50 logs/month = manageable with SQLite)
- ❌ No distributed tracing (single-server deployment)
- ❌ No Grafana dashboards (CSV exports sufficient)

**Implementation:**

**Storage:**
```
/var/app/logs/normalize/
  2024-10/
    UL_abc123.log.json
    UL_def456.log.json
  2024-11/
    UL_xyz789.log.json

/var/app/db/normalization_metrics.db (SQLite)
```

**Code Example (250 LOC):**
```python
# normalization_log.py (Small Business)

import json
import sqlite3
from datetime import datetime
from pathlib import Path
from typing import Optional
import smtplib
from email.message import EmailMessage

class NormalizationLog:
    """JSON files + SQLite metrics for quality monitoring."""

    def __init__(self, base_path="/var/app/logs/normalize", db_path="/var/app/db/normalization_metrics.db"):
        self.base_path = Path(base_path)
        self.db_path = db_path
        self._init_db()

    def _init_db(self):
        """Create metrics table if not exists."""
        conn = sqlite3.connect(self.db_path)
        conn.execute("""
            CREATE TABLE IF NOT EXISTS normalization_metrics (
                upload_id TEXT PRIMARY KEY,
                normalizer_version TEXT,
                rule_set_version TEXT,
                observations_processed INTEGER,
                canonicals_created INTEGER,
                failures INTEGER,
                warnings INTEGER,
                duration_ms INTEGER,
                started_at TIMESTAMP,
                completed_at TIMESTAMP,
                success_rate REAL,  -- (canonicals / observations) * 100
                warning_rate REAL   -- (warnings / observations) * 100
            )
        """)
        conn.execute("CREATE INDEX IF NOT EXISTS idx_completed ON normalization_metrics(completed_at)")
        conn.commit()
        conn.close()

    def write(self, upload_id: str, log_entry: dict):
        """Write log to JSON + insert metrics to SQLite."""
        # 1. Write JSON file (for detailed error inspection)
        month_dir = self.base_path / datetime.now().strftime("%Y-%m")
        month_dir.mkdir(parents=True, exist_ok=True)

        log_path = month_dir / f"{upload_id}.log.json"
        with open(log_path, "w") as f:
            json.dump(log_entry, f, indent=2)

        # 2. Insert metrics to SQLite (for aggregation queries)
        conn = sqlite3.connect(self.db_path)

        success_rate = 0
        warning_rate = 0
        if log_entry.get("observations_processed", 0) > 0:
            success_rate = (log_entry["canonicals_created"] / log_entry["observations_processed"]) * 100
            warning_rate = (log_entry.get("warnings", 0) / log_entry["observations_processed"]) * 100

        conn.execute("""
            INSERT OR REPLACE INTO normalization_metrics
            (upload_id, normalizer_version, rule_set_version, observations_processed,
             canonicals_created, failures, warnings, duration_ms, started_at, completed_at,
             success_rate, warning_rate)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            upload_id,
            log_entry.get("normalizer_version"),
            log_entry.get("rule_set_version"),
            log_entry.get("observations_processed", 0),
            log_entry.get("canonicals_created", 0),
            log_entry.get("failures", 0),
            log_entry.get("warnings", 0),
            log_entry.get("duration_ms", 0),
            log_entry.get("started_at"),
            log_entry.get("completed_at"),
            success_rate,
            warning_rate
        ))
        conn.commit()
        conn.close()

        # 3. Check alert thresholds
        self._check_alerts(log_entry)

    def _check_alerts(self, log_entry: dict):
        """Send email if failure rate exceeds threshold."""
        obs = log_entry.get("observations_processed", 0)
        failures = log_entry.get("failures", 0)

        if obs == 0:
            return

        failure_rate = (failures / obs) * 100

        if failure_rate > 5.0:
            self._send_alert_email(
                subject=f"High Normalization Failure Rate: {failure_rate:.1f}%",
                body=f"""
                Upload: {log_entry['upload_id']}
                Observations: {obs}
                Failures: {failures}
                Failure Rate: {failure_rate:.1f}%

                Details: /var/app/logs/normalize/{log_entry['upload_id']}.log.json
                """
            )

    def _send_alert_email(self, subject: str, body: str):
        """Send email via SMTP."""
        msg = EmailMessage()
        msg["From"] = "alerts@accounting-firm.com"
        msg["To"] = "admin@accounting-firm.com"
        msg["Subject"] = subject
        msg.set_content(body)

        with smtplib.SMTP("localhost", 1025) as smtp:
            smtp.send_message(msg)

    def get_metrics_summary(self, days: int = 30) -> dict:
        """Get aggregated metrics for last N days."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.execute(f"""
            SELECT
                COUNT(*) as total_uploads,
                AVG(success_rate) as avg_success_rate,
                AVG(warning_rate) as avg_warning_rate,
                AVG(duration_ms) as avg_duration_ms,
                MAX(duration_ms) as p95_duration_ms
            FROM normalization_metrics
            WHERE completed_at >= datetime('now', '-{days} days')
        """)

        row = cursor.fetchone()
        conn.close()

        return {
            "total_uploads": row[0],
            "avg_success_rate": round(row[1], 2) if row[1] else 0,
            "avg_warning_rate": round(row[2], 2) if row[2] else 0,
            "avg_duration_ms": int(row[3]) if row[3] else 0,
            "p95_duration_ms": row[4]
        }

    def export_to_csv(self, output_path: str, days: int = 30):
        """Export metrics to CSV for analysis."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.execute(f"""
            SELECT upload_id, normalizer_version, rule_set_version,
                   observations_processed, canonicals_created, failures, warnings,
                   success_rate, warning_rate, duration_ms, completed_at
            FROM normalization_metrics
            WHERE completed_at >= datetime('now', '-{days} days')
            ORDER BY completed_at DESC
        """)

        import csv
        with open(output_path, "w", newline="") as f:
            writer = csv.writer(f)
            writer.writerow(["Upload ID", "Normalizer", "Rules", "Observations",
                           "Canonicals", "Failures", "Warnings", "Success %",
                           "Warning %", "Duration (ms)", "Completed"])
            writer.writerows(cursor.fetchall())

        conn.close()
```

**Usage:**
```python
# After normalization completes
log = NormalizationLog()

log.write("UL_abc123", {
    "upload_id": "UL_abc123",
    "normalizer_version": "1.2.0",
    "rule_set_version": "2024.11",
    "observations_processed": 250,
    "canonicals_created": 245,
    "failures": 5,
    "warnings": 18,
    "failure_details": [...],
    "warning_details": [...],
    "duration_ms": 8200,
    "started_at": "2024-10-27T10:00:00",
    "completed_at": "2024-10-27T10:00:08"
})

# Monthly quality report
summary = log.get_metrics_summary(days=30)
print(f"Success Rate: {summary['avg_success_rate']}%")
print(f"P95 Latency: {summary['p95_duration_ms']} ms")

# Export for accounting review
log.export_to_csv("/tmp/normalization_metrics_oct.csv", days=30)
```

**Testing:**
- ✅ Unit tests for metrics calculation (success_rate, warning_rate)
- ✅ Integration test for email alerts (mock SMTP)
- ✅ CSV export validation

**Monitoring:**
- Weekly CSV export reviewed by operations manager
- Email alerts for high failure rates
- Monthly trend analysis (is quality improving?)

---

### Enterprise Profile (900 LOC)

**Use Case:** National bank processing 5M transactions/day with real-time quality monitoring

**What they need (vs Small Business):**
- ✅ All Small Business features (metrics, alerts, rule tracking)
- ✅ **Elasticsearch** with daily index rotation (logs.normalization-2024.10.27)
- ✅ **Distributed tracing** (trace_id/span_id for correlation)
- ✅ **Prometheus metrics** (histograms for latency, counters for failures)
- ✅ **Real-time alerting** (PagerDuty if failure rate spikes)
- ✅ **Grafana dashboards** (p50/p95/p99 latency, success rate trends)
- ✅ **Confidence score tracking** (ML model confidence per normalization)
- ✅ **Rule change audit** (which rule version was active?)

**Implementation:**

**Storage:**
```
Elasticsearch:
  Index: logs.normalization-2024.10.27
  Retention: 90 days
  Shards: 5 (distributed across cluster)

Prometheus Metrics:
  normalization_observations_total (counter)
  normalization_failures_total (counter)
  normalization_warnings_total (counter)
  normalization_duration_seconds (histogram)
  normalization_confidence_score (histogram)
```

**Code Example (900 LOC - excerpt):**
```python
# normalization_log.py (Enterprise)

import json
import time
from datetime import datetime
from typing import Optional, Dict, List
from elasticsearch import Elasticsearch
from prometheus_client import Counter, Histogram
import uuid

# Prometheus metrics
OBSERVATIONS_PROCESSED = Counter(
    'normalization_observations_total',
    'Total observations processed',
    ['normalizer_version', 'rule_set_version']
)

FAILURES = Counter(
    'normalization_failures_total',
    'Total normalization failures',
    ['normalizer_version', 'error_type']
)

WARNINGS = Counter(
    'normalization_warnings_total',
    'Total normalization warnings',
    ['warning_flag']
)

DURATION = Histogram(
    'normalization_duration_seconds',
    'Normalization duration in seconds',
    buckets=[0.1, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0]
)

CONFIDENCE_SCORE = Histogram(
    'normalization_confidence_score',
    'ML model confidence scores',
    ['field_name'],
    buckets=[0.5, 0.6, 0.7, 0.8, 0.9, 0.95, 0.99, 1.0]
)

class NormalizationLog:
    """Enterprise logging with Elasticsearch, distributed tracing, Prometheus."""

    def __init__(self, es_hosts: List[str], pagerduty_key: Optional[str] = None):
        self.es = Elasticsearch(es_hosts)
        self.pagerduty_key = pagerduty_key
        self._ensure_index_template()

    def _ensure_index_template(self):
        """Create Elasticsearch index template for normalization logs."""
        template = {
            "index_patterns": ["logs.normalization-*"],
            "settings": {
                "number_of_shards": 5,
                "number_of_replicas": 2,
                "index.lifecycle.name": "logs-90day-retention"
            },
            "mappings": {
                "properties": {
                    "upload_id": {"type": "keyword"},
                    "trace_id": {"type": "keyword"},
                    "span_id": {"type": "keyword"},
                    "parent_span_id": {"type": "keyword"},
                    "normalizer_version": {"type": "keyword"},
                    "rule_set_version": {"type": "keyword"},
                    "observations_processed": {"type": "integer"},
                    "canonicals_created": {"type": "integer"},
                    "failures": {"type": "integer"},
                    "warnings": {"type": "integer"},
                    "duration_ms": {"type": "integer"},
                    "started_at": {"type": "date"},
                    "completed_at": {"type": "date"},
                    "success_rate": {"type": "float"},
                    "warning_rate": {"type": "float"},
                    "failure_details": {
                        "type": "nested",
                        "properties": {
                            "observation_id": {"type": "keyword"},
                            "field": {"type": "keyword"},
                            "error": {"type": "text"},
                            "error_type": {"type": "keyword"}
                        }
                    },
                    "warning_details": {
                        "type": "nested",
                        "properties": {
                            "canonical_id": {"type": "keyword"},
                            "flag": {"type": "keyword"},
                            "message": {"type": "text"},
                            "confidence_score": {"type": "float"}
                        }
                    }
                }
            }
        }

        self.es.indices.put_index_template(
            name="normalization-logs",
            body=template
        )

    def write(
        self,
        upload_id: str,
        log_entry: dict,
        trace_id: Optional[str] = None,
        parent_span_id: Optional[str] = None
    ):
        """Write log to Elasticsearch with distributed tracing."""
        # Generate span ID for this normalization operation
        span_id = f"normalize-{uuid.uuid4().hex[:16]}"

        # Add tracing context
        log_entry["trace_id"] = trace_id or f"trace-{uuid.uuid4()}"
        log_entry["span_id"] = span_id
        log_entry["parent_span_id"] = parent_span_id

        # Calculate metrics
        obs = log_entry.get("observations_processed", 0)
        success = log_entry.get("canonicals_created", 0)
        failures = log_entry.get("failures", 0)
        warnings = log_entry.get("warnings", 0)

        if obs > 0:
            log_entry["success_rate"] = (success / obs) * 100
            log_entry["warning_rate"] = (warnings / obs) * 100
        else:
            log_entry["success_rate"] = 0
            log_entry["warning_rate"] = 0

        # Index to Elasticsearch (daily indices)
        index_name = f"logs.normalization-{datetime.now().strftime('%Y.%m.%d')}"

        self.es.index(
            index=index_name,
            document=log_entry,
            id=upload_id
        )

        # Update Prometheus metrics
        self._update_metrics(log_entry)

        # Check alert thresholds
        self._check_alerts(log_entry)

    def _update_metrics(self, log_entry: dict):
        """Update Prometheus metrics."""
        normalizer_ver = log_entry.get("normalizer_version", "unknown")
        rule_ver = log_entry.get("rule_set_version", "unknown")

        # Observations processed
        obs = log_entry.get("observations_processed", 0)
        OBSERVATIONS_PROCESSED.labels(
            normalizer_version=normalizer_ver,
            rule_set_version=rule_ver
        ).inc(obs)

        # Failures by type
        for failure in log_entry.get("failure_details", []):
            error_type = failure.get("error_type", "unknown")
            FAILURES.labels(
                normalizer_version=normalizer_ver,
                error_type=error_type
            ).inc()

        # Warnings by flag
        for warning in log_entry.get("warning_details", []):
            flag = warning.get("flag", "unknown")
            WARNINGS.labels(warning_flag=flag).inc()

            # Confidence scores
            if "confidence_score" in warning:
                field_name = warning.get("field", "unknown")
                CONFIDENCE_SCORE.labels(field_name=field_name).observe(
                    warning["confidence_score"]
                )

        # Duration
        duration_s = log_entry.get("duration_ms", 0) / 1000.0
        DURATION.observe(duration_s)

    def _check_alerts(self, log_entry: dict):
        """Trigger PagerDuty alert if failure rate exceeds threshold."""
        obs = log_entry.get("observations_processed", 0)
        failures = log_entry.get("failures", 0)

        if obs == 0:
            return

        failure_rate = (failures / obs) * 100

        # Critical: > 10% failure rate
        if failure_rate > 10.0:
            self._trigger_pagerduty(
                severity="critical",
                summary=f"Normalization failure rate: {failure_rate:.1f}%",
                details={
                    "upload_id": log_entry["upload_id"],
                    "observations": obs,
                    "failures": failures,
                    "failure_rate": failure_rate,
                    "trace_id": log_entry.get("trace_id")
                }
            )

        # Warning: > 5% failure rate
        elif failure_rate > 5.0:
            self._trigger_pagerduty(
                severity="warning",
                summary=f"Elevated normalization failure rate: {failure_rate:.1f}%",
                details={
                    "upload_id": log_entry["upload_id"],
                    "failure_rate": failure_rate
                }
            )

    def _trigger_pagerduty(self, severity: str, summary: str, details: dict):
        """Send alert to PagerDuty."""
        if not self.pagerduty_key:
            return

        import requests

        payload = {
            "routing_key": self.pagerduty_key,
            "event_action": "trigger",
            "payload": {
                "summary": summary,
                "severity": severity,
                "source": "normalization-pipeline",
                "custom_details": details
            }
        }

        requests.post(
            "https://events.pagerduty.com/v2/enqueue",
            json=payload
        )

    def query_success_rate(self, hours: int = 24) -> Dict:
        """Query success rate from Elasticsearch."""
        query = {
            "size": 0,
            "query": {
                "range": {
                    "completed_at": {
                        "gte": f"now-{hours}h"
                    }
                }
            },
            "aggs": {
                "avg_success_rate": {"avg": {"field": "success_rate"}},
                "avg_warning_rate": {"avg": {"field": "warning_rate"}},
                "p50_duration": {"percentiles": {"field": "duration_ms", "percents": [50]}},
                "p95_duration": {"percentiles": {"field": "duration_ms", "percents": [95]}},
                "p99_duration": {"percentiles": {"field": "duration_ms", "percents": [99]}}
            }
        }

        result = self.es.search(
            index="logs.normalization-*",
            body=query
        )

        aggs = result["aggregations"]

        return {
            "avg_success_rate": round(aggs["avg_success_rate"]["value"], 2),
            "avg_warning_rate": round(aggs["avg_warning_rate"]["value"], 2),
            "p50_duration_ms": int(aggs["p50_duration"]["values"]["50.0"]),
            "p95_duration_ms": int(aggs["p95_duration"]["values"]["95.0"]),
            "p99_duration_ms": int(aggs["p99_duration"]["values"]["99.0"])
        }
```

**Grafana Dashboard Queries:**
```promql
# Success rate (last 24h)
100 - (
  rate(normalization_failures_total[1h]) /
  rate(normalization_observations_total[1h]) * 100
)

# P95 latency
histogram_quantile(0.95,
  rate(normalization_duration_seconds_bucket[5m])
)

# Warning rate by flag
rate(normalization_warnings_total[5m])

# Low confidence normalizations (< 0.8)
histogram_quantile(0.5,
  normalization_confidence_score_bucket{le="0.8"}
)
```

**Testing:**
- ✅ Unit tests for all metrics calculations
- ✅ Integration tests with Elasticsearch test cluster
- ✅ PagerDuty alert mock testing
- ✅ Load testing (10K logs/minute ingestion)
- ✅ Distributed tracing validation (trace_id propagation)

**Monitoring:**
- Real-time Grafana dashboards (success rate, latency percentiles, error types)
- PagerDuty alerts for critical failure rates
- Weekly quality review (trend analysis over 30 days)
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
