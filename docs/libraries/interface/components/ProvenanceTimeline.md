# ProvenanceTimeline (IL Component)

## Definition

**ProvenanceTimeline** is a visual timeline component that displays the complete audit trail (upload → parse → normalize → correct) for a canonical record. It shows events chronologically with timestamps, actors, and status indicators.

**Problem it solves:**
- Provenance data is hard to read as raw JSON
- Users need visual representation of "what happened when"
- Timestamps alone don't show duration (parse took 15s, normalize took 3s)
- Error events need visual distinction from success events

**Solution:**
- Vertical timeline with event cards
- Color-coded status (success=green, error=red, warning=yellow)
- Duration indicators (time between events)
- Expandable metadata (click event to see details)
- Responsive design (desktop/mobile)

---

## Interface Contract

```typescript
interface ProvenanceTimelineProps {
  // Data
  entries: ProvenanceEntry[];  // Array of provenance events

  // UI customization
  variant?: "vertical" | "horizontal";  // Layout direction (default: vertical)
  showDurations?: boolean;  // Show time between events (default: true)
  showMetadata?: boolean;  // Show expandable metadata (default: true)
  maxHeight?: string;  // Max height before scrolling (default: "600px")
  theme?: "light" | "dark";
}

interface ProvenanceEntry {
  event: string;  // Event name (e.g., "uploaded", "parsed", "normalized")
  timestamp: string;  // ISO 8601 timestamp
  actor: string;  // Who/what performed this event (e.g., "eugenio@example.com", "system:parser_v1.2")
  status?: "success" | "error" | "warning";  // Event status
  metadata?: Record<string, any>;  // Additional context
}
```

---

## Component Structure

```tsx
import React, { useState } from 'react';
import { formatDistanceToNow, differenceInSeconds } from 'date-fns';

export const ProvenanceTimeline: React.FC<ProvenanceTimelineProps> = ({
  entries,
  variant = "vertical",
  showDurations = true,
  showMetadata = true,
  maxHeight = "600px",
  theme = "light"
}) => {
  const [expandedIndex, setExpandedIndex] = useState<number | null>(null);

  // Sort entries chronologically
  const sortedEntries = [...entries].sort((a, b) =>
    new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime()
  );

  // Calculate durations between events
  const getDuration = (index: number): number | null => {
    if (index === 0) return null;
    const prev = sortedEntries[index - 1];
    const curr = sortedEntries[index];
    return differenceInSeconds(new Date(curr.timestamp), new Date(prev.timestamp));
  };

  return (
    <div
      className={`provenance-timeline ${variant} ${theme}`}
      style={{ maxHeight }}
    >
      {sortedEntries.map((entry, index) => (
        <div key={index} className="timeline-entry">
          {/* Duration Indicator */}
          {showDurations && index > 0 && (
            <div className="timeline-duration">
              {formatDuration(getDuration(index))}
            </div>
          )}

          {/* Timeline Dot */}
          <div className={`timeline-dot ${entry.status || 'success'}`}>
            {getStatusIcon(entry.status)}
          </div>

          {/* Event Card */}
          <div
            className="timeline-card"
            onClick={() => setExpandedIndex(expandedIndex === index ? null : index)}
          >
            {/* Event Header */}
            <div className="timeline-card-header">
              <span className="event-name">{formatEventName(entry.event)}</span>
              <span className="event-timestamp">
                {formatDistanceToNow(new Date(entry.timestamp), { addSuffix: true })}
              </span>
            </div>

            {/* Event Actor */}
            <div className="timeline-card-actor">
              {formatActor(entry.actor)}
            </div>

            {/* Expandable Metadata */}
            {showMetadata && entry.metadata && expandedIndex === index && (
              <div className="timeline-card-metadata">
                <pre>{JSON.stringify(entry.metadata, null, 2)}</pre>
              </div>
            )}
          </div>
        </div>
      ))}
    </div>
  );
};
```

---

## Helper Functions

```tsx
function formatEventName(event: string): string {
  const eventNames = {
    "uploaded": "📤 Uploaded",
    "parsed": "🔍 Parsed",
    "normalized": "✓ Normalized",
    "corrected": "✏️ Corrected",
    "error": "✗ Error"
  };
  return eventNames[event] || event;
}

function formatActor(actor: string): string {
  if (actor.startsWith("system:")) {
    return `System: ${actor.replace("system:", "")}`;
  }
  return `User: ${actor}`;
}

function getStatusIcon(status?: string): string {
  switch (status) {
    case "success": return "✓";
    case "error": return "✗";
    case "warning": return "⚠";
    default: return "•";
  }
}

function formatDuration(seconds: number | null): string {
  if (!seconds) return "";
  if (seconds < 1) return "<1s";
  if (seconds < 60) return `${seconds}s`;
  return `${Math.floor(seconds / 60)}m ${seconds % 60}s`;
}
```

---

## Visual Design

### Vertical Timeline (Desktop)

```
┌─────────────────────────────────┐
│ 📤 Uploaded                     │
│ User: eugenio@example.com       │
│ 2 hours ago                     │
└─────────────────────────────────┘
        ↓ 15s
┌─────────────────────────────────┐
│ 🔍 Parsed                       │
│ System: bofa_pdf_parser_v1.2    │
│ 2 hours ago                     │
│ [Click to expand metadata]      │
└─────────────────────────────────┘
        ↓ 3s
┌─────────────────────────────────┐
│ ✓ Normalized                    │
│ System: normalizer_v1.0         │
│ 2 hours ago                     │
└─────────────────────────────────┘
```

### Horizontal Timeline (Mobile)

```
📤────15s────🔍────3s────✓
```

---

## CSS Styling

```css
/* Vertical Timeline */
.provenance-timeline.vertical {
  display: flex;
  flex-direction: column;
  gap: 0;
  overflow-y: auto;
}

.timeline-entry {
  display: grid;
  grid-template-columns: auto 1fr;
  gap: 16px;
  position: relative;
}

.timeline-dot {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  z-index: 1;
}

.timeline-dot.success {
  background: #10b981;
  color: white;
}

.timeline-dot.error {
  background: #ef4444;
  color: white;
}

.timeline-dot.warning {
  background: #f59e0b;
  color: white;
}

/* Connecting Line */
.timeline-entry::before {
  content: '';
  position: absolute;
  left: 16px;
  top: 32px;
  bottom: -16px;
  width: 2px;
  background: #e5e7eb;
}

.timeline-entry:last-child::before {
  display: none;
}

/* Event Card */
.timeline-card {
  background: #f9fafb;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 12px 16px;
  cursor: pointer;
  transition: all 0.2s;
}

.timeline-card:hover {
  background: #f3f4f6;
  border-color: #d1d5db;
}

/* Duration Indicator */
.timeline-duration {
  grid-column: 1;
  text-align: center;
  font-size: 12px;
  color: #6b7280;
  padding: 4px 0;
}
```

---

## Accessibility

**Keyboard Navigation:**
- `Tab` - Focus next event
- `Enter` / `Space` - Expand/collapse metadata
- Screen reader announces: "Event: Uploaded, by eugenio@example.com, 2 hours ago"

**ARIA Attributes:**
```tsx
<div
  role="list"
  aria-label="Provenance timeline"
>
  <div
    role="listitem"
    aria-label={`${entry.event} by ${entry.actor} at ${entry.timestamp}`}
    tabIndex={0}
    onClick={() => toggleMetadata(index)}
  >
    ...
  </div>
</div>
```

---

## Multi-Domain Applicability

**Finance:**
```tsx
<ProvenanceTimeline
  entries={[
    { event: "uploaded", timestamp: "2025-05-23T10:00:00Z", actor: "eugenio@example.com" },
    { event: "parsed", timestamp: "2025-05-23T10:00:15Z", actor: "system:bofa_parser_v1.2" },
    { event: "normalized", timestamp: "2025-05-23T10:00:18Z", actor: "system:normalizer_v1.0" }
  ]}
/>
// Shows: Bank statement processing timeline
```

**Healthcare:**
```tsx
<ProvenanceTimeline
  entries={[
    { event: "collected", timestamp: "...", actor: "lab_tech_01" },
    { event: "analyzed", timestamp: "...", actor: "system:lab_analyzer_v2.1" },
    { event: "reviewed", timestamp: "...", actor: "dr_smith@hospital.com" }
  ]}
/>
// Shows: Lab result processing timeline
```

**Legal:**
```tsx
<ProvenanceTimeline
  entries={[
    { event: "drafted", timestamp: "...", actor: "lawyer_01@firm.com" },
    { event: "reviewed", timestamp: "...", actor: "senior_partner@firm.com" },
    { event: "approved", timestamp: "...", actor: "system:compliance_check_v1.0" }
  ]}
/>
// Shows: Contract approval timeline
```

**Domain-agnostic nature:** Same component, different event names. Timeline structure remains identical.

---

## Related Components

**Used by:**
- **DrillDownPanel** (IL) - Renders provenance timeline in Provenance tab
- **AuditLog** (IL) - Shows timeline for corrections history (vertical 4.3)

**Uses:**
- No dependencies (standalone component)

---

## Example Usage

### Basic Timeline

```tsx
import { ProvenanceTimeline } from '@/components/ProvenanceTimeline';

function DrillDownPanel({ drilldownData }) {
  return (
    <div>
      <h3>Provenance</h3>
      <ProvenanceTimeline
        entries={drilldownData.provenance}
        showDurations={true}
        showMetadata={true}
      />
    </div>
  );
}
```

### With Metadata Expansion

```tsx
<ProvenanceTimeline
  entries={[
    {
      event: "parsed",
      timestamp: "2025-05-23T10:00:15Z",
      actor: "system:bofa_parser_v1.2",
      status: "success",
      metadata: {
        observations_extracted: 47,
        errors: 0,
        warnings: 2,
        execution_time_ms: 15234
      }
    }
  ]}
  showMetadata={true}  // Click event to see metadata
/>
```

### Error Handling

```tsx
<ProvenanceTimeline
  entries={[
    {
      event: "uploaded",
      timestamp: "2025-05-23T10:00:00Z",
      actor: "eugenio@example.com",
      status: "success"
    },
    {
      event: "parsed",
      timestamp: "2025-05-23T10:00:15Z",
      actor: "system:bofa_parser_v1.2",
      status: "error",  // Red indicator
      metadata: {
        error: "PDF_CORRUPT",
        message: "PDF file corrupted or encrypted"
      }
    }
  ]}
/>
// Shows red dot for error event
```

### Horizontal Timeline (Mobile)

```tsx
<ProvenanceTimeline
  entries={provenance}
  variant="horizontal"  // Switch to horizontal on mobile
  showDurations={false}  // Hide durations in compact view
  showMetadata={false}   // No metadata expansion in mobile
/>
```
