# PerformanceAnalyzer OL Primitive

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
8. [Aggregation Strategies](#aggregation-strategies)
9. [Statistical Methods](#statistical-methods)
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

The **PerformanceAnalyzer** queries and aggregates metrics collected by the MetricsCollector to provide actionable insights into document processing pipeline performance. It computes latency percentiles, success rates, error rates, and throughput for parsers and rules, enabling performance optimization and bottleneck identification.

### Key Capabilities

- **Fast Aggregation**: <50ms p95 single parser stats, <200ms all parsers
- **Statistical Analysis**: Computes p50/p95/p99 latency percentiles
- **Success/Error Rates**: Tracks parser and rule success rates
- **Throughput Metrics**: Measures processing rate over time
- **Bottleneck Detection**: Identifies slowest parsers and rules
- **Time-Range Queries**: Flexible time window analysis

### Design Philosophy

The PerformanceAnalyzer follows four core principles:

1. **Query Optimization**: Pre-aggregate common queries for fast response
2. **Statistical Rigor**: Use standard statistical methods (percentiles, rates)
3. **Actionable Insights**: Surface metrics that drive optimization decisions
4. **Flexible Time Windows**: Support real-time and historical analysis

---

## Purpose & Scope

### Problem Statement

In document processing pipelines, performance analysis poses critical challenges:

- **No Visibility**: Can't see which parsers are slow or failing
- **Manual Analysis**: Requires SQL expertise to query metrics
- **Delayed Detection**: Bottlenecks discovered days after they occur
- **Incomplete Context**: Metrics lack business context (e.g., cost impact)
- **No Benchmarking**: Can't compare performance across parsers or time periods

Traditional approaches have significant limitations:

**Approach 1: Raw Metric Queries**
- ❌ Slow queries (seconds to minutes)
- ❌ Requires SQL expertise
- ❌ No statistical analysis

**Approach 2: Pre-Computed Dashboards**
- ❌ Fixed time windows
- ❌ Can't drill down
- ❌ Stale data (refreshed hourly)

**Approach 3: Performance Analyzer**
- ✅ Fast queries (<200ms)
- ✅ Statistical analysis built-in
- ✅ Flexible time windows
- ✅ Drill-down support
- ✅ Real-time insights

### Solution

The PerformanceAnalyzer implements a **fast query engine** with:

1. **Materialized Views**: Pre-computed aggregates for common queries
2. **Statistical Functions**: Built-in percentile and rate calculations
3. **Caching**: In-memory cache for frequently accessed stats
4. **Time-Series Indexing**: Optimized for time-range queries
5. **Comparative Analysis**: Benchmark parsers against each other

---

## Multi-Domain Applicability

The PerformanceAnalyzer is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Analyze bank statement parser performance and identify slow parsers.

**Analysis Queries**:
- Which bank parser is slowest? (Chase: 1250ms p95, Wells Fargo: 950ms p95)
- What's the failure rate for OCR parsers? (Scanned checks: 5% failure rate)
- How many transactions processed per hour? (Peak: 50K transactions/hour)
- Which categorization rules are slowest? (ML categorization: 12ms p95)

**Example**:
```typescript
// Get Chase parser stats
const stats = await analyzer.getParserStats({
  parser_id: "chase_pdf_parser_v2",
  time_window: "24h"
});

console.log(`p50: ${stats.duration_p50_ms}ms`);
console.log(`p95: ${stats.duration_p95_ms}ms`);
console.log(`p99: ${stats.duration_p99_ms}ms`);
console.log(`Success rate: ${stats.success_rate * 100}%`);
console.log(`Throughput: ${stats.executions_per_hour} executions/hour`);

// Find slowest parsers
const slowest = await analyzer.getSlowestParsers({
  time_window: "7d",
  limit: 10
});

slowest.forEach(parser => {
  console.log(`${parser.parser_id}: ${parser.duration_p95_ms}ms p95`);
});

// Get error breakdown
const errors = await analyzer.getErrorBreakdown({
  parser_id: "chase_pdf_parser_v2",
  time_window: "24h"
});

errors.forEach(error => {
  console.log(`${error.error_type}: ${error.count} occurrences (${error.percentage}%)`);
});
```

**Performance Target**: <50ms for single parser stats, <200ms for all parsers.

---

### 2. Healthcare

**Use Case**: Monitor HL7 parser performance and FHIR validation rule efficiency.

**Analysis Queries**:
- Which HL7 message types are slowest to parse? (ORU^R01: 85ms p95)
- What's the validation failure rate? (Patient resource: 2% failure rate)
- How many messages processed per minute? (Peak: 1000 messages/min)
- Which validation rules are most expensive? (us-core profile: 18ms p95)

**Example**:
```typescript
// Get HL7 parser stats by message type
const hl7Stats = await analyzer.getParserStats({
  parser_id: "hl7_oru_r01_parser",
  time_window: "7d"
});

// Get validation rule stats
const ruleStats = await analyzer.getRuleStats({
  rule_id: "fhir_patient_validation",
  rule_type: "validation",
  time_window: "24h"
});

console.log(`Validation rule execution time: ${ruleStats.execution_time_p95_ms}ms p95`);
console.log(`Match rate: ${ruleStats.match_rate * 100}%`);

// Compare parser performance across message types
const comparison = await analyzer.compareParserPerformance({
  parser_ids: ["hl7_adt_a01_parser", "hl7_oru_r01_parser", "hl7_orm_o01_parser"],
  time_window: "7d"
});
```

**Performance Target**: <80ms for rule stats, <150ms for parser comparison.

---

### 3. Legal

**Use Case**: Analyze contract OCR performance and document classification accuracy.

**Analysis Queries**:
- What's the OCR confidence distribution? (Mean: 0.92, p50: 0.94)
- Which document types have lowest classification accuracy? (Exhibits: 78% accuracy)
- How long does OCR take per page? (Average: 245ms/page)
- Which extraction rules are most effective? (Party extraction: 95% match rate)

**Example**:
```typescript
// Get OCR parser stats
const ocrStats = await analyzer.getParserStats({
  parser_id: "tesseract_legal_ocr",
  time_window: "30d"
});

// Get classification rule effectiveness
const classifierStats = await analyzer.getRuleStats({
  rule_id: "doc_type_classifier_v3",
  rule_type: "classification",
  time_window: "7d"
});

console.log(`Classification accuracy: ${classifierStats.match_rate * 100}%`);
console.log(`Average confidence: ${classifierStats.avg_confidence}`);

// Get slowest extraction rules
const slowestRules = await analyzer.getSlowestRules({
  rule_type: "extraction",
  time_window: "7d",
  limit: 10
});
```

**Performance Target**: <60ms for rule stats, <120ms for slowest rules query.

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Monitor founder fact extraction from web scraping and entity resolution efficiency.

**Analysis Queries**:
- What's the TechCrunch parser success rate? (Success: 88%, Failure: 12% - broken links)
- How many facts extracted per web page? (Average: 15 facts/page)
- Which fact type extractor is most accurate? (Investment: 92% accuracy)
- What's the processing rate for bulk scraping? (Peak: 100 web pages/min)

**Example**:
```typescript
// Get TechCrunch parser stats
const webScraperStats = await analyzer.getParserStats({
  parser_id: "techcrunch_web_parser",
  time_window: "7d"
});

console.log(`Web pages parsed: ${webScraperStats.total_executions}`);
console.log(`Average facts per page: ${webScraperStats.avg_observations_extracted}`);
console.log(`Failure rate: ${(1 - webScraperStats.success_rate) * 100}%`);

// Get fact extractor effectiveness by type
const factStats = await analyzer.compareRulePerformance({
  rule_ids: ["investment_fact_extractor", "founding_fact_extractor", "employment_fact_extractor"],
  rule_type: "extraction",
  time_window: "30d"
});
```

**Performance Target**: <50ms for parser stats, <100ms for rule comparison.

---

### 5. E-commerce

**Use Case**: Analyze product catalog import and categorization performance.

**Analysis Queries**:
- Which marketplace feed parser is fastest? (Amazon: 450ms p95, Shopify: 320ms p95)
- What's the categorization accuracy? (Taxonomy mapper: 91% accuracy)
- How many products processed per second? (Peak: 5000 products/sec)
- Which validation rules catch most errors? (SKU validator: 12% error rate)

**Example**:
```typescript
// Get product feed parser stats
const feedStats = await analyzer.getParserStats({
  parser_id: "amazon_catalog_csv_parser",
  time_window: "24h"
});

console.log(`Products parsed: ${feedStats.total_executions}`);
console.log(`Throughput: ${feedStats.executions_per_second} products/sec`);
console.log(`p95 latency: ${feedStats.duration_p95_ms}ms`);

// Get categorization rule stats
const categorizationStats = await analyzer.getRuleStats({
  rule_id: "product_taxonomy_mapper",
  rule_type: "normalization",
  time_window: "7d"
});

console.log(`Categorization match rate: ${categorizationStats.match_rate * 100}%`);
```

**Performance Target**: <40ms for parser stats, <80ms for rule stats.

---

### 6. SaaS

**Use Case**: Monitor webhook payload parsing and API request processing.

**Analysis Queries**:
- What's the webhook processing latency? (Stripe: 25ms p95)
- Which API endpoints have highest error rates? (Payment API: 1.5% error rate)
- How many webhooks processed per second? (Peak: 10K webhooks/sec)
- Which validation rules are bottlenecks? (Request validator: 2ms p95)

**Example**:
```typescript
// Get webhook parser stats
const webhookStats = await analyzer.getParserStats({
  parser_id: "stripe_webhook_parser",
  time_window: "1h"
});

console.log(`Webhooks processed: ${webhookStats.total_executions}`);
console.log(`p95 latency: ${webhookStats.duration_p95_ms}ms`);
console.log(`Error rate: ${(1 - webhookStats.success_rate) * 100}%`);

// Get validation rule performance
const validationStats = await analyzer.getRuleStats({
  rule_id: "api_request_validator",
  rule_type: "validation",
  time_window: "24h"
});
```

**Performance Target**: <30ms for parser stats, <60ms for rule stats.

---

### 7. Insurance

**Use Case**: Analyze claims form extraction and policy document parsing.

**Analysis Queries**:
- What's the ACORD form OCR accuracy? (Mean confidence: 0.89)
- Which validation rules reject most claims? (Coverage limit: 8% rejection rate)
- How long does claims intake take? (Average: 2800ms per claim)
- What's the processing rate during peak? (Peak: 1000 claims/hour)

**Example**:
```typescript
// Get claims form parser stats
const claimsStats = await analyzer.getParserStats({
  parser_id: "acord_form_ocr_parser",
  time_window: "7d"
});

console.log(`Claims processed: ${claimsStats.total_executions}`);
console.log(`Average processing time: ${claimsStats.duration_p50_ms}ms`);
console.log(`OCR confidence: ${claimsStats.avg_metadata_value('ocr_confidence')}`);

// Get validation rule effectiveness
const validationStats = await analyzer.getRuleStats({
  rule_id: "coverage_limit_validator",
  rule_type: "validation",
  time_window: "30d"
});

console.log(`Validation match rate: ${validationStats.match_rate * 100}%`);
```

**Performance Target**: <50ms for parser stats, <100ms for rule stats.

---

### Cross-Domain Benefits

**Consistent Analysis**: All domains benefit from:
- Standardized performance metrics across pipeline stages
- Fast query performance for real-time monitoring
- Statistical rigor in metric calculations
- Comparative analysis across parsers and rules

**Operational Insights**: Supports:
- Capacity planning (throughput trends)
- SLA monitoring (latency percentiles)
- Cost optimization (identify expensive parsers)
- Quality assurance (error rate tracking)

---

## Core Concepts

### Performance Metrics

The PerformanceAnalyzer computes five categories of metrics:

1. **Latency Metrics**: Duration percentiles (p50, p95, p99)
2. **Throughput Metrics**: Executions per second/minute/hour
3. **Success Metrics**: Success rate, failure rate, error breakdown
4. **Efficiency Metrics**: Observations per execution, cost per execution
5. **Comparative Metrics**: Performance vs baseline, trends over time

### Time Windows

Supported time window formats:
- **Relative**: "1h", "24h", "7d", "30d"
- **Absolute**: "2025-01-01T00:00:00Z" to "2025-01-31T23:59:59Z"
- **Rolling**: Last N hours/days from now

### Statistical Methods

**Percentiles**: Use PERCENTILE_CONT (linear interpolation)
```sql
PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms)
```

**Rates**: Calculate as ratio of success/total
```sql
SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END)::float / COUNT(*)
```

**Throughput**: Count executions per time unit
```sql
COUNT(*) / EXTRACT(EPOCH FROM (MAX(created_at) - MIN(created_at))) * 3600
```

---

## Interface Definition

### TypeScript Interface

```typescript
interface PerformanceAnalyzer {
  /**
   * Get parser statistics for a single parser
   *
   * @param query - Parser stats query parameters
   * @returns Parser performance statistics
   */
  getParserStats(query: ParserStatsQuery): Promise<ParserStats>;

  /**
   * Get statistics for all parsers
   *
   * @param query - All parsers query parameters
   * @returns Array of parser statistics
   */
  getAllParserStats(query: AllParsersQuery): Promise<ParserStats[]>;

  /**
   * Get rule statistics for a single rule
   *
   * @param query - Rule stats query parameters
   * @returns Rule performance statistics
   */
  getRuleStats(query: RuleStatsQuery): Promise<RuleStats>;

  /**
   * Get statistics for all rules
   *
   * @param query - All rules query parameters
   * @returns Array of rule statistics
   */
  getAllRuleStats(query: AllRulesQuery): Promise<RuleStats[]>;

  /**
   * Get slowest parsers by p95 latency
   *
   * @param query - Slowest parsers query parameters
   * @returns Array of parsers sorted by p95 latency
   */
  getSlowestParsers(query: SlowestQuery): Promise<ParserRanking[]>;

  /**
   * Get slowest rules by p95 execution time
   *
   * @param query - Slowest rules query parameters
   * @returns Array of rules sorted by p95 execution time
   */
  getSlowestRules(query: SlowestQuery): Promise<RuleRanking[]>;

  /**
   * Get error breakdown for a parser
   *
   * @param query - Error breakdown query parameters
   * @returns Array of error types with counts
   */
  getErrorBreakdown(query: ErrorBreakdownQuery): Promise<ErrorBreakdown[]>;

  /**
   * Compare performance across multiple parsers
   *
   * @param query - Parser comparison query parameters
   * @returns Comparative performance statistics
   */
  compareParserPerformance(
    query: ParserComparisonQuery
  ): Promise<ParserComparison[]>;

  /**
   * Compare performance across multiple rules
   *
   * @param query - Rule comparison query parameters
   * @returns Comparative performance statistics
   */
  compareRulePerformance(
    query: RuleComparisonQuery
  ): Promise<RuleComparison[]>;

  /**
   * Get throughput trend over time
   *
   * @param query - Throughput trend query parameters
   * @returns Time-series throughput data
   */
  getThroughputTrend(query: ThroughputTrendQuery): Promise<ThroughputTrend[]>;

  /**
   * Get latency trend over time
   *
   * @param query - Latency trend query parameters
   * @returns Time-series latency data
   */
  getLatencyTrend(query: LatencyTrendQuery): Promise<LatencyTrend[]>;
}
```

---

## Data Model

### ParserStatsQuery Type

```typescript
interface ParserStatsQuery {
  parser_id: string;              // Parser identifier
  time_window: string;            // Time window (e.g., "24h", "7d")
}
```

### ParserStats Type

```typescript
interface ParserStats {
  // Identity
  parser_id: string;

  // Time Window
  time_window: string;
  start_time: string;             // ISO timestamp
  end_time: string;               // ISO timestamp

  // Execution Metrics
  total_executions: number;
  successful_executions: number;
  failed_executions: number;

  // Latency Metrics
  duration_p50_ms: number;
  duration_p95_ms: number;
  duration_p99_ms: number;
  duration_avg_ms: number;
  duration_min_ms: number;
  duration_max_ms: number;

  // Success Metrics
  success_rate: number;           // 0.0 to 1.0
  failure_rate: number;           // 0.0 to 1.0

  // Output Metrics
  total_observations_extracted: number;
  avg_observations_extracted: number;

  // Throughput Metrics
  executions_per_second: number;
  executions_per_minute: number;
  executions_per_hour: number;
}
```

### RuleStatsQuery Type

```typescript
interface RuleStatsQuery {
  rule_id: string;                // Rule identifier
  rule_type?: string;             // Optional rule type filter
  time_window: string;            // Time window
}
```

### RuleStats Type

```typescript
interface RuleStats {
  // Identity
  rule_id: string;
  rule_type: string;

  // Time Window
  time_window: string;
  start_time: string;
  end_time: string;

  // Execution Metrics
  total_executions: number;
  matched_executions: number;
  unmatched_executions: number;

  // Latency Metrics
  execution_time_p50_ms: number;
  execution_time_p95_ms: number;
  execution_time_p99_ms: number;
  execution_time_avg_ms: number;

  // Effectiveness Metrics
  match_rate: number;             // 0.0 to 1.0
  avg_confidence?: number;        // Optional confidence score

  // Throughput Metrics
  executions_per_second: number;
}
```

### ParserRanking Type

```typescript
interface ParserRanking {
  parser_id: string;
  duration_p95_ms: number;
  total_executions: number;
  success_rate: number;
}
```

### RuleRanking Type

```typescript
interface RuleRanking {
  rule_id: string;
  rule_type: string;
  execution_time_p95_ms: number;
  total_executions: number;
  match_rate: number;
}
```

### ErrorBreakdown Type

```typescript
interface ErrorBreakdown {
  error_type: string;
  count: number;
  percentage: number;             // 0.0 to 100.0
  recent_examples: string[];      // Sample error messages
}
```

### ParserComparison Type

```typescript
interface ParserComparison {
  parser_id: string;
  duration_p95_ms: number;
  success_rate: number;
  throughput: number;
  rank: number;                   // 1 = fastest
}
```

### ThroughputTrend Type

```typescript
interface ThroughputTrend {
  timestamp: string;              // ISO timestamp (bucket start)
  executions: number;
  executions_per_second: number;
}
```

### LatencyTrend Type

```typescript
interface LatencyTrend {
  timestamp: string;              // ISO timestamp (bucket start)
  duration_p50_ms: number;
  duration_p95_ms: number;
  duration_p99_ms: number;
}
```

---

## Core Functionality

### 1. getParserStats()

Get comprehensive statistics for a single parser.

#### Signature

```typescript
getParserStats(query: ParserStatsQuery): Promise<ParserStats>
```

#### Behavior

1. **Parse Time Window**:
   - Convert relative time window to absolute timestamps
   - Example: "24h" → (now - 24h, now)

2. **Query Metrics**:
   - Aggregate metrics from `parser_execution_metrics` table
   - Calculate percentiles using PERCENTILE_CONT
   - Calculate success/failure rates

3. **Return**:
   - Return comprehensive statistics

#### SQL Query

```sql
SELECT
  parser_id,
  COUNT(*) AS total_executions,
  SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) AS successful_executions,
  SUM(CASE WHEN status != 'success' THEN 1 ELSE 0 END) AS failed_executions,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY duration_ms) AS duration_p50_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS duration_p95_ms,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms) AS duration_p99_ms,
  AVG(duration_ms) AS duration_avg_ms,
  MIN(duration_ms) AS duration_min_ms,
  MAX(duration_ms) AS duration_max_ms,
  SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END)::float / COUNT(*) AS success_rate,
  SUM(observations_extracted) AS total_observations_extracted,
  AVG(observations_extracted) AS avg_observations_extracted,
  COUNT(*) / EXTRACT(EPOCH FROM (MAX(created_at) - MIN(created_at))) AS executions_per_second
FROM parser_execution_metrics
WHERE parser_id = $1
  AND created_at >= $2
  AND created_at <= $3
GROUP BY parser_id;
```

#### Example

```typescript
const stats = await analyzer.getParserStats({
  parser_id: "chase_pdf_parser_v2",
  time_window: "24h"
});

console.log(`Total executions: ${stats.total_executions}`);
console.log(`Success rate: ${(stats.success_rate * 100).toFixed(2)}%`);
console.log(`p50: ${stats.duration_p50_ms}ms`);
console.log(`p95: ${stats.duration_p95_ms}ms`);
console.log(`p99: ${stats.duration_p99_ms}ms`);
console.log(`Throughput: ${stats.executions_per_hour.toFixed(0)} executions/hour`);
```

#### Performance

- **Target Latency**: <50ms p95
- **Cache**: Cache results for 60 seconds

---

### 2. getAllParserStats()

Get statistics for all parsers.

#### Signature

```typescript
getAllParserStats(query: AllParsersQuery): Promise<ParserStats[]>
```

#### Behavior

1. **Query All Parsers**:
   - Aggregate metrics grouped by parser_id

2. **Sort**:
   - Sort by total_executions DESC (most active first)

3. **Return**:
   - Return array of parser statistics

#### Example

```typescript
const allStats = await analyzer.getAllParserStats({
  time_window: "7d"
});

allStats.forEach(stats => {
  console.log(`${stats.parser_id}:`);
  console.log(`  Executions: ${stats.total_executions}`);
  console.log(`  p95: ${stats.duration_p95_ms}ms`);
  console.log(`  Success rate: ${(stats.success_rate * 100).toFixed(2)}%`);
});
```

#### Performance

- **Target Latency**: <200ms p95
- **Optimization**: Use materialized view for faster queries

---

### 3. getRuleStats()

Get comprehensive statistics for a single rule.

#### Signature

```typescript
getRuleStats(query: RuleStatsQuery): Promise<RuleStats>
```

#### Behavior

1. **Parse Time Window**:
   - Convert relative time window to absolute timestamps

2. **Query Metrics**:
   - Aggregate metrics from `rule_execution_metrics` table
   - Calculate match rate (matched / total)

3. **Return**:
   - Return rule statistics

#### SQL Query

```sql
SELECT
  rule_id,
  rule_type,
  COUNT(*) AS total_executions,
  SUM(CASE WHEN matched = true THEN 1 ELSE 0 END) AS matched_executions,
  SUM(CASE WHEN matched = false THEN 1 ELSE 0 END) AS unmatched_executions,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY execution_time_ms) AS execution_time_p50_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY execution_time_ms) AS execution_time_p95_ms,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms) AS execution_time_p99_ms,
  AVG(execution_time_ms) AS execution_time_avg_ms,
  SUM(CASE WHEN matched = true THEN 1 ELSE 0 END)::float / COUNT(*) AS match_rate,
  COUNT(*) / EXTRACT(EPOCH FROM (MAX(created_at) - MIN(created_at))) AS executions_per_second
FROM rule_execution_metrics
WHERE rule_id = $1
  AND created_at >= $2
  AND created_at <= $3
GROUP BY rule_id, rule_type;
```

#### Example

```typescript
const ruleStats = await analyzer.getRuleStats({
  rule_id: "merchant_categorization_ml",
  rule_type: "normalization",
  time_window: "24h"
});

console.log(`Total executions: ${ruleStats.total_executions}`);
console.log(`Match rate: ${(ruleStats.match_rate * 100).toFixed(2)}%`);
console.log(`p95 execution time: ${ruleStats.execution_time_p95_ms}ms`);
```

#### Performance

- **Target Latency**: <50ms p95

---

### 4. getSlowestParsers()

Get slowest parsers ranked by p95 latency.

#### Signature

```typescript
getSlowestParsers(query: SlowestQuery): Promise<ParserRanking[]>
```

#### Behavior

1. **Query All Parsers**:
   - Calculate p95 latency for each parser

2. **Sort**:
   - Sort by duration_p95_ms DESC

3. **Limit**:
   - Return top N parsers

#### SQL Query

```sql
SELECT
  parser_id,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS duration_p95_ms,
  COUNT(*) AS total_executions,
  SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END)::float / COUNT(*) AS success_rate
FROM parser_execution_metrics
WHERE created_at >= $1 AND created_at <= $2
GROUP BY parser_id
ORDER BY duration_p95_ms DESC
LIMIT $3;
```

#### Example

```typescript
const slowest = await analyzer.getSlowestParsers({
  time_window: "7d",
  limit: 10
});

slowest.forEach((parser, index) => {
  console.log(`${index + 1}. ${parser.parser_id}: ${parser.duration_p95_ms}ms p95`);
});
```

#### Performance

- **Target Latency**: <100ms p95

---

### 5. getSlowestRules()

Get slowest rules ranked by p95 execution time.

#### Signature

```typescript
getSlowestRules(query: SlowestQuery): Promise<RuleRanking[]>
```

#### Behavior

Similar to getSlowestParsers(), but for rules.

#### Example

```typescript
const slowestRules = await analyzer.getSlowestRules({
  rule_type: "normalization",
  time_window: "7d",
  limit: 10
});

slowestRules.forEach((rule, index) => {
  console.log(`${index + 1}. ${rule.rule_id}: ${rule.execution_time_p95_ms}ms p95`);
});
```

#### Performance

- **Target Latency**: <80ms p95

---

### 6. getErrorBreakdown()

Get error breakdown by error type.

#### Signature

```typescript
getErrorBreakdown(query: ErrorBreakdownQuery): Promise<ErrorBreakdown[]>
```

#### Behavior

1. **Query Failed Executions**:
   - Filter by status != 'success'

2. **Group By Error Type**:
   - Count occurrences of each error type

3. **Calculate Percentages**:
   - Percentage = (count / total errors) * 100

4. **Sample Examples**:
   - Get recent error messages for each type

#### SQL Query

```sql
WITH error_counts AS (
  SELECT
    error_type,
    COUNT(*) AS count
  FROM parser_execution_metrics
  WHERE parser_id = $1
    AND status != 'success'
    AND created_at >= $2
    AND created_at <= $3
  GROUP BY error_type
),
total_errors AS (
  SELECT SUM(count) AS total FROM error_counts
),
recent_examples AS (
  SELECT
    error_type,
    ARRAY_AGG(metadata->>'error_message' ORDER BY created_at DESC LIMIT 3) AS examples
  FROM parser_execution_metrics
  WHERE parser_id = $1
    AND status != 'success'
    AND created_at >= $2
    AND created_at <= $3
  GROUP BY error_type
)
SELECT
  ec.error_type,
  ec.count,
  (ec.count::float / te.total * 100) AS percentage,
  re.examples AS recent_examples
FROM error_counts ec
CROSS JOIN total_errors te
LEFT JOIN recent_examples re ON ec.error_type = re.error_type
ORDER BY ec.count DESC;
```

#### Example

```typescript
const errors = await analyzer.getErrorBreakdown({
  parser_id: "chase_pdf_parser_v2",
  time_window: "24h"
});

errors.forEach(error => {
  console.log(`${error.error_type}: ${error.count} occurrences (${error.percentage.toFixed(2)}%)`);
  console.log(`  Recent examples:`);
  error.recent_examples.forEach(example => {
    console.log(`    - ${example}`);
  });
});
```

#### Performance

- **Target Latency**: <100ms p95

---

### 7. compareParserPerformance()

Compare performance across multiple parsers.

#### Signature

```typescript
compareParserPerformance(
  query: ParserComparisonQuery
): Promise<ParserComparison[]>
```

#### Behavior

1. **Query Each Parser**:
   - Get p95 latency, success rate, throughput

2. **Rank**:
   - Rank by p95 latency (1 = fastest)

3. **Return**:
   - Return array sorted by rank

#### Example

```typescript
const comparison = await analyzer.compareParserPerformance({
  parser_ids: [
    "chase_pdf_parser_v2",
    "wells_fargo_parser",
    "bofa_parser"
  ],
  time_window: "7d"
});

comparison.forEach(parser => {
  console.log(`Rank ${parser.rank}: ${parser.parser_id}`);
  console.log(`  p95: ${parser.duration_p95_ms}ms`);
  console.log(`  Success rate: ${(parser.success_rate * 100).toFixed(2)}%`);
  console.log(`  Throughput: ${parser.throughput.toFixed(0)} exec/hour`);
});
```

#### Performance

- **Target Latency**: <150ms p95

---

### 8. getThroughputTrend()

Get throughput trend over time.

#### Signature

```typescript
getThroughputTrend(query: ThroughputTrendQuery): Promise<ThroughputTrend[]>
```

#### Behavior

1. **Bucket By Time**:
   - Group metrics into time buckets (e.g., hourly)

2. **Calculate Throughput**:
   - Count executions per bucket

3. **Return**:
   - Return time-series data

#### SQL Query

```sql
SELECT
  DATE_TRUNC('hour', created_at) AS timestamp,
  COUNT(*) AS executions,
  COUNT(*) / 3600.0 AS executions_per_second
FROM parser_execution_metrics
WHERE parser_id = $1
  AND created_at >= $2
  AND created_at <= $3
GROUP BY DATE_TRUNC('hour', created_at)
ORDER BY timestamp;
```

#### Example

```typescript
const trend = await analyzer.getThroughputTrend({
  parser_id: "chase_pdf_parser_v2",
  time_window: "7d",
  bucket_size: "1h"
});

trend.forEach(point => {
  console.log(`${point.timestamp}: ${point.executions_per_second.toFixed(2)} exec/sec`);
});
```

#### Performance

- **Target Latency**: <120ms p95

---

### 9. getLatencyTrend()

Get latency trend over time.

#### Signature

```typescript
getLatencyTrend(query: LatencyTrendQuery): Promise<LatencyTrend[]>
```

#### Behavior

1. **Bucket By Time**:
   - Group metrics into time buckets

2. **Calculate Percentiles**:
   - Calculate p50/p95/p99 for each bucket

3. **Return**:
   - Return time-series data

#### SQL Query

```sql
SELECT
  DATE_TRUNC('hour', created_at) AS timestamp,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY duration_ms) AS duration_p50_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS duration_p95_ms,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms) AS duration_p99_ms
FROM parser_execution_metrics
WHERE parser_id = $1
  AND created_at >= $2
  AND created_at <= $3
GROUP BY DATE_TRUNC('hour', created_at)
ORDER BY timestamp;
```

#### Example

```typescript
const trend = await analyzer.getLatencyTrend({
  parser_id: "chase_pdf_parser_v2",
  time_window: "7d",
  bucket_size: "1h"
});

trend.forEach(point => {
  console.log(`${point.timestamp}:`);
  console.log(`  p50: ${point.duration_p50_ms}ms`);
  console.log(`  p95: ${point.duration_p95_ms}ms`);
  console.log(`  p99: ${point.duration_p99_ms}ms`);
});
```

#### Performance

- **Target Latency**: <150ms p95

---

## Aggregation Strategies

### Pre-Aggregation with Materialized Views

Create materialized views for common queries:

```sql
-- Hourly parser stats
CREATE MATERIALIZED VIEW parser_stats_hourly AS
SELECT
  parser_id,
  DATE_TRUNC('hour', created_at) AS hour,
  COUNT(*) AS total_executions,
  SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) AS successful_executions,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY duration_ms) AS duration_p50_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS duration_p95_ms,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms) AS duration_p99_ms,
  AVG(duration_ms) AS duration_avg_ms
FROM parser_execution_metrics
GROUP BY parser_id, hour;

CREATE INDEX idx_parser_stats_hourly_parser_id ON parser_stats_hourly(parser_id);
CREATE INDEX idx_parser_stats_hourly_hour ON parser_stats_hourly(hour DESC);

-- Refresh every 5 minutes
REFRESH MATERIALIZED VIEW parser_stats_hourly;
```

### Continuous Aggregation

Use PostgreSQL timescaledb extension for continuous aggregation:

```sql
-- Convert table to hypertable (timescaledb)
SELECT create_hypertable('parser_execution_metrics', 'created_at');

-- Create continuous aggregate
CREATE MATERIALIZED VIEW parser_stats_hourly_continuous
WITH (timescaledb.continuous) AS
SELECT
  parser_id,
  time_bucket('1 hour', created_at) AS bucket,
  COUNT(*) AS total_executions,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS duration_p95_ms
FROM parser_execution_metrics
GROUP BY parser_id, bucket;

-- Refresh policy (automatic)
SELECT add_continuous_aggregate_policy('parser_stats_hourly_continuous',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '5 minutes');
```

---

## Statistical Methods

### Percentile Calculation

Use PERCENTILE_CONT for continuous percentiles:

```sql
-- p50 (median)
PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY duration_ms)

-- p95 (95th percentile)
PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms)

-- p99 (99th percentile)
PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms)
```

### Rate Calculation

```sql
-- Success rate
SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END)::float / COUNT(*)

-- Error rate
SUM(CASE WHEN status != 'success' THEN 1 ELSE 0 END)::float / COUNT(*)

-- Match rate (rules)
SUM(CASE WHEN matched = true THEN 1 ELSE 0 END)::float / COUNT(*)
```

### Throughput Calculation

```sql
-- Executions per second
COUNT(*) / EXTRACT(EPOCH FROM (MAX(created_at) - MIN(created_at)))

-- Executions per hour
COUNT(*) / EXTRACT(EPOCH FROM (MAX(created_at) - MIN(created_at))) * 3600
```

---

## Storage & Indexing

### Indexes for Fast Queries

```sql
-- Composite index for time-range queries by parser
CREATE INDEX idx_parser_metrics_parser_time ON parser_execution_metrics(parser_id, created_at DESC);

-- Composite index for time-range queries by rule
CREATE INDEX idx_rule_metrics_rule_time ON rule_execution_metrics(rule_id, created_at DESC);

-- Index for status filtering
CREATE INDEX idx_parser_metrics_status ON parser_execution_metrics(status);

-- Index for error type queries
CREATE INDEX idx_parser_metrics_error_type ON parser_execution_metrics(error_type) WHERE status != 'success';
```

### Query Optimization

```sql
-- Use EXPLAIN ANALYZE to verify index usage
EXPLAIN ANALYZE
SELECT
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms)
FROM parser_execution_metrics
WHERE parser_id = 'chase_pdf_parser_v2'
  AND created_at >= NOW() - INTERVAL '24 hours';

-- Expected: Index Scan using idx_parser_metrics_parser_time
```

---

## Edge Cases

### Edge Case 1: No Data for Time Window

**Scenario**: Query for parser with no executions in time window.

**Detection**:
```typescript
if (stats.total_executions === 0) {
  return null; // Or return stats with null percentiles
}
```

**Resolution**:
- Return null or empty stats object
- Document behavior in API

**Example**:
```typescript
const stats = await analyzer.getParserStats({
  parser_id: "unused_parser",
  time_window: "24h"
});

if (stats === null) {
  console.log("No data available for parser in time window");
}
```

---

### Edge Case 2: Single Data Point

**Scenario**: Parser has only 1 execution in time window.

**Detection**:
```typescript
if (stats.total_executions === 1) {
  // All percentiles equal to single value
}
```

**Resolution**:
- Set p50 = p95 = p99 = single value
- Display warning in UI

---

### Edge Case 3: All Executions Failed

**Scenario**: Parser has 100% failure rate.

**Detection**:
```typescript
if (stats.success_rate === 0) {
  console.warn(`Parser ${stats.parser_id} has 100% failure rate`);
}
```

**Resolution**:
- Highlight in dashboard
- Alert operations team

---

### Edge Case 4: Extreme Outliers

**Scenario**: One execution takes 10x longer than others, skewing p99.

**Detection**:
```sql
-- Detect outliers (> 3 standard deviations)
WITH stats AS (
  SELECT
    AVG(duration_ms) AS mean,
    STDDEV(duration_ms) AS stddev
  FROM parser_execution_metrics
  WHERE parser_id = $1
)
SELECT *
FROM parser_execution_metrics
WHERE parser_id = $1
  AND ABS(duration_ms - (SELECT mean FROM stats)) > 3 * (SELECT stddev FROM stats);
```

**Resolution**:
- Show p99 and max separately
- Investigate outliers manually

---

### Edge Case 5: Time Zone Issues

**Scenario**: Query time window in wrong timezone.

**Resolution**:
- Always use UTC for storage and queries
- Convert to local timezone only for display

**Example**:
```typescript
// Always use UTC
const now = new Date().toISOString(); // Includes 'Z' suffix

const stats = await analyzer.getParserStats({
  parser_id: "chase_parser",
  time_window: "24h" // Interpreted as last 24 hours in UTC
});
```

---

### Edge Case 6: Very Long Time Windows

**Scenario**: Query for "365d" time window with millions of metrics.

**Detection**:
```typescript
if (timeWindowDays > 90) {
  console.warn("Long time window may cause slow queries");
}
```

**Resolution**:
- Use materialized views for long time windows
- Recommend shorter time windows or pre-aggregated data

---

### Edge Case 7: Concurrent Materialized View Refresh

**Scenario**: Query runs during materialized view refresh, getting stale data.

**Resolution**:
- Use CONCURRENTLY option for non-blocking refresh
- Accept eventual consistency (data may be 5 minutes old)

**Example**:
```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY parser_stats_hourly;
```

---

### Edge Case 8: Division by Zero

**Scenario**: Calculate executions_per_second when time range is 0.

**Detection**:
```sql
-- Prevent division by zero
SELECT
  COUNT(*) / GREATEST(EXTRACT(EPOCH FROM (MAX(created_at) - MIN(created_at))), 1) AS executions_per_second
FROM parser_execution_metrics;
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `getParserStats()` | 1 parser, 24h | < 50ms | Single parser, small window |
| `getAllParserStats()` | All parsers, 7d | < 200ms | All parsers, medium window |
| `getRuleStats()` | 1 rule, 24h | < 50ms | Single rule |
| `getAllRuleStats()` | All rules, 7d | < 150ms | All rules |
| `getSlowestParsers()` | Top 10, 7d | < 100ms | Requires full scan |
| `getSlowestRules()` | Top 10, 7d | < 80ms | Fewer rules than parsers |
| `getErrorBreakdown()` | 1 parser, 24h | < 100ms | Error filtering + grouping |
| `compareParserPerformance()` | 3 parsers, 7d | < 150ms | Multiple parser queries |
| `getThroughputTrend()` | 1 parser, 7d, hourly | < 120ms | Time bucketing |
| `getLatencyTrend()` | 1 parser, 7d, hourly | < 150ms | Percentile calculation |

### Throughput

- **Concurrent Queries**: 100+ queries/sec (with caching)
- **Cache Hit Rate**: >80% for frequently accessed stats

### Scalability

| Metrics Stored | Query Performance | Notes |
|---------------|------------------|-------|
| < 1M | Excellent (< 50ms) | All queries fast |
| 1M - 10M | Excellent (< 100ms) | Use materialized views |
| 10M - 100M | Good (< 500ms) | Partitioning required |
| > 100M | Moderate (< 2s) | Use continuous aggregation |

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';
import LRU from 'lru-cache';

export class PostgresPerformanceAnalyzer implements PerformanceAnalyzer {
  private pool: Pool;
  private cache: LRU<string, any>;

  constructor(pool: Pool) {
    this.pool = pool;

    // Initialize cache (60 second TTL)
    this.cache = new LRU({
      max: 1000,
      ttl: 60000 // 60 seconds
    });
  }

  async getParserStats(query: ParserStatsQuery): Promise<ParserStats> {
    // Check cache
    const cacheKey = `parser_stats:${query.parser_id}:${query.time_window}`;
    const cached = this.cache.get(cacheKey);
    if (cached) {
      return cached;
    }

    // Parse time window
    const { start_time, end_time } = this.parseTimeWindow(query.time_window);

    // Query metrics
    const sql = `
      SELECT
        parser_id,
        COUNT(*) AS total_executions,
        SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) AS successful_executions,
        SUM(CASE WHEN status != 'success' THEN 1 ELSE 0 END) AS failed_executions,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY duration_ms) AS duration_p50_ms,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS duration_p95_ms,
        PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms) AS duration_p99_ms,
        AVG(duration_ms) AS duration_avg_ms,
        MIN(duration_ms) AS duration_min_ms,
        MAX(duration_ms) AS duration_max_ms,
        SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END)::float / COUNT(*) AS success_rate,
        SUM(observations_extracted) AS total_observations_extracted,
        AVG(observations_extracted) AS avg_observations_extracted,
        MIN(created_at) AS start_time,
        MAX(created_at) AS end_time
      FROM parser_execution_metrics
      WHERE parser_id = $1
        AND created_at >= $2
        AND created_at <= $3
      GROUP BY parser_id
    `;

    const result = await this.pool.query(sql, [
      query.parser_id,
      start_time,
      end_time
    ]);

    if (result.rows.length === 0) {
      return null;
    }

    const row = result.rows[0];

    // Calculate throughput
    const durationSeconds = (new Date(row.end_time).getTime() - new Date(row.start_time).getTime()) / 1000;
    const executions_per_second = row.total_executions / durationSeconds;

    const stats: ParserStats = {
      parser_id: row.parser_id,
      time_window: query.time_window,
      start_time: start_time.toISOString(),
      end_time: end_time.toISOString(),
      total_executions: parseInt(row.total_executions),
      successful_executions: parseInt(row.successful_executions),
      failed_executions: parseInt(row.failed_executions),
      duration_p50_ms: parseFloat(row.duration_p50_ms),
      duration_p95_ms: parseFloat(row.duration_p95_ms),
      duration_p99_ms: parseFloat(row.duration_p99_ms),
      duration_avg_ms: parseFloat(row.duration_avg_ms),
      duration_min_ms: parseInt(row.duration_min_ms),
      duration_max_ms: parseInt(row.duration_max_ms),
      success_rate: parseFloat(row.success_rate),
      failure_rate: 1 - parseFloat(row.success_rate),
      total_observations_extracted: parseInt(row.total_observations_extracted),
      avg_observations_extracted: parseFloat(row.avg_observations_extracted),
      executions_per_second,
      executions_per_minute: executions_per_second * 60,
      executions_per_hour: executions_per_second * 3600
    };

    // Cache result
    this.cache.set(cacheKey, stats);

    return stats;
  }

  async getAllParserStats(query: AllParsersQuery): Promise<ParserStats[]> {
    // Implementation similar to getParserStats, but GROUP BY parser_id
    // Omitted for brevity
  }

  async getSlowestParsers(query: SlowestQuery): Promise<ParserRanking[]> {
    const { start_time, end_time } = this.parseTimeWindow(query.time_window);

    const sql = `
      SELECT
        parser_id,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS duration_p95_ms,
        COUNT(*) AS total_executions,
        SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END)::float / COUNT(*) AS success_rate
      FROM parser_execution_metrics
      WHERE created_at >= $1 AND created_at <= $2
      GROUP BY parser_id
      ORDER BY duration_p95_ms DESC
      LIMIT $3
    `;

    const result = await this.pool.query(sql, [
      start_time,
      end_time,
      query.limit || 10
    ]);

    return result.rows.map(row => ({
      parser_id: row.parser_id,
      duration_p95_ms: parseFloat(row.duration_p95_ms),
      total_executions: parseInt(row.total_executions),
      success_rate: parseFloat(row.success_rate)
    }));
  }

  private parseTimeWindow(timeWindow: string): { start_time: Date; end_time: Date } {
    const now = new Date();
    const regex = /^(\d+)([hdwmy])$/;
    const match = timeWindow.match(regex);

    if (!match) {
      throw new Error(`Invalid time window format: ${timeWindow}`);
    }

    const value = parseInt(match[1]);
    const unit = match[2];

    let start_time: Date;

    switch (unit) {
      case 'h':
        start_time = new Date(now.getTime() - value * 3600 * 1000);
        break;
      case 'd':
        start_time = new Date(now.getTime() - value * 86400 * 1000);
        break;
      case 'w':
        start_time = new Date(now.getTime() - value * 7 * 86400 * 1000);
        break;
      case 'm':
        start_time = new Date(now.getTime() - value * 30 * 86400 * 1000);
        break;
      case 'y':
        start_time = new Date(now.getTime() - value * 365 * 86400 * 1000);
        break;
      default:
        throw new Error(`Unknown time unit: ${unit}`);
    }

    return { start_time, end_time: now };
  }
}
```

---

## Security Considerations

### 1. Query Injection Prevention

**Prevent SQL Injection**:
```typescript
// Always use parameterized queries
const sql = `
  SELECT * FROM parser_execution_metrics
  WHERE parser_id = $1
`;

await this.pool.query(sql, [parser_id]);
```

### 2. Rate Limiting

**Prevent Excessive Queries**:
```typescript
class RateLimitedAnalyzer implements PerformanceAnalyzer {
  private requestCounts = new Map<string, number>();
  private maxRequestsPerMinute = 100;

  async getParserStats(query: ParserStatsQuery): Promise<ParserStats> {
    const key = `parser_stats:${query.parser_id}`;
    const count = this.requestCounts.get(key) || 0;

    if (count >= this.maxRequestsPerMinute) {
      throw new RateLimitError(`Rate limit exceeded for ${key}`);
    }

    this.requestCounts.set(key, count + 1);

    return this.analyzer.getParserStats(query);
  }
}
```

### 3. Access Control

**Role-Based Query Access**:
```typescript
async getParserStats(
  query: ParserStatsQuery,
  user: User
): Promise<ParserStats> {
  // Check if user has permission to view parser stats
  if (!user.hasRole('metrics_reader')) {
    throw new UnauthorizedError('User does not have permission to view metrics');
  }

  return this.analyzer.getParserStats(query);
}
```

---

## Integration Patterns

### Pattern 1: Dashboard Integration

```typescript
// Real-time dashboard that refreshes every 60 seconds
class PerformanceDashboard {
  private analyzer: PerformanceAnalyzer;

  async refreshDashboard(): Promise<DashboardData> {
    // Get all parser stats
    const parserStats = await this.analyzer.getAllParserStats({
      time_window: "1h"
    });

    // Get slowest parsers
    const slowest = await this.analyzer.getSlowestParsers({
      time_window: "1h",
      limit: 5
    });

    // Get overall throughput
    const totalThroughput = parserStats.reduce(
      (sum, stats) => sum + stats.executions_per_second,
      0
    );

    return {
      parserStats,
      slowest,
      totalThroughput
    };
  }
}
```

### Pattern 2: Alert Generation

```typescript
// Alert on degraded performance
class PerformanceAlerter {
  async checkForAlerts(): Promise<Alert[]> {
    const alerts: Alert[] = [];

    // Check all parser stats
    const stats = await this.analyzer.getAllParserStats({
      time_window: "1h"
    });

    for (const parserStat of stats) {
      // Alert if p95 exceeds threshold
      if (parserStat.duration_p95_ms > 5000) {
        alerts.push({
          type: "HIGH_LATENCY",
          parser_id: parserStat.parser_id,
          message: `Parser ${parserStat.parser_id} p95 latency is ${parserStat.duration_p95_ms}ms (threshold: 5000ms)`
        });
      }

      // Alert if success rate below threshold
      if (parserStat.success_rate < 0.95) {
        alerts.push({
          type: "LOW_SUCCESS_RATE",
          parser_id: parserStat.parser_id,
          message: `Parser ${parserStat.parser_id} success rate is ${(parserStat.success_rate * 100).toFixed(2)}% (threshold: 95%)`
        });
      }
    }

    return alerts;
  }
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section with 7 domains)

---

## Testing Strategy

### Unit Tests

```typescript
describe('PerformanceAnalyzer', () => {
  let analyzer: PerformanceAnalyzer;
  let pool: Pool;

  beforeEach(async () => {
    pool = new Pool({ /* test config */ });
    analyzer = new PostgresPerformanceAnalyzer(pool);
    await seedTestData();
  });

  describe('getParserStats()', () => {
    it('should return parser stats', async () => {
      const stats = await analyzer.getParserStats({
        parser_id: 'test_parser',
        time_window: '24h'
      });

      expect(stats.parser_id).toBe('test_parser');
      expect(stats.total_executions).toBeGreaterThan(0);
      expect(stats.duration_p95_ms).toBeGreaterThan(0);
      expect(stats.success_rate).toBeGreaterThanOrEqual(0);
      expect(stats.success_rate).toBeLessThanOrEqual(1);
    });

    it('should return null for parser with no data', async () => {
      const stats = await analyzer.getParserStats({
        parser_id: 'nonexistent_parser',
        time_window: '24h'
      });

      expect(stats).toBeNull();
    });
  });

  describe('getSlowestParsers()', () => {
    it('should return parsers sorted by p95 latency', async () => {
      const slowest = await analyzer.getSlowestParsers({
        time_window: '7d',
        limit: 5
      });

      expect(slowest.length).toBeLessThanOrEqual(5);

      // Verify sorted descending
      for (let i = 1; i < slowest.length; i++) {
        expect(slowest[i].duration_p95_ms).toBeLessThanOrEqual(
          slowest[i - 1].duration_p95_ms
        );
      }
    });
  });

  describe('getErrorBreakdown()', () => {
    it('should return error breakdown with percentages', async () => {
      const errors = await analyzer.getErrorBreakdown({
        parser_id: 'test_parser',
        time_window: '24h'
      });

      const totalPercentage = errors.reduce(
        (sum, error) => sum + error.percentage,
        0
      );

      expect(totalPercentage).toBeCloseTo(100, 1);
    });
  });
});
```

### Performance Tests

```typescript
describe('PerformanceAnalyzer Performance', () => {
  it('should get parser stats in <50ms', async () => {
    const startTime = Date.now();

    await analyzer.getParserStats({
      parser_id: 'test_parser',
      time_window: '24h'
    });

    const duration = Date.now() - startTime;
    console.log(`Query duration: ${duration}ms`);
    expect(duration).toBeLessThan(50);
  });

  it('should get all parser stats in <200ms', async () => {
    const startTime = Date.now();

    await analyzer.getAllParserStats({
      time_window: '7d'
    });

    const duration = Date.now() - startTime;
    console.log(`Query duration: ${duration}ms`);
    expect(duration).toBeLessThan(200);
  });
});
```

---

## Configuration

### Tunable Parameters

```typescript
interface AnalyzerConfig {
  // Cache Settings
  cacheEnabled: boolean;          // Default: true
  cacheTTLSeconds: number;        // Default: 60

  // Query Settings
  defaultTimeWindow: string;      // Default: "24h"
  maxTimeWindowDays: number;      // Default: 365

  // Pagination
  defaultLimit: number;           // Default: 10
  maxLimit: number;               // Default: 100
}
```

---

## Observability

### Metrics

```typescript
interface AnalyzerMetrics {
  queries_total: Counter;
  query_duration_ms: Histogram;
  cache_hits_total: Counter;
  cache_misses_total: Counter;
}
```

---

## Future Enhancements

### Phase 1: Predictive Analytics (3 months)
- Forecast future performance degradation
- Capacity planning recommendations

### Phase 2: Anomaly Detection (6 months)
- Automatic anomaly detection using ML
- Root cause analysis

### Phase 3: Cost Analysis (12 months)
- Calculate cost per execution
- ROI analysis for parser optimization

---

**End of PerformanceAnalyzer OL Primitive Specification**
