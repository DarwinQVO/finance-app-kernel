# OL Primitive: ParseLog

**Type**: Observability / Structured Log
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.2 (Extraction)

---

## Purpose

Structured execution log for parser operations. Records what happened during parsing: success/failure, rows extracted, execution time, warnings, errors. Enables debugging, monitoring, and audit trails.

---

## Simplicity Profiles

**This section shows how the same universal logging primitive is configured differently based on usage scale.**

### Profile 1: Personal Use (Finance App - Simple JSON Files)

**Context:** Darwin uploads 1-2 statements/month, needs basic debugging when parser fails

**Configuration:**
```yaml
parse_log:
  format: "json"
  destination: "file"
  log_directory: "~/.finance-app/logs/parse/"
  filename_pattern: "{upload_id}.log.json"
  retention_days: null  # Keep forever (logs tiny: ~1KB/month)
  structured: true
  aggregation: false  # No metrics collection
  alerting: false
```

**What's Used:**
- ✅ Write JSON log file - Simple dict → JSON file
- ✅ Basic fields - upload_id, success, rows_extracted, duration_ms, errors
- ✅ Immutable logs - Write-once, never update
- ✅ Filesystem storage - Local directory

**What's NOT Used:**
- ❌ Centralized logging - No Elasticsearch/CloudWatch
- ❌ Metrics aggregation - No success rate calculations
- ❌ Alerting - No notifications on failure
- ❌ Log rotation - Keep all logs (tiny footprint)
- ❌ Structured search - No log indexing
- ❌ Correlation IDs - Single user, no distributed tracing

**Implementation Complexity:** **LOW**
- ~40 lines Python
- Uses stdlib json module
- Write dict to file
- No tests

**Narrative Example:**
> Darwin uploads "BoFA_Oct2024.pdf". Parser runs for 3.4 seconds, extracts 42 transactions. At completion:
> ```python
> parse_log = {
>     "upload_id": "UL_abc123",
>     "parser_id": "bofa_pdf_parser",
>     "parser_version": "1.0.0",
>     "success": True,
>     "rows_extracted": 42,
>     "duration_ms": 3421,
>     "warnings": [],
>     "errors": [],
>     "started_at": "2024-10-27T10:00:00Z",
>     "completed_at": "2024-10-27T10:00:03Z"
> }
>
> # Write to file
> with open("~/.finance-app/logs/parse/UL_abc123.log.json", "w") as f:
>     json.dump(parse_log, f, indent=2)
> ```
>
> Week later, parser fails on corrupted PDF:
> ```python
> parse_log = {
>     "upload_id": "UL_def456",
>     "parser_id": "bofa_pdf_parser",
>     "parser_version": "1.0.0",
>     "success": False,
>     "rows_extracted": 0,
>     "duration_ms": 150,
>     "warnings": [],
>     "errors": ["PDF structure invalid: Cannot extract text"],
>     "started_at": "2024-11-03T14:30:00Z",
>     "completed_at": "2024-11-03T14:30:00Z"
> }
> ```
> Darwin sees error message in UI: "Parsing failed". Clicks "View Details" → UI reads `/logs/parse/UL_def456.log.json` → Displays error: "PDF structure invalid" → Darwin re-downloads PDF from bank, re-uploads.

---

### Profile 2: Small Business (Accounting Firm - Aggregated Metrics, Basic Monitoring)

**Context:** 50-100 uploads/day, need to monitor parser success rate and debug failures

**Configuration:**
```yaml
parse_log:
  format: "json"
  destination: "file"  # Still filesystem (sufficient for 36K logs/year)
  log_directory: "/var/accounting-data/logs/parse/"
  filename_pattern: "{upload_id}.log.json"
  retention_days: 365  # 1 year retention
  structured: true
  aggregation:
    enabled: true
    metrics:
      - success_rate
      - p95_latency
      - warning_rate
      - errors_by_type
    update_interval_minutes: 5  # Recalculate every 5 min
  alerting:
    enabled: true
    rules:
      - condition: "success_rate < 0.95"  # Alert if <95% success
        notification: "email"
        recipients: ["admin@accounting-firm.com"]
  dashboard:
    enabled: true
    refresh_interval_seconds: 30
```

**What's Used:**
- ✅ JSON file logging - Same format as Personal
- ✅ Retention policy - Delete logs >1 year old
- ✅ Metrics aggregation - Calculate success rate, latency, warnings
- ✅ Simple dashboard - HTML page with metrics
- ✅ Email alerting - Notify on success rate drop

**What's NOT Used:**
- ❌ Centralized logging - Still filesystem
- ❌ Real-time streaming - Batch aggregation (5min intervals)
- ❌ Advanced analytics - Basic counts only
- ❌ Distributed tracing - No correlation IDs

**Implementation Complexity:** **MEDIUM**
- ~200 lines Python
- Background job: Aggregate metrics every 5 min
- Simple HTML dashboard (Flask endpoint)
- Email notification (SMTP)
- Unit tests (test aggregation logic)

**Narrative Example:**
> Accountant uploads 85 documents in one day. Each parser execution writes log file. Background aggregation job runs every 5 minutes:
> ```python
> # Query logs from last 24 hours
> logs = [json.load(open(f)) for f in glob("logs/parse/*.log.json") if recent(f, hours=24)]
>
> # Calculate metrics
> total = len(logs)  # 85
> successes = sum(1 for log in logs if log["success"])  # 82
> success_rate = successes / total  # 0.965 (96.5%)
>
> latencies = [log["duration_ms"] for log in logs if log["success"]]
> p95_latency = percentile(latencies, 95)  # 4200ms
>
> warnings = sum(len(log["warnings"]) for log in logs)  # 12
> warning_rate = warnings / total  # 0.141 (14.1%)
>
> # Group errors by type
> errors_by_type = {}
> for log in logs:
>     for error in log["errors"]:
>         error_type = error.split(":")[0]  # "PDF structure invalid"
>         errors_by_type[error_type] = errors_by_type.get(error_type, 0) + 1
> # {"PDF structure invalid": 3}
> ```
>
> Dashboard displays:
> ```
> Parser Success Rate: 96.5% ✅ (threshold: 95%)
> P95 Latency: 4.2s
> Warning Rate: 14.1%
> Errors (last 24h):
>   - PDF structure invalid: 3 occurrences
> ```
>
> Next day, success rate drops to 92% (8 failures out of 100). Alerting rule triggers:
> ```
> Condition: success_rate (0.92) < 0.95 ✅
> Send email to: admin@accounting-firm.com
> Subject: "Parser Success Rate Alert: 92%"
> Body: "Parser success rate dropped below 95% threshold. Current: 92%. Failures: 8/100 in last 24h."
> ```
> Admin investigates → Discovers new bank statement format not recognized by parser → Updates parser rules → Success rate recovers to 98%.

---

### Profile 3: Enterprise (Bank - Elasticsearch, Prometheus, Distributed Tracing)

**Context:** 10,000 uploads/day = 3.6M logs/year, need real-time monitoring, correlation with distributed services

**Configuration:**
```yaml
parse_log:
  format: "json"
  destination: "elasticsearch"
  elasticsearch:
    hosts: ["es-logs-1.internal:9200", "es-logs-2.internal:9200"]
    index: "parse-logs"
    index_rotation: "daily"  # Daily indexes (parse-logs-2024-10-27)
    retention_days: 90  # Delete indexes older than 90 days
  structured: true
  fields:
    required: [upload_id, parser_id, parser_version, success, rows_extracted, duration_ms, started_at, completed_at]
    optional: [warnings, errors, trace_id, span_id, customer_id, document_type]
  distributed_tracing:
    enabled: true
    trace_id_field: "trace_id"
    span_id_field: "span_id"
    parent_span_id_field: "parent_span_id"
  metrics:
    enabled: true
    backend: "prometheus"
    labels: [parser_id, parser_version, document_type, success]
    histograms:
      - name: "parse_duration_seconds"
        buckets: [0.1, 0.5, 1, 2, 5, 10, 30]
      - name: "rows_extracted"
        buckets: [0, 10, 50, 100, 500, 1000, 5000]
  alerting:
    enabled: true
    backend: "pagerduty"
    rules:
      - name: "parser_failure_rate_high"
        condition: "success_rate < 0.99"  # Alert if <99%
        severity: "P2"
      - name: "parser_latency_high"
        condition: "p95_latency_ms > 10000"  # >10s
        severity: "P3"
      - name: "parser_errors_spike"
        condition: "error_rate > 0.05"  # >5% errors
        severity: "P1"
  dashboard:
    enabled: true
    backend: "grafana"
    panels:
      - parser_success_rate_by_version
      - parser_latency_heatmap
      - parser_errors_by_type
      - parser_throughput_per_hour
```

**What's Used:**
- ✅ Elasticsearch storage - Centralized, searchable, daily indexes
- ✅ Distributed tracing - trace_id/span_id for correlation
- ✅ Prometheus metrics - Histograms for latency/rows extracted
- ✅ PagerDuty alerting - P1/P2/P3 severity levels
- ✅ Grafana dashboards - Real-time visualization
- ✅ Index rotation - Daily indexes, 90-day retention
- ✅ Structured fields - Required + optional fields
- ✅ Rich metadata - customer_id, document_type for segmentation

**Implementation Complexity:** **HIGH**
- ~800 lines Python/TypeScript
- Elasticsearch client (bulk insert for performance)
- Prometheus client (histogram/counter metrics)
- OpenTelemetry integration (distributed tracing)
- PagerDuty API client
- Grafana dashboard definitions (JSON)
- Comprehensive test suite
- Monitoring runbooks

**Narrative Example:**
> Bank processes credit card statement (1,200 transactions). Parser executes within distributed system:
> ```python
> # Distributed tracing context
> trace_id = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"  # From API gateway
> span_id = "parser-span-001"
> parent_span_id = "upload-span-001"
>
> # Start parsing
> started_at = datetime.now()
>
> # ... parser execution ...
>
> # Complete parsing
> completed_at = datetime.now()
> duration_ms = (completed_at - started_at).total_seconds() * 1000
>
> # Build log entry
> parse_log = {
>     "upload_id": "UL_12345",
>     "parser_id": "universal_credit_card_parser",
>     "parser_version": "3.5.0",
>     "success": True,
>     "rows_extracted": 1200,
>     "duration_ms": 3200,
>     "warnings": ["Row 450: Date format variant detected"],
>     "errors": [],
>     "started_at": started_at.isoformat(),
>     "completed_at": completed_at.isoformat(),
>     "trace_id": trace_id,
>     "span_id": span_id,
>     "parent_span_id": parent_span_id,
>     "customer_id": "CUST_12345",
>     "document_type": "credit_card_statement"
> }
>
> # Write to Elasticsearch
> es_client.index(
>     index="parse-logs-2024-10-27",
>     document=parse_log
> )
>
> # Record Prometheus metrics
> parse_duration_histogram.labels(
>     parser_id="universal_credit_card_parser",
>     parser_version="3.5.0",
>     document_type="credit_card_statement",
>     success="true"
> ).observe(3.2)  # seconds
>
> rows_extracted_histogram.labels(
>     parser_id="universal_credit_card_parser",
>     document_type="credit_card_statement"
> ).observe(1200)
>
> parse_total_counter.labels(
>     parser_id="universal_credit_card_parser",
>     parser_version="3.5.0",
>     success="true"
> ).inc()
> ```
>
> **Distributed tracing query** (OpenTelemetry):
> ```
> Trace ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
> ├─ API Gateway (50ms) [upload-span-000]
> ├─ Upload Service (200ms) [upload-span-001]
> │  ├─ File Storage (100ms) [storage-span-001]
> │  └─ Parser Execution (3200ms) [parser-span-001] ← Parse log here
> │     ├─ PDF Text Extraction (1500ms)
> │     ├─ Transaction Pattern Matching (1200ms)
> │     └─ Observation Store Writes (500ms)
> └─ Normalization Service (2000ms) [normalize-span-001]
> ```
> Total request time: 5.45s (parse = 3.2s = 59% of total time)
>
> **Grafana dashboard visualization:**
> ```
> Parser Success Rate (last 1 hour):
> - universal_credit_card_parser v3.5.0: 99.2% ✅
> - universal_credit_card_parser v3.4.0: 98.8% ✅
> - bank_statement_parser v2.1.0: 97.5% ⚠️
>
> P95 Latency Heatmap:
> [Heat map showing latency distribution by hour, parser version]
> - 10 AM - 12 PM: 3.5s (normal)
> - 12 PM - 2 PM: 8.2s (high load, acceptable)
> - 2 PM - 4 PM: 12.5s (⚠️ exceeds 10s threshold)
>
> Errors by Type (last 24h):
> - PDF structure invalid: 45 occurrences
> - Timeout after 30s: 12 occurrences
> - Unsupported encoding: 8 occurrences
> ```
>
> **Alert triggered** (2 PM - latency spike):
> ```
> PagerDuty Incident: #12345
> Severity: P3
> Title: "Parser Latency High"
> Description: "P95 latency for universal_credit_card_parser v3.5.0 exceeded 10s threshold (current: 12.5s)"
> Assigned to: On-call engineer
> Runbook: https://internal-wiki.com/runbooks/parser-latency
> ```
>
> Engineer investigates:
> 1. Query Elasticsearch: `GET /parse-logs-*/_search?q=parser_id:universal_credit_card_parser AND duration_ms:>10000`
> 2. Finds 150 slow parses (all from same time window: 2-4 PM)
> 3. Correlation: High CPU usage on parser workers during peak hours
> 4. Resolution: Scale parser workers from 10 → 15 instances
> 5. Latency recovers to 3.5s within 10 minutes ✅
>
> **Retention cleanup** (runs daily):
> ```bash
> # Delete Elasticsearch indexes older than 90 days
> curl -X DELETE "es-logs-1.internal:9200/parse-logs-2024-07-27"
> # Success: Index deleted (freed 2.5GB disk space)
> ```
> Retention met, logs archived to S3 Glacier for compliance (7-year audit trail).

---

**Key Insight:** The same `ParseLog` structured format works across all 3 profiles. Personal writes JSON files (40 lines); Small business adds aggregation + email alerts (200 lines); Enterprise uses Elasticsearch + Prometheus + distributed tracing + PagerDuty (800 lines). All record the same core fields (upload_id, success, duration, errors).

---

## Schema

See [`parse-log.schema.json`](../../schemas/parse-log.schema.json) for complete schema.

**Key fields:**
```typescript
interface ParseLog {
  upload_id: string
  parser_id: string
  parser_version: string
  success: boolean
  rows_extracted: number
  duration_ms: number
  warnings: string[]
  errors: string[]
  started_at: ISO8601DateTime
  completed_at: ISO8601DateTime
}
```

---

## Behavior

**Success case:**
```json
{
  "upload_id": "UL_abc123",
  "parser_id": "bofa_pdf_parser",
  "parser_version": "1.2.0",
  "success": true,
  "rows_extracted": 42,
  "duration_ms": 3421,
  "warnings": ["Row 15: Ambiguous date format"],
  "errors": [],
  "started_at": "2025-10-23T14:35:00Z",
  "completed_at": "2025-10-23T14:35:03Z"
}
```

**Failure case:**
```json
{
  "upload_id": "UL_def456",
  "parser_id": "bofa_pdf_parser",
  "parser_version": "1.2.0",
  "success": false,
  "rows_extracted": 0,
  "duration_ms": 150,
  "warnings": [],
  "errors": ["PDF structure invalid", "Cannot extract table"],
  "started_at": "2025-10-23T14:40:00Z",
  "completed_at": "2025-10-23T14:40:00Z"
}
```

---

## Storage

**Path:** `/logs/parse/{upload_id}.log.json`

**Persistence:** Write-once, immutable. Never update existing log.

---

## Multi-Domain Applicability

**Finance:** Log bank statement parsing (success rate, extraction latency)
**Healthcare:** Log lab result extraction (which tests extracted, errors)
**Legal:** Log contract parsing (clauses found, parsing time)
**Research (RSRCH - Utilitario):** Log fact extraction from web pages (facts found, parsing errors, source credibility score)
**Manufacturing:** Log sensor data parsing (data points extracted, corruption detected)

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Log bank statement parsing execution for debugging and monitoring
**Example:** User uploads BoFA PDF → Parser runs → ParseLog records {upload_id: "UL_abc123", parser_id: "bofa_pdf_parser", parser_version: "1.2.0", success: true, rows_extracted: 42, duration_ms: 3421, warnings: ["Row 15: Ambiguous date format"], started_at: "2025-10-23T14:35:00Z"} → Stored at /logs/parse/UL_abc123.log.json → Monitoring dashboard queries logs → Calculates parser success rate (98.5%), p95 latency (4.2s), warning rate (12%)
**Operations:** Write immutable log (write-once), query logs (debugging, monitoring), aggregate metrics (success rate, latency percentiles)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Log lab result parsing for audit trail and error tracking
**Example:** Hospital uploads Quest lab report → Parser extracts 8 tests → ParseLog records {success: true, rows_extracted: 8, duration_ms: 2100, warnings: []} → Later patient queries fail (0 results) → Admin checks parse log → Sees warning "Test name format changed, using fallback extraction" → Fixes parser
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Log contract parsing for audit trail (regulatory compliance)
**Example:** Law firm uploads contract → Parser extracts 25 clauses → ParseLog records {parser_id: "docusign_pdf_parser", parser_version: "2.1.0", rows_extracted: 25, duration_ms: 5500} → Audit requires proof of extraction → Parse log provides immutable record with timestamp, parser version, extraction details
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Log web page fact extraction for debugging and credibility tracking
**Example:** Scraper downloads TechCrunch article → TechCrunchHTMLParser extracts 12 founder facts → ParseLog records {upload_id: "UL_tc_123", parser_id: "techcrunch_html_parser", parser_version: "1.5.0", success: true, rows_extracted: 12, duration_ms: 1850, warnings: ["Author field missing, using 'Unknown'"], metadata: {source_credibility_score: 0.92, publication_date: "2024-02-25"}} → Dashboard queries logs to calculate parser reliability (TechCrunch parser: 99.2% success rate, avg 12 facts per article)
**Operations:** Logs include domain-specific metadata (source_credibility_score for research quality tracking), warnings track parsing anomalies (missing fields, format changes), duration tracking for scraper performance optimization
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Log supplier catalog parsing for monitoring and error alerts
**Example:** Merchant imports supplier CSV (1,500 products) → SupplierCSVParser extracts observations → ParseLog records {rows_extracted: 1500, duration_ms: 6200, warnings: ["Row 342: Missing SKU, using product_id as fallback"]} → Alert fires if success: false (notify admin) → Monitoring tracks catalog import latency (p95: 7.5s)
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (structured execution log pattern, no domain-specific code)
**Reusability:** High (same log schema works for bank parsers, lab parsers, contract parsers, web scrapers, catalog parsers; only domain-specific metadata fields differ)

---

## Related Primitives

- **Parser**: Execution is logged by ParseLog
- **UploadRecord**: References ParseLog via `parse_log_ref`
- **ProvenanceLedger**: Records parse events using data from ParseLog

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
