# DrillDownPanel (IL Component)

## Definition

**DrillDownPanel** is a reusable UI component that displays complete data lineage in a slide-in panel (desktop) or full-screen view (mobile). It renders canonical data, raw observations, artifact links, normalization decisions, and provenance timeline.

**Problem it solves:**
- Drill-down data is complex (5+ data layers)
- Modal overlays block context (user loses list view)
- Mobile full-screen modals are better UX than small panels
- Repeating drill-down UI code across features leads to inconsistency

**Solution:**
- Single reusable component for all drill-down views
- Responsive: slide-in panel (desktop) vs full-screen (mobile)
- Tabbed interface (Overview, Raw Data, Provenance, Artifact)
- Keyboard navigation (ESC to close, arrow keys for tabs)
- Loading states (skeleton loader, not spinner)

---

## Interface Contract

```typescript
interface DrillDownPanelProps {
  // Data
  drilldownData: DrillDownData | null;  // From ProvenanceTracer
  loading?: boolean;
  error?: string | null;

  // Callbacks
  onClose: () => void;  // Called when user closes panel
  onDownloadArtifact?: (uploadId: string) => void;
  onFlagForReview?: (canonicalId: string) => void;  // Deferred to 4.3

  // UI customization
  defaultTab?: "overview" | "raw-data" | "provenance" | "artifact";
  showArtifactTab?: boolean;  // Hide if artifact deleted
  theme?: "light" | "dark";
}

interface DrillDownData {
  canonical: Record<string, any>;  // Canonical record (all fields)
  observation: Record<string, any> | null;  // Raw observation (may be null)
  artifact: ArtifactMetadata;  // Artifact metadata
  decisions: DecisionExplanation[];  // Normalization decisions
  provenance: ProvenanceEntry[];  // Provenance chain
  validation: ValidationResults;  // Validation results
  logs: LogURLs;  // Log URLs
}
```

---

## Component Structure

```tsx
import React, { useState } from 'react';

export const DrillDownPanel: React.FC<DrillDownPanelProps> = ({
  drilldownData,
  loading = false,
  error = null,
  onClose,
  onDownloadArtifact,
  onFlagForReview,
  defaultTab = "overview",
  showArtifactTab = true,
  theme = "light"
}) => {
  const [activeTab, setActiveTab] = useState(defaultTab);

  // Keyboard navigation
  useEffect(() => {
    const handleKeydown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
      if (e.key === 'ArrowRight') nextTab();
      if (e.key === 'ArrowLeft') prevTab();
    };
    window.addEventListener('keydown', handleKeydown);
    return () => window.removeEventListener('keydown', handleKeydown);
  }, [activeTab]);

  if (loading) {
    return <DrillDownSkeleton />;
  }

  if (error) {
    return (
      <DrillDownError
        message={error}
        onClose={onClose}
      />
    );
  }

  if (!drilldownData) {
    return null;  // Panel closed
  }

  return (
    <div className="drilldown-panel">
      {/* Header */}
      <div className="drilldown-header">
        <h2>Transaction Details</h2>
        <button onClick={onClose} aria-label="Close">
          ✕
        </button>
      </div>

      {/* Tabs */}
      <div className="drilldown-tabs" role="tablist">
        <button
          role="tab"
          aria-selected={activeTab === "overview"}
          onClick={() => setActiveTab("overview")}
        >
          Overview
        </button>
        <button
          role="tab"
          aria-selected={activeTab === "raw-data"}
          onClick={() => setActiveTab("raw-data")}
        >
          Raw Data
        </button>
        <button
          role="tab"
          aria-selected={activeTab === "provenance"}
          onClick={() => setActiveTab("provenance")}
        >
          Provenance
        </button>
        {showArtifactTab && (
          <button
            role="tab"
            aria-selected={activeTab === "artifact"}
            onClick={() => setActiveTab("artifact")}
          >
            Artifact
          </button>
        )}
      </div>

      {/* Tab Content */}
      <div className="drilldown-content">
        {activeTab === "overview" && (
          <OverviewTab
            canonical={drilldownData.canonical}
            decisions={drilldownData.decisions}
            validation={drilldownData.validation}
          />
        )}
        {activeTab === "raw-data" && (
          <RawDataTab
            canonical={drilldownData.canonical}
            observation={drilldownData.observation}
          />
        )}
        {activeTab === "provenance" && (
          <ProvenanceTab
            provenance={drilldownData.provenance}
            logs={drilldownData.logs}
          />
        )}
        {activeTab === "artifact" && showArtifactTab && (
          <ArtifactTab
            artifact={drilldownData.artifact}
            onDownload={onDownloadArtifact}
          />
        )}
      </div>

      {/* Footer Actions */}
      <div className="drilldown-footer">
        <button onClick={onClose}>Close</button>
        {onFlagForReview && (
          <button onClick={() => onFlagForReview(drilldownData.canonical.canonical_id)}>
            Flag for Review
          </button>
        )}
      </div>
    </div>
  );
};
```

---

## Tab Components

### OverviewTab

```tsx
const OverviewTab: React.FC<{
  canonical: Record<string, any>;
  decisions: DecisionExplanation[];
  validation: ValidationResults;
}> = ({ canonical, decisions, validation }) => {
  return (
    <div className="overview-tab">
      {/* Canonical Data */}
      <section>
        <h3>Canonical Data</h3>
        <dl>
          <dt>Date:</dt>
          <dd>{canonical.transaction_date}</dd>
          <dt>Merchant:</dt>
          <dd>{canonical.merchant}</dd>
          <dt>Amount:</dt>
          <dd>{canonical.amount} {canonical.currency}</dd>
          <dt>Category:</dt>
          <dd>{canonical.category}</dd>
        </dl>
      </section>

      {/* Decision Explanations */}
      <section>
        <h3>Normalization Decisions</h3>
        {decisions.map((decision, i) => (
          <RuleExplanationCard key={i} decision={decision} />
        ))}
      </section>

      {/* Validation Results */}
      <section>
        <h3>Validation</h3>
        <div className={`validation-status ${validation.status}`}>
          {validation.status === "valid" && "✓ All checks passed"}
          {validation.status === "warning" && "⚠ Warnings detected"}
          {validation.status === "error" && "✗ Validation failed"}
        </div>
        {validation.warnings.length > 0 && (
          <ul>
            {validation.warnings.map((warning, i) => (
              <li key={i}>{warning}</li>
            ))}
          </ul>
        )}
      </section>
    </div>
  );
};
```

### RawDataTab

```tsx
const RawDataTab: React.FC<{
  canonical: Record<string, any>;
  observation: Record<string, any> | null;
}> = ({ canonical, observation }) => {
  if (!observation) {
    return (
      <div className="empty-state">
        <p>Raw observation data unavailable (parse error).</p>
        <p>Showing canonical data only.</p>
      </div>
    );
  }

  return (
    <FieldComparisonTable
      observation={observation}
      canonical={canonical}
    />
  );
};
```

### ProvenanceTab

```tsx
const ProvenanceTab: React.FC<{
  provenance: ProvenanceEntry[];
  logs: LogURLs;
}> = ({ provenance, logs }) => {
  return (
    <div className="provenance-tab">
      {/* Timeline Visualization */}
      <ProvenanceTimeline entries={provenance} />

      {/* Log Links */}
      <section>
        <h3>Execution Logs</h3>
        <ul>
          <li>
            <a href={logs.parse_log_url} target="_blank">
              Parse Log →
            </a>
          </li>
          <li>
            <a href={logs.normalization_log_url} target="_blank">
              Normalization Log →
            </a>
          </li>
        </ul>
      </section>
    </div>
  );
};
```

### ArtifactTab

```tsx
const ArtifactTab: React.FC<{
  artifact: ArtifactMetadata;
  onDownload?: (uploadId: string) => void;
}> = ({ artifact, onDownload }) => {
  if (artifact.status === "deleted") {
    return (
      <div className="empty-state">
        <p>Original file deleted (retention policy).</p>
        <p>Deleted on: {artifact.deleted_at}</p>
        <p>Hash (for audit): {artifact.hash}</p>
      </div>
    );
  }

  return (
    <div className="artifact-tab">
      {/* Artifact Metadata */}
      <section>
        <h3>File Information</h3>
        <dl>
          <dt>Filename:</dt>
          <dd>{artifact.filename}</dd>
          <dt>Size:</dt>
          <dd>{formatBytes(artifact.size_bytes)}</dd>
          <dt>Uploaded by:</dt>
          <dd>{artifact.uploaded_by}</dd>
          <dt>Uploaded at:</dt>
          <dd>{formatDate(artifact.uploaded_at)}</dd>
        </dl>
      </section>

      {/* PDF Viewer */}
      <section>
        <h3>Preview</h3>
        <ArtifactViewer
          url={artifact.view_url}
          mimeType={artifact.mime_type}
        />
      </section>

      {/* Download Button */}
      <button onClick={() => onDownload?.(artifact.upload_id)}>
        Download PDF
      </button>
    </div>
  );
};
```

---

## Responsive Behavior

### Desktop (≥768px)

```css
.drilldown-panel {
  position: fixed;
  top: 0;
  right: 0;
  width: 600px;
  height: 100vh;
  background: white;
  box-shadow: -2px 0 8px rgba(0,0,0,0.1);
  transform: translateX(100%);
  transition: transform 0.3s ease;
}

.drilldown-panel.open {
  transform: translateX(0);
}
```

### Mobile (<768px)

```css
.drilldown-panel {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: white;
  z-index: 1000;
}
```

---

## Loading States

```tsx
const DrillDownSkeleton: React.FC = () => {
  return (
    <div className="drilldown-skeleton">
      <div className="skeleton-header">
        <div className="skeleton-title"></div>
        <div className="skeleton-close"></div>
      </div>
      <div className="skeleton-tabs">
        <div className="skeleton-tab"></div>
        <div className="skeleton-tab"></div>
        <div className="skeleton-tab"></div>
      </div>
      <div className="skeleton-content">
        <div className="skeleton-line"></div>
        <div className="skeleton-line"></div>
        <div className="skeleton-line"></div>
      </div>
    </div>
  );
};
```

---

## Accessibility

**Keyboard Navigation:**
- `ESC` - Close panel
- `→` - Next tab
- `←` - Previous tab
- `Tab` - Focus next element
- `Shift+Tab` - Focus previous element

**ARIA Attributes:**
```tsx
<div
  role="dialog"
  aria-labelledby="drilldown-title"
  aria-modal="true"
>
  <h2 id="drilldown-title">Transaction Details</h2>
  <div role="tablist">
    <button role="tab" aria-selected={active} aria-controls="panel-1">
      Overview
    </button>
  </div>
  <div id="panel-1" role="tabpanel" aria-labelledby="tab-1">
    ...
  </div>
</div>
```

**Screen Reader Support:**
- Announce panel open/close
- Announce tab changes
- Read decision explanations
- Read provenance timeline events

---

## Multi-Domain Applicability

**Finance:**
```tsx
<DrillDownPanel
  drilldownData={transactionDrilldown}
  onClose={() => setOpen(false)}
/>
// Displays: Transaction details, bank statement PDF
```

**Healthcare:**
```tsx
<DrillDownPanel
  drilldownData={labResultDrilldown}
  onClose={() => setOpen(false)}
/>
// Displays: Lab result details, lab report PDF
```

**Legal:**
```tsx
<DrillDownPanel
  drilldownData={clauseDrilldown}
  onClose={() => setOpen(false)}
/>
// Displays: Contract clause details, contract PDF
```

**Research:**
```tsx
<DrillDownPanel
  drilldownData={citationDrilldown}
  onClose={() => setOpen(false)}
/>
// Displays: Citation details, paper PDF
```

**Domain-agnostic nature:** Same component, different data. Tabs and structure remain identical.

---

## Related Components

**Used by:**
- Transaction list view (2.1) - Opens drill-down panel on row click
- Finance dashboard (2.3) - Shows drill-down in dashboard context
- Corrections flow (4.3) - Opens drill-down before editing

**Uses:**
- **ProvenanceTimeline** (IL) - Renders provenance tab
- **FieldComparisonTable** (IL) - Renders raw data tab
- **RuleExplanationCard** (IL) - Renders decision explanations
- **ArtifactViewer** (IL) - Renders PDF/image viewer

---

## Example Usage

```tsx
import { DrillDownPanel } from '@/components/DrillDownPanel';
import { ProvenanceTracer } from '@/primitives/ol';

function TransactionListView() {
  const [selectedCanonicalId, setSelectedCanonicalId] = useState<string | null>(null);
  const [drilldownData, setDrilldownData] = useState<DrillDownData | null>(null);
  const [loading, setLoading] = useState(false);

  // Fetch drill-down data when transaction clicked
  useEffect(() => {
    if (!selectedCanonicalId) return;

    setLoading(true);
    fetch(`/api/transactions/${selectedCanonicalId}/drilldown`)
      .then(res => res.json())
      .then(data => {
        setDrilldownData(data);
        setLoading(false);
      });
  }, [selectedCanonicalId]);

  return (
    <div>
      {/* Transaction Table */}
      <TransactionTable
        onRowClick={(canonicalId) => setSelectedCanonicalId(canonicalId)}
      />

      {/* Drill-Down Panel */}
      {selectedCanonicalId && (
        <DrillDownPanel
          drilldownData={drilldownData}
          loading={loading}
          onClose={() => {
            setSelectedCanonicalId(null);
            setDrilldownData(null);
          }}
          onDownloadArtifact={(uploadId) => {
            window.open(`/api/uploads/${uploadId}/artifact?download=true`, '_blank');
          }}
        />
      )}
    </div>
  );
}
```
