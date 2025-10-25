# TimelineReconstructor OL Primitive

**Domain:** Provenance Ledger (Vertical 5.1)
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
8. [Timeline Reconstruction](#timeline-reconstruction)
9. [Visualization Output](#visualization-output)
10. [Snapshot Interpolation](#snapshot-interpolation)
11. [Edge Cases](#edge-cases)
12. [Performance Characteristics](#performance-characteristics)
13. [Implementation Notes](#implementation-notes)
14. [Visualization Integration](#visualization-integration)
15. [Integration Patterns](#integration-patterns)
16. [Multi-Domain Examples](#multi-domain-examples)
17. [Testing Strategy](#testing-strategy)
18. [Migration Guide](#migration-guide)
19. [Related Primitives](#related-primitives)
20. [References](#references)

---

## Overview

The **TimelineReconstructor** primitive reconstructs complete entity histories from bitemporal events, transforming raw provenance data into rich, queryable timelines suitable for visualization and analysis.

### Key Capabilities

- **Complete History Reconstruction**: Build full timeline from provenance events
- **Bitemporal Timeline Support**: Track both transaction time and valid time dimensions
- **Snapshot Generation**: Create point-in-time snapshots at any moment
- **Value Interpolation**: Fill gaps in temporal data
- **Visualization-Ready Output**: Generate D3.js-compatible data structures
- **High Performance**: <200ms timeline generation for typical entities (p95)

### Design Philosophy

The TimelineReconstructor follows four core principles:

1. **Completeness**: Every event captured in temporal context
2. **Accuracy**: Precise temporal ordering and relationships
3. **Usability**: Output optimized for consumption by visualization libraries
4. **Performance**: Efficient algorithms for timeline construction

---

## Purpose & Scope

### Problem Statement

Raw bitemporal events in the provenance ledger are difficult to:
- **Visualize**: Need transformation for timeline UI components
- **Analyze**: Temporal relationships not immediately apparent
- **Query**: Point-in-time snapshots require complex event replay
- **Understand**: Retroactive corrections obscure timeline clarity

**Traditional Approaches**:
- L Manual event iteration and state building
- L No interpolation for sparse timelines
- L Inefficient snapshot generation
- L No built-in visualization support

**TimelineReconstructor Solution**:
-  Automated timeline reconstruction from events
-  Intelligent interpolation for smooth timelines
-  Fast snapshot generation at any point
-  D3.js and React-compatible output formats

### Solution

TimelineReconstructor implements comprehensive timeline reconstruction with:

1. **Event Ordering**: Chronological timeline with both time dimensions
2. **Snapshot Generation**: Instant state at any temporal coordinate
3. **Interpolation**: Fill temporal gaps intelligently
4. **Visualization Data**: Format for D3.js, Chart.js, timeline libraries
5. **Retroactive Highlighting**: Mark corrections visually

---

## Multi-Domain Applicability

TimelineReconstructor is universally applicable for temporal visualization. 7+ domains:

### 1. Finance

**Use Case**: Visualize transaction history with corrections.

**Timeline Output**:
- Transaction amount changes over time
- Category classification timeline
- Merchant normalization history
- Retroactive corrections highlighted

**Example**:
```typescript
// Reconstruct transaction timeline
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001",
  include_retroactive: true
});

// Visualize in UI
<TimelineChart data={timeline.events} />

// Timeline shows:
// - Jan 15: Created with amount $45.00
// - Jan 20: Amount corrected to $47.00 (retroactive)
// - Jan 22: Category changed from "Uncategorized" to "Shopping"
```

### 2. Healthcare

**Use Case**: Patient diagnosis evolution timeline.

**Timeline Output**:
- Initial diagnosis on admission
- Updated diagnoses from test results
- Treatment plan changes
- Retroactive corrections from chart reviews

**Example**:
```typescript
// Reconstruct patient encounter timeline
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "encounter_12345",
  field_names: ["diagnosis_code", "treatment_plan"]
});

// Generate visualization data
const vizData = await reconstructor.buildVisualization(timeline, {
  format: 'd3_timeline',
  highlight_retroactive: true
});

// Render in medical records UI
<DiagnosisTimeline data={vizData} />
```

### 3. Legal

**Use Case**: Case evidence timeline for discovery.

**Timeline Output**:
- Evidence collection dates
- Document filing timeline
- Case status transitions
- Attorney assignment changes

**Example**:
```typescript
// Reconstruct case timeline
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "case_12345",
  entity_type: "legal_case",
  start_time: "2024-01-01T00:00:00Z",
  end_time: "2025-01-01T00:00:00Z"
});

// Export for legal review
const report = await reconstructor.export(timeline, {
  format: 'pdf_timeline',
  include_metadata: true
});
```

### 4. Research

**Use Case**: Citation metadata evolution timeline.

**Timeline Output**:
- Initial extraction from PDF
- Author name corrections
- DOI assignment
- Publication date updates

**Example**:
```typescript
// Reconstruct citation history
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "cite_001",
  field_names: ["authors", "title", "year"]
});

// Highlight data quality improvements
const improvements = timeline.events.filter(e =>
  e.event_type === 'corrected' && e.is_retroactive
);

console.log(`${improvements.length} data quality corrections`);
```

### 5. E-commerce

**Use Case**: Product price history visualization.

**Timeline Output**:
- Price changes over time
- Promotional periods
- Inventory adjustments
- Category reassignments

**Example**:
```typescript
// Reconstruct product price timeline
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "prod_001",
  field_name: "price",
  start_time: "2024-01-01T00:00:00Z",
  end_time: "2025-01-01T00:00:00Z"
});

// Generate price chart data
const chartData = await reconstructor.buildVisualization(timeline, {
  format: 'chart_js',
  interpolate: true, // Smooth price curve
  interval: '1day'
});

<LineChart data={chartData} />
```

### 6. SaaS

**Use Case**: Subscription plan timeline for customer view.

**Timeline Output**:
- Plan upgrades/downgrades
- Feature flag changes
- Billing adjustments
- Trial conversions

**Example**:
```typescript
// Reconstruct subscription timeline
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "sub_001",
  field_names: ["plan", "status", "mrr"]
});

// Customer-facing timeline
<SubscriptionHistory timeline={timeline} />
```

### 7. Insurance

**Use Case**: Policy premium timeline with claim impact.

**Timeline Output**:
- Initial premium quote
- Premium adjustments
- Claim filings (events)
- Coverage changes

**Example**:
```typescript
// Reconstruct policy timeline
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "policy_789",
  field_name: "premium",
  include_related_events: ["claim_filed", "coverage_change"]
});

// Visualize premium evolution
<PremiumTimeline
  data={timeline}
  highlightClaims={true}
/>
```

### Cross-Domain Benefits

All domains benefit from:
- **Visual Understanding**: See how data evolved over time
- **Audit Support**: Complete timeline for compliance
- **Error Detection**: Retroactive corrections visible
- **User Confidence**: Transparency into data changes

---

## Core Concepts

### Timeline Structure

A timeline is an ordered sequence of events across two time dimensions:

```typescript
interface Timeline {
  entity_id: string;
  entity_type: string;
  field_names: string[];

  // Temporal bounds
  start_time: string;
  end_time: string;

  // Events sorted by transaction_time
  events: TimelineEvent[];

  // Snapshots at key moments
  snapshots: TimelineSnapshot[];

  // Metadata
  total_events: number;
  retroactive_count: number;
  field_count: number;
}
```

### Timeline Event

Each event represents a state transition:

```typescript
interface TimelineEvent {
  sequence_number: number;
  timestamp: string;              // transaction_time (when recorded)
  effective_time: string;         // valid_time (when effective)

  field_name: string;
  old_value: any;
  new_value: any;

  event_type: string;
  user_id: string;
  reason?: string;

  is_retroactive: boolean;        // timestamp > effective_time
  time_lag_ms?: number;           // timestamp - effective_time
}
```

### Timeline Snapshot

Point-in-time state of entity:

```typescript
interface TimelineSnapshot {
  timestamp: string;              // When this snapshot is valid
  state: Record<string, any>;     // Complete entity state at this time
  event_count: number;            // Events applied to reach this state
}
```

---

## Interface Definition

### TypeScript Interface

```typescript
interface TimelineReconstructor {
  /**
   * Reconstruct complete timeline for entity
   *
   * @param filters - Entity and temporal filters
   * @returns Complete timeline with events and snapshots
   */
  reconstructTimeline(
    filters: TimelineReconstructionFilters
  ): Promise<Timeline>;

  /**
   * Get snapshot at specific point in time
   *
   * @param entityId - Entity to query
   * @param timestamp - Point in time
   * @param timeType - Transaction or valid time dimension
   * @returns Entity state at that moment
   */
  getSnapshot(
    entityId: string,
    timestamp: string,
    timeType: 'transaction' | 'valid'
  ): Promise<TimelineSnapshot>;

  /**
   * Interpolate value at specific time
   *
   * @param entityId - Entity to query
   * @param fieldName - Field to interpolate
   * @param timestamp - Time to interpolate at
   * @returns Interpolated value
   */
  interpolateValue(
    entityId: string,
    fieldName: string,
    timestamp: string
  ): Promise<any>;

  /**
   * Build visualization-ready data structure
   *
   * @param timeline - Input timeline
   * @param options - Visualization options
   * @returns Formatted data for visualization library
   */
  buildVisualization(
    timeline: Timeline,
    options: VisualizationOptions
  ): Promise<VisualizationData>;

  /**
   * Export timeline in various formats
   *
   * @param timeline - Timeline to export
   * @param options - Export options
   * @returns Serialized timeline data
   */
  export(
    timeline: Timeline,
    options: ExportOptions
  ): Promise<string>;

  /**
   * Get timeline for field across multiple entities
   *
   * @param filters - Multi-entity filters
   * @returns Aggregated timeline
   */
  reconstructAggregateTimeline(
    filters: AggregateTimelineFilters
  ): Promise<AggregateTimeline>;
}
```

---

## Data Model

### TimelineReconstructionFilters Type

```typescript
interface TimelineReconstructionFilters {
  // Entity Selection
  entity_id: string;
  entity_type?: string;

  // Field Selection
  field_name?: string;             // Single field
  field_names?: string[];          // Multiple fields

  // Temporal Bounds
  start_time?: string;             // Default: first event
  end_time?: string;               // Default: now
  time_dimension?: 'transaction' | 'valid'; // Default: 'transaction'

  // Options
  include_retroactive?: boolean;   // Default: true
  include_snapshots?: boolean;     // Default: true
  snapshot_interval?: string;      // e.g., '1day', '1hour'
  interpolate?: boolean;           // Default: false
}
```

### VisualizationOptions Type

```typescript
interface VisualizationOptions {
  // Output Format
  format: 'd3_timeline' | 'chart_js' | 'custom';

  // Visual Settings
  highlight_retroactive?: boolean;
  color_scheme?: 'default' | 'categorical' | 'sequential';

  // Data Processing
  interpolate?: boolean;
  interval?: string;               // e.g., '1day', '1hour'

  // Filtering
  event_types?: string[];
  field_names?: string[];
}
```

### ExportOptions Type

```typescript
interface ExportOptions {
  format: 'json' | 'csv' | 'pdf_timeline' | 'excel';
  include_metadata?: boolean;
  include_snapshots?: boolean;
  pretty_print?: boolean;
}
```

### VisualizationData Type

```typescript
interface VisualizationData {
  format: string;

  // D3.js timeline format
  lanes?: TimelineLane[];

  // Chart.js format
  datasets?: ChartDataset[];
  labels?: string[];

  // Custom format
  data?: any;

  // Metadata
  bounds: { start: string; end: string };
  event_count: number;
  retroactive_count: number;
}

interface TimelineLane {
  id: string;
  label: string;
  events: {
    id: string;
    start: string;
    end?: string;
    label: string;
    className?: string;
  }[];
}

interface ChartDataset {
  label: string;
  data: { x: string; y: any }[];
  borderColor?: string;
  backgroundColor?: string;
}
```

---

## Core Functionality

### 1. reconstructTimeline()

Reconstruct complete timeline for entity from provenance events.

#### Signature

```typescript
reconstructTimeline(
  filters: TimelineReconstructionFilters
): Promise<Timeline>
```

#### Behavior

1. **Fetch Events**: Query provenance ledger for matching events
2. **Sort Events**: Order by transaction_time ASC
3. **Build Timeline**: Create Timeline object with events
4. **Generate Snapshots**: If requested, create snapshots at intervals
5. **Mark Retroactive**: Flag events where transaction_time > valid_time
6. **Return Timeline**: Complete timeline with metadata

#### Example

```typescript
// Reconstruct timeline for entity
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001",
  start_time: "2025-01-01T00:00:00Z",
  end_time: "2025-01-31T23:59:59Z",
  include_snapshots: true,
  snapshot_interval: "1day"
});

console.log(`Timeline for ${timeline.entity_id}:`);
console.log(`- ${timeline.total_events} events`);
console.log(`- ${timeline.retroactive_count} retroactive corrections`);
console.log(`- ${timeline.snapshots.length} snapshots`);

// Iterate events
for (const event of timeline.events) {
  const retro = event.is_retroactive ? "[RETROACTIVE]" : "";
  console.log(`${event.timestamp} ${retro}: ${event.field_name}`);
  console.log(`  ${event.old_value} ’ ${event.new_value}`);
}
```

#### Implementation

```typescript
async reconstructTimeline(
  filters: TimelineReconstructionFilters
): Promise<Timeline> {
  // 1. Fetch events from provenance ledger
  const events = await bitemporalQuery.getTimeline(
    filters.entity_id,
    filters.field_name
  );

  // 2. Filter by time bounds
  const filteredEvents = events.filter(e => {
    const timeValue = filters.time_dimension === 'valid'
      ? e.valid_time
      : e.transaction_time;

    if (filters.start_time && timeValue < filters.start_time) return false;
    if (filters.end_time && timeValue > filters.end_time) return false;

    return true;
  });

  // 3. Transform to timeline events
  const timelineEvents: TimelineEvent[] = filteredEvents.map(e => ({
    sequence_number: e.sequence_number,
    timestamp: e.transaction_time,
    effective_time: e.valid_time,
    field_name: e.field_name,
    old_value: e.old_value,
    new_value: e.new_value,
    event_type: e.event_type,
    user_id: e.user_id,
    reason: e.reason,
    is_retroactive: e.transaction_time > e.valid_time,
    time_lag_ms: e.transaction_time > e.valid_time
      ? new Date(e.transaction_time).getTime() - new Date(e.valid_time).getTime()
      : undefined
  }));

  // 4. Generate snapshots if requested
  let snapshots: TimelineSnapshot[] = [];
  if (filters.include_snapshots) {
    snapshots = await this.generateSnapshots(
      filters.entity_id,
      filters.start_time,
      filters.end_time,
      filters.snapshot_interval || '1day'
    );
  }

  // 5. Build timeline
  const timeline: Timeline = {
    entity_id: filters.entity_id,
    entity_type: filters.entity_type || 'unknown',
    field_names: [...new Set(timelineEvents.map(e => e.field_name))],
    start_time: filters.start_time || timelineEvents[0]?.timestamp,
    end_time: filters.end_time || new Date().toISOString(),
    events: timelineEvents,
    snapshots: snapshots,
    total_events: timelineEvents.length,
    retroactive_count: timelineEvents.filter(e => e.is_retroactive).length,
    field_count: new Set(timelineEvents.map(e => e.field_name)).size
  };

  return timeline;
}
```

---

### 2. getSnapshot()

Get entity state at specific point in time.

#### Signature

```typescript
getSnapshot(
  entityId: string,
  timestamp: string,
  timeType: 'transaction' | 'valid'
): Promise<TimelineSnapshot>
```

#### Behavior

1. **Query State**: Use BitemporalQuery to get state at timestamp
2. **Count Events**: Count events applied to reach this state
3. **Build Snapshot**: Create TimelineSnapshot object

#### Example

```typescript
// Get snapshot on Jan 15, 2025
const snapshot = await reconstructor.getSnapshot(
  "txn_001",
  "2025-01-15T23:59:59Z",
  'transaction'
);

console.log(`Snapshot at ${snapshot.timestamp}:`);
console.log(JSON.stringify(snapshot.state, null, 2));
console.log(`Based on ${snapshot.event_count} events`);
```

---

### 3. interpolateValue()

Interpolate value at specific time (for numeric fields).

#### Signature

```typescript
interpolateValue(
  entityId: string,
  fieldName: string,
  timestamp: string
): Promise<any>
```

#### Behavior

1. **Get Timeline**: Fetch events for field
2. **Find Bounding Events**: Events before and after timestamp
3. **Interpolate**: Linear interpolation for numeric values
4. **Return Value**: Interpolated or exact value

#### Example

```typescript
// Interpolate price at specific time
const price = await reconstructor.interpolateValue(
  "prod_001",
  "price",
  "2025-01-10T12:00:00Z"
);

// If price was $29.99 on Jan 1 and $24.99 on Jan 15
// Interpolated price on Jan 10 might be ~$27.99
console.log(`Interpolated price: $${price.toFixed(2)}`);
```

#### Implementation

```typescript
async interpolateValue(
  entityId: string,
  fieldName: string,
  timestamp: string
): Promise<any> {
  // Get events for field
  const events = await bitemporalQuery.getTimeline(entityId, fieldName);

  // Find events before and after timestamp
  const before = events.filter(e => e.transaction_time <= timestamp);
  const after = events.filter(e => e.transaction_time > timestamp);

  if (before.length === 0 && after.length === 0) {
    return null; // No data
  }

  if (before.length > 0 && after.length === 0) {
    return before[before.length - 1].new_value; // Use last known value
  }

  if (before.length === 0 && after.length > 0) {
    return null; // No value yet at this time
  }

  const beforeEvent = before[before.length - 1];
  const afterEvent = after[0];

  // Check if values are numeric
  if (typeof beforeEvent.new_value !== 'number' ||
      typeof afterEvent.new_value !== 'number') {
    return beforeEvent.new_value; // Return last known value for non-numeric
  }

  // Linear interpolation
  const beforeTime = new Date(beforeEvent.transaction_time).getTime();
  const afterTime = new Date(afterEvent.transaction_time).getTime();
  const targetTime = new Date(timestamp).getTime();

  const ratio = (targetTime - beforeTime) / (afterTime - beforeTime);
  const interpolated = beforeEvent.new_value +
    ratio * (afterEvent.new_value - beforeEvent.new_value);

  return interpolated;
}
```

---

### 4. buildVisualization()

Build visualization-ready data structure from timeline.

#### Signature

```typescript
buildVisualization(
  timeline: Timeline,
  options: VisualizationOptions
): Promise<VisualizationData>
```

#### Behavior

Based on `options.format`, generate appropriate data structure:
- **d3_timeline**: Timeline lanes for D3.js timeline component
- **chart_js**: Datasets for Chart.js line/bar charts
- **custom**: Flexible custom format

#### Example: D3.js Timeline

```typescript
const vizData = await reconstructor.buildVisualization(timeline, {
  format: 'd3_timeline',
  highlight_retroactive: true
});

// vizData.lanes:
// [
//   {
//     id: "merchant",
//     label: "Merchant",
//     events: [
//       {
//         id: "event_1",
//         start: "2025-01-15T10:00:00Z",
//         label: "AMZN MKTP US*1234",
//         className: "normal"
//       },
//       {
//         id: "event_2",
//         start: "2025-01-20T14:30:00Z",
//         label: "Amazon.com",
//         className: "retroactive"
//       }
//     ]
//   }
// ]

// Render with D3.js
<D3Timeline lanes={vizData.lanes} />
```

#### Example: Chart.js Format

```typescript
const chartData = await reconstructor.buildVisualization(timeline, {
  format: 'chart_js',
  interpolate: true,
  interval: '1day'
});

// chartData.datasets:
// [
//   {
//     label: "Price",
//     data: [
//       { x: "2025-01-01", y: 29.99 },
//       { x: "2025-01-02", y: 29.99 },
//       ...
//       { x: "2025-01-15", y: 24.99 }
//     ],
//     borderColor: "#3b82f6"
//   }
// ]

<Line data={chartData} />
```

---

### 5. export()

Export timeline in various formats.

#### Signature

```typescript
export(
  timeline: Timeline,
  options: ExportOptions
): Promise<string>
```

#### Behavior

Serialize timeline to requested format:
- **json**: JSON representation
- **csv**: CSV with events
- **pdf_timeline**: PDF with visual timeline
- **excel**: Excel workbook with events and snapshots

#### Example

```typescript
// Export as JSON
const json = await reconstructor.export(timeline, {
  format: 'json',
  pretty_print: true,
  include_snapshots: true
});

fs.writeFileSync('timeline.json', json);

// Export as CSV
const csv = await reconstructor.export(timeline, {
  format: 'csv',
  include_metadata: true
});

fs.writeFileSync('timeline.csv', csv);
```

---

## Timeline Reconstruction

### Algorithm

```
ALGORITHM: ReconstructTimeline(entity_id, filters)

1. FETCH events from provenance ledger WHERE:
   - entity_id = entity_id
   - field_name IN filters.field_names (if specified)
   - transaction_time >= filters.start_time (if specified)
   - transaction_time <= filters.end_time (if specified)

2. SORT events by transaction_time ASC

3. FOR EACH event:
   - Calculate is_retroactive = (transaction_time > valid_time)
   - Calculate time_lag_ms if retroactive
   - Add to timeline.events

4. IF filters.include_snapshots:
   - Generate snapshots at intervals
   - For each snapshot time:
     - Query state at that time
     - Add to timeline.snapshots

5. COMPUTE metadata:
   - total_events = count(events)
   - retroactive_count = count(events WHERE is_retroactive)
   - field_count = count(DISTINCT field_name)
   - field_names = DISTINCT field_names from events

6. RETURN Timeline object
```

### Complexity

- **Time**: O(n log n) where n = number of events (sorting)
- **Space**: O(n + s) where s = number of snapshots

---

## Visualization Output

### D3.js Timeline Format

```typescript
interface D3TimelineData {
  lanes: TimelineLane[];
  bounds: { start: string; end: string };
}

interface TimelineLane {
  id: string;
  label: string;
  events: TimelineMarker[];
}

interface TimelineMarker {
  id: string;
  start: string;
  end?: string;
  label: string;
  className?: string; // e.g., "normal", "retroactive", "corrected"
  metadata?: Record<string, any>;
}
```

**Usage**:
```typescript
const vizData = await reconstructor.buildVisualization(timeline, {
  format: 'd3_timeline'
});

// Render with D3.js
d3.select("#timeline")
  .datum(vizData)
  .call(d3Timeline());
```

### Chart.js Format

```typescript
interface ChartJsData {
  labels: string[];      // X-axis labels (timestamps)
  datasets: ChartDataset[];
}

interface ChartDataset {
  label: string;
  data: { x: string; y: any }[];
  borderColor?: string;
  backgroundColor?: string;
  fill?: boolean;
}
```

**Usage**:
```typescript
const chartData = await reconstructor.buildVisualization(timeline, {
  format: 'chart_js',
  interpolate: true
});

new Chart(ctx, {
  type: 'line',
  data: chartData,
  options: { /* ... */ }
});
```

---

## Snapshot Interpolation

### Snapshot Generation

```typescript
async generateSnapshots(
  entityId: string,
  startTime: string,
  endTime: string,
  interval: string
): Promise<TimelineSnapshot[]> {
  const snapshots: TimelineSnapshot[] = [];

  // Parse interval (e.g., "1day", "1hour")
  const intervalMs = parseInterval(interval);

  const start = new Date(startTime).getTime();
  const end = new Date(endTime).getTime();

  for (let time = start; time <= end; time += intervalMs) {
    const timestamp = new Date(time).toISOString();

    // Get state at this time
    const state = await bitemporalQuery.queryTransactionTime({
      entity_id: entityId,
      transaction_time: timestamp
    });

    // Count events up to this time
    const events = await bitemporalQuery.getTimeline(entityId);
    const eventCount = events.filter(
      e => e.transaction_time <= timestamp
    ).length;

    snapshots.push({
      timestamp,
      state,
      event_count: eventCount
    });
  }

  return snapshots;
}
```

### Interpolation Strategies

**Linear Interpolation** (for numeric values):
```
value(t) = value(t1) + (value(t2) - value(t1)) * (t - t1) / (t2 - t1)
```

**Step Interpolation** (for categorical values):
```
value(t) = value(t1)  // Last known value before t
```

**Forward Fill**:
```
value(t) = last known value before t
```

---

## Edge Cases

### Edge Case 1: No Events in Time Range

**Scenario**: Timeline requested for time range with no events.

**Handling**: Return empty timeline with metadata:

```typescript
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001",
  start_time: "2020-01-01T00:00:00Z",
  end_time: "2020-12-31T23:59:59Z"
});

console.log(timeline.total_events); // 0
console.log(timeline.events); // []
```

---

### Edge Case 2: Retroactive Events Only

**Scenario**: All events in timeline are retroactive corrections.

**Handling**: Mark all as retroactive, highlight in visualization:

```typescript
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001"
});

if (timeline.retroactive_count === timeline.total_events) {
  console.log("All events are retroactive corrections!");
}
```

---

### Edge Case 3: Sparse Timeline with Large Gaps

**Scenario**: Long periods with no events.

**Handling**: Use interpolation to fill gaps:

```typescript
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "prod_001",
  field_name: "price",
  interpolate: true
});

// Interpolation fills gaps with estimated values
const vizData = await reconstructor.buildVisualization(timeline, {
  format: 'chart_js',
  interpolate: true,
  interval: '1day'
});
```

---

### Edge Case 4: Future Events

**Scenario**: Timeline includes scheduled future changes.

**Handling**: Include future events with special marker:

```typescript
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "prod_001",
  end_time: "2026-01-01T00:00:00Z" // Future date
});

const futureEvents = timeline.events.filter(
  e => new Date(e.effective_time) > new Date()
);

console.log(`${futureEvents.length} scheduled future changes`);
```

---

### Edge Case 5: Concurrent Events

**Scenario**: Multiple events at exact same timestamp.

**Handling**: Order by sequence_number:

```typescript
// Events with identical timestamps ordered by sequence
timeline.events.sort((a, b) => {
  if (a.timestamp === b.timestamp) {
    return a.sequence_number - b.sequence_number;
  }
  return a.timestamp.localeCompare(b.timestamp);
});
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `reconstructTimeline()` | 100 events | < 50ms | Simple timeline |
| `reconstructTimeline()` | 1,000 events | < 200ms | Complex timeline |
| `getSnapshot()` | Any | < 100ms | Delegates to BitemporalQuery |
| `interpolateValue()` | Any | < 50ms | Linear interpolation |
| `buildVisualization()` | 100 events | < 100ms | D3.js format |
| `export()` | 1,000 events | < 500ms | JSON/CSV export |

### Throughput

- **Timeline Reconstruction**: 100-500 timelines/sec
- **Snapshot Generation**: 1,000-5,000 snapshots/sec

### Optimization Tips

**1. Limit Event Count**

```typescript
// Bad: Fetch entire history
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001"
  // No time bounds = all events
});

// Good: Limit to recent history
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001",
  start_time: "2025-01-01T00:00:00Z",
  end_time: "2025-01-31T23:59:59Z"
});
```

**2. Disable Snapshots When Not Needed**

```typescript
// Bad: Generate snapshots when not using them
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001",
  include_snapshots: true, // Expensive
  snapshot_interval: "1hour" // Very expensive
});

// Good: Only generate if needed
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001",
  include_snapshots: false
});
```

**3. Cache Timelines**

```typescript
const timelineCache = new Map<string, Timeline>();

async function getCachedTimeline(entityId: string): Promise<Timeline> {
  if (timelineCache.has(entityId)) {
    return timelineCache.get(entityId)!;
  }

  const timeline = await reconstructor.reconstructTimeline({ entity_id: entityId });
  timelineCache.set(entityId, timeline);

  return timeline;
}
```

---

## Implementation Notes

### TypeScript Implementation

```typescript
import { BitemporalQuery } from './BitemporalQuery';
import { ProvenanceLedger } from './ProvenanceLedger';

export class TimelineReconstructor {
  constructor(
    private bitemporalQuery: BitemporalQuery,
    private provenanceLedger: ProvenanceLedger
  ) {}

  async reconstructTimeline(
    filters: TimelineReconstructionFilters
  ): Promise<Timeline> {
    // Implementation shown in Core Functionality section
    // ...
  }

  async getSnapshot(
    entityId: string,
    timestamp: string,
    timeType: 'transaction' | 'valid'
  ): Promise<TimelineSnapshot> {
    const state = timeType === 'transaction'
      ? await this.bitemporalQuery.queryTransactionTime({
          entity_id: entityId,
          transaction_time: timestamp
        })
      : await this.bitemporalQuery.queryValidTime({
          entity_id: entityId,
          valid_time: timestamp
        });

    const events = await this.provenanceLedger.getHistory(entityId);
    const eventCount = events.filter(
      e => timeType === 'transaction'
        ? e.transaction_time <= timestamp
        : e.valid_time <= timestamp
    ).length;

    return {
      timestamp,
      state,
      event_count: eventCount
    };
  }

  async buildVisualization(
    timeline: Timeline,
    options: VisualizationOptions
  ): Promise<VisualizationData> {
    if (options.format === 'd3_timeline') {
      return this.buildD3Timeline(timeline, options);
    } else if (options.format === 'chart_js') {
      return this.buildChartJs(timeline, options);
    } else {
      throw new Error(`Unsupported format: ${options.format}`);
    }
  }

  private buildD3Timeline(
    timeline: Timeline,
    options: VisualizationOptions
  ): VisualizationData {
    const lanes: TimelineLane[] = [];

    // Group events by field
    const byField = timeline.events.reduce((acc, event) => {
      if (!acc[event.field_name]) acc[event.field_name] = [];
      acc[event.field_name].push(event);
      return acc;
    }, {} as Record<string, TimelineEvent[]>);

    // Create lane for each field
    for (const [fieldName, events] of Object.entries(byField)) {
      const lane: TimelineLane = {
        id: fieldName,
        label: fieldName,
        events: events.map(e => ({
          id: `event_${e.sequence_number}`,
          start: e.timestamp,
          label: String(e.new_value),
          className: e.is_retroactive && options.highlight_retroactive
            ? 'retroactive'
            : 'normal'
        }))
      };
      lanes.push(lane);
    }

    return {
      format: 'd3_timeline',
      lanes,
      bounds: {
        start: timeline.start_time,
        end: timeline.end_time
      },
      event_count: timeline.total_events,
      retroactive_count: timeline.retroactive_count
    };
  }

  private buildChartJs(
    timeline: Timeline,
    options: VisualizationOptions
  ): VisualizationData {
    const datasets: ChartDataset[] = [];

    // Group events by field
    const byField = timeline.events.reduce((acc, event) => {
      if (!acc[event.field_name]) acc[event.field_name] = [];
      acc[event.field_name].push(event);
      return acc;
    }, {} as Record<string, TimelineEvent[]>);

    // Create dataset for each field
    for (const [fieldName, events] of Object.entries(byField)) {
      const data = events.map(e => ({
        x: e.timestamp,
        y: e.new_value
      }));

      datasets.push({
        label: fieldName,
        data,
        borderColor: this.getColor(fieldName),
        fill: false
      });
    }

    // Extract labels
    const labels = [...new Set(timeline.events.map(e => e.timestamp))].sort();

    return {
      format: 'chart_js',
      labels,
      datasets,
      bounds: {
        start: timeline.start_time,
        end: timeline.end_time
      },
      event_count: timeline.total_events,
      retroactive_count: timeline.retroactive_count
    };
  }

  private getColor(fieldName: string): string {
    // Simple hash-based color assignment
    const colors = [
      '#3b82f6', '#ef4444', '#10b981', '#f59e0b',
      '#8b5cf6', '#ec4899', '#14b8a6', '#f97316'
    ];

    let hash = 0;
    for (let i = 0; i < fieldName.length; i++) {
      hash = fieldName.charCodeAt(i) + ((hash << 5) - hash);
    }

    return colors[Math.abs(hash) % colors.length];
  }
}
```

---

## Visualization Integration

### React Component Example

```typescript
import { useState, useEffect } from 'react';
import { Timeline as D3Timeline } from 'd3-timeline';

function TimelineView({ entityId }: { entityId: string }) {
  const [timeline, setTimeline] = useState<Timeline | null>(null);

  useEffect(() => {
    async function loadTimeline() {
      const timeline = await reconstructor.reconstructTimeline({
        entity_id: entityId,
        include_snapshots: true
      });
      setTimeline(timeline);
    }

    loadTimeline();
  }, [entityId]);

  if (!timeline) return <div>Loading...</div>;

  return (
    <div>
      <h2>Timeline for {entityId}</h2>
      <div className="stats">
        <span>{timeline.total_events} events</span>
        <span>{timeline.retroactive_count} retroactive</span>
      </div>
      <D3Timeline data={timeline} />
    </div>
  );
}
```

---

## Integration Patterns

### Pattern 1: Audit Trail Viewer

```typescript
class AuditTrailViewer {
  async viewEntityHistory(entityId: string): Promise<void> {
    const timeline = await reconstructor.reconstructTimeline({
      entity_id: entityId,
      include_retroactive: true
    });

    console.log(`Audit Trail for ${entityId}:`);
    console.log(`Total changes: ${timeline.total_events}`);
    console.log(`Retroactive corrections: ${timeline.retroactive_count}`);

    for (const event of timeline.events) {
      console.log(`\n${event.timestamp}`);
      console.log(`  User: ${event.user_id}`);
      console.log(`  Field: ${event.field_name}`);
      console.log(`  Change: ${event.old_value} ’ ${event.new_value}`);
      if (event.is_retroactive) {
        console.log(`     Retroactive (effective ${event.effective_time})`);
      }
    }
  }
}
```

---

## Multi-Domain Examples

(Already covered extensively in Multi-Domain Applicability section with 7 domains)

---

## Testing Strategy

### Unit Tests

```typescript
describe('TimelineReconstructor', () => {
  let reconstructor: TimelineReconstructor;

  beforeEach(() => {
    reconstructor = new TimelineReconstructor(bitemporalQuery, provenanceLedger);
  });

  describe('reconstructTimeline()', () => {
    it('should reconstruct complete timeline', async () => {
      const timeline = await reconstructor.reconstructTimeline({
        entity_id: 'txn_001'
      });

      expect(timeline.events).toHaveLength(3);
      expect(timeline.total_events).toBe(3);
    });

    it('should mark retroactive events', async () => {
      const timeline = await reconstructor.reconstructTimeline({
        entity_id: 'txn_001'
      });

      const retroactive = timeline.events.filter(e => e.is_retroactive);
      expect(retroactive).toHaveLength(1);
    });
  });

  describe('buildVisualization()', () => {
    it('should build D3 timeline format', async () => {
      const timeline = await reconstructor.reconstructTimeline({
        entity_id: 'txn_001'
      });

      const viz = await reconstructor.buildVisualization(timeline, {
        format: 'd3_timeline'
      });

      expect(viz.lanes).toBeDefined();
      expect(viz.bounds).toBeDefined();
    });
  });
});
```

---

## Migration Guide

### From Manual Timeline Construction

```typescript
// Before: Manual event iteration
const events = await provenanceLedger.getHistory("txn_001");
const timeline = {
  events: events.map(e => ({
    timestamp: e.transaction_time,
    field: e.field_name,
    value: e.new_value
  }))
};

// After: Use TimelineReconstructor
const timeline = await reconstructor.reconstructTimeline({
  entity_id: "txn_001"
});
```

---

## Related Primitives

- **ProvenanceLedger**: Source of bitemporal events
- **BitemporalQuery**: Provides temporal queries for snapshots
- **RetroactiveCorrector**: Creates retroactive events that appear in timelines

---

## References

- [ProvenanceLedger Primitive](./ProvenanceLedger.md)
- [BitemporalQuery Primitive](./BitemporalQuery.md)
- [D3.js Timeline Component](https://github.com/jiahuang/d3-timeline)
- [Chart.js Documentation](https://www.chartjs.org/)

---

**End of TimelineReconstructor OL Primitive Specification**
