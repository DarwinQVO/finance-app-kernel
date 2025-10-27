# MetricsCollector OL Primitive

**Domain:** Rule Performance & Logs (Vertical 5.3)
**Layer:** Objective Layer (OL)
**Version:** 1.0.0
**Status:** Specification

---

## Table of Contents

1. [Overview](#overview)
2. [Purpose & Scope](#purpose--scope)
3. [Multi-Domain Applicability](#multi-domain-applicability)
4. [Core Concepts](#core-concepts)
5. [Interface Definition](#interface-definition)
6. [Data Model](#data-model)
7. [Core Functionality](#core-functionality)
8. [Batch Processing](#batch-processing)
9. [Metric Types](#metric-types)
10. [Storage & Indexing](#storage--indexing)
11. [Edge Cases](#edge-cases)
12. [Performance Characteristics](#performance-characteristics)
13. [Implementation Notes](#implementation-notes)
14. [Security Considerations](#security-considerations)
15. [Integration Patterns](#integration-patterns)
16. [Multi-Domain Examples](#multi-domain-examples)
17. [Testing Strategy](#testing-strategy)
18. [Configuration](#configuration)
19. [Observability](#observability)
20. [Future Enhancements](#future-enhancements)

---

## Overview

The **MetricsCollector** is a high-performance observability primitive that records execution metrics for document processing pipelines with minimal overhead. It captures parser execution times, normalization rule performance, queue depths, and error logs to enable comprehensive monitoring and performance analysis.

### Key Capabilities

- **Low-Latency Ingestion**: <10ms p95 metric recording with minimal overhead
- **Batch Processing**: Supports bulk inserts up to 10K metrics/sec
- **Multi-Metric Types**: Parser, rule, queue, and error metrics
- **Asynchronous Writing**: Non-blocking metric collection
- **Time-Series Optimized**: Designed for high-volume time-series data
- **Automatic Aggregation**: Pre-computed rollups for common queries

### Design Philosophy

The MetricsCollector follows four core principles:

1. **Minimal Overhead**: Metric collection must not slow down pipeline processing
2. **High Throughput**: Support thousands of metrics per second
3. **Reliability**: Never lose critical performance data
4. **Queryability**: Optimize storage for fast aggregation queries

---

## Purpose & Scope

### Problem Statement

In document processing pipelines, observability poses critical challenges:

- **Performance Blind Spots**: No visibility into parser/rule execution times
- **Bottleneck Detection**: Can't identify slow parsers or inefficient rules
- **Error Tracking**: Errors are scattered across logs without context
- **Queue Monitoring**: No real-time view of processing backlogs
- **Compliance Requirements**: Need audit trail of all processing activities

Traditional approaches have significant limitations:

**Approach 1: Application Logging**
- ❌ High overhead (10-50ms per log write)
- ❌ Unstructured data, hard to query
- ❌ No aggregation support

**Approach 2: External Monitoring Tools**
- ❌ Expensive (per-metric pricing)
- ❌ Limited retention (30-90 days)
- ❌ Network latency to external service

**Approach 3: Custom Metrics Collector**
- ✅ Optimized for time-series data
- ✅ <10ms p95 ingestion latency
- ✅ Batch processing support
- ✅ Unlimited retention in PostgreSQL
- ✅ Domain-specific metric types

### Solution

The MetricsCollector implements a **high-performance metrics pipeline** with:

1. **PostgreSQL Storage**: Time-series optimized tables with partitioning
2. **Asynchronous Collection**: Non-blocking writes with in-memory buffering
3. **Batch Inserts**: Bulk insert up to 1000 metrics at once
4. **Pre-Aggregation**: Automatic rollups for common queries
5. **Context Preservation**: Full context (upload_id, parser_id, rule_id)

---

## Multi-Domain Applicability

The MetricsCollector is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Track performance of bank statement parsers and transaction categorization rules.

**Metrics Collected**:
- PDF parser execution time for Chase, Wells Fargo, BoA statements
- OFX parser processing time for investment accounts
- Transaction categorization rule performance (average 5-15ms per transaction)
- Error rates for OCR failures on scanned checks
- Queue depth for pending statement uploads (peak: 1000+ during month-end)

**Example**:
```typescript
// Record bank statement parser execution
await collector.recordParserExecution({
  parser_id: "chase_pdf_parser_v2",
  upload_id: "upload_123",
  duration_ms: 1250,
  status: "success",
  observations_extracted: 87,
  metadata: {
    file_size_bytes: 524288,
    page_count: 8,
    ocr_required: false
  }
});

// Record transaction categorization rule
await collector.recordRuleExecution({
  rule_id: "merchant_categorization_ml",
  rule_type: "normalization",
  execution_time_ms: 12,
  matched: true,
  transformation_applied: "category:groceries",
  context: {
    merchant: "WHOLE FOODS",
    amount: 87.45
  }
});

// Record queue snapshot
await collector.recordQueueSnapshot({
  queue_name: "statement_processing",
  pending: 234,
  in_progress: 12,
  stuck: 2,
  health_status: "healthy"
});
```

**Performance Target**: <5ms p95 per metric, 500+ parsers/sec during peak.

---

### 2. Healthcare

**Use Case**: Monitor HL7 message parsing and FHIR resource validation performance.

**Metrics Collected**:
- HL7 v2.x message parser execution time (ADT, ORM, ORU messages)
- FHIR resource validation rule performance (Patient, Observation, Condition)
- Claims processing rule execution (X12 837 parsing)
- Error rates for invalid LOINC/SNOMED codes
- Queue depth for inbound lab result messages

**Example**:
```typescript
// Record HL7 parser execution
await collector.recordParserExecution({
  parser_id: "hl7_oru_r01_parser",
  upload_id: "msg_456",
  duration_ms: 85,
  status: "success",
  observations_extracted: 24,
  metadata: {
    message_type: "ORU^R01",
    segments_count: 34,
    version: "2.5.1"
  }
});

// Record FHIR validation rule
await collector.recordRuleExecution({
  rule_id: "fhir_patient_validation",
  rule_type: "validation",
  execution_time_ms: 18,
  matched: true,
  transformation_applied: null,
  context: {
    resource_type: "Patient",
    profile: "us-core"
  }
});

// Record error for invalid code
await collector.recordError({
  error_type: "INVALID_CODE",
  error_message: "Invalid LOINC code: 1234-X",
  context: {
    parser_id: "hl7_oru_r01_parser",
    upload_id: "msg_456",
    field: "OBX-3",
    value: "1234-X"
  },
  severity: "error",
  stack_trace: null
});
```

**Performance Target**: <8ms p95 per metric, handle 1000+ messages/sec.

---

### 3. Legal

**Use Case**: Track e-discovery document processing and contract OCR performance.

**Metrics Collected**:
- PDF OCR parser execution time for scanned contracts
- Document classification rule performance (motion, complaint, exhibit)
- Entity extraction rule performance (party names, dates, amounts)
- Error rates for corrupted or password-protected files
- Queue depth for large document productions (100K+ documents)

**Example**:
```typescript
// Record contract OCR parser
await collector.recordParserExecution({
  parser_id: "tesseract_legal_ocr",
  upload_id: "doc_789",
  duration_ms: 8500,
  status: "success",
  observations_extracted: 142,
  metadata: {
    file_size_bytes: 4194304,
    page_count: 35,
    ocr_confidence: 0.92
  }
});

// Record document classification rule
await collector.recordRuleExecution({
  rule_id: "doc_type_classifier_v3",
  rule_type: "classification",
  execution_time_ms: 45,
  matched: true,
  transformation_applied: "doc_type:motion",
  context: {
    text_sample: "PLAINTIFF'S MOTION FOR SUMMARY JUDGMENT",
    confidence: 0.95
  }
});
```

**Performance Target**: <15ms p95 per metric, support 10K+ documents/hour.

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Monitor founder fact extraction from web scraping and entity resolution performance.

**Metrics Collected**:
- Web page parser execution time (TechCrunch, Twitter, podcast transcripts)
- Fact extraction rule performance (investment facts, founding facts, employment facts)
- Entity name normalization rules (@sama → Sam Altman)
- Error rates for 404s or broken web pages
- Queue depth for bulk web scraping (1000+ URLs)

**Example**:
```typescript
// Record web page fact extraction
await collector.recordParserExecution({
  parser_id: "techcrunch_web_parser",
  upload_id: "article_techcrunch_sama_helion",
  duration_ms: 1800,
  status: "success",
  observations_extracted: 15,
  metadata: {
    source_url: "https://techcrunch.com/2025/01/15/sama-helion",
    page_size_bytes: 524288,
    facts_found: 15,
    entities_detected: 3
  }
});

// Record fact extraction rule
await collector.recordRuleExecution({
  rule_id: "investment_fact_extractor",
  rule_type: "extraction",
  execution_time_ms: 12,
  matched: true,
  transformation_applied: "fact_type:investment",
  context: {
    claim: "Sam Altman invested $375M in Helion Energy",
    subject_entity: "Sam Altman",
    confidence: 0.92
  }
});
```

**Performance Target**: <10ms p95 per metric, 100+ web pages/min during bulk scraping.

---

### 5. E-commerce

**Use Case**: Track product catalog import and categorization rule performance.

**Metrics Collected**:
- CSV parser execution time for product feeds (Amazon, Shopify)
- Product categorization rule performance (taxonomy mapping)
- Image validation rule performance
- Error rates for invalid SKUs or missing required fields
- Queue depth for marketplace integrations (real-time sync)

**Example**:
```typescript
// Record product feed parser
await collector.recordParserExecution({
  parser_id: "amazon_catalog_csv_parser",
  upload_id: "feed_567",
  duration_ms: 450,
  status: "success",
  observations_extracted: 1200,
  metadata: {
    file_size_bytes: 2097152,
    row_count: 1200,
    marketplace: "amazon"
  }
});

// Record categorization rule
await collector.recordRuleExecution({
  rule_id: "product_taxonomy_mapper",
  rule_type: "normalization",
  execution_time_ms: 3,
  matched: true,
  transformation_applied: "category:electronics>computers>laptops",
  context: {
    product_title: "Dell XPS 13 Laptop",
    keywords: ["laptop", "computer", "dell"]
  }
});
```

**Performance Target**: <5ms p95 per metric, 5K+ products/sec during sync.

---

### 6. SaaS

**Use Case**: Monitor webhook payload parsing and API request processing.

**Metrics Collected**:
- Webhook payload parser execution time (Stripe, Twilio, SendGrid)
- API request validation rule performance
- Rate limiting rule execution
- Error rates for malformed payloads or authentication failures
- Queue depth for async webhook processing

**Example**:
```typescript
// Record webhook parser
await collector.recordParserExecution({
  parser_id: "stripe_webhook_parser",
  upload_id: "webhook_890",
  duration_ms: 25,
  status: "success",
  observations_extracted: 1,
  metadata: {
    event_type: "charge.succeeded",
    api_version: "2023-10-16"
  }
});

// Record validation rule
await collector.recordRuleExecution({
  rule_id: "api_request_validator",
  rule_type: "validation",
  execution_time_ms: 2,
  matched: true,
  transformation_applied: null,
  context: {
    endpoint: "/api/v1/payments",
    method: "POST"
  }
});
```

**Performance Target**: <3ms p95 per metric, 10K+ webhooks/sec.

---

### 7. Insurance

**Use Case**: Track claims form extraction and policy document parsing performance.

**Metrics Collected**:
- Claims form OCR parser execution time (ACORD forms)
- Policy document parser performance (PDF, Word)
- Claims validation rule performance (coverage limits, exclusions)
- Error rates for missing signatures or incomplete forms
- Queue depth for claims intake (peak during catastrophe events)

**Example**:
```typescript
// Record claims form parser
await collector.recordParserExecution({
  parser_id: "acord_form_ocr_parser",
  upload_id: "claim_345",
  duration_ms: 2800,
  status: "success",
  observations_extracted: 52,
  metadata: {
    form_type: "ACORD_125",
    page_count: 6,
    ocr_confidence: 0.89
  }
});

// Record claims validation rule
await collector.recordRuleExecution({
  rule_id: "coverage_limit_validator",
  rule_type: "validation",
  execution_time_ms: 15,
  matched: true,
  transformation_applied: "status:approved",
  context: {
    claim_amount: 5000,
    policy_limit: 10000,
    deductible: 500
  }
});
```

**Performance Target**: <10ms p95 per metric, 1K+ claims/hour during peak.

---

### Cross-Domain Benefits

**Consistent Observability**: All domains benefit from:
- Single metrics pipeline for all processing types
- Standardized metric schema across parsers and rules
- Real-time performance monitoring
- Historical trend analysis

**Regulatory Compliance**: Supports:
- SOX (Finance): Audit trail of transaction processing
- HIPAA (Healthcare): PHI processing activity logs
- GDPR (Privacy): Data processing performance records
- ASC 606 (Revenue): Contract processing metrics

---

## Core Concepts

### Metric Types

The MetricsCollector supports four primary metric types:

1. **Parser Execution Metrics**: Record parser performance
2. **Rule Execution Metrics**: Record normalization/validation rule performance
3. **Queue Depth Snapshots**: Record queue state at intervals
4. **Error Logs**: Record processing errors with context

### Metric Identity

Every metric has:
- **metric_id**: Auto-generated UUID (primary key)
- **timestamp**: ISO-8601 timestamp (indexed for time-series queries)
- **type**: Metric type (parser, rule, queue, error)
- **context**: Full context (upload_id, parser_id, rule_id, etc.)

### Performance Targets

- **Ingestion Latency**: <10ms p95 per metric
- **Batch Throughput**: 10K metrics/sec
- **Storage Efficiency**: <1KB per metric (compressed)
- **Query Latency**: <50ms for aggregations over 1M metrics

### Metric Lifecycle

1. **Collection**: Metric recorded during pipeline execution
2. **Buffering**: Metric buffered in-memory (async)
3. **Batch Insert**: Metrics inserted in batches (100-1000 at a time)
4. **Indexing**: Automatic indexing for fast queries
5. **Aggregation**: Pre-computed rollups for common queries
6. **Retention**: Retained indefinitely (with partitioning for old data)

---

## Interface Definition

### TypeScript Interface

```typescript
interface MetricsCollector {
  /**
   * Record parser execution metrics
   *
   * @param metrics - Parser execution details
   * @returns Promise resolving to metric ID
   * @throws ValidationError if metrics are invalid
   */
  recordParserExecution(metrics: ParserExecutionMetrics): Promise<string>;

  /**
   * Record rule execution metrics
   *
   * @param metrics - Rule execution details
   * @returns Promise resolving to metric ID
   * @throws ValidationError if metrics are invalid
   */
  recordRuleExecution(metrics: RuleExecutionMetrics): Promise<string>;

  /**
   * Record queue depth snapshot
   *
   * @param snapshot - Queue state at point in time
   * @returns Promise resolving to snapshot ID
   * @throws ValidationError if snapshot is invalid
   */
  recordQueueSnapshot(snapshot: QueueDepthSnapshot): Promise<string>;

  /**
   * Record error with context
   *
   * @param error - Error details with context
   * @returns Promise resolving to error log ID
   * @throws ValidationError if error is invalid
   */
  recordError(error: ErrorLog): Promise<string>;

  /**
   * Batch record parser executions
   *
   * @param metrics - Array of parser execution metrics
   * @returns Promise resolving to array of metric IDs
   * @throws ValidationError if any metric is invalid
   */
  batchRecordParserExecutions(
    metrics: ParserExecutionMetrics[]
  ): Promise<string[]>;

  /**
   * Batch record rule executions
   *
   * @param metrics - Array of rule execution metrics
   * @returns Promise resolving to array of metric IDs
   * @throws ValidationError if any metric is invalid
   */
  batchRecordRuleExecutions(
    metrics: RuleExecutionMetrics[]
  ): Promise<string[]>;

  /**
   * Flush buffered metrics to storage
   *
   * @returns Promise resolving when flush completes
   */
  flush(): Promise<void>;

  /**
   * Get collector statistics
   *
   * @returns Collector performance stats
   */
  getStats(): CollectorStats;
}
```

---

## Data Model

### ParserExecutionMetrics Type

```typescript
interface ParserExecutionMetrics {
  // Identity
  parser_id: string;              // Parser identifier (e.g., "chase_pdf_parser_v2")
  upload_id: string;              // Upload/document identifier

  // Performance
  duration_ms: number;            // Execution time in milliseconds
  status: ExecutionStatus;        // success | failure | timeout | cancelled

  // Output
  observations_extracted: number; // Number of observations extracted
  error_type?: string;            // Error type if status=failure

  // Context (optional)
  metadata?: Record<string, any>; // Additional context (file size, page count, etc.)
}
```

### RuleExecutionMetrics Type

```typescript
interface RuleExecutionMetrics {
  // Identity
  rule_id: string;                // Rule identifier (e.g., "merchant_categorization_ml")
  rule_type: RuleType;            // normalization | validation | classification | extraction

  // Performance
  execution_time_ms: number;      // Rule execution time in milliseconds

  // Output
  matched: boolean;               // Whether rule matched/applied
  transformation_applied?: string; // Transformation result (if matched)

  // Context (optional)
  context?: Record<string, any>;  // Input data and additional context
}
```

### QueueDepthSnapshot Type

```typescript
interface QueueDepthSnapshot {
  // Identity
  queue_name: string;             // Queue identifier (e.g., "statement_processing")

  // Metrics
  pending: number;                // Items waiting to be processed
  in_progress: number;            // Items currently processing
  stuck: number;                  // Items stuck (>5 retries)

  // Health
  health_status: HealthStatus;    // healthy | degraded | critical
}
```

### ErrorLog Type

```typescript
interface ErrorLog {
  // Error Details
  error_type: string;             // Error classification (e.g., "VALIDATION_ERROR")
  error_message: string;          // Human-readable error message

  // Context
  context: Record<string, any>;   // Full context (parser_id, upload_id, etc.)

  // Severity
  severity: Severity;             // debug | info | warning | error | critical

  // Stack Trace (optional)
  stack_trace?: string;           // Stack trace for debugging
}
```

### CollectorStats Type

```typescript
interface CollectorStats {
  // Throughput
  metrics_collected_total: number;        // Total metrics collected since start
  metrics_per_second: number;             // Current collection rate

  // Latency
  ingestion_latency_p50_ms: number;       // p50 ingestion latency
  ingestion_latency_p95_ms: number;       // p95 ingestion latency
  ingestion_latency_p99_ms: number;       // p99 ingestion latency

  // Buffer
  buffer_size: number;                    // Current buffer size
  buffer_capacity: number;                // Max buffer capacity
  buffer_utilization_percent: number;     // Buffer usage percentage

  // Errors
  collection_errors_total: number;        // Total collection errors
}
```

### Database Schema (PostgreSQL)

```sql
-- Parser execution metrics table
CREATE TABLE parser_execution_metrics (
  -- Primary Key
  metric_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Identity
  parser_id VARCHAR(128) NOT NULL,
  upload_id VARCHAR(128) NOT NULL,

  -- Performance
  duration_ms INTEGER NOT NULL,
  status VARCHAR(32) NOT NULL,

  -- Output
  observations_extracted INTEGER NOT NULL DEFAULT 0,
  error_type VARCHAR(64),

  -- Context
  metadata JSONB,

  -- Timestamp
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Rule execution metrics table
CREATE TABLE rule_execution_metrics (
  -- Primary Key
  metric_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Identity
  rule_id VARCHAR(128) NOT NULL,
  rule_type VARCHAR(32) NOT NULL,

  -- Performance
  execution_time_ms INTEGER NOT NULL,

  -- Output
  matched BOOLEAN NOT NULL,
  transformation_applied TEXT,

  -- Context
  context JSONB,

  -- Timestamp
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Queue depth snapshots table
CREATE TABLE queue_depth_snapshots (
  -- Primary Key
  snapshot_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Identity
  queue_name VARCHAR(128) NOT NULL,

  -- Metrics
  pending INTEGER NOT NULL,
  in_progress INTEGER NOT NULL,
  stuck INTEGER NOT NULL,

  -- Health
  health_status VARCHAR(32) NOT NULL,

  -- Timestamp
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Error logs table
CREATE TABLE error_logs (
  -- Primary Key
  error_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Error Details
  error_type VARCHAR(64) NOT NULL,
  error_message TEXT NOT NULL,

  -- Context
  context JSONB NOT NULL,

  -- Severity
  severity VARCHAR(32) NOT NULL,

  -- Stack Trace
  stack_trace TEXT,

  -- Timestamp
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes for Performance
CREATE INDEX idx_parser_metrics_parser_id ON parser_execution_metrics(parser_id);
CREATE INDEX idx_parser_metrics_upload_id ON parser_execution_metrics(upload_id);
CREATE INDEX idx_parser_metrics_status ON parser_execution_metrics(status);
CREATE INDEX idx_parser_metrics_created_at ON parser_execution_metrics(created_at DESC);

CREATE INDEX idx_rule_metrics_rule_id ON rule_execution_metrics(rule_id);
CREATE INDEX idx_rule_metrics_rule_type ON rule_execution_metrics(rule_type);
CREATE INDEX idx_rule_metrics_matched ON rule_execution_metrics(matched);
CREATE INDEX idx_rule_metrics_created_at ON rule_execution_metrics(created_at DESC);

CREATE INDEX idx_queue_snapshots_queue_name ON queue_depth_snapshots(queue_name);
CREATE INDEX idx_queue_snapshots_health_status ON queue_depth_snapshots(health_status);
CREATE INDEX idx_queue_snapshots_created_at ON queue_depth_snapshots(created_at DESC);

CREATE INDEX idx_error_logs_error_type ON error_logs(error_type);
CREATE INDEX idx_error_logs_severity ON error_logs(severity);
CREATE INDEX idx_error_logs_created_at ON error_logs(created_at DESC);

-- GIN indexes for JSONB columns
CREATE INDEX idx_parser_metrics_metadata ON parser_execution_metrics USING GIN(metadata);
CREATE INDEX idx_rule_metrics_context ON rule_execution_metrics USING GIN(context);
CREATE INDEX idx_error_logs_context ON error_logs USING GIN(context);

-- Partitioning for time-series data (monthly partitions)
-- Example: CREATE TABLE parser_execution_metrics_2025_01 PARTITION OF parser_execution_metrics
--          FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

---

## Core Functionality

### 1. recordParserExecution()

Record parser execution metrics with minimal overhead.

#### Signature

```typescript
recordParserExecution(metrics: ParserExecutionMetrics): Promise<string>
```

#### Behavior

1. **Validation**:
   - Validate parser_id is not empty
   - Validate upload_id is not empty
   - Validate duration_ms >= 0
   - Validate observations_extracted >= 0

2. **Buffering** (Async):
   - Add metric to in-memory buffer
   - If buffer reaches threshold (100 metrics), trigger batch insert
   - Return immediately (non-blocking)

3. **Batch Insert**:
   - INSERT batch into `parser_execution_metrics` table
   - Return metric_id

4. **Error Handling**:
   - Throw `ValidationError` if validation fails
   - Log collection errors (don't throw if insert fails)

#### Example

```typescript
// Record successful parser execution
const metricId = await collector.recordParserExecution({
  parser_id: "chase_pdf_parser_v2",
  upload_id: "upload_123",
  duration_ms: 1250,
  status: "success",
  observations_extracted: 87,
  metadata: {
    file_size_bytes: 524288,
    page_count: 8
  }
});

console.log(`Recorded metric: ${metricId}`);

// Record parser failure
await collector.recordParserExecution({
  parser_id: "wells_fargo_parser",
  upload_id: "upload_456",
  duration_ms: 500,
  status: "failure",
  observations_extracted: 0,
  error_type: "INVALID_FORMAT",
  metadata: {
    error_details: "Unsupported PDF version"
  }
});
```

#### Performance

- **Target Latency**: <10ms p95 (async buffering)
- **Throughput**: 10K metrics/sec (with batching)

---

### 2. recordRuleExecution()

Record normalization/validation rule execution metrics.

#### Signature

```typescript
recordRuleExecution(metrics: RuleExecutionMetrics): Promise<string>
```

#### Behavior

1. **Validation**:
   - Validate rule_id is not empty
   - Validate rule_type is valid enum value
   - Validate execution_time_ms >= 0

2. **Buffering** (Async):
   - Add metric to in-memory buffer
   - Trigger batch insert if threshold reached

3. **Batch Insert**:
   - INSERT batch into `rule_execution_metrics` table

#### Example

```typescript
// Record normalization rule execution
await collector.recordRuleExecution({
  rule_id: "merchant_categorization_ml",
  rule_type: "normalization",
  execution_time_ms: 12,
  matched: true,
  transformation_applied: "category:groceries",
  context: {
    merchant: "WHOLE FOODS",
    amount: 87.45
  }
});

// Record validation rule (no match)
await collector.recordRuleExecution({
  rule_id: "amount_range_validator",
  rule_type: "validation",
  execution_time_ms: 2,
  matched: false,
  context: {
    amount: -500,
    rule: "amount > 0"
  }
});
```

#### Performance

- **Target Latency**: <8ms p95
- **Throughput**: 15K metrics/sec

---

### 3. recordQueueSnapshot()

Record queue depth snapshot at point in time.

#### Signature

```typescript
recordQueueSnapshot(snapshot: QueueDepthSnapshot): Promise<string>
```

#### Behavior

1. **Validation**:
   - Validate queue_name is not empty
   - Validate pending, in_progress, stuck >= 0

2. **Health Status Detection**:
   - Calculate health based on metrics:
     - `critical`: stuck > 10 OR pending > 1000
     - `degraded`: stuck > 5 OR pending > 500
     - `healthy`: otherwise

3. **Insert**:
   - INSERT into `queue_depth_snapshots` table

#### Example

```typescript
// Record queue snapshot
await collector.recordQueueSnapshot({
  queue_name: "statement_processing",
  pending: 234,
  in_progress: 12,
  stuck: 2,
  health_status: "healthy"
});

// Record critical queue state
await collector.recordQueueSnapshot({
  queue_name: "ocr_processing",
  pending: 1500,
  in_progress: 20,
  stuck: 15,
  health_status: "critical"
});
```

#### Performance

- **Target Latency**: <15ms p95
- **Frequency**: Every 60 seconds (configurable)

---

### 4. recordError()

Record error with full context for debugging.

#### Signature

```typescript
recordError(error: ErrorLog): Promise<string>
```

#### Behavior

1. **Validation**:
   - Validate error_type is not empty
   - Validate error_message is not empty
   - Validate severity is valid enum value

2. **Context Enrichment**:
   - Add timestamp
   - Add hostname
   - Add process ID

3. **Insert**:
   - INSERT into `error_logs` table

#### Example

```typescript
// Record validation error
await collector.recordError({
  error_type: "VALIDATION_ERROR",
  error_message: "Invalid date format in transaction",
  context: {
    parser_id: "chase_pdf_parser_v2",
    upload_id: "upload_123",
    field: "transaction_date",
    value: "13/45/2025"
  },
  severity: "error",
  stack_trace: null
});

// Record critical error with stack trace
await collector.recordError({
  error_type: "DATABASE_CONNECTION_ERROR",
  error_message: "Failed to connect to PostgreSQL",
  context: {
    service: "parser_worker",
    host: "db.example.com",
    port: 5432
  },
  severity: "critical",
  stack_trace: "Error: Connection refused\n  at Socket.connect (...)"
});
```

#### Performance

- **Target Latency**: <20ms p95
- **Throughput**: 1K errors/sec

---

### 5. batchRecordParserExecutions()

Batch record multiple parser executions at once.

#### Signature

```typescript
batchRecordParserExecutions(
  metrics: ParserExecutionMetrics[]
): Promise<string[]>
```

#### Behavior

1. **Validation**:
   - Validate array is not empty
   - Validate each metric individually
   - Validate batch size <= 1000

2. **Bulk Insert**:
   - INSERT all metrics in single query
   - Use PostgreSQL COPY or multi-row INSERT

3. **Return**:
   - Return array of metric IDs

#### Example

```typescript
// Batch record parser executions
const metrics = [
  {
    parser_id: "chase_parser",
    upload_id: "upload_1",
    duration_ms: 1200,
    status: "success",
    observations_extracted: 85
  },
  {
    parser_id: "wells_fargo_parser",
    upload_id: "upload_2",
    duration_ms: 950,
    status: "success",
    observations_extracted: 42
  },
  // ... 998 more metrics
];

const metricIds = await collector.batchRecordParserExecutions(metrics);
console.log(`Recorded ${metricIds.length} metrics`);
```

#### Performance

- **Target Latency**: <100ms for 1000 metrics
- **Throughput**: 10K metrics/sec

---

### 6. flush()

Flush buffered metrics to storage immediately.

#### Signature

```typescript
flush(): Promise<void>
```

#### Behavior

1. **Drain Buffer**:
   - Take all buffered metrics
   - Batch insert to database

2. **Wait**:
   - Wait for all inserts to complete

3. **Return**:
   - Return when flush is complete

#### Example

```typescript
// Collect metrics
await collector.recordParserExecution({ ... });
await collector.recordRuleExecution({ ... });

// Flush before shutdown
await collector.flush();
console.log("All metrics flushed to storage");
```

#### Performance

- **Target Latency**: <500ms (depends on buffer size)

---

## Batch Processing

### Batch Insert Strategy

**Trigger Conditions**:
1. Buffer reaches 100 metrics (configurable)
2. 5 seconds elapsed since last flush (configurable)
3. Manual flush() call

**Batch Size**: 100-1000 metrics per batch (configurable)

**Implementation**:
```typescript
class BufferedMetricsCollector implements MetricsCollector {
  private buffer: ParserExecutionMetrics[] = [];
  private batchSize = 100;
  private flushInterval = 5000; // 5 seconds

  async recordParserExecution(
    metrics: ParserExecutionMetrics
  ): Promise<string> {
    // Add to buffer
    this.buffer.push(metrics);

    // Trigger batch insert if threshold reached
    if (this.buffer.length >= this.batchSize) {
      await this.flushBuffer();
    }

    return "pending"; // Return immediately
  }

  private async flushBuffer(): Promise<void> {
    if (this.buffer.length === 0) return;

    const batch = this.buffer.splice(0, this.batchSize);

    // Bulk insert
    const query = `
      INSERT INTO parser_execution_metrics
      (parser_id, upload_id, duration_ms, status, observations_extracted, metadata)
      SELECT * FROM json_populate_recordset(NULL::parser_execution_metrics, $1)
    `;

    await this.pool.query(query, [JSON.stringify(batch)]);
  }

  // Start periodic flush timer
  private startFlushTimer(): void {
    setInterval(() => {
      this.flushBuffer().catch(err => {
        console.error("Failed to flush buffer:", err);
      });
    }, this.flushInterval);
  }
}
```

---

## Metric Types

### ExecutionStatus Enum

```typescript
enum ExecutionStatus {
  SUCCESS = "success",
  FAILURE = "failure",
  TIMEOUT = "timeout",
  CANCELLED = "cancelled"
}
```

### RuleType Enum

```typescript
enum RuleType {
  NORMALIZATION = "normalization",
  VALIDATION = "validation",
  CLASSIFICATION = "classification",
  EXTRACTION = "extraction"
}
```

### HealthStatus Enum

```typescript
enum HealthStatus {
  HEALTHY = "healthy",
  DEGRADED = "degraded",
  CRITICAL = "critical"
}
```

### Severity Enum

```typescript
enum Severity {
  DEBUG = "debug",
  INFO = "info",
  WARNING = "warning",
  ERROR = "error",
  CRITICAL = "critical"
}
```

---

## Storage & Indexing

### Time-Series Partitioning

Partition tables by month for efficient queries:

```sql
-- Create monthly partition
CREATE TABLE parser_execution_metrics_2025_01
PARTITION OF parser_execution_metrics
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Automatic partition creation (using pg_partman or custom function)
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
  partition_date DATE;
  partition_name TEXT;
BEGIN
  partition_date := DATE_TRUNC('month', NOW() + INTERVAL '1 month');
  partition_name := 'parser_execution_metrics_' || TO_CHAR(partition_date, 'YYYY_MM');

  EXECUTE format(
    'CREATE TABLE IF NOT EXISTS %I PARTITION OF parser_execution_metrics
     FOR VALUES FROM (%L) TO (%L)',
    partition_name,
    partition_date,
    partition_date + INTERVAL '1 month'
  );
END;
$$ LANGUAGE plpgsql;
```

### Index Strategy

**B-tree Indexes**: For exact matches and range queries
```sql
CREATE INDEX idx_parser_metrics_created_at ON parser_execution_metrics(created_at DESC);
CREATE INDEX idx_parser_metrics_parser_id ON parser_execution_metrics(parser_id);
```

**GIN Indexes**: For JSONB columns
```sql
CREATE INDEX idx_parser_metrics_metadata ON parser_execution_metrics USING GIN(metadata);
```

**Composite Indexes**: For common query patterns
```sql
CREATE INDEX idx_parser_metrics_composite ON parser_execution_metrics(parser_id, created_at DESC);
```

### Retention Policy

```sql
-- Delete metrics older than 1 year
DELETE FROM parser_execution_metrics
WHERE created_at < NOW() - INTERVAL '1 year';

-- Or drop old partitions
DROP TABLE parser_execution_metrics_2024_01;
```

---

## Edge Cases

### Edge Case 1: Buffer Overflow

**Scenario**: Metrics collected faster than they can be flushed.

**Detection**:
```typescript
if (this.buffer.length > this.maxBufferSize) {
  console.warn(`Buffer overflow: ${this.buffer.length} metrics buffered`);
}
```

**Resolution**:
- Increase batch size
- Decrease flush interval
- Apply backpressure to producers
- Log dropped metrics for analysis

**Example**:
```typescript
async recordParserExecution(
  metrics: ParserExecutionMetrics
): Promise<string> {
  if (this.buffer.length >= this.maxBufferSize) {
    // Backpressure: flush synchronously
    await this.flushBuffer();
  }

  this.buffer.push(metrics);
  return "buffered";
}
```

---

### Edge Case 2: Database Connection Failure

**Scenario**: Cannot connect to PostgreSQL during metric insert.

**Detection**:
```typescript
try {
  await this.pool.query(query, values);
} catch (error) {
  if (error.code === 'ECONNREFUSED') {
    console.error("Database connection failed");
  }
}
```

**Resolution**:
- Retry with exponential backoff (3 attempts)
- Persist failed metrics to disk
- Alert operations team
- Continue collecting metrics in-memory

**Example**:
```typescript
async flushBuffer(): Promise<void> {
  const maxRetries = 3;
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      await this.bulkInsert(this.buffer);
      return;
    } catch (error) {
      attempt++;
      if (attempt >= maxRetries) {
        // Persist to disk
        await this.persistToDisk(this.buffer);
        throw error;
      }
      // Exponential backoff
      await sleep(Math.pow(2, attempt) * 1000);
    }
  }
}
```

---

### Edge Case 3: Invalid Metric Data

**Scenario**: Metric contains invalid or malformed data.

**Detection**:
```typescript
if (!metrics.parser_id || metrics.duration_ms < 0) {
  throw new ValidationError("Invalid metric data");
}
```

**Resolution**:
- Validate before buffering
- Log validation errors
- Skip invalid metrics
- Alert if validation error rate exceeds threshold

**Example**:
```typescript
async recordParserExecution(
  metrics: ParserExecutionMetrics
): Promise<string> {
  // Validation
  if (!metrics.parser_id) {
    await this.recordError({
      error_type: "INVALID_METRIC",
      error_message: "Missing parser_id",
      context: { metrics },
      severity: "warning"
    });
    throw new ValidationError("parser_id is required");
  }

  if (metrics.duration_ms < 0) {
    throw new ValidationError("duration_ms must be >= 0");
  }

  this.buffer.push(metrics);
  return "buffered";
}
```

---

### Edge Case 4: Clock Skew

**Scenario**: Server clocks are out of sync, causing incorrect timestamps.

**Detection**:
```typescript
const serverTime = new Date();
const metricTime = new Date(metrics.created_at);

if (Math.abs(serverTime.getTime() - metricTime.getTime()) > 60000) {
  console.warn("Clock skew detected: " + (serverTime.getTime() - metricTime.getTime()) + "ms");
}
```

**Resolution**:
- Use database timestamps (NOW()) instead of client timestamps
- Synchronize server clocks with NTP
- Log clock skew warnings

**Example**:
```sql
-- Use database timestamp
INSERT INTO parser_execution_metrics (parser_id, upload_id, duration_ms, status, created_at)
VALUES ($1, $2, $3, $4, NOW()); -- Always use NOW()
```

---

### Edge Case 5: Extremely Large Metadata

**Scenario**: Metadata JSONB field exceeds reasonable size (>1MB).

**Detection**:
```typescript
const metadataSize = Buffer.byteLength(JSON.stringify(metrics.metadata), 'utf8');

if (metadataSize > 1048576) { // 1MB
  console.warn(`Metadata too large: ${metadataSize} bytes`);
}
```

**Resolution**:
- Truncate large metadata fields
- Store large data separately (S3, etc.)
- Log warning for oversized metadata

**Example**:
```typescript
function truncateMetadata(metadata: any, maxSize: number = 102400): any {
  const json = JSON.stringify(metadata);
  if (Buffer.byteLength(json, 'utf8') <= maxSize) {
    return metadata;
  }

  // Truncate
  return {
    ...metadata,
    _truncated: true,
    _original_size: Buffer.byteLength(json, 'utf8')
  };
}
```

---

### Edge Case 6: High Cardinality Parser IDs

**Scenario**: Thousands of unique parser_id values, causing index bloat.

**Detection**:
```sql
SELECT COUNT(DISTINCT parser_id) FROM parser_execution_metrics;
-- Result: 50,000+ unique parser IDs
```

**Resolution**:
- Normalize parser IDs (remove version suffixes)
- Use separate dimension table for parser metadata
- Partition by parser_id_hash

**Example**:
```sql
-- Create parser dimension table
CREATE TABLE parsers (
  parser_id VARCHAR(128) PRIMARY KEY,
  parser_name VARCHAR(128),
  parser_version VARCHAR(32),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Reference by ID in metrics table
ALTER TABLE parser_execution_metrics
ADD CONSTRAINT fk_parser_id
FOREIGN KEY (parser_id) REFERENCES parsers(parser_id);
```

---

### Edge Case 7: Duplicate Metrics

**Scenario**: Same metric recorded multiple times due to retries.

**Detection**:
```sql
-- Find duplicates
SELECT parser_id, upload_id, created_at, COUNT(*)
FROM parser_execution_metrics
GROUP BY parser_id, upload_id, created_at
HAVING COUNT(*) > 1;
```

**Resolution**:
- Add unique constraint on (parser_id, upload_id, created_at)
- Use idempotent inserts (INSERT ... ON CONFLICT DO NOTHING)
- Deduplicate during query time

**Example**:
```sql
-- Add unique constraint
ALTER TABLE parser_execution_metrics
ADD CONSTRAINT uq_parser_execution
UNIQUE (parser_id, upload_id, created_at);

-- Idempotent insert
INSERT INTO parser_execution_metrics (parser_id, upload_id, duration_ms, status)
VALUES ($1, $2, $3, $4)
ON CONFLICT (parser_id, upload_id, created_at) DO NOTHING;
```

---

### Edge Case 8: Time Zone Confusion

**Scenario**: Timestamps in different time zones cause incorrect queries.

**Detection**:
```typescript
// Check if timestamp has timezone
if (!metrics.created_at.includes('Z') && !metrics.created_at.includes('+')) {
  console.warn("Timestamp missing timezone info");
}
```

**Resolution**:
- Always use TIMESTAMP WITH TIME ZONE
- Store all timestamps in UTC
- Convert to local timezone only for display

**Example**:
```typescript
// Always use UTC
const timestamp = new Date().toISOString(); // Includes 'Z' suffix

await collector.recordParserExecution({
  parser_id: "chase_parser",
  upload_id: "upload_123",
  duration_ms: 1200,
  status: "success",
  observations_extracted: 87,
  metadata: {
    timestamp // UTC timestamp
  }
});
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `recordParserExecution()` | 1 metric | < 10ms | Async buffering |
| `recordRuleExecution()` | 1 metric | < 8ms | Async buffering |
| `recordQueueSnapshot()` | 1 snapshot | < 15ms | Direct insert |
| `recordError()` | 1 error | < 20ms | Direct insert |
| `batchRecordParserExecutions()` | 1000 metrics | < 100ms | Bulk insert |
| `flush()` | All buffered | < 500ms | Depends on buffer size |

### Throughput

- **Parser Metrics**: 10K metrics/sec (with batching)
- **Rule Metrics**: 15K metrics/sec (with batching)
- **Queue Snapshots**: 100 snapshots/sec
- **Error Logs**: 1K errors/sec

### Scalability

| Metrics Stored | Query Performance | Notes |
|---------------|------------------|-------|
| < 1M | Excellent (< 50ms) | All queries fast |
| 1M - 10M | Excellent (< 100ms) | Indexed queries fast |
| 10M - 100M | Good (< 500ms) | Aggregations slower |
| > 100M | Moderate (< 2s) | Use partitioning + pre-aggregation |

### Storage Efficiency

- **Compressed Size**: ~500 bytes per metric (JSONB compression)
- **Index Overhead**: ~200 bytes per metric
- **Total**: ~700 bytes per metric
- **1M metrics**: ~700MB storage

### Optimization Tips

**1. Increase Batch Size**

```typescript
// Bad: Small batches (high overhead)
collector.batchSize = 10;

// Good: Larger batches (better throughput)
collector.batchSize = 1000;
```

**2. Use Asynchronous Collection**

```typescript
// Bad: Synchronous (blocks pipeline)
await collector.recordParserExecution(metrics);

// Good: Fire-and-forget (non-blocking)
collector.recordParserExecution(metrics).catch(err => {
  console.error("Failed to record metric:", err);
});
```

**3. Pre-Aggregate Common Queries**

```sql
-- Create materialized view for hourly aggregates
CREATE MATERIALIZED VIEW parser_metrics_hourly AS
SELECT
  parser_id,
  DATE_TRUNC('hour', created_at) AS hour,
  COUNT(*) AS executions,
  AVG(duration_ms) AS avg_duration_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_duration_ms
FROM parser_execution_metrics
GROUP BY parser_id, hour;

-- Refresh periodically
REFRESH MATERIALIZED VIEW parser_metrics_hourly;
```

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';

export class PostgresMetricsCollector implements MetricsCollector {
  private pool: Pool;
  private buffer: ParserExecutionMetrics[] = [];
  private batchSize = 100;
  private maxBufferSize = 10000;
  private flushInterval = 5000; // 5 seconds
  private flushTimer: NodeJS.Timer;

  constructor(pool: Pool) {
    this.pool = pool;
    this.startFlushTimer();
  }

  async recordParserExecution(
    metrics: ParserExecutionMetrics
  ): Promise<string> {
    // Validation
    this.validateParserMetrics(metrics);

    // Add to buffer
    this.buffer.push(metrics);

    // Trigger flush if threshold reached
    if (this.buffer.length >= this.batchSize) {
      await this.flushBuffer();
    }

    return "buffered";
  }

  async recordRuleExecution(
    metrics: RuleExecutionMetrics
  ): Promise<string> {
    // Validation
    this.validateRuleMetrics(metrics);

    // Direct insert (rules are less frequent than parsers)
    const query = `
      INSERT INTO rule_execution_metrics
      (rule_id, rule_type, execution_time_ms, matched, transformation_applied, context)
      VALUES ($1, $2, $3, $4, $5, $6)
      RETURNING metric_id
    `;

    const values = [
      metrics.rule_id,
      metrics.rule_type,
      metrics.execution_time_ms,
      metrics.matched,
      metrics.transformation_applied,
      JSON.stringify(metrics.context || {})
    ];

    const result = await this.pool.query(query, values);
    return result.rows[0].metric_id;
  }

  async recordQueueSnapshot(
    snapshot: QueueDepthSnapshot
  ): Promise<string> {
    // Validation
    if (!snapshot.queue_name) {
      throw new ValidationError("queue_name is required");
    }

    // Insert
    const query = `
      INSERT INTO queue_depth_snapshots
      (queue_name, pending, in_progress, stuck, health_status)
      VALUES ($1, $2, $3, $4, $5)
      RETURNING snapshot_id
    `;

    const values = [
      snapshot.queue_name,
      snapshot.pending,
      snapshot.in_progress,
      snapshot.stuck,
      snapshot.health_status
    ];

    const result = await this.pool.query(query, values);
    return result.rows[0].snapshot_id;
  }

  async recordError(error: ErrorLog): Promise<string> {
    // Validation
    if (!error.error_type || !error.error_message) {
      throw new ValidationError("error_type and error_message are required");
    }

    // Insert
    const query = `
      INSERT INTO error_logs
      (error_type, error_message, context, severity, stack_trace)
      VALUES ($1, $2, $3, $4, $5)
      RETURNING error_id
    `;

    const values = [
      error.error_type,
      error.error_message,
      JSON.stringify(error.context),
      error.severity,
      error.stack_trace
    ];

    const result = await this.pool.query(query, values);
    return result.rows[0].error_id;
  }

  async batchRecordParserExecutions(
    metrics: ParserExecutionMetrics[]
  ): Promise<string[]> {
    if (metrics.length === 0) {
      return [];
    }

    if (metrics.length > 1000) {
      throw new ValidationError("Batch size exceeds maximum (1000)");
    }

    // Validate all metrics
    metrics.forEach(m => this.validateParserMetrics(m));

    // Bulk insert using json_populate_recordset
    const query = `
      INSERT INTO parser_execution_metrics
      (parser_id, upload_id, duration_ms, status, observations_extracted, error_type, metadata)
      SELECT parser_id, upload_id, duration_ms, status, observations_extracted, error_type, metadata
      FROM json_populate_recordset(NULL::parser_execution_metrics, $1)
      RETURNING metric_id
    `;

    const result = await this.pool.query(query, [JSON.stringify(metrics)]);
    return result.rows.map(row => row.metric_id);
  }

  async flush(): Promise<void> {
    await this.flushBuffer();
  }

  getStats(): CollectorStats {
    // Implementation omitted for brevity
    return {
      metrics_collected_total: 0,
      metrics_per_second: 0,
      ingestion_latency_p50_ms: 0,
      ingestion_latency_p95_ms: 0,
      ingestion_latency_p99_ms: 0,
      buffer_size: this.buffer.length,
      buffer_capacity: this.maxBufferSize,
      buffer_utilization_percent: (this.buffer.length / this.maxBufferSize) * 100,
      collection_errors_total: 0
    };
  }

  private async flushBuffer(): Promise<void> {
    if (this.buffer.length === 0) return;

    const batch = this.buffer.splice(0, this.batchSize);

    try {
      await this.batchRecordParserExecutions(batch);
    } catch (error) {
      console.error("Failed to flush buffer:", error);
      // Re-add to buffer
      this.buffer.unshift(...batch);
    }
  }

  private startFlushTimer(): void {
    this.flushTimer = setInterval(() => {
      this.flushBuffer().catch(err => {
        console.error("Periodic flush failed:", err);
      });
    }, this.flushInterval);
  }

  private validateParserMetrics(metrics: ParserExecutionMetrics): void {
    if (!metrics.parser_id) {
      throw new ValidationError("parser_id is required");
    }
    if (!metrics.upload_id) {
      throw new ValidationError("upload_id is required");
    }
    if (metrics.duration_ms < 0) {
      throw new ValidationError("duration_ms must be >= 0");
    }
    if (metrics.observations_extracted < 0) {
      throw new ValidationError("observations_extracted must be >= 0");
    }
  }

  private validateRuleMetrics(metrics: RuleExecutionMetrics): void {
    if (!metrics.rule_id) {
      throw new ValidationError("rule_id is required");
    }
    if (!metrics.rule_type) {
      throw new ValidationError("rule_type is required");
    }
    if (metrics.execution_time_ms < 0) {
      throw new ValidationError("execution_time_ms must be >= 0");
    }
  }
}
```

---

## Security Considerations

### 1. Input Validation

**Prevent SQL Injection**:
```typescript
// Always use parameterized queries
const query = `
  INSERT INTO parser_execution_metrics (parser_id, upload_id, duration_ms)
  VALUES ($1, $2, $3)
`;

await this.pool.query(query, [metrics.parser_id, metrics.upload_id, metrics.duration_ms]);
```

### 2. Rate Limiting

**Prevent DoS Attacks**:
```typescript
class RateLimitedCollector implements MetricsCollector {
  private requestCounts = new Map<string, number>();
  private maxRequestsPerMinute = 10000;

  async recordParserExecution(
    metrics: ParserExecutionMetrics
  ): Promise<string> {
    const key = metrics.parser_id;
    const count = this.requestCounts.get(key) || 0;

    if (count >= this.maxRequestsPerMinute) {
      throw new RateLimitError(`Rate limit exceeded for parser ${key}`);
    }

    this.requestCounts.set(key, count + 1);

    return this.collector.recordParserExecution(metrics);
  }
}
```

### 3. Data Encryption

**Encrypt Sensitive Context**:
```typescript
import { encrypt, decrypt } from './encryption';

async recordError(error: ErrorLog): Promise<string> {
  // Encrypt sensitive context data
  const encryptedContext = encrypt(JSON.stringify(error.context));

  const query = `
    INSERT INTO error_logs (error_type, error_message, context, severity)
    VALUES ($1, $2, $3, $4)
    RETURNING error_id
  `;

  const values = [
    error.error_type,
    error.error_message,
    encryptedContext,
    error.severity
  ];

  const result = await this.pool.query(query, values);
  return result.rows[0].error_id;
}
```

### 4. Access Control

**Role-Based Metric Collection**:
```typescript
async recordParserExecution(
  metrics: ParserExecutionMetrics,
  user: User
): Promise<string> {
  // Check if user has permission to record metrics
  if (!user.hasRole('metrics_writer')) {
    throw new UnauthorizedError('User does not have permission to record metrics');
  }

  return this.collector.recordParserExecution(metrics);
}
```

---

## Integration Patterns

### Pattern 1: Automatic Instrumentation

```typescript
// Wrap parser execution with automatic metric collection
class InstrumentedParser implements Parser {
  constructor(
    private parser: Parser,
    private collector: MetricsCollector
  ) {}

  async parse(upload: Upload): Promise<Observation[]> {
    const startTime = Date.now();
    let status: ExecutionStatus = "success";
    let observations: Observation[] = [];

    try {
      observations = await this.parser.parse(upload);
      return observations;
    } catch (error) {
      status = "failure";

      // Record error
      await this.collector.recordError({
        error_type: error.constructor.name,
        error_message: error.message,
        context: {
          parser_id: this.parser.id,
          upload_id: upload.id
        },
        severity: "error",
        stack_trace: error.stack
      });

      throw error;
    } finally {
      const duration = Date.now() - startTime;

      // Record parser execution
      await this.collector.recordParserExecution({
        parser_id: this.parser.id,
        upload_id: upload.id,
        duration_ms: duration,
        status,
        observations_extracted: observations.length
      });
    }
  }
}
```

### Pattern 2: Middleware Integration

```typescript
// Express middleware for API request metrics
function metricsMiddleware(collector: MetricsCollector) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const startTime = Date.now();

    // Continue to next handler
    res.on('finish', async () => {
      const duration = Date.now() - startTime;

      await collector.recordParserExecution({
        parser_id: `api_${req.method}_${req.path}`,
        upload_id: req.headers['x-request-id'] as string,
        duration_ms: duration,
        status: res.statusCode < 400 ? "success" : "failure",
        observations_extracted: 1,
        metadata: {
          method: req.method,
          path: req.path,
          status_code: res.statusCode
        }
      });
    });

    next();
  };
}

// Usage
app.use(metricsMiddleware(collector));
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section with 7 domains: Finance, Healthcare, Legal, Research, E-commerce, SaaS, Insurance)

---

## Testing Strategy

### Unit Tests

```typescript
describe('MetricsCollector', () => {
  let collector: MetricsCollector;
  let pool: Pool;

  beforeEach(async () => {
    pool = new Pool({ /* test config */ });
    collector = new PostgresMetricsCollector(pool);
    await cleanDatabase();
  });

  describe('recordParserExecution()', () => {
    it('should record valid parser execution', async () => {
      const metricId = await collector.recordParserExecution({
        parser_id: 'test_parser',
        upload_id: 'upload_123',
        duration_ms: 1000,
        status: 'success',
        observations_extracted: 50
      });

      expect(metricId).toBeDefined();
    });

    it('should reject negative duration', async () => {
      await expect(
        collector.recordParserExecution({
          parser_id: 'test_parser',
          upload_id: 'upload_123',
          duration_ms: -100,
          status: 'success',
          observations_extracted: 0
        })
      ).rejects.toThrow('duration_ms must be >= 0');
    });

    it('should reject missing parser_id', async () => {
      await expect(
        collector.recordParserExecution({
          parser_id: '',
          upload_id: 'upload_123',
          duration_ms: 1000,
          status: 'success',
          observations_extracted: 0
        })
      ).rejects.toThrow('parser_id is required');
    });
  });

  describe('batchRecordParserExecutions()', () => {
    it('should batch insert 1000 metrics', async () => {
      const metrics = Array.from({ length: 1000 }, (_, i) => ({
        parser_id: 'test_parser',
        upload_id: `upload_${i}`,
        duration_ms: 1000,
        status: 'success' as ExecutionStatus,
        observations_extracted: 50
      }));

      const metricIds = await collector.batchRecordParserExecutions(metrics);
      expect(metricIds.length).toBe(1000);
    });

    it('should reject batch larger than 1000', async () => {
      const metrics = Array.from({ length: 1001 }, (_, i) => ({
        parser_id: 'test_parser',
        upload_id: `upload_${i}`,
        duration_ms: 1000,
        status: 'success' as ExecutionStatus,
        observations_extracted: 50
      }));

      await expect(
        collector.batchRecordParserExecutions(metrics)
      ).rejects.toThrow('Batch size exceeds maximum');
    });
  });

  describe('recordQueueSnapshot()', () => {
    it('should record queue snapshot', async () => {
      const snapshotId = await collector.recordQueueSnapshot({
        queue_name: 'test_queue',
        pending: 100,
        in_progress: 10,
        stuck: 2,
        health_status: 'healthy'
      });

      expect(snapshotId).toBeDefined();
    });
  });

  describe('recordError()', () => {
    it('should record error with context', async () => {
      const errorId = await collector.recordError({
        error_type: 'VALIDATION_ERROR',
        error_message: 'Invalid data',
        context: {
          parser_id: 'test_parser',
          upload_id: 'upload_123'
        },
        severity: 'error'
      });

      expect(errorId).toBeDefined();
    });
  });
});
```

### Integration Tests

```typescript
describe('MetricsCollector Integration', () => {
  it('should collect metrics during parser execution', async () => {
    const parser = new ChaseParser();
    const collector = new MetricsCollector(pool);
    const instrumentedParser = new InstrumentedParser(parser, collector);

    // Parse document
    const observations = await instrumentedParser.parse(upload);

    // Verify metric was recorded
    const query = `
      SELECT * FROM parser_execution_metrics
      WHERE parser_id = $1 AND upload_id = $2
    `;

    const result = await pool.query(query, [parser.id, upload.id]);
    expect(result.rows.length).toBe(1);

    const metric = result.rows[0];
    expect(metric.status).toBe('success');
    expect(metric.observations_extracted).toBe(observations.length);
  });
});
```

### Performance Tests

```typescript
describe('MetricsCollector Performance', () => {
  it('should handle 10K metrics/sec', async () => {
    const collector = new MetricsCollector(pool);
    const startTime = Date.now();
    const metricsCount = 10000;

    // Generate metrics
    const metrics = Array.from({ length: metricsCount }, (_, i) => ({
      parser_id: 'perf_test_parser',
      upload_id: `upload_${i}`,
      duration_ms: Math.random() * 1000,
      status: 'success' as ExecutionStatus,
      observations_extracted: Math.floor(Math.random() * 100)
    }));

    // Batch insert
    await collector.batchRecordParserExecutions(metrics);

    const duration = Date.now() - startTime;
    const throughput = (metricsCount / duration) * 1000;

    console.log(`Throughput: ${throughput.toFixed(0)} metrics/sec`);
    expect(throughput).toBeGreaterThan(10000);
  });

  it('should maintain <10ms p95 latency', async () => {
    const collector = new MetricsCollector(pool);
    const latencies: number[] = [];

    // Collect 1000 metrics
    for (let i = 0; i < 1000; i++) {
      const startTime = Date.now();

      await collector.recordParserExecution({
        parser_id: 'latency_test_parser',
        upload_id: `upload_${i}`,
        duration_ms: 1000,
        status: 'success',
        observations_extracted: 50
      });

      latencies.push(Date.now() - startTime);
    }

    // Calculate p95
    latencies.sort((a, b) => a - b);
    const p95Index = Math.floor(latencies.length * 0.95);
    const p95Latency = latencies[p95Index];

    console.log(`p95 latency: ${p95Latency}ms`);
    expect(p95Latency).toBeLessThan(10);
  });
});
```

---

## Configuration

### Tunable Parameters

```typescript
interface CollectorConfig {
  // Buffer Settings
  batchSize: number;              // Default: 100
  maxBufferSize: number;          // Default: 10000
  flushInterval: number;          // Default: 5000 (5 seconds)

  // Performance Settings
  asyncWrites: boolean;           // Default: true
  compressionEnabled: boolean;    // Default: true

  // Retry Settings
  maxRetries: number;             // Default: 3
  retryBackoffMs: number;         // Default: 1000

  // Storage Settings
  partitioningEnabled: boolean;   // Default: true
  retentionDays: number;          // Default: 365
}
```

### Configuration Examples

```typescript
// High-throughput configuration
const highThroughputConfig: CollectorConfig = {
  batchSize: 1000,
  maxBufferSize: 50000,
  flushInterval: 10000,
  asyncWrites: true,
  compressionEnabled: true,
  maxRetries: 3,
  retryBackoffMs: 1000,
  partitioningEnabled: true,
  retentionDays: 365
};

// Low-latency configuration
const lowLatencyConfig: CollectorConfig = {
  batchSize: 10,
  maxBufferSize: 1000,
  flushInterval: 1000,
  asyncWrites: false,
  compressionEnabled: false,
  maxRetries: 1,
  retryBackoffMs: 100,
  partitioningEnabled: false,
  retentionDays: 90
};

// Create collector with config
const collector = new PostgresMetricsCollector(pool, highThroughputConfig);
```

---

## Observability

### Metrics for the Collector Itself

```typescript
interface CollectorMetrics {
  // Throughput
  metrics_collected_total: Counter;
  metrics_per_second: Gauge;

  // Latency
  ingestion_latency_ms: Histogram;

  // Buffer
  buffer_size: Gauge;
  buffer_flushes_total: Counter;

  // Errors
  collection_errors_total: Counter;
  flush_errors_total: Counter;
}
```

### Logging

```typescript
class LoggingMetricsCollector implements MetricsCollector {
  async recordParserExecution(
    metrics: ParserExecutionMetrics
  ): Promise<string> {
    logger.debug('Recording parser execution', {
      parser_id: metrics.parser_id,
      upload_id: metrics.upload_id,
      duration_ms: metrics.duration_ms
    });

    try {
      const metricId = await this.collector.recordParserExecution(metrics);

      logger.debug('Parser execution recorded', {
        metric_id: metricId
      });

      return metricId;
    } catch (error) {
      logger.error('Failed to record parser execution', {
        error: error.message,
        parser_id: metrics.parser_id,
        upload_id: metrics.upload_id
      });

      throw error;
    }
  }
}
```

### Tracing

```typescript
import { trace } from '@opentelemetry/api';

class TracingMetricsCollector implements MetricsCollector {
  async recordParserExecution(
    metrics: ParserExecutionMetrics
  ): Promise<string> {
    const tracer = trace.getTracer('metrics-collector');

    return tracer.startActiveSpan('record_parser_execution', async (span) => {
      span.setAttribute('parser_id', metrics.parser_id);
      span.setAttribute('upload_id', metrics.upload_id);

      try {
        const metricId = await this.collector.recordParserExecution(metrics);

        span.setStatus({ code: SpanStatusCode.OK });
        return metricId;
      } catch (error) {
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error.message
        });

        throw error;
      } finally {
        span.end();
      }
    });
  }
}
```

---

## Simplicity Profiles

**Personal - ~15 LOC:** Print statements for debugging (no metrics)
**Small Business - ~70 LOC:** Simple counters, log to file
**Enterprise - ~350 LOC:** Prometheus metrics, Grafana dashboards, alerting

---

## Future Enhancements

### Phase 1: Advanced Analytics (3 months)

**Features**:
- Real-time anomaly detection for parser performance
- Automatic alert generation for degraded parsers
- Predictive capacity forecasting

**Implementation**:
```typescript
class AnomalyDetector {
  async detectAnomalies(
    parser_id: string,
    timeWindow: string = '1 hour'
  ): Promise<Anomaly[]> {
    // Query recent metrics
    const metrics = await this.queryMetrics(parser_id, timeWindow);

    // Calculate z-score for duration
    const mean = this.calculateMean(metrics.map(m => m.duration_ms));
    const stdDev = this.calculateStdDev(metrics.map(m => m.duration_ms));

    // Detect anomalies (z-score > 2.5)
    const anomalies = metrics.filter(m => {
      const zScore = Math.abs((m.duration_ms - mean) / stdDev);
      return zScore > 2.5;
    });

    return anomalies;
  }
}
```

### Phase 2: Distributed Tracing (6 months)

**Features**:
- End-to-end request tracing across parsers and rules
- Trace ID propagation through pipeline
- Distributed context correlation

**Implementation**:
```typescript
interface DistributedMetrics extends ParserExecutionMetrics {
  trace_id: string;               // Distributed trace ID
  span_id: string;                // Span ID
  parent_span_id?: string;        // Parent span ID
}
```

### Phase 3: Machine Learning Insights (12 months)

**Features**:
- ML-powered performance regression detection
- Automatic root cause analysis for errors
- Capacity planning recommendations

**Implementation**:
```typescript
class MLInsights {
  async detectPerformanceRegression(
    parser_id: string
  ): Promise<RegressionReport> {
    // Train model on historical data
    const model = await this.trainModel(parser_id);

    // Predict expected performance
    const prediction = model.predict(currentMetrics);

    // Compare with actual performance
    const regression = this.comparePerformance(prediction, currentMetrics);

    return regression;
  }
}
```

---

**End of MetricsCollector OL Primitive Specification**
