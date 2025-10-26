# QueueMonitor OL Primitive

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
8. [Health Indicators](#health-indicators)
9. [Alert Thresholds](#alert-thresholds)
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

The **QueueMonitor** provides real-time visibility into document processing queue health. It monitors queue depths, detects stuck documents, tracks processing rates, and generates alerts when queues become unhealthy. This primitive enables proactive identification of bottlenecks and backlog issues before they impact SLAs.

### Key Capabilities

- **Real-Time Queue Depth**: <20ms p95 current queue depth query
- **Backlog Detection**: Alert when pending count exceeds 500 documents
- **Stuck Document Detection**: Identify documents with >5 failed retries
- **Processing Rate Tracking**: Measure documents processed per minute
- **Health Status**: Automatic health classification (healthy, degraded, critical)
- **Historical Snapshots**: Snapshot queue state every 60 seconds

### Design Philosophy

The QueueMonitor follows four core principles:

1. **Real-Time Visibility**: Instant access to current queue state
2. **Proactive Alerting**: Detect issues before they impact users
3. **Minimal Overhead**: <20ms monitoring queries
4. **Historical Context**: Track queue trends over time

---

## Purpose & Scope

### Problem Statement

In document processing pipelines, queue monitoring poses critical challenges:

- **No Visibility**: Can't see queue depths or stuck documents
- **Reactive Response**: Issues discovered after SLA breaches
- **No Backlog Alerts**: Queue backlogs grow unnoticed
- **Stuck Documents**: Documents fail repeatedly without intervention
- **No Processing Rate**: Can't measure throughput or forecast completion

Traditional approaches have significant limitations:

**Approach 1: Application Logs**
- ❌ Scattered across log files
- ❌ No aggregation or trends
- ❌ Manual analysis required

**Approach 2: Database Queries**
- ❌ Slow queries (1-5 seconds)
- ❌ No real-time updates
- ❌ No health indicators

**Approach 3: Queue Monitor**
- ✅ Real-time queue depth (<20ms)
- ✅ Automatic health classification
- ✅ Stuck document detection
- ✅ Processing rate metrics
- ✅ Historical trends

### Solution

The QueueMonitor implements a **real-time queue observability system** with:

1. **Snapshot Collection**: Record queue state every 60 seconds
2. **Fast Queries**: Indexed queries for <20ms response
3. **Health Classification**: Automatic status (healthy/degraded/critical)
4. **Stuck Detection**: Identify documents stuck in processing
5. **Rate Calculation**: Measure processing throughput

---

## Multi-Domain Applicability

The QueueMonitor is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Monitor bank statement processing queues and detect backlog during month-end.

**Queue Types**:
- `statement_processing`: Bank statement uploads (peak: 1000+ pending)
- `transaction_extraction`: Transaction parsing queue
- `ocr_processing`: Scanned check OCR queue
- `reconciliation`: Account reconciliation queue

**Health Thresholds**:
- **Healthy**: pending < 100, stuck = 0
- **Degraded**: pending 100-500, stuck 1-5
- **Critical**: pending > 500, stuck > 5

**Example**:
```typescript
// Get current queue depth
const queueDepth = await monitor.getQueueDepth("statement_processing");

console.log(`Pending: ${queueDepth.pending}`);
console.log(`In progress: ${queueDepth.in_progress}`);
console.log(`Stuck: ${queueDepth.stuck}`);
console.log(`Health: ${queueDepth.health_status}`);

if (queueDepth.health_status === "critical") {
  // Alert operations team
  await alerting.sendAlert({
    type: "QUEUE_CRITICAL",
    queue_name: "statement_processing",
    pending: queueDepth.pending,
    stuck: queueDepth.stuck
  });
}

// Get stuck documents
const stuckDocs = await monitor.getStuckDocuments({
  queue_name: "statement_processing",
  min_retries: 3
});

stuckDocs.forEach(doc => {
  console.log(`Stuck document: ${doc.document_id}`);
  console.log(`  Retries: ${doc.retry_count}`);
  console.log(`  Last error: ${doc.last_error}`);
});

// Get processing rate
const rate = await monitor.getProcessingRate({
  queue_name: "statement_processing",
  time_window: "1h"
});

console.log(`Processing rate: ${rate.documents_per_minute.toFixed(1)} docs/min`);
```

**Performance Target**: <20ms for queue depth, <100ms for stuck documents.

---

### 2. Healthcare

**Use Case**: Monitor HL7 message processing queues and ensure timely lab result delivery.

**Queue Types**:
- `hl7_inbound`: Inbound HL7 messages (real-time)
- `fhir_validation`: FHIR resource validation
- `claims_processing`: Insurance claim processing
- `lab_results`: Lab result delivery queue

**Health Thresholds**:
- **Healthy**: pending < 50, processing rate > 1000 msgs/min
- **Degraded**: pending 50-200, processing rate 500-1000 msgs/min
- **Critical**: pending > 200, processing rate < 500 msgs/min

**Example**:
```typescript
// Get all queue depths
const allQueues = await monitor.getAllQueues();

allQueues.forEach(queue => {
  console.log(`${queue.queue_name}:`);
  console.log(`  Pending: ${queue.pending}`);
  console.log(`  Health: ${queue.health_status}`);
});

// Get processing rate trend
const trend = await monitor.getProcessingRateTrend({
  queue_name: "hl7_inbound",
  time_window: "24h",
  bucket_size: "1h"
});

trend.forEach(point => {
  console.log(`${point.timestamp}: ${point.documents_per_minute.toFixed(1)} msgs/min`);
});
```

**Performance Target**: <15ms for all queues, <80ms for rate trend.

---

### 3. Legal

**Use Case**: Monitor e-discovery document processing queues during large productions.

**Queue Types**:
- `document_intake`: Document uploads (100K+ documents)
- `ocr_processing`: OCR for scanned documents
- `classification`: Document type classification
- `review_assignment`: Document assignment to reviewers

**Health Thresholds**:
- **Healthy**: pending < 1000, stuck < 10
- **Degraded**: pending 1000-10000, stuck 10-50
- **Critical**: pending > 10000, stuck > 50

**Example**:
```typescript
// Monitor large document production
const queueDepth = await monitor.getQueueDepth("document_intake");

console.log(`Production status:`);
console.log(`  Total pending: ${queueDepth.pending}`);
console.log(`  Processing: ${queueDepth.in_progress}`);
console.log(`  Stuck: ${queueDepth.stuck}`);

// Estimate completion time
const rate = await monitor.getProcessingRate({
  queue_name: "document_intake",
  time_window: "1h"
});

const estimatedMinutes = queueDepth.pending / rate.documents_per_minute;
console.log(`Estimated completion: ${estimatedMinutes.toFixed(0)} minutes`);
```

**Performance Target**: <25ms for queue depth, <120ms for processing rate.

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Monitor founder fact extraction queues during bulk web scraping.

**Queue Types**:
- `web_scraping`: URL scraping queue (1000+ URLs)
- `html_parsing`: HTML fact extraction
- `entity_resolution`: Entity name normalization (@sama → Sam Altman)
- `fact_deduplication`: Multi-source fact merging

**Health Thresholds**:
- **Healthy**: pending < 100, processing rate > 100 pages/min
- **Degraded**: pending 100-500, processing rate 50-100 pages/min
- **Critical**: pending > 500, processing rate < 50 pages/min

**Example**:
```typescript
// Monitor bulk web scraping
const queueDepth = await monitor.getQueueDepth("web_scraping");

if (queueDepth.health_status === "degraded") {
  // Scale up workers
  await workers.scaleUp("web_scraping", 5);
}

// Get oldest pending URLs
const oldestURLs = await monitor.getOldestPendingDocuments({
  queue_name: "web_scraping",
  limit: 10
});

oldestURLs.forEach(url => {
  console.log(`Pending for ${url.age_minutes} minutes: ${url.document_id}`);
});
```

**Performance Target**: <20ms for queue depth, <90ms for oldest documents.

---

### 5. E-commerce

**Use Case**: Monitor product catalog import queues during marketplace syncs.

**Queue Types**:
- `product_import`: Product feed imports (5000+ products/sec)
- `image_download`: Product image downloads
- `category_mapping`: Product categorization
- `inventory_sync`: Inventory updates

**Health Thresholds**:
- **Healthy**: pending < 500, processing rate > 5000 products/sec
- **Degraded**: pending 500-2000, processing rate 2000-5000 products/sec
- **Critical**: pending > 2000, processing rate < 2000 products/sec

**Example**:
```typescript
// Monitor real-time marketplace sync
const queueDepth = await monitor.getQueueDepth("product_import");

console.log(`Marketplace sync status:`);
console.log(`  Products pending: ${queueDepth.pending}`);
console.log(`  Products processing: ${queueDepth.in_progress}`);

// Get processing rate
const rate = await monitor.getProcessingRate({
  queue_name: "product_import",
  time_window: "5m"
});

console.log(`Current throughput: ${rate.documents_per_second.toFixed(0)} products/sec`);
```

**Performance Target**: <15ms for queue depth, <60ms for processing rate.

---

### 6. SaaS

**Use Case**: Monitor webhook processing queues for API integrations.

**Queue Types**:
- `webhook_inbound`: Incoming webhooks (10K+ webhooks/sec)
- `webhook_retry`: Failed webhook retries
- `event_processing`: Event stream processing
- `notification_delivery`: Notification sending

**Health Thresholds**:
- **Healthy**: pending < 100, stuck = 0
- **Degraded**: pending 100-500, stuck 1-3
- **Critical**: pending > 500, stuck > 3

**Example**:
```typescript
// Monitor webhook processing
const queueDepth = await monitor.getQueueDepth("webhook_inbound");

if (queueDepth.stuck > 0) {
  // Investigate stuck webhooks
  const stuckWebhooks = await monitor.getStuckDocuments({
    queue_name: "webhook_inbound",
    min_retries: 3
  });

  stuckWebhooks.forEach(webhook => {
    console.log(`Stuck webhook: ${webhook.document_id}`);
    console.log(`  Source: ${webhook.metadata.source}`);
    console.log(`  Event type: ${webhook.metadata.event_type}`);
    console.log(`  Retries: ${webhook.retry_count}`);
  });
}
```

**Performance Target**: <10ms for queue depth, <70ms for stuck documents.

---

### 7. Insurance

**Use Case**: Monitor claims processing queues during catastrophe events.

**Queue Types**:
- `claims_intake`: Claims submissions (1000+ claims/hour during CAT)
- `document_extraction`: Claims form extraction
- `adjudication`: Claims adjudication
- `payout_processing`: Payment processing

**Health Thresholds**:
- **Healthy**: pending < 200, stuck < 5
- **Degraded**: pending 200-1000, stuck 5-20
- **Critical**: pending > 1000, stuck > 20

**Example**:
```typescript
// Monitor catastrophe claims surge
const queueDepth = await monitor.getQueueDepth("claims_intake");

console.log(`CAT claims status:`);
console.log(`  Claims pending: ${queueDepth.pending}`);
console.log(`  Claims processing: ${queueDepth.in_progress}`);
console.log(`  Health: ${queueDepth.health_status}`);

if (queueDepth.health_status === "critical") {
  // Scale up claims processing workers
  await workers.scaleUp("claims_intake", 10);

  // Alert claims management
  await alerting.sendAlert({
    type: "CAT_SURGE",
    queue_name: "claims_intake",
    pending: queueDepth.pending
  });
}

// Get processing rate
const rate = await monitor.getProcessingRate({
  queue_name: "claims_intake",
  time_window: "1h"
});

console.log(`Processing rate: ${rate.documents_per_hour.toFixed(0)} claims/hour`);
```

**Performance Target**: <20ms for queue depth, <100ms for processing rate.

---

### Cross-Domain Benefits

**Consistent Monitoring**: All domains benefit from:
- Real-time queue visibility across all processing stages
- Automatic health classification and alerting
- Historical trends for capacity planning
- Stuck document detection and remediation

**Operational Excellence**: Supports:
- SLA compliance monitoring
- Capacity planning (scale workers based on queue depth)
- Incident response (identify and fix stuck documents)
- Performance optimization (identify bottlenecks)

---

## Core Concepts

### Queue States

A document in a queue can be in one of three states:

1. **Pending**: Waiting to be processed
2. **In Progress**: Currently being processed by a worker
3. **Stuck**: Failed multiple times (>5 retries) and requires intervention

### Health Status

Queue health is automatically classified based on metrics:

- **Healthy**: Queue is processing normally
- **Degraded**: Queue is slower than expected or has some stuck documents
- **Critical**: Queue has large backlog or many stuck documents

### Processing Rate

Measured in three units:
- **Documents per second**: Real-time throughput
- **Documents per minute**: Short-term throughput
- **Documents per hour**: Long-term throughput

### Snapshot Collection

QueueMonitor records snapshots every 60 seconds for:
- Historical trend analysis
- Performance regression detection
- Capacity planning

---

## Interface Definition

### TypeScript Interface

```typescript
interface QueueMonitor {
  /**
   * Get current queue depth for a single queue
   *
   * @param queueName - Queue identifier
   * @returns Current queue state
   */
  getQueueDepth(queueName: string): Promise<QueueDepth>;

  /**
   * Get queue depths for all queues
   *
   * @returns Array of queue states
   */
  getAllQueues(): Promise<QueueDepth[]>;

  /**
   * Get stuck documents (>= min_retries failures)
   *
   * @param query - Stuck documents query parameters
   * @returns Array of stuck documents
   */
  getStuckDocuments(query: StuckDocumentsQuery): Promise<StuckDocument[]>;

  /**
   * Get processing rate over time window
   *
   * @param query - Processing rate query parameters
   * @returns Processing rate metrics
   */
  getProcessingRate(query: ProcessingRateQuery): Promise<ProcessingRate>;

  /**
   * Get processing rate trend over time
   *
   * @param query - Processing rate trend query parameters
   * @returns Time-series processing rate data
   */
  getProcessingRateTrend(
    query: ProcessingRateTrendQuery
  ): Promise<ProcessingRateTrend[]>;

  /**
   * Get queue depth trend over time
   *
   * @param query - Queue depth trend query parameters
   * @returns Time-series queue depth data
   */
  getQueueDepthTrend(
    query: QueueDepthTrendQuery
  ): Promise<QueueDepthTrend[]>;

  /**
   * Get oldest pending documents
   *
   * @param query - Oldest documents query parameters
   * @returns Array of oldest pending documents
   */
  getOldestPendingDocuments(
    query: OldestDocumentsQuery
  ): Promise<PendingDocument[]>;

  /**
   * Record queue snapshot (called periodically)
   *
   * @param queueName - Queue identifier
   * @returns Promise resolving when snapshot recorded
   */
  recordSnapshot(queueName: string): Promise<void>;

  /**
   * Get queue health summary
   *
   * @returns Health summary for all queues
   */
  getHealthSummary(): Promise<HealthSummary>;
}
```

---

## Data Model

### QueueDepth Type

```typescript
interface QueueDepth {
  // Identity
  queue_name: string;

  // Metrics
  pending: number;                // Documents waiting to be processed
  in_progress: number;            // Documents currently processing
  stuck: number;                  // Documents with >5 failed retries

  // Health
  health_status: HealthStatus;    // healthy | degraded | critical

  // Timestamp
  snapshot_time: string;          // ISO timestamp
}
```

### StuckDocument Type

```typescript
interface StuckDocument {
  document_id: string;
  queue_name: string;
  retry_count: number;
  last_error: string;
  last_retry_at: string;          // ISO timestamp
  first_attempt_at: string;       // ISO timestamp
  metadata?: Record<string, any>;
}
```

### ProcessingRate Type

```typescript
interface ProcessingRate {
  queue_name: string;
  time_window: string;

  // Rate Metrics
  documents_processed: number;
  documents_per_second: number;
  documents_per_minute: number;
  documents_per_hour: number;

  // Time Range
  start_time: string;             // ISO timestamp
  end_time: string;               // ISO timestamp
}
```

### ProcessingRateTrend Type

```typescript
interface ProcessingRateTrend {
  timestamp: string;              // ISO timestamp (bucket start)
  documents_processed: number;
  documents_per_minute: number;
}
```

### QueueDepthTrend Type

```typescript
interface QueueDepthTrend {
  timestamp: string;              // ISO timestamp
  pending: number;
  in_progress: number;
  stuck: number;
  health_status: HealthStatus;
}
```

### PendingDocument Type

```typescript
interface PendingDocument {
  document_id: string;
  queue_name: string;
  queued_at: string;              // ISO timestamp
  age_minutes: number;
  metadata?: Record<string, any>;
}
```

### HealthSummary Type

```typescript
interface HealthSummary {
  total_queues: number;
  healthy_queues: number;
  degraded_queues: number;
  critical_queues: number;
  total_pending: number;
  total_stuck: number;
  queues: QueueDepth[];
}
```

### Database Schema (PostgreSQL)

```sql
-- Queue state table (updated in real-time)
CREATE TABLE queue_state (
  queue_name VARCHAR(128) PRIMARY KEY,
  pending INTEGER NOT NULL DEFAULT 0,
  in_progress INTEGER NOT NULL DEFAULT 0,
  stuck INTEGER NOT NULL DEFAULT 0,
  health_status VARCHAR(32) NOT NULL DEFAULT 'healthy',
  last_updated TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Document queue table (tracks individual documents)
CREATE TABLE document_queue (
  document_id VARCHAR(128) PRIMARY KEY,
  queue_name VARCHAR(128) NOT NULL,
  status VARCHAR(32) NOT NULL DEFAULT 'pending',
  retry_count INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  last_retry_at TIMESTAMP WITH TIME ZONE,
  queued_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  started_at TIMESTAMP WITH TIME ZONE,
  completed_at TIMESTAMP WITH TIME ZONE,
  metadata JSONB
);

-- Indexes
CREATE INDEX idx_document_queue_queue_name ON document_queue(queue_name);
CREATE INDEX idx_document_queue_status ON document_queue(status);
CREATE INDEX idx_document_queue_retry_count ON document_queue(retry_count);
CREATE INDEX idx_document_queue_queued_at ON document_queue(queued_at);

-- Queue depth snapshots (from MetricsCollector)
-- Reuse queue_depth_snapshots table from MetricsCollector
```

---

## Core Functionality

### 1. getQueueDepth()

Get current queue depth for a single queue.

#### Signature

```typescript
getQueueDepth(queueName: string): Promise<QueueDepth>
```

#### Behavior

1. **Query Current State**:
   - Query `queue_state` table for queue

2. **Calculate Health**:
   - Apply health rules based on pending/stuck counts

3. **Return**:
   - Return current queue depth

#### SQL Query

```sql
SELECT
  queue_name,
  pending,
  in_progress,
  stuck,
  health_status,
  last_updated AS snapshot_time
FROM queue_state
WHERE queue_name = $1;
```

#### Example

```typescript
const queueDepth = await monitor.getQueueDepth("statement_processing");

console.log(`Queue: ${queueDepth.queue_name}`);
console.log(`Pending: ${queueDepth.pending}`);
console.log(`In progress: ${queueDepth.in_progress}`);
console.log(`Stuck: ${queueDepth.stuck}`);
console.log(`Health: ${queueDepth.health_status}`);
```

#### Performance

- **Target Latency**: <20ms p95
- **Index Used**: PRIMARY KEY (queue_name)

---

### 2. getAllQueues()

Get queue depths for all queues.

#### Signature

```typescript
getAllQueues(): Promise<QueueDepth[]>
```

#### Behavior

1. **Query All Queues**:
   - SELECT * FROM queue_state

2. **Sort**:
   - Sort by health_status (critical first)

3. **Return**:
   - Return array of queue depths

#### SQL Query

```sql
SELECT
  queue_name,
  pending,
  in_progress,
  stuck,
  health_status,
  last_updated AS snapshot_time
FROM queue_state
ORDER BY
  CASE health_status
    WHEN 'critical' THEN 1
    WHEN 'degraded' THEN 2
    WHEN 'healthy' THEN 3
  END,
  pending DESC;
```

#### Example

```typescript
const allQueues = await monitor.getAllQueues();

allQueues.forEach(queue => {
  console.log(`${queue.queue_name}: ${queue.health_status}`);
  console.log(`  Pending: ${queue.pending}`);
  console.log(`  Stuck: ${queue.stuck}`);
});
```

#### Performance

- **Target Latency**: <30ms p95

---

### 3. getStuckDocuments()

Get documents with >= min_retries failures.

#### Signature

```typescript
getStuckDocuments(query: StuckDocumentsQuery): Promise<StuckDocument[]>
```

#### Behavior

1. **Query Document Queue**:
   - Filter by queue_name and retry_count >= min_retries

2. **Sort**:
   - Sort by retry_count DESC (most retries first)

3. **Return**:
   - Return array of stuck documents

#### SQL Query

```sql
SELECT
  document_id,
  queue_name,
  retry_count,
  last_error,
  last_retry_at,
  queued_at AS first_attempt_at,
  metadata
FROM document_queue
WHERE queue_name = $1
  AND status = 'pending'
  AND retry_count >= $2
ORDER BY retry_count DESC, last_retry_at ASC
LIMIT $3;
```

#### Example

```typescript
const stuckDocs = await monitor.getStuckDocuments({
  queue_name: "statement_processing",
  min_retries: 3,
  limit: 10
});

stuckDocs.forEach(doc => {
  console.log(`Document: ${doc.document_id}`);
  console.log(`  Retries: ${doc.retry_count}`);
  console.log(`  Last error: ${doc.last_error}`);
  console.log(`  First attempt: ${doc.first_attempt_at}`);
});
```

#### Performance

- **Target Latency**: <100ms p95

---

### 4. getProcessingRate()

Get processing rate over time window.

#### Signature

```typescript
getProcessingRate(query: ProcessingRateQuery): Promise<ProcessingRate>
```

#### Behavior

1. **Parse Time Window**:
   - Convert relative time window to absolute timestamps

2. **Count Completed Documents**:
   - Count documents with completed_at in time range

3. **Calculate Rate**:
   - Rate = count / duration

4. **Return**:
   - Return processing rate metrics

#### SQL Query

```sql
SELECT
  COUNT(*) AS documents_processed,
  MIN(completed_at) AS start_time,
  MAX(completed_at) AS end_time
FROM document_queue
WHERE queue_name = $1
  AND status = 'completed'
  AND completed_at >= $2
  AND completed_at <= $3;
```

#### Example

```typescript
const rate = await monitor.getProcessingRate({
  queue_name: "statement_processing",
  time_window: "1h"
});

console.log(`Documents processed: ${rate.documents_processed}`);
console.log(`Rate: ${rate.documents_per_minute.toFixed(1)} docs/min`);
console.log(`Rate: ${rate.documents_per_hour.toFixed(0)} docs/hour`);
```

#### Performance

- **Target Latency**: <80ms p95

---

### 5. getProcessingRateTrend()

Get processing rate trend over time.

#### Signature

```typescript
getProcessingRateTrend(
  query: ProcessingRateTrendQuery
): Promise<ProcessingRateTrend[]>
```

#### Behavior

1. **Bucket By Time**:
   - Group documents into time buckets

2. **Count Per Bucket**:
   - Count completed documents per bucket

3. **Calculate Rate**:
   - Rate per bucket

4. **Return**:
   - Return time-series data

#### SQL Query

```sql
SELECT
  DATE_TRUNC('hour', completed_at) AS timestamp,
  COUNT(*) AS documents_processed,
  COUNT(*) / 60.0 AS documents_per_minute
FROM document_queue
WHERE queue_name = $1
  AND status = 'completed'
  AND completed_at >= $2
  AND completed_at <= $3
GROUP BY DATE_TRUNC('hour', completed_at)
ORDER BY timestamp;
```

#### Example

```typescript
const trend = await monitor.getProcessingRateTrend({
  queue_name: "statement_processing",
  time_window: "24h",
  bucket_size: "1h"
});

trend.forEach(point => {
  console.log(`${point.timestamp}: ${point.documents_per_minute.toFixed(1)} docs/min`);
});
```

#### Performance

- **Target Latency**: <120ms p95

---

### 6. getQueueDepthTrend()

Get queue depth trend over time.

#### Signature

```typescript
getQueueDepthTrend(
  query: QueueDepthTrendQuery
): Promise<QueueDepthTrend[]>
```

#### Behavior

1. **Query Snapshots**:
   - Query queue_depth_snapshots table

2. **Filter By Time**:
   - Filter by time window

3. **Return**:
   - Return time-series data

#### SQL Query

```sql
SELECT
  created_at AS timestamp,
  pending,
  in_progress,
  stuck,
  health_status
FROM queue_depth_snapshots
WHERE queue_name = $1
  AND created_at >= $2
  AND created_at <= $3
ORDER BY created_at;
```

#### Example

```typescript
const trend = await monitor.getQueueDepthTrend({
  queue_name: "statement_processing",
  time_window: "24h"
});

trend.forEach(point => {
  console.log(`${point.timestamp}:`);
  console.log(`  Pending: ${point.pending}`);
  console.log(`  Health: ${point.health_status}`);
});
```

#### Performance

- **Target Latency**: <100ms p95

---

### 7. getOldestPendingDocuments()

Get oldest pending documents.

#### Signature

```typescript
getOldestPendingDocuments(
  query: OldestDocumentsQuery
): Promise<PendingDocument[]>
```

#### Behavior

1. **Query Pending Documents**:
   - Filter by status = 'pending'

2. **Sort By Age**:
   - Sort by queued_at ASC (oldest first)

3. **Calculate Age**:
   - Age = now - queued_at

4. **Return**:
   - Return oldest documents

#### SQL Query

```sql
SELECT
  document_id,
  queue_name,
  queued_at,
  EXTRACT(EPOCH FROM (NOW() - queued_at)) / 60 AS age_minutes,
  metadata
FROM document_queue
WHERE queue_name = $1
  AND status = 'pending'
ORDER BY queued_at ASC
LIMIT $2;
```

#### Example

```typescript
const oldestDocs = await monitor.getOldestPendingDocuments({
  queue_name: "statement_processing",
  limit: 10
});

oldestDocs.forEach(doc => {
  console.log(`Document: ${doc.document_id}`);
  console.log(`  Age: ${doc.age_minutes.toFixed(0)} minutes`);
  console.log(`  Queued at: ${doc.queued_at}`);
});
```

#### Performance

- **Target Latency**: <90ms p95

---

### 8. recordSnapshot()

Record queue snapshot (called periodically).

#### Signature

```typescript
recordSnapshot(queueName: string): Promise<void>
```

#### Behavior

1. **Count Documents**:
   - Count pending, in_progress, stuck documents

2. **Determine Health**:
   - Apply health rules

3. **Update Queue State**:
   - UPSERT into queue_state table

4. **Insert Snapshot**:
   - INSERT into queue_depth_snapshots table

#### Example

```typescript
// Run every 60 seconds
setInterval(async () => {
  await monitor.recordSnapshot("statement_processing");
  await monitor.recordSnapshot("transaction_extraction");
  // ... other queues
}, 60000);
```

#### Performance

- **Target Latency**: <50ms p95

---

### 9. getHealthSummary()

Get health summary for all queues.

#### Signature

```typescript
getHealthSummary(): Promise<HealthSummary>
```

#### Behavior

1. **Query All Queues**:
   - Get all queue depths

2. **Aggregate**:
   - Count healthy/degraded/critical queues
   - Sum total pending/stuck

3. **Return**:
   - Return summary

#### Example

```typescript
const summary = await monitor.getHealthSummary();

console.log(`Total queues: ${summary.total_queues}`);
console.log(`Healthy: ${summary.healthy_queues}`);
console.log(`Degraded: ${summary.degraded_queues}`);
console.log(`Critical: ${summary.critical_queues}`);
console.log(`Total pending: ${summary.total_pending}`);
console.log(`Total stuck: ${summary.total_stuck}`);
```

#### Performance

- **Target Latency**: <40ms p95

---

## Health Indicators

### Health Classification Rules

```typescript
function calculateHealth(
  pending: number,
  stuck: number
): HealthStatus {
  // Critical thresholds
  if (stuck > 10 || pending > 1000) {
    return "critical";
  }

  // Degraded thresholds
  if (stuck > 5 || pending > 500) {
    return "degraded";
  }

  // Healthy
  return "healthy";
}
```

### Configurable Thresholds

```typescript
interface HealthThresholds {
  critical_pending: number;       // Default: 1000
  critical_stuck: number;         // Default: 10
  degraded_pending: number;       // Default: 500
  degraded_stuck: number;         // Default: 5
}
```

---

## Alert Thresholds

### Backlog Alert

**Trigger**: pending > 500
**Action**: Scale up workers, notify operations

```typescript
if (queueDepth.pending > 500) {
  await alerting.sendAlert({
    type: "BACKLOG_ALERT",
    queue_name: queueDepth.queue_name,
    pending: queueDepth.pending,
    threshold: 500
  });
}
```

### Stuck Document Alert

**Trigger**: stuck > 5
**Action**: Investigate errors, notify engineering

```typescript
if (queueDepth.stuck > 5) {
  await alerting.sendAlert({
    type: "STUCK_DOCUMENTS",
    queue_name: queueDepth.queue_name,
    stuck: queueDepth.stuck,
    threshold: 5
  });
}
```

### Stale Queue Alert

**Trigger**: Last snapshot > 5 minutes old
**Action**: Check worker health, restart workers

```typescript
const lastSnapshot = new Date(queueDepth.snapshot_time);
const ageMinutes = (Date.now() - lastSnapshot.getTime()) / 60000;

if (ageMinutes > 5) {
  await alerting.sendAlert({
    type: "STALE_QUEUE",
    queue_name: queueDepth.queue_name,
    age_minutes: ageMinutes
  });
}
```

---

## Storage & Indexing

### Indexes for Fast Queries

```sql
-- Primary key for fast single queue lookup
ALTER TABLE queue_state ADD PRIMARY KEY (queue_name);

-- Composite index for queue + status queries
CREATE INDEX idx_document_queue_queue_status ON document_queue(queue_name, status);

-- Index for stuck document queries
CREATE INDEX idx_document_queue_stuck ON document_queue(queue_name, retry_count) WHERE retry_count >= 3;

-- Index for oldest pending queries
CREATE INDEX idx_document_queue_oldest ON document_queue(queue_name, queued_at) WHERE status = 'pending';
```

---

## Edge Cases

### Edge Case 1: Queue Does Not Exist

**Scenario**: Query for non-existent queue.

**Detection**:
```typescript
const queueDepth = await monitor.getQueueDepth("nonexistent_queue");

if (queueDepth === null) {
  console.log("Queue does not exist");
}
```

**Resolution**:
- Return null or empty queue depth
- Create queue on first document

---

### Edge Case 2: No Documents Processed

**Scenario**: Processing rate query returns 0.

**Detection**:
```typescript
if (rate.documents_processed === 0) {
  console.log("No documents processed in time window");
}
```

**Resolution**:
- Return rate of 0
- Display "N/A" in UI

---

### Edge Case 3: All Documents Stuck

**Scenario**: pending = 0, stuck = 100.

**Detection**:
```typescript
if (queueDepth.pending === 0 && queueDepth.stuck > 0) {
  console.warn("All documents are stuck");
}
```

**Resolution**:
- Alert engineering team
- Investigate common error
- Manually retry or fix documents

---

### Edge Case 4: Very Old Pending Documents

**Scenario**: Document pending for >24 hours.

**Detection**:
```typescript
const oldestDoc = await monitor.getOldestPendingDocuments({
  queue_name: "statement_processing",
  limit: 1
});

if (oldestDoc[0].age_minutes > 1440) { // 24 hours
  console.warn(`Document pending for ${oldestDoc[0].age_minutes} minutes`);
}
```

**Resolution**:
- Investigate why document not processed
- Manually trigger processing
- Check worker availability

---

### Edge Case 5: Negative Queue Depth

**Scenario**: Bug causes pending count to go negative.

**Detection**:
```typescript
if (queueDepth.pending < 0) {
  console.error("Negative queue depth detected");
}
```

**Resolution**:
- Log error
- Reset queue depth to 0
- Investigate bug

---

### Edge Case 6: Snapshot Collection Failure

**Scenario**: recordSnapshot() fails due to database error.

**Detection**:
```typescript
try {
  await monitor.recordSnapshot("statement_processing");
} catch (error) {
  console.error("Failed to record snapshot:", error);
}
```

**Resolution**:
- Retry with exponential backoff
- Alert operations if repeated failures
- Continue with stale data

---

### Edge Case 7: Race Condition in Queue Depth

**Scenario**: Document completes while counting queue depth.

**Resolution**:
- Accept eventual consistency
- Use database transactions for critical updates
- Snapshot is point-in-time, not real-time

---

### Edge Case 8: Queue Name Collision

**Scenario**: Two systems use same queue name.

**Resolution**:
- Use namespace prefixes (e.g., "prod_statement_processing")
- Document queue naming conventions
- Validate queue names on creation

---

## Performance Characteristics

### Latency Targets

| Operation | Target Latency (p95) | Notes |
|-----------|---------------------|-------|
| `getQueueDepth()` | < 20ms | PRIMARY KEY lookup |
| `getAllQueues()` | < 30ms | Full table scan (small table) |
| `getStuckDocuments()` | < 100ms | Indexed query |
| `getProcessingRate()` | < 80ms | Aggregation query |
| `getProcessingRateTrend()` | < 120ms | Time bucketing |
| `getQueueDepthTrend()` | < 100ms | Snapshot query |
| `getOldestPendingDocuments()` | < 90ms | Sorted query |
| `recordSnapshot()` | < 50ms | UPSERT + INSERT |
| `getHealthSummary()` | < 40ms | Aggregate all queues |

### Snapshot Frequency

- **Default**: Every 60 seconds
- **High-Volume Queues**: Every 30 seconds
- **Low-Volume Queues**: Every 5 minutes

### Storage Requirements

- **queue_state table**: ~1KB per queue (100 queues = 100KB)
- **document_queue table**: ~500 bytes per document
- **queue_depth_snapshots**: ~200 bytes per snapshot

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';

export class PostgresQueueMonitor implements QueueMonitor {
  private pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  async getQueueDepth(queueName: string): Promise<QueueDepth> {
    const query = `
      SELECT
        queue_name,
        pending,
        in_progress,
        stuck,
        health_status,
        last_updated AS snapshot_time
      FROM queue_state
      WHERE queue_name = $1
    `;

    const result = await this.pool.query(query, [queueName]);

    if (result.rows.length === 0) {
      return null;
    }

    const row = result.rows[0];

    return {
      queue_name: row.queue_name,
      pending: parseInt(row.pending),
      in_progress: parseInt(row.in_progress),
      stuck: parseInt(row.stuck),
      health_status: row.health_status,
      snapshot_time: row.snapshot_time.toISOString()
    };
  }

  async getAllQueues(): Promise<QueueDepth[]> {
    const query = `
      SELECT
        queue_name,
        pending,
        in_progress,
        stuck,
        health_status,
        last_updated AS snapshot_time
      FROM queue_state
      ORDER BY
        CASE health_status
          WHEN 'critical' THEN 1
          WHEN 'degraded' THEN 2
          WHEN 'healthy' THEN 3
        END,
        pending DESC
    `;

    const result = await this.pool.query(query);

    return result.rows.map(row => ({
      queue_name: row.queue_name,
      pending: parseInt(row.pending),
      in_progress: parseInt(row.in_progress),
      stuck: parseInt(row.stuck),
      health_status: row.health_status,
      snapshot_time: row.snapshot_time.toISOString()
    }));
  }

  async getStuckDocuments(
    query: StuckDocumentsQuery
  ): Promise<StuckDocument[]> {
    const sql = `
      SELECT
        document_id,
        queue_name,
        retry_count,
        last_error,
        last_retry_at,
        queued_at AS first_attempt_at,
        metadata
      FROM document_queue
      WHERE queue_name = $1
        AND status = 'pending'
        AND retry_count >= $2
      ORDER BY retry_count DESC, last_retry_at ASC
      LIMIT $3
    `;

    const result = await this.pool.query(sql, [
      query.queue_name,
      query.min_retries || 5,
      query.limit || 100
    ]);

    return result.rows.map(row => ({
      document_id: row.document_id,
      queue_name: row.queue_name,
      retry_count: parseInt(row.retry_count),
      last_error: row.last_error,
      last_retry_at: row.last_retry_at?.toISOString(),
      first_attempt_at: row.first_attempt_at.toISOString(),
      metadata: row.metadata
    }));
  }

  async recordSnapshot(queueName: string): Promise<void> {
    // Count documents in each state
    const countQuery = `
      SELECT
        COUNT(*) FILTER (WHERE status = 'pending' AND retry_count < 5) AS pending,
        COUNT(*) FILTER (WHERE status = 'in_progress') AS in_progress,
        COUNT(*) FILTER (WHERE status = 'pending' AND retry_count >= 5) AS stuck
      FROM document_queue
      WHERE queue_name = $1
    `;

    const countResult = await this.pool.query(countQuery, [queueName]);
    const counts = countResult.rows[0];

    const pending = parseInt(counts.pending);
    const in_progress = parseInt(counts.in_progress);
    const stuck = parseInt(counts.stuck);

    // Calculate health status
    const health_status = this.calculateHealth(pending, stuck);

    // Update queue state
    const upsertQuery = `
      INSERT INTO queue_state (queue_name, pending, in_progress, stuck, health_status, last_updated)
      VALUES ($1, $2, $3, $4, $5, NOW())
      ON CONFLICT (queue_name)
      DO UPDATE SET
        pending = EXCLUDED.pending,
        in_progress = EXCLUDED.in_progress,
        stuck = EXCLUDED.stuck,
        health_status = EXCLUDED.health_status,
        last_updated = NOW()
    `;

    await this.pool.query(upsertQuery, [
      queueName,
      pending,
      in_progress,
      stuck,
      health_status
    ]);

    // Insert snapshot
    const snapshotQuery = `
      INSERT INTO queue_depth_snapshots (queue_name, pending, in_progress, stuck, health_status)
      VALUES ($1, $2, $3, $4, $5)
    `;

    await this.pool.query(snapshotQuery, [
      queueName,
      pending,
      in_progress,
      stuck,
      health_status
    ]);
  }

  private calculateHealth(pending: number, stuck: number): string {
    if (stuck > 10 || pending > 1000) {
      return "critical";
    }

    if (stuck > 5 || pending > 500) {
      return "degraded";
    }

    return "healthy";
  }
}
```

---

## Security Considerations

### 1. Queue Name Validation

**Prevent SQL Injection**:
```typescript
function validateQueueName(queueName: string): void {
  if (!/^[a-z0-9_-]+$/i.test(queueName)) {
    throw new ValidationError("Invalid queue name");
  }
}
```

### 2. Rate Limiting

**Prevent Excessive Queries**:
```typescript
// Limit to 100 queries per minute per user
```

### 3. Access Control

**Role-Based Access**:
```typescript
// Only allow metrics_reader role to view queue depths
```

---

## Integration Patterns

### Pattern 1: Auto-Scaling

```typescript
// Scale workers based on queue depth
setInterval(async () => {
  const queueDepth = await monitor.getQueueDepth("statement_processing");

  if (queueDepth.pending > 500) {
    await workers.scaleUp("statement_processing", 10);
  } else if (queueDepth.pending < 50) {
    await workers.scaleDown("statement_processing", 2);
  }
}, 60000);
```

### Pattern 2: SLA Monitoring

```typescript
// Alert if processing rate below SLA
const rate = await monitor.getProcessingRate({
  queue_name: "statement_processing",
  time_window: "1h"
});

const SLA_DOCUMENTS_PER_HOUR = 1000;

if (rate.documents_per_hour < SLA_DOCUMENTS_PER_HOUR) {
  await alerting.sendAlert({
    type: "SLA_BREACH",
    queue_name: "statement_processing",
    actual_rate: rate.documents_per_hour,
    sla_rate: SLA_DOCUMENTS_PER_HOUR
  });
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section)

---

## Testing Strategy

### Unit Tests

```typescript
describe('QueueMonitor', () => {
  it('should get queue depth', async () => {
    const queueDepth = await monitor.getQueueDepth("test_queue");

    expect(queueDepth.queue_name).toBe("test_queue");
    expect(queueDepth.pending).toBeGreaterThanOrEqual(0);
  });

  it('should get stuck documents', async () => {
    const stuckDocs = await monitor.getStuckDocuments({
      queue_name: "test_queue",
      min_retries: 3,
      limit: 10
    });

    stuckDocs.forEach(doc => {
      expect(doc.retry_count).toBeGreaterThanOrEqual(3);
    });
  });
});
```

---

## Configuration

```typescript
interface QueueMonitorConfig {
  snapshotIntervalSeconds: number;        // Default: 60
  healthThresholds: HealthThresholds;
  stuckDocumentThreshold: number;         // Default: 5
}
```

---

## Observability

### Metrics

```typescript
interface MonitorMetrics {
  queue_depth_queries_total: Counter;
  query_duration_ms: Histogram;
  snapshots_recorded_total: Counter;
}
```

---

## Future Enhancements

### Phase 1: Predictive Alerts (3 months)
- Predict queue backlog before it occurs
- Forecast completion time

### Phase 2: Automatic Remediation (6 months)
- Auto-retry stuck documents
- Auto-scale workers

### Phase 3: ML-Based Capacity Planning (12 months)
- ML models for optimal worker scaling
- Cost optimization recommendations

---

**End of QueueMonitor OL Primitive Specification**
