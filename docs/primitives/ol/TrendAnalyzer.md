# TrendAnalyzer OL Primitive

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
8. [Statistical Algorithms](#statistical-algorithms)
9. [Anomaly Detection](#anomaly-detection)
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

The **TrendAnalyzer** analyzes historical performance metrics to detect trends, forecast capacity, and identify anomalies. It uses statistical methods (z-score, linear regression, baseline comparison) to provide predictive insights into pipeline performance, enabling proactive capacity planning and early anomaly detection.

### Key Capabilities

- **Trend Detection**: <150ms p95 performance trend queries
- **Anomaly Detection**: <300ms p95 anomaly detection (z-score > 2.5)
- **Capacity Forecasting**: Predict future capacity needs using linear regression
- **Baseline Comparison**: Compare current performance vs historical baseline
- **Error Trend Analysis**: Track error rate trends over time
- **Seasonal Pattern Detection**: Identify daily/weekly patterns

### Design Philosophy

The TrendAnalyzer follows four core principles:

1. **Statistical Rigor**: Use proven statistical methods (z-score, regression)
2. **Predictive Insights**: Forecast future performance based on trends
3. **Actionable Alerts**: Surface anomalies that require intervention
4. **Performance Optimization**: <300ms for complex anomaly detection

---

## Purpose & Scope

### Problem Statement

In document processing pipelines, trend analysis poses critical challenges:

- **No Historical Context**: Can't compare current performance to past
- **Reactive Monitoring**: Issues detected after they occur
- **No Capacity Planning**: Can't predict future resource needs
- **Manual Analysis**: Requires data science expertise to analyze trends
- **Missed Anomalies**: Performance degradation goes unnoticed

Traditional approaches have significant limitations:

**Approach 1: Manual Analysis**
- ❌ Requires data science expertise
- ❌ Time-consuming (hours to days)
- ❌ Not real-time

**Approach 2: Static Thresholds**
- ❌ Don't adapt to changing baselines
- ❌ High false positive rate
- ❌ No predictive capability

**Approach 3: Trend Analyzer**
- ✅ Automatic trend detection
- ✅ Statistical anomaly detection (z-score)
- ✅ Capacity forecasting (linear regression)
- ✅ Real-time analysis (<300ms)
- ✅ Adaptive baselines

### Solution

The TrendAnalyzer implements a **statistical trend analysis engine** with:

1. **Time-Series Analysis**: Analyze metrics over time windows
2. **Z-Score Anomaly Detection**: Detect outliers (z-score > 2.5)
3. **Linear Regression Forecasting**: Predict future capacity needs
4. **Baseline Comparison**: Compare vs rolling 7-day average
5. **Seasonal Decomposition**: Identify daily/weekly patterns

---

## Multi-Domain Applicability

The TrendAnalyzer is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Detect performance degradation in bank statement parsers and forecast capacity for month-end.

**Trend Analysis**:
- **Performance Trend**: Chase parser p95 latency increased 25% over 7 days
- **Error Trend**: OCR failure rate spiked to 8% (baseline: 2%)
- **Capacity Forecast**: Need 50% more capacity for month-end processing
- **Anomaly Detection**: Transaction categorization rule 5x slower than baseline

**Example**:
```typescript
// Get performance trend
const trend = await analyzer.getPerformanceTrend({
  parser_id: "chase_pdf_parser_v2",
  time_window: "30d",
  metric: "duration_p95_ms"
});

console.log(`Trend: ${trend.direction} (${trend.percentage_change}%)`);
console.log(`Current: ${trend.current_value}ms`);
console.log(`Baseline: ${trend.baseline_value}ms`);

if (trend.direction === "increasing" && trend.percentage_change > 20) {
  console.log("⚠️  Performance degradation detected");
}

// Get error trend
const errorTrend = await analyzer.getErrorTrend({
  parser_id: "chase_pdf_parser_v2",
  time_window: "7d",
  error_type: "OCR_FAILURE"
});

console.log(`Error rate trend: ${errorTrend.direction}`);
console.log(`Current error rate: ${(errorTrend.current_rate * 100).toFixed(2)}%`);
console.log(`Baseline error rate: ${(errorTrend.baseline_rate * 100).toFixed(2)}%`);

// Forecast capacity
const forecast = await analyzer.forecastCapacity({
  parser_id: "chase_pdf_parser_v2",
  forecast_days: 30,
  metric: "executions_per_hour"
});

console.log(`Current capacity: ${forecast.current_value} exec/hour`);
console.log(`Forecasted capacity (30d): ${forecast.forecasted_value} exec/hour`);
console.log(`Capacity increase needed: ${forecast.percentage_increase}%`);

// Detect anomalies
const anomalies = await analyzer.detectAnomalies({
  parser_id: "chase_pdf_parser_v2",
  time_window: "24h",
  metric: "duration_ms",
  threshold_z_score: 2.5
});

console.log(`Anomalies detected: ${anomalies.length}`);

anomalies.forEach(anomaly => {
  console.log(`Anomaly: ${anomaly.metric_value}ms (z-score: ${anomaly.z_score.toFixed(2)})`);
  console.log(`  Timestamp: ${anomaly.timestamp}`);
  console.log(`  Upload ID: ${anomaly.upload_id}`);
});
```

**Performance Target**: <150ms for trend queries, <300ms for anomaly detection.

---

### 2. Healthcare

**Use Case**: Monitor HL7 parser performance trends and detect anomalies in message processing.

**Trend Analysis**:
- **Performance Trend**: HL7 ORU parser 15% slower during night shift
- **Error Trend**: FHIR validation failures increased 3x
- **Capacity Forecast**: Peak message volume predicted to increase 40%
- **Anomaly Detection**: 5 messages took >10 seconds (z-score: 3.2)

**Example**:
```typescript
// Detect seasonal patterns
const patterns = await analyzer.detectSeasonalPatterns({
  parser_id: "hl7_oru_r01_parser",
  time_window: "7d",
  metric: "duration_p95_ms"
});

console.log(`Daily patterns detected: ${patterns.daily_pattern_detected}`);
console.log(`Peak hours: ${patterns.peak_hours.join(", ")}`);
console.log(`Off-peak hours: ${patterns.off_peak_hours.join(", ")}`);

// Compare current vs baseline
const comparison = await analyzer.compareVsBaseline({
  parser_id: "hl7_oru_r01_parser",
  current_window: "1h",
  baseline_window: "7d",
  metric: "duration_p95_ms"
});

console.log(`Current p95: ${comparison.current_value}ms`);
console.log(`Baseline p95: ${comparison.baseline_value}ms`);
console.log(`Difference: ${comparison.percentage_difference}%`);

if (comparison.percentage_difference > 30) {
  console.log("⚠️  Significant performance degradation");
}
```

**Performance Target**: <120ms for trend queries, <250ms for pattern detection.

---

### 3. Legal

**Use Case**: Forecast e-discovery processing capacity and detect OCR performance degradation.

**Trend Analysis**:
- **Performance Trend**: OCR processing time increased 40% over 14 days
- **Error Trend**: Document classification accuracy decreased to 75% (baseline: 90%)
- **Capacity Forecast**: Need 2x capacity for upcoming 100K document production
- **Anomaly Detection**: 15 documents took >10 minutes to OCR (z-score: 4.1)

**Example**:
```typescript
// Forecast capacity for large production
const forecast = await analyzer.forecastCapacity({
  parser_id: "tesseract_legal_ocr",
  forecast_days: 7,
  metric: "executions_per_hour",
  target_volume: 100000
});

console.log(`Current throughput: ${forecast.current_value} docs/hour`);
console.log(`Target volume: 100,000 documents`);
console.log(`Estimated completion: ${forecast.estimated_completion_hours} hours`);
console.log(`Capacity increase needed: ${forecast.percentage_increase}%`);

if (forecast.percentage_increase > 50) {
  console.log("⚠️  Significant capacity increase needed");
  console.log(`Recommendation: Scale up workers by ${Math.ceil(forecast.percentage_increase / 10)}x`);
}

// Get error rate trend
const errorTrend = await analyzer.getErrorTrend({
  parser_id: "doc_type_classifier_v3",
  time_window: "14d",
  error_type: "CLASSIFICATION_ERROR"
});

if (errorTrend.direction === "increasing" && errorTrend.percentage_change > 20) {
  console.log("⚠️  Classification accuracy declining");
  console.log(`Current error rate: ${(errorTrend.current_rate * 100).toFixed(2)}%`);
  console.log(`Recommendation: Retrain classification model`);
}
```

**Performance Target**: <180ms for forecast queries, <300ms for anomaly detection.

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Monitor founder fact extraction trends and detect web scraping anomalies.

**Trend Analysis**:
- **Performance Trend**: TechCrunch parser 15% slower on recent articles (more ads/JS)
- **Error Trend**: Entity resolution failure rate increased to 8% (new founder names)
- **Capacity Forecast**: Bulk scraping of 5000 URLs requires 50 minutes
- **Anomaly Detection**: 12 web pages took >10 seconds (z-score: 3.2)

**Example**:
```typescript
// Get performance trend
const trend = await analyzer.getPerformanceTrend({
  parser_id: "techcrunch_web_parser",
  time_window: "30d",
  metric: "duration_p95_ms"
});

console.log(`Performance trend: ${trend.direction}`);
console.log(`Change: ${trend.percentage_change}%`);

// Detect anomalies in fact extraction
const anomalies = await analyzer.detectAnomalies({
  parser_id: "investment_fact_extractor",
  time_window: "24h",
  metric: "execution_time_ms",
  threshold_z_score: 2.5
});

if (anomalies.length > 10) {
  console.log("⚠️  Multiple fact extraction anomalies detected");
  console.log(`Total anomalies: ${anomalies.length}`);
  console.log(`Recommendation: Investigate recent articles with complex investment structures`);
}
```

**Performance Target**: <140ms for trend queries, <280ms for anomaly detection.

---

### 5. E-commerce

**Use Case**: Forecast product catalog import capacity and detect categorization anomalies.

**Trend Analysis**:
- **Performance Trend**: Product feed parser 30% slower during peak hours
- **Error Trend**: Categorization error rate spiked to 15% (baseline: 5%)
- **Capacity Forecast**: Black Friday requires 5x capacity
- **Anomaly Detection**: 20 products took >5 seconds (z-score: 3.8)

**Example**:
```typescript
// Forecast Black Friday capacity
const forecast = await analyzer.forecastCapacity({
  parser_id: "amazon_catalog_csv_parser",
  forecast_days: 7,
  metric: "executions_per_second",
  seasonal_adjustment: "black_friday"
});

console.log(`Current throughput: ${forecast.current_value} products/sec`);
console.log(`Black Friday forecast: ${forecast.forecasted_value} products/sec`);
console.log(`Capacity multiplier: ${forecast.capacity_multiplier}x`);

// Detect categorization anomalies
const anomalies = await analyzer.detectAnomalies({
  rule_id: "product_taxonomy_mapper",
  time_window: "1h",
  metric: "execution_time_ms",
  threshold_z_score: 3.0
});

console.log(`Categorization anomalies: ${anomalies.length}`);
```

**Performance Target**: <130ms for forecast queries, <260ms for anomaly detection.

---

### 6. SaaS

**Use Case**: Monitor webhook processing trends and detect delivery anomalies.

**Trend Analysis**:
- **Performance Trend**: Webhook processing 10% slower over 7 days
- **Error Trend**: Webhook delivery failure rate doubled to 4%
- **Capacity Forecast**: New integration requires 30% more capacity
- **Anomaly Detection**: 12 webhooks took >10 seconds (z-score: 4.0)

**Example**:
```typescript
// Get webhook processing trend
const trend = await analyzer.getPerformanceTrend({
  parser_id: "stripe_webhook_parser",
  time_window: "7d",
  metric: "duration_p95_ms"
});

if (trend.percentage_change > 10) {
  console.log("⚠️  Webhook processing slowing down");
  console.log(`Increase: ${trend.percentage_change}%`);
}

// Forecast capacity for new integration
const forecast = await analyzer.forecastCapacity({
  parser_id: "stripe_webhook_parser",
  forecast_days: 30,
  metric: "executions_per_second",
  growth_rate: 0.3 // 30% growth expected
});

console.log(`Current: ${forecast.current_value} webhooks/sec`);
console.log(`Forecast (30d): ${forecast.forecasted_value} webhooks/sec`);
```

**Performance Target**: <100ms for trend queries, <200ms for anomaly detection.

---

### 7. Insurance

**Use Case**: Forecast catastrophe claims processing capacity and detect adjudication anomalies.

**Trend Analysis**:
- **Performance Trend**: Claims processing 50% slower during CAT event
- **Error Trend**: Adjudication error rate increased to 12% (baseline: 3%)
- **Capacity Forecast**: Hurricane season requires 3x capacity
- **Anomaly Detection**: 25 claims took >10 minutes (z-score: 3.9)

**Example**:
```typescript
// Forecast hurricane season capacity
const forecast = await analyzer.forecastCapacity({
  parser_id: "acord_form_ocr_parser",
  forecast_days: 90,
  metric: "executions_per_hour",
  seasonal_adjustment: "hurricane_season"
});

console.log(`Current capacity: ${forecast.current_value} claims/hour`);
console.log(`Hurricane season forecast: ${forecast.forecasted_value} claims/hour`);
console.log(`Capacity multiplier: ${forecast.capacity_multiplier}x`);

// Detect adjudication anomalies
const anomalies = await analyzer.detectAnomalies({
  rule_id: "coverage_limit_validator",
  time_window: "24h",
  metric: "execution_time_ms",
  threshold_z_score: 2.5
});

if (anomalies.length > 20) {
  console.log("⚠️  Multiple adjudication anomalies during CAT event");
}
```

**Performance Target**: <150ms for forecast queries, <300ms for anomaly detection.

---

### Cross-Domain Benefits

**Predictive Insights**: All domains benefit from:
- Performance trend detection (identify degradation early)
- Capacity forecasting (plan for future growth)
- Anomaly detection (catch outliers automatically)
- Baseline comparison (adapt to changing workloads)

**Operational Excellence**: Supports:
- Proactive capacity planning
- Early anomaly detection
- Performance regression identification
- Cost optimization (right-size capacity)

---

## Core Concepts

### Trend Types

1. **Performance Trend**: Change in latency/throughput over time
2. **Error Trend**: Change in error rate over time
3. **Capacity Trend**: Change in processing volume over time

### Statistical Methods

**Z-Score (Anomaly Detection)**:
```
z = (x - μ) / σ
```
- x = observed value
- μ = mean
- σ = standard deviation
- Anomaly: |z| > 2.5

**Linear Regression (Forecasting)**:
```
y = mx + b
```
- y = forecasted value
- x = time
- m = slope (trend)
- b = intercept (baseline)

**Baseline Comparison**:
```
percentage_change = ((current - baseline) / baseline) * 100
```

### Time Windows

- **Current Window**: Recent data (e.g., last 1 hour)
- **Baseline Window**: Historical data (e.g., last 7 days)
- **Forecast Window**: Future period (e.g., next 30 days)

---

## Interface Definition

### TypeScript Interface

```typescript
interface TrendAnalyzer {
  /**
   * Get performance trend over time window
   *
   * @param query - Performance trend query parameters
   * @returns Performance trend analysis
   */
  getPerformanceTrend(query: PerformanceTrendQuery): Promise<PerformanceTrend>;

  /**
   * Get error rate trend over time window
   *
   * @param query - Error trend query parameters
   * @returns Error rate trend analysis
   */
  getErrorTrend(query: ErrorTrendQuery): Promise<ErrorTrend>;

  /**
   * Forecast future capacity needs
   *
   * @param query - Capacity forecast query parameters
   * @returns Capacity forecast with recommendations
   */
  forecastCapacity(query: CapacityForecastQuery): Promise<CapacityForecast>;

  /**
   * Detect performance anomalies
   *
   * @param query - Anomaly detection query parameters
   * @returns Array of detected anomalies
   */
  detectAnomalies(query: AnomalyDetectionQuery): Promise<Anomaly[]>;

  /**
   * Compare current performance vs baseline
   *
   * @param query - Baseline comparison query parameters
   * @returns Baseline comparison result
   */
  compareVsBaseline(query: BaselineComparisonQuery): Promise<BaselineComparison>;

  /**
   * Detect seasonal patterns
   *
   * @param query - Seasonal pattern query parameters
   * @returns Seasonal pattern analysis
   */
  detectSeasonalPatterns(
    query: SeasonalPatternQuery
  ): Promise<SeasonalPattern>;

  /**
   * Get trend summary for multiple metrics
   *
   * @param query - Trend summary query parameters
   * @returns Summary of trends across metrics
   */
  getTrendSummary(query: TrendSummaryQuery): Promise<TrendSummary>;
}
```

---

## Data Model

### PerformanceTrendQuery Type

```typescript
interface PerformanceTrendQuery {
  parser_id?: string;             // Parser identifier (optional)
  rule_id?: string;               // Rule identifier (optional)
  time_window: string;            // Time window (e.g., "30d")
  metric: string;                 // Metric name (e.g., "duration_p95_ms")
}
```

### PerformanceTrend Type

```typescript
interface PerformanceTrend {
  // Identity
  parser_id?: string;
  rule_id?: string;
  metric: string;

  // Trend
  direction: TrendDirection;      // increasing | decreasing | stable
  percentage_change: number;      // Percentage change vs baseline

  // Values
  current_value: number;
  baseline_value: number;

  // Statistical
  slope: number;                  // Linear regression slope
  r_squared: number;              // Goodness of fit (0-1)

  // Time Range
  time_window: string;
  start_time: string;
  end_time: string;
}
```

### ErrorTrendQuery Type

```typescript
interface ErrorTrendQuery {
  parser_id?: string;
  rule_id?: string;
  time_window: string;
  error_type?: string;            // Optional error type filter
}
```

### ErrorTrend Type

```typescript
interface ErrorTrend {
  // Identity
  parser_id?: string;
  rule_id?: string;
  error_type?: string;

  // Trend
  direction: TrendDirection;
  percentage_change: number;

  // Error Rates
  current_rate: number;           // 0.0 to 1.0
  baseline_rate: number;          // 0.0 to 1.0

  // Counts
  current_errors: number;
  baseline_errors: number;
  current_total: number;
  baseline_total: number;

  // Time Range
  time_window: string;
  start_time: string;
  end_time: string;
}
```

### CapacityForecastQuery Type

```typescript
interface CapacityForecastQuery {
  parser_id?: string;
  rule_id?: string;
  forecast_days: number;          // Number of days to forecast
  metric: string;                 // Metric to forecast
  growth_rate?: number;           // Optional growth rate override
  seasonal_adjustment?: string;   // Optional seasonal adjustment
  target_volume?: number;         // Optional target volume
}
```

### CapacityForecast Type

```typescript
interface CapacityForecast {
  // Identity
  parser_id?: string;
  rule_id?: string;
  metric: string;

  // Current
  current_value: number;

  // Forecast
  forecasted_value: number;
  forecast_date: string;          // Date of forecast
  confidence_interval_low: number;
  confidence_interval_high: number;

  // Capacity Planning
  percentage_increase: number;
  capacity_multiplier: number;
  estimated_completion_hours?: number;

  // Recommendation
  recommendation: string;
}
```

### AnomalyDetectionQuery Type

```typescript
interface AnomalyDetectionQuery {
  parser_id?: string;
  rule_id?: string;
  time_window: string;
  metric: string;
  threshold_z_score?: number;     // Default: 2.5
}
```

### Anomaly Type

```typescript
interface Anomaly {
  // Identity
  metric_id: string;              // Parser or rule metric ID
  timestamp: string;

  // Anomaly Details
  metric_value: number;
  expected_value: number;
  z_score: number;
  severity: AnomalySeverity;      // low | medium | high | critical

  // Context
  upload_id?: string;
  document_id?: string;
  metadata?: Record<string, any>;
}
```

### BaselineComparisonQuery Type

```typescript
interface BaselineComparisonQuery {
  parser_id?: string;
  rule_id?: string;
  current_window: string;         // Recent window (e.g., "1h")
  baseline_window: string;        // Historical window (e.g., "7d")
  metric: string;
}
```

### BaselineComparison Type

```typescript
interface BaselineComparison {
  // Identity
  parser_id?: string;
  rule_id?: string;
  metric: string;

  // Values
  current_value: number;
  baseline_value: number;

  // Comparison
  percentage_difference: number;
  is_degraded: boolean;           // True if performance worse than baseline

  // Time Windows
  current_window: string;
  baseline_window: string;
}
```

### SeasonalPatternQuery Type

```typescript
interface SeasonalPatternQuery {
  parser_id?: string;
  rule_id?: string;
  time_window: string;            // Must be >= 7 days for pattern detection
  metric: string;
}
```

### SeasonalPattern Type

```typescript
interface SeasonalPattern {
  // Identity
  parser_id?: string;
  rule_id?: string;
  metric: string;

  // Pattern Detection
  daily_pattern_detected: boolean;
  weekly_pattern_detected: boolean;

  // Peak Times
  peak_hours: number[];           // Hours with highest values
  off_peak_hours: number[];       // Hours with lowest values
  peak_days: string[];            // Days with highest values (e.g., ["monday", "tuesday"])

  // Statistics
  peak_value: number;
  off_peak_value: number;
  pattern_strength: number;       // 0-1 (1 = strong pattern)
}
```

---

## Core Functionality

### 1. getPerformanceTrend()

Analyze performance trend over time window.

#### Signature

```typescript
getPerformanceTrend(query: PerformanceTrendQuery): Promise<PerformanceTrend>
```

#### Behavior

1. **Parse Time Window**:
   - Convert to absolute timestamps

2. **Fetch Historical Data**:
   - Query metrics for time window

3. **Calculate Baseline**:
   - Baseline = average of first 50% of data

4. **Calculate Current**:
   - Current = average of last 10% of data

5. **Linear Regression**:
   - Fit line to data points
   - Calculate slope and R²

6. **Determine Trend**:
   - Increasing: slope > 0 and percentage_change > 5%
   - Decreasing: slope < 0 and percentage_change < -5%
   - Stable: |percentage_change| <= 5%

7. **Return**:
   - Return trend analysis

#### SQL Query

```sql
WITH time_series AS (
  SELECT
    created_at,
    duration_ms,
    ROW_NUMBER() OVER (ORDER BY created_at) AS row_num,
    COUNT(*) OVER () AS total_rows
  FROM parser_execution_metrics
  WHERE parser_id = $1
    AND created_at >= $2
    AND created_at <= $3
),
baseline AS (
  SELECT AVG(duration_ms) AS baseline_value
  FROM time_series
  WHERE row_num <= total_rows * 0.5
),
current AS (
  SELECT AVG(duration_ms) AS current_value
  FROM time_series
  WHERE row_num > total_rows * 0.9
),
regression AS (
  SELECT
    REGR_SLOPE(duration_ms, EXTRACT(EPOCH FROM created_at)) AS slope,
    REGR_R2(duration_ms, EXTRACT(EPOCH FROM created_at)) AS r_squared
  FROM time_series
)
SELECT
  b.baseline_value,
  c.current_value,
  r.slope,
  r.r_squared,
  ((c.current_value - b.baseline_value) / b.baseline_value * 100) AS percentage_change
FROM baseline b, current c, regression r;
```

#### Example

```typescript
const trend = await analyzer.getPerformanceTrend({
  parser_id: "chase_pdf_parser_v2",
  time_window: "30d",
  metric: "duration_p95_ms"
});

console.log(`Trend direction: ${trend.direction}`);
console.log(`Percentage change: ${trend.percentage_change.toFixed(2)}%`);
console.log(`Current p95: ${trend.current_value}ms`);
console.log(`Baseline p95: ${trend.baseline_value}ms`);
console.log(`R²: ${trend.r_squared.toFixed(3)}`);

if (trend.direction === "increasing" && trend.percentage_change > 20) {
  console.log("⚠️  Significant performance degradation detected");
}
```

#### Performance

- **Target Latency**: <150ms p95

---

### 2. getErrorTrend()

Analyze error rate trend over time window.

#### Signature

```typescript
getErrorTrend(query: ErrorTrendQuery): Promise<ErrorTrend>
```

#### Behavior

Similar to getPerformanceTrend(), but for error rates.

#### Example

```typescript
const errorTrend = await analyzer.getErrorTrend({
  parser_id: "chase_pdf_parser_v2",
  time_window: "7d",
  error_type: "OCR_FAILURE"
});

console.log(`Error trend: ${errorTrend.direction}`);
console.log(`Current error rate: ${(errorTrend.current_rate * 100).toFixed(2)}%`);
console.log(`Baseline error rate: ${(errorTrend.baseline_rate * 100).toFixed(2)}%`);

if (errorTrend.percentage_change > 50) {
  console.log("⚠️  Error rate doubled");
}
```

#### Performance

- **Target Latency**: <120ms p95

---

### 3. forecastCapacity()

Forecast future capacity needs using linear regression.

#### Signature

```typescript
forecastCapacity(query: CapacityForecastQuery): Promise<CapacityForecast>
```

#### Behavior

1. **Fetch Historical Data**:
   - Query metrics for past 30 days

2. **Linear Regression**:
   - Fit line to data: y = mx + b
   - Extrapolate to forecast date

3. **Confidence Interval**:
   - Calculate 95% confidence interval

4. **Capacity Planning**:
   - Calculate percentage increase needed
   - Generate recommendation

5. **Return**:
   - Return forecast

#### Example

```typescript
const forecast = await analyzer.forecastCapacity({
  parser_id: "chase_pdf_parser_v2",
  forecast_days: 30,
  metric: "executions_per_hour"
});

console.log(`Current: ${forecast.current_value} exec/hour`);
console.log(`Forecast (30d): ${forecast.forecasted_value} exec/hour`);
console.log(`Increase needed: ${forecast.percentage_increase}%`);
console.log(`Recommendation: ${forecast.recommendation}`);
```

#### Performance

- **Target Latency**: <200ms p95

---

### 4. detectAnomalies()

Detect anomalies using z-score analysis.

#### Signature

```typescript
detectAnomalies(query: AnomalyDetectionQuery): Promise<Anomaly[]>
```

#### Behavior

1. **Fetch Historical Data**:
   - Query metrics for time window

2. **Calculate Statistics**:
   - Mean (μ) = average
   - Standard deviation (σ)

3. **Calculate Z-Scores**:
   - For each data point: z = (x - μ) / σ

4. **Detect Anomalies**:
   - Anomaly: |z| > threshold (default: 2.5)

5. **Classify Severity**:
   - Low: 2.5 < |z| < 3.0
   - Medium: 3.0 <= |z| < 3.5
   - High: 3.5 <= |z| < 4.0
   - Critical: |z| >= 4.0

6. **Return**:
   - Return array of anomalies

#### SQL Query

```sql
WITH stats AS (
  SELECT
    AVG(duration_ms) AS mean,
    STDDEV(duration_ms) AS stddev
  FROM parser_execution_metrics
  WHERE parser_id = $1
    AND created_at >= $2
    AND created_at <= $3
)
SELECT
  metric_id,
  created_at AS timestamp,
  duration_ms AS metric_value,
  (SELECT mean FROM stats) AS expected_value,
  (duration_ms - (SELECT mean FROM stats)) / (SELECT stddev FROM stats) AS z_score,
  upload_id,
  metadata
FROM parser_execution_metrics
WHERE parser_id = $1
  AND created_at >= $2
  AND created_at <= $3
  AND ABS((duration_ms - (SELECT mean FROM stats)) / (SELECT stddev FROM stats)) > $4
ORDER BY z_score DESC;
```

#### Example

```typescript
const anomalies = await analyzer.detectAnomalies({
  parser_id: "chase_pdf_parser_v2",
  time_window: "24h",
  metric: "duration_ms",
  threshold_z_score: 2.5
});

console.log(`Anomalies detected: ${anomalies.length}`);

anomalies.forEach(anomaly => {
  console.log(`Anomaly (${anomaly.severity}):`);
  console.log(`  Value: ${anomaly.metric_value}ms`);
  console.log(`  Expected: ${anomaly.expected_value}ms`);
  console.log(`  Z-score: ${anomaly.z_score.toFixed(2)}`);
  console.log(`  Upload: ${anomaly.upload_id}`);
});
```

#### Performance

- **Target Latency**: <300ms p95

---

### 5. compareVsBaseline()

Compare current performance vs historical baseline.

#### Signature

```typescript
compareVsBaseline(query: BaselineComparisonQuery): Promise<BaselineComparison>
```

#### Behavior

1. **Fetch Current Data**:
   - Query metrics for current window (e.g., last 1 hour)

2. **Fetch Baseline Data**:
   - Query metrics for baseline window (e.g., last 7 days)

3. **Calculate Averages**:
   - Current average
   - Baseline average

4. **Compare**:
   - Percentage difference = ((current - baseline) / baseline) * 100
   - is_degraded = current > baseline * 1.2 (20% worse)

5. **Return**:
   - Return comparison

#### Example

```typescript
const comparison = await analyzer.compareVsBaseline({
  parser_id: "chase_pdf_parser_v2",
  current_window: "1h",
  baseline_window: "7d",
  metric: "duration_p95_ms"
});

console.log(`Current p95: ${comparison.current_value}ms`);
console.log(`Baseline p95: ${comparison.baseline_value}ms`);
console.log(`Difference: ${comparison.percentage_difference}%`);

if (comparison.is_degraded) {
  console.log("⚠️  Performance degraded vs baseline");
}
```

#### Performance

- **Target Latency**: <100ms p95

---

### 6. detectSeasonalPatterns()

Detect daily/weekly seasonal patterns.

#### Signature

```typescript
detectSeasonalPatterns(
  query: SeasonalPatternQuery
): Promise<SeasonalPattern>
```

#### Behavior

1. **Fetch Historical Data**:
   - Query metrics for time window (>= 7 days)

2. **Group By Hour**:
   - Calculate average value per hour of day

3. **Detect Daily Pattern**:
   - Check if variance between hours is significant (CV > 0.2)

4. **Group By Day of Week**:
   - Calculate average value per day

5. **Detect Weekly Pattern**:
   - Check if variance between days is significant

6. **Identify Peaks**:
   - Find peak hours/days

7. **Return**:
   - Return pattern analysis

#### Example

```typescript
const patterns = await analyzer.detectSeasonalPatterns({
  parser_id: "chase_pdf_parser_v2",
  time_window: "30d",
  metric: "executions_per_hour"
});

console.log(`Daily pattern: ${patterns.daily_pattern_detected}`);
console.log(`Peak hours: ${patterns.peak_hours.join(", ")}`);
console.log(`Off-peak hours: ${patterns.off_peak_hours.join(", ")}`);
console.log(`Pattern strength: ${patterns.pattern_strength.toFixed(2)}`);

if (patterns.daily_pattern_detected) {
  console.log("💡 Scale workers during peak hours for cost optimization");
}
```

#### Performance

- **Target Latency**: <250ms p95

---

## Statistical Algorithms

### Z-Score Calculation

```typescript
function calculateZScore(value: number, mean: number, stddev: number): number {
  if (stddev === 0) return 0;
  return (value - mean) / stddev;
}

// Example
const mean = 1000; // Average duration
const stddev = 200; // Standard deviation
const value = 1500; // Observed value

const zScore = calculateZScore(value, mean, stddev);
console.log(`Z-score: ${zScore}`); // 2.5

if (Math.abs(zScore) > 2.5) {
  console.log("Anomaly detected!");
}
```

### Linear Regression

```typescript
function linearRegression(
  data: Array<{ x: number; y: number }>
): { slope: number; intercept: number; r_squared: number } {
  const n = data.length;
  const sumX = data.reduce((sum, point) => sum + point.x, 0);
  const sumY = data.reduce((sum, point) => sum + point.y, 0);
  const sumXY = data.reduce((sum, point) => sum + point.x * point.y, 0);
  const sumXX = data.reduce((sum, point) => sum + point.x * point.x, 0);
  const sumYY = data.reduce((sum, point) => sum + point.y * point.y, 0);

  const slope = (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX);
  const intercept = (sumY - slope * sumX) / n;

  // Calculate R²
  const meanY = sumY / n;
  const ssTotal = data.reduce((sum, point) => sum + Math.pow(point.y - meanY, 2), 0);
  const ssResidual = data.reduce(
    (sum, point) => sum + Math.pow(point.y - (slope * point.x + intercept), 2),
    0
  );
  const r_squared = 1 - ssResidual / ssTotal;

  return { slope, intercept, r_squared };
}
```

### Baseline Comparison

```typescript
function compareToBaseline(
  current: number,
  baseline: number
): { percentage_difference: number; is_degraded: boolean } {
  const percentage_difference = ((current - baseline) / baseline) * 100;
  const is_degraded = current > baseline * 1.2; // 20% worse

  return { percentage_difference, is_degraded };
}
```

---

## Anomaly Detection

### Anomaly Severity Classification

```typescript
function classifyAnomalySeverity(zScore: number): AnomalySeverity {
  const absZScore = Math.abs(zScore);

  if (absZScore >= 4.0) return "critical";
  if (absZScore >= 3.5) return "high";
  if (absZScore >= 3.0) return "medium";
  return "low";
}
```

### Anomaly Filtering

```typescript
// Filter anomalies by severity
const criticalAnomalies = anomalies.filter(a => a.severity === "critical");

// Group by upload_id
const anomaliesByUpload = anomalies.reduce((acc, anomaly) => {
  const key = anomaly.upload_id || "unknown";
  if (!acc[key]) acc[key] = [];
  acc[key].push(anomaly);
  return acc;
}, {} as Record<string, Anomaly[]>);
```

---

## Storage & Indexing

### Indexes for Fast Queries

```sql
-- Composite index for time-series queries
CREATE INDEX idx_parser_metrics_time_series ON parser_execution_metrics(parser_id, created_at DESC, duration_ms);

-- Index for anomaly detection queries
CREATE INDEX idx_parser_metrics_anomaly ON parser_execution_metrics(parser_id, created_at DESC) INCLUDE (duration_ms);
```

---

## Edge Cases

### Edge Case 1: Insufficient Data

**Scenario**: Not enough data points for trend analysis (<10 data points).

**Detection**:
```typescript
if (dataPoints.length < 10) {
  throw new ValidationError("Insufficient data for trend analysis (minimum: 10 points)");
}
```

**Resolution**:
- Return error or null
- Suggest longer time window

---

### Edge Case 2: Zero Standard Deviation

**Scenario**: All data points have same value, stddev = 0.

**Detection**:
```typescript
if (stddev === 0) {
  return []; // No anomalies (all values identical)
}
```

**Resolution**:
- Return no anomalies
- Skip z-score calculation

---

### Edge Case 3: Negative Forecasts

**Scenario**: Linear regression predicts negative value.

**Resolution**:
```typescript
const forecasted_value = Math.max(0, slope * future_time + intercept);
```

---

### Edge Case 4: Very High Z-Scores

**Scenario**: Z-score > 10 (extreme outlier).

**Resolution**:
- Classify as critical
- Investigate data quality issue

---

### Edge Case 5: No Seasonal Pattern

**Scenario**: Data is completely random, no pattern detected.

**Resolution**:
```typescript
return {
  daily_pattern_detected: false,
  weekly_pattern_detected: false,
  pattern_strength: 0,
  peak_hours: [],
  off_peak_hours: []
};
```

---

## Performance Characteristics

### Latency Targets

| Operation | Target Latency (p95) | Notes |
|-----------|---------------------|-------|
| `getPerformanceTrend()` | < 150ms | Time-series query + regression |
| `getErrorTrend()` | < 120ms | Simpler than performance trend |
| `forecastCapacity()` | < 200ms | Regression + confidence interval |
| `detectAnomalies()` | < 300ms | Full table scan + z-score calc |
| `compareVsBaseline()` | < 100ms | Two aggregation queries |
| `detectSeasonalPatterns()` | < 250ms | Multiple groupings |
| `getTrendSummary()` | < 400ms | Multiple trend queries |

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';

export class PostgresTrendAnalyzer implements TrendAnalyzer {
  private pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  async getPerformanceTrend(
    query: PerformanceTrendQuery
  ): Promise<PerformanceTrend> {
    const { start_time, end_time } = this.parseTimeWindow(query.time_window);

    const sql = `
      WITH time_series AS (
        SELECT
          created_at,
          duration_ms,
          ROW_NUMBER() OVER (ORDER BY created_at) AS row_num,
          COUNT(*) OVER () AS total_rows
        FROM parser_execution_metrics
        WHERE parser_id = $1
          AND created_at >= $2
          AND created_at <= $3
      ),
      baseline AS (
        SELECT AVG(duration_ms) AS baseline_value
        FROM time_series
        WHERE row_num <= total_rows * 0.5
      ),
      current AS (
        SELECT AVG(duration_ms) AS current_value
        FROM time_series
        WHERE row_num > total_rows * 0.9
      ),
      regression AS (
        SELECT
          REGR_SLOPE(duration_ms, EXTRACT(EPOCH FROM created_at)) AS slope,
          REGR_R2(duration_ms, EXTRACT(EPOCH FROM created_at)) AS r_squared
        FROM time_series
      )
      SELECT
        b.baseline_value,
        c.current_value,
        r.slope,
        r.r_squared,
        ((c.current_value - b.baseline_value) / NULLIF(b.baseline_value, 0) * 100) AS percentage_change
      FROM baseline b, current c, regression r
    `;

    const result = await this.pool.query(sql, [
      query.parser_id,
      start_time,
      end_time
    ]);

    if (result.rows.length === 0) {
      throw new Error("Insufficient data for trend analysis");
    }

    const row = result.rows[0];
    const percentage_change = parseFloat(row.percentage_change);

    // Determine trend direction
    let direction: TrendDirection;
    if (percentage_change > 5) {
      direction = "increasing";
    } else if (percentage_change < -5) {
      direction = "decreasing";
    } else {
      direction = "stable";
    }

    return {
      parser_id: query.parser_id,
      metric: query.metric,
      direction,
      percentage_change,
      current_value: parseFloat(row.current_value),
      baseline_value: parseFloat(row.baseline_value),
      slope: parseFloat(row.slope),
      r_squared: parseFloat(row.r_squared),
      time_window: query.time_window,
      start_time: start_time.toISOString(),
      end_time: end_time.toISOString()
    };
  }

  async detectAnomalies(
    query: AnomalyDetectionQuery
  ): Promise<Anomaly[]> {
    const { start_time, end_time } = this.parseTimeWindow(query.time_window);
    const threshold_z_score = query.threshold_z_score || 2.5;

    const sql = `
      WITH stats AS (
        SELECT
          AVG(duration_ms) AS mean,
          STDDEV(duration_ms) AS stddev
        FROM parser_execution_metrics
        WHERE parser_id = $1
          AND created_at >= $2
          AND created_at <= $3
      )
      SELECT
        metric_id,
        created_at AS timestamp,
        duration_ms AS metric_value,
        (SELECT mean FROM stats) AS expected_value,
        (duration_ms - (SELECT mean FROM stats)) / NULLIF((SELECT stddev FROM stats), 0) AS z_score,
        upload_id,
        metadata
      FROM parser_execution_metrics
      WHERE parser_id = $1
        AND created_at >= $2
        AND created_at <= $3
        AND (SELECT stddev FROM stats) > 0
        AND ABS((duration_ms - (SELECT mean FROM stats)) / (SELECT stddev FROM stats)) > $4
      ORDER BY ABS(z_score) DESC
    `;

    const result = await this.pool.query(sql, [
      query.parser_id,
      start_time,
      end_time,
      threshold_z_score
    ]);

    return result.rows.map(row => ({
      metric_id: row.metric_id,
      timestamp: row.timestamp.toISOString(),
      metric_value: parseFloat(row.metric_value),
      expected_value: parseFloat(row.expected_value),
      z_score: parseFloat(row.z_score),
      severity: this.classifyAnomalySeverity(parseFloat(row.z_score)),
      upload_id: row.upload_id,
      metadata: row.metadata
    }));
  }

  private classifyAnomalySeverity(zScore: number): AnomalySeverity {
    const absZScore = Math.abs(zScore);

    if (absZScore >= 4.0) return "critical";
    if (absZScore >= 3.5) return "high";
    if (absZScore >= 3.0) return "medium";
    return "low";
  }

  private parseTimeWindow(timeWindow: string): { start_time: Date; end_time: Date } {
    // Implementation omitted for brevity (same as PerformanceAnalyzer)
  }
}
```

---

## Security Considerations

### 1. Query Injection Prevention
### 2. Rate Limiting
### 3. Access Control

(Same as PerformanceAnalyzer)

---

## Integration Patterns

### Pattern 1: Automated Alerting

```typescript
// Check for performance degradation every hour
setInterval(async () => {
  const trend = await analyzer.getPerformanceTrend({
    parser_id: "chase_pdf_parser_v2",
    time_window: "7d",
    metric: "duration_p95_ms"
  });

  if (trend.direction === "increasing" && trend.percentage_change > 20) {
    await alerting.sendAlert({
      type: "PERFORMANCE_DEGRADATION",
      parser_id: "chase_pdf_parser_v2",
      percentage_change: trend.percentage_change,
      current_value: trend.current_value,
      baseline_value: trend.baseline_value
    });
  }
}, 3600000);
```

### Pattern 2: Capacity Planning Dashboard

```typescript
// Generate capacity forecast dashboard
async function generateCapacityDashboard() {
  const parsers = ["chase_parser", "wells_fargo_parser", "bofa_parser"];

  const forecasts = await Promise.all(
    parsers.map(parser_id =>
      analyzer.forecastCapacity({
        parser_id,
        forecast_days: 30,
        metric: "executions_per_hour"
      })
    )
  );

  return {
    forecasts,
    totalCapacityIncrease: forecasts.reduce(
      (sum, f) => sum + f.percentage_increase,
      0
    ) / forecasts.length
  };
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section)

---

## Testing Strategy

### Unit Tests

```typescript
describe('TrendAnalyzer', () => {
  it('should detect increasing trend', async () => {
    const trend = await analyzer.getPerformanceTrend({
      parser_id: 'test_parser',
      time_window: '7d',
      metric: 'duration_p95_ms'
    });

    expect(trend.direction).toBe('increasing');
    expect(trend.percentage_change).toBeGreaterThan(0);
  });

  it('should detect anomalies', async () => {
    const anomalies = await analyzer.detectAnomalies({
      parser_id: 'test_parser',
      time_window: '24h',
      metric: 'duration_ms',
      threshold_z_score: 2.5
    });

    anomalies.forEach(anomaly => {
      expect(Math.abs(anomaly.z_score)).toBeGreaterThan(2.5);
    });
  });
});
```

---

## Configuration

```typescript
interface TrendAnalyzerConfig {
  defaultZScoreThreshold: number;         // Default: 2.5
  minDataPointsForTrend: number;          // Default: 10
  forecastConfidenceLevel: number;        // Default: 0.95
  seasonalPatternMinDays: number;         // Default: 7
}
```

---

## Observability

```typescript
interface TrendAnalyzerMetrics {
  trend_queries_total: Counter;
  anomalies_detected_total: Counter;
  query_duration_ms: Histogram;
}
```

---

## Future Enhancements

### Phase 1: Machine Learning (6 months)
- ARIMA time-series forecasting
- Prophet for seasonal forecasting
- Isolation Forest for anomaly detection

### Phase 2: Real-Time Streaming (12 months)
- Stream processing for real-time anomaly detection
- Online learning for adaptive baselines

### Phase 3: Automated Remediation (18 months)
- Auto-scale workers based on forecasts
- Auto-rollback deployments on performance degradation

---

**End of TrendAnalyzer OL Primitive Specification**
