# ReconciliationDashboard (IL Component)

## Definition

**ReconciliationDashboard** is a comprehensive UI component for managing multi-source reconciliation. It displays unmatched items, suggested matches with confidence scores, and confirmed matches. Supports accepting/rejecting suggestions, manual matching, threshold adjustment, and re-running reconciliation after configuration changes.

**Problem it solves:**
- No centralized view of reconciliation status (matched vs unmatched items)
- Users unsure which items need attention
- No easy way to review auto-suggested matches before acceptance
- Cannot manually match items when auto-detection fails
- No visibility into reconciliation progress (% matched)
- Difficult to adjust thresholds and re-run reconciliation

**Solution:**
- Three-column layout: Unmatched Items | Suggested Matches | Matched Items
- Progress indicator showing reconciliation completion (% matched)
- One-click accept/reject for suggested matches with confidence scores
- Manual match creation for items auto-detection missed
- Threshold adjustment UI with instant preview of impact
- Batch operations (accept all high-confidence, unmatch all low-confidence)
- Filter/search across all sections
- Export reconciliation report for audit trail

---

## Interface Contract

```typescript
interface ReconciliationDashboardProps {
  // Data
  unmatchedItems: UnmatchedItemsData;
  suggestedMatches: MatchCandidate[];
  matchedItems: ReconciliationMatch[];

  // Configuration
  config: ReconciliationConfig;

  // Callbacks
  onAcceptMatch: (candidateId: string) => Promise<void>;
  onRejectMatch: (candidateId: string, reason?: string) => Promise<void>;
  onManualMatch: (source1Items: string[], source2Items: string[]) => Promise<void>;
  onUnmatch: (matchId: string, reason?: string) => Promise<void>;
  onUpdateConfig: (config: ReconciliationConfig) => Promise<void>;
  onRerunReconciliation: () => Promise<void>;
  onExportReport: () => Promise<Blob>;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Display options
  theme?: "light" | "dark";
  showConfidenceBreakdown?: boolean;  // Show individual feature scores
  allowBatchOperations?: boolean;
}

interface UnmatchedItemsData {
  source_1: UnmatchedItem[];
  source_2: UnmatchedItem[];
  total_source_1: number;
  total_source_2: number;
  unmatched_source_1: number;
  unmatched_source_2: number;
}

interface UnmatchedItem {
  item_id: string;
  display_text: string;  // Primary description
  amount?: number;
  date?: string;
  metadata: Record<string, any>;  // Domain-specific fields
}

interface MatchCandidate {
  candidate_id: string;
  source_1_item: UnmatchedItem;
  source_2_item: UnmatchedItem;
  similarity_score: number;  // Overall confidence (0.0 to 1.0)
  feature_scores: Record<string, number>;  // Individual feature scores
  decision: "auto_link" | "suggest" | "manual" | "no_match";
  match_details: Record<string, any>;
}

interface ReconciliationMatch {
  match_id: string;
  source_1_items: UnmatchedItem[];
  source_2_items: UnmatchedItem[];
  cardinality: "one_to_one" | "one_to_many" | "many_to_one" | "many_to_many";
  confidence: number;
  match_method: "auto" | "manual" | "assisted";
  matched_at: string;  // ISO timestamp
  matched_by?: string;  // User ID for manual matches
}

interface ReconciliationConfig {
  thresholds: {
    auto_link: number;      // e.g., 0.95 (auto-accept if confidence >= 95%)
    auto_suggest: number;   // e.g., 0.70 (suggest if 70% <= confidence < 95%)
    manual: number;         // e.g., 0.50 (allow manual if confidence >= 50%)
  };
  tolerances: {
    amount_percent?: number;   // e.g., 0.05 (5% amount tolerance)
    amount_fixed?: number;     // e.g., 0.01 ($0.01 tolerance for rounding)
    date_days?: number;        // e.g., 3 (allow ±3 days difference)
  };
  weights: {
    amount: number;        // e.g., 0.50 (50% weight)
    date: number;          // e.g., 0.30 (30% weight)
    counterparty?: number; // e.g., 0.15 (15% weight)
    description?: number;  // e.g., 0.05 (5% weight)
  };
  blocking: {
    enabled: boolean;
    fields: string[];  // e.g., ["date", "amount_bucket"]
  };
}

type DashboardState = "idle" | "loading" | "reconciling" | "reviewing" | "error";
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useEffect } from 'react';

export const ReconciliationDashboard: React.FC<ReconciliationDashboardProps> = ({
  unmatchedItems,
  suggestedMatches,
  matchedItems,
  config,
  onAcceptMatch,
  onRejectMatch,
  onManualMatch,
  onUnmatch,
  onUpdateConfig,
  onRerunReconciliation,
  onExportReport,
  loading = false,
  error = null,
  theme = "light",
  showConfidenceBreakdown = true,
  allowBatchOperations = true
}) => {
  // State
  const [state, setState] = useState<DashboardState>("idle");
  const [selectedTab, setSelectedTab] = useState<"unmatched" | "suggested" | "matched">("suggested");
  const [searchQuery, setSearchQuery] = useState("");
  const [selectedItems, setSelectedItems] = useState<Set<string>>(new Set());
  const [showConfigDialog, setShowConfigDialog] = useState(false);
  const [processingItems, setProcessingItems] = useState<Set<string>>(new Set());

  // Calculate progress
  const progress = useMemo(() => {
    const totalItems = unmatchedItems.total_source_1 + unmatchedItems.total_source_2;
    const matchedCount = matchedItems.length * 2;  // Each match covers 2 items
    const matchedPercent = totalItems > 0 ? (matchedCount / totalItems) * 100 : 0;

    return {
      totalItems,
      matchedCount,
      unmatchedCount: unmatchedItems.unmatched_source_1 + unmatchedItems.unmatched_source_2,
      suggestedCount: suggestedMatches.length,
      matchedPercent: Math.round(matchedPercent)
    };
  }, [unmatchedItems, matchedItems, suggestedMatches]);

  // Filter suggested matches by confidence tier
  const matchesByTier = useMemo(() => {
    return {
      high: suggestedMatches.filter(m => m.similarity_score >= config.thresholds.auto_link),
      medium: suggestedMatches.filter(
        m => m.similarity_score >= config.thresholds.auto_suggest && m.similarity_score < config.thresholds.auto_link
      ),
      low: suggestedMatches.filter(
        m => m.similarity_score >= config.thresholds.manual && m.similarity_score < config.thresholds.auto_suggest
      )
    };
  }, [suggestedMatches, config.thresholds]);

  // Handle accept match
  const handleAcceptMatch = async (candidateId: string) => {
    setProcessingItems(prev => new Set(prev).add(candidateId));
    try {
      await onAcceptMatch(candidateId);
    } catch (err) {
      console.error("Failed to accept match:", err);
    } finally {
      setProcessingItems(prev => {
        const next = new Set(prev);
        next.delete(candidateId);
        return next;
      });
    }
  };

  // Handle reject match
  const handleRejectMatch = async (candidateId: string, reason?: string) => {
    setProcessingItems(prev => new Set(prev).add(candidateId));
    try {
      await onRejectMatch(candidateId, reason);
    } catch (err) {
      console.error("Failed to reject match:", err);
    } finally {
      setProcessingItems(prev => {
        const next = new Set(prev);
        next.delete(candidateId);
        return next;
      });
    }
  };

  // Handle batch accept all high-confidence
  const handleBatchAcceptHighConfidence = async () => {
    setState("reconciling");
    try {
      for (const match of matchesByTier.high) {
        await onAcceptMatch(match.candidate_id);
      }
    } finally {
      setState("idle");
    }
  };

  // Handle rerun reconciliation
  const handleRerunReconciliation = async () => {
    setState("reconciling");
    try {
      await onRerunReconciliation();
    } finally {
      setState("idle");
    }
  };

  return (
    <div className={`reconciliation-dashboard theme-${theme}`}>
      {/* Header */}
      <div className="dashboard-header">
        <div className="header-content">
          <h2>Reconciliation Dashboard</h2>
          <div className="header-stats">
            <span className="stat">
              {progress.matchedCount}/{progress.totalItems} items matched
            </span>
            <span className="stat-separator">•</span>
            <span className="stat">
              {progress.matchedPercent}% complete
            </span>
          </div>
        </div>

        <div className="header-actions">
          <button
            className="config-button"
            onClick={() => setShowConfigDialog(true)}
          >
            ⚙️ Configure
          </button>
          <button
            className="rerun-button"
            onClick={handleRerunReconciliation}
            disabled={state === "reconciling"}
          >
            🔄 Re-run Reconciliation
          </button>
          <button
            className="export-button"
            onClick={onExportReport}
          >
            📊 Export Report
          </button>
        </div>
      </div>

      {/* Progress Bar */}
      <div className="progress-section">
        <div className="progress-bar-container">
          <div
            className="progress-bar-fill"
            style={{ width: `${progress.matchedPercent}%` }}
          />
        </div>
        <div className="progress-legend">
          <span className="legend-item matched">
            Matched: {matchedItems.length}
          </span>
          <span className="legend-item suggested">
            Suggested: {progress.suggestedCount}
          </span>
          <span className="legend-item unmatched">
            Unmatched: {progress.unmatchedCount}
          </span>
        </div>
      </div>

      {/* Batch Operations (if enabled) */}
      {allowBatchOperations && matchesByTier.high.length > 0 && (
        <div className="batch-operations">
          <div className="batch-message">
            {matchesByTier.high.length} high-confidence matches (≥{Math.round(config.thresholds.auto_link * 100)}%)
            ready to accept
          </div>
          <button
            className="batch-accept-button"
            onClick={handleBatchAcceptHighConfidence}
            disabled={state === "reconciling"}
          >
            Accept All High-Confidence Matches
          </button>
        </div>
      )}

      {/* Tabs */}
      <div className="dashboard-tabs">
        <button
          className={`tab ${selectedTab === "suggested" ? "active" : ""}`}
          onClick={() => setSelectedTab("suggested")}
        >
          Suggested Matches ({progress.suggestedCount})
        </button>
        <button
          className={`tab ${selectedTab === "unmatched" ? "active" : ""}`}
          onClick={() => setSelectedTab("unmatched")}
        >
          Unmatched ({progress.unmatchedCount})
        </button>
        <button
          className={`tab ${selectedTab === "matched" ? "active" : ""}`}
          onClick={() => setSelectedTab("matched")}
        >
          Matched ({matchedItems.length})
        </button>
      </div>

      {/* Tab Content */}
      <div className="dashboard-content">
        {selectedTab === "suggested" && (
          <SuggestedMatchesTab
            matches={suggestedMatches}
            matchesByTier={matchesByTier}
            config={config}
            onAcceptMatch={handleAcceptMatch}
            onRejectMatch={handleRejectMatch}
            processingItems={processingItems}
            showConfidenceBreakdown={showConfidenceBreakdown}
          />
        )}

        {selectedTab === "unmatched" && (
          <UnmatchedItemsTab
            unmatchedItems={unmatchedItems}
            onManualMatch={onManualMatch}
          />
        )}

        {selectedTab === "matched" && (
          <MatchedItemsTab
            matchedItems={matchedItems}
            onUnmatch={onUnmatch}
          />
        )}
      </div>

      {/* Config Dialog */}
      {showConfigDialog && (
        <ConfigDialog
          config={config}
          onSave={onUpdateConfig}
          onClose={() => setShowConfigDialog(false)}
        />
      )}

      {/* Loading Overlay */}
      {state === "reconciling" && (
        <div className="loading-overlay">
          <div className="spinner" />
          <div className="loading-text">Running reconciliation...</div>
        </div>
      )}

      {/* Error Banner */}
      {error && (
        <div className="error-banner">
          <span className="error-icon">⚠️</span>
          <span className="error-text">{error}</span>
        </div>
      )}
    </div>
  );
};

// Suggested Matches Tab Component
const SuggestedMatchesTab: React.FC<{
  matches: MatchCandidate[];
  matchesByTier: { high: MatchCandidate[]; medium: MatchCandidate[]; low: MatchCandidate[] };
  config: ReconciliationConfig;
  onAcceptMatch: (candidateId: string) => void;
  onRejectMatch: (candidateId: string, reason?: string) => void;
  processingItems: Set<string>;
  showConfidenceBreakdown: boolean;
}> = ({
  matches,
  matchesByTier,
  config,
  onAcceptMatch,
  onRejectMatch,
  processingItems,
  showConfidenceBreakdown
}) => {
  if (matches.length === 0) {
    return (
      <div className="empty-state">
        <div className="empty-icon">✓</div>
        <div className="empty-text">No suggested matches</div>
        <div className="empty-hint">
          All items are either matched or below the suggestion threshold
        </div>
      </div>
    );
  }

  return (
    <div className="suggested-matches-tab">
      {/* High Confidence Section */}
      {matchesByTier.high.length > 0 && (
        <div className="confidence-tier high">
          <div className="tier-header">
            <span className="tier-badge high">High Confidence</span>
            <span className="tier-count">
              {matchesByTier.high.length} matches ≥ {Math.round(config.thresholds.auto_link * 100)}%
            </span>
          </div>
          <div className="matches-list">
            {matchesByTier.high.map(match => (
              <MatchCandidateCard
                key={match.candidate_id}
                match={match}
                tier="high"
                onAccept={() => onAcceptMatch(match.candidate_id)}
                onReject={() => onRejectMatch(match.candidate_id)}
                processing={processingItems.has(match.candidate_id)}
                showConfidenceBreakdown={showConfidenceBreakdown}
              />
            ))}
          </div>
        </div>
      )}

      {/* Medium Confidence Section */}
      {matchesByTier.medium.length > 0 && (
        <div className="confidence-tier medium">
          <div className="tier-header">
            <span className="tier-badge medium">Medium Confidence</span>
            <span className="tier-count">
              {matchesByTier.medium.length} matches {Math.round(config.thresholds.auto_suggest * 100)}%-
              {Math.round(config.thresholds.auto_link * 100 - 1)}%
            </span>
          </div>
          <div className="matches-list">
            {matchesByTier.medium.map(match => (
              <MatchCandidateCard
                key={match.candidate_id}
                match={match}
                tier="medium"
                onAccept={() => onAcceptMatch(match.candidate_id)}
                onReject={() => onRejectMatch(match.candidate_id)}
                processing={processingItems.has(match.candidate_id)}
                showConfidenceBreakdown={showConfidenceBreakdown}
              />
            ))}
          </div>
        </div>
      )}

      {/* Low Confidence Section */}
      {matchesByTier.low.length > 0 && (
        <div className="confidence-tier low">
          <div className="tier-header">
            <span className="tier-badge low">Low Confidence</span>
            <span className="tier-count">
              {matchesByTier.low.length} matches {Math.round(config.thresholds.manual * 100)}%-
              {Math.round(config.thresholds.auto_suggest * 100 - 1)}%
            </span>
          </div>
          <div className="matches-list">
            {matchesByTier.low.map(match => (
              <MatchCandidateCard
                key={match.candidate_id}
                match={match}
                tier="low"
                onAccept={() => onAcceptMatch(match.candidate_id)}
                onReject={() => onRejectMatch(match.candidate_id)}
                processing={processingItems.has(match.candidate_id)}
                showConfidenceBreakdown={showConfidenceBreakdown}
              />
            ))}
          </div>
        </div>
      )}
    </div>
  );
};

// Match Candidate Card Component
const MatchCandidateCard: React.FC<{
  match: MatchCandidate;
  tier: "high" | "medium" | "low";
  onAccept: () => void;
  onReject: () => void;
  processing: boolean;
  showConfidenceBreakdown: boolean;
}> = ({ match, tier, onAccept, onReject, processing, showConfidenceBreakdown }) => {
  const [showDetails, setShowDetails] = useState(false);

  return (
    <div className={`match-candidate-card tier-${tier}`}>
      <div className="card-header">
        <div className="confidence-indicator">
          <div className="confidence-score">{Math.round(match.similarity_score * 100)}%</div>
          <div className="confidence-bar">
            <div
              className={`confidence-fill tier-${tier}`}
              style={{ width: `${match.similarity_score * 100}%` }}
            />
          </div>
        </div>
        <button
          className="details-toggle"
          onClick={() => setShowDetails(!showDetails)}
        >
          {showDetails ? "Hide Details ▴" : "Show Details ▾"}
        </button>
      </div>

      <div className="card-body">
        <div className="match-items">
          <div className="source-item source-1">
            <div className="source-label">Source 1</div>
            <div className="item-text">{match.source_1_item.display_text}</div>
            {match.source_1_item.amount !== undefined && (
              <div className="item-amount">${match.source_1_item.amount.toFixed(2)}</div>
            )}
            {match.source_1_item.date && (
              <div className="item-date">{match.source_1_item.date}</div>
            )}
          </div>

          <div className="match-arrow">↔</div>

          <div className="source-item source-2">
            <div className="source-label">Source 2</div>
            <div className="item-text">{match.source_2_item.display_text}</div>
            {match.source_2_item.amount !== undefined && (
              <div className="item-amount">${match.source_2_item.amount.toFixed(2)}</div>
            )}
            {match.source_2_item.date && (
              <div className="item-date">{match.source_2_item.date}</div>
            )}
          </div>
        </div>

        {showDetails && showConfidenceBreakdown && (
          <div className="feature-scores">
            <div className="feature-scores-header">Confidence Breakdown:</div>
            {Object.entries(match.feature_scores).map(([feature, score]) => (
              <div key={feature} className="feature-score-row">
                <span className="feature-name">{feature.replace(/_/g, " ")}</span>
                <div className="feature-score-bar">
                  <div
                    className="feature-score-fill"
                    style={{ width: `${score * 100}%` }}
                  />
                </div>
                <span className="feature-score-value">{Math.round(score * 100)}%</span>
              </div>
            ))}
          </div>
        )}
      </div>

      <div className="card-footer">
        <button
          className="reject-button"
          onClick={onReject}
          disabled={processing}
        >
          Reject
        </button>
        <button
          className="accept-button"
          onClick={onAccept}
          disabled={processing}
        >
          {processing ? "Accepting..." : "Accept Match"}
        </button>
      </div>
    </div>
  );
};
```

---

## Visual Wireframes

### Main Dashboard View (Suggested Matches Tab)

```
┌────────────────────────────────────────────────────────────────────────┐
│ Reconciliation Dashboard                                               │
│ 142/200 items matched • 71% complete                                   │
│                                              [⚙️ Configure] [🔄 Re-run] │
├────────────────────────────────────────────────────────────────────────┤
│ Progress: ████████████████████████████████░░░░░░░░░░░░ 71%            │
│ Matched: 71  Suggested: 25  Unmatched: 33                             │
├────────────────────────────────────────────────────────────────────────┤
│ ℹ️ 15 high-confidence matches (≥95%) ready to accept                  │
│                              [Accept All High-Confidence Matches]      │
├────────────────────────────────────────────────────────────────────────┤
│ [Suggested Matches (25)] [Unmatched (33)] [Matched (71)]              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ HIGH CONFIDENCE • 15 matches ≥ 95%                                    │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ 98% ████████████████████▓                         [Show Details ▾]│  │
│ │                                                                  │  │
│ │ Source 1                   ↔                   Source 2          │  │
│ │ Payment to Vendor A            Bank Transfer - Vendor A         │  │
│ │ $1,234.56 • Oct 15, 2024      $1,234.56 • Oct 15, 2024         │  │
│ │                                                                  │  │
│ │                                   [Reject]  [Accept Match]      │  │
│ └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ 96% ████████████████████▒                         [Show Details ▾]│  │
│ │                                                                  │  │
│ │ Source 1                   ↔                   Source 2          │  │
│ │ Invoice #12345                 Payment Received - INV12345      │  │
│ │ $567.89 • Oct 16, 2024        $567.89 • Oct 16, 2024           │  │
│ │                                                                  │  │
│ │                                   [Reject]  [Accept Match]      │  │
│ └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│ MEDIUM CONFIDENCE • 8 matches 70%-94%                                 │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ 85% ████████████████▓░░░░░░                       [Show Details ▾]│  │
│ │                                                                  │  │
│ │ Source 1                   ↔                   Source 2          │  │
│ │ Utility Payment                Electricity Bill Payment         │  │
│ │ $123.45 • Oct 14, 2024        $123.00 • Oct 15, 2024           │  │
│ │                                                                  │  │
│ │                                   [Reject]  [Accept Match]      │  │
│ └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│ LOW CONFIDENCE • 2 matches 50%-69%                                    │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ 62% ████████░░░░░░░░░░░░░░                       [Show Details ▾]│  │
│ │                                                                  │  │
│ │ Source 1                   ↔                   Source 2          │  │
│ │ Misc Expense                   Office Supplies                  │  │
│ │ $45.00 • Oct 10, 2024         $52.00 • Oct 12, 2024            │  │
│ │                                                                  │  │
│ │                                   [Reject]  [Accept Match]      │  │
│ └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

### Match Candidate with Confidence Breakdown

```
┌──────────────────────────────────────────────────────────────────┐
│ 85% ████████████████▓░░░░░░                      [Hide Details ▴]│
│                                                                  │
│ Source 1                   ↔                   Source 2          │
│ Utility Payment                Electricity Bill Payment         │
│ $123.45 • Oct 14, 2024        $123.00 • Oct 15, 2024           │
│                                                                  │
│ Confidence Breakdown:                                            │
│ amount score      ████████████████░░░░  90%                     │
│ date score        ██████████████▓░░░░░  80%                     │
│ description score ████████████████████  95%                     │
│ counterparty score ████████▓░░░░░░░░░░  65%                     │
│                                                                  │
│                                   [Reject]  [Accept Match]      │
└──────────────────────────────────────────────────────────────────┘
```

### Unmatched Items Tab

```
┌────────────────────────────────────────────────────────────────────────┐
│ [Suggested Matches (25)] [Unmatched (33)] [Matched (71)]              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ Source 1 Unmatched (18)                                                │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ □ Payment to Vendor B            $2,345.67 • Oct 18, 2024       │  │
│ │ □ Refund from Client C           $456.78 • Oct 19, 2024         │  │
│ │ □ Bank Fee                       $15.00 • Oct 20, 2024          │  │
│ └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│ Source 2 Unmatched (15)                                                │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ □ Wire Transfer Received         $2,345.67 • Oct 18, 2024       │  │
│ │ □ Customer Refund                $456.78 • Oct 19, 2024         │  │
│ │ □ Monthly Service Charge         $15.00 • Oct 20, 2024          │  │
│ └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│ [Manual Match Selected Items]                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Matched Items Tab

```
┌────────────────────────────────────────────────────────────────────────┐
│ [Suggested Matches (25)] [Unmatched (33)] [Matched (71)]              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ ✓ Payment to Vendor A ↔ Bank Transfer - Vendor A                │  │
│ │ $1,234.56 • Oct 15, 2024                                         │  │
│ │ Confidence: 98% • Method: Auto • Matched at: Oct 25, 2024 2:30 PM│ │
│ │                                                        [Unmatch] │  │
│ └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │ ✓ Invoice #12345 ↔ Payment Received - INV12345                  │  │
│ │ $567.89 • Oct 16, 2024                                           │  │
│ │ Confidence: 96% • Method: Auto • Matched at: Oct 25, 2024 2:31 PM│ │
│ │                                                        [Unmatch] │  │
│ └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Multi-Domain Examples

### Finance: Bank Reconciliation

```tsx
<ReconciliationDashboard
  unmatchedItems={{
    source_1: [
      {
        item_id: "bank_txn_001",
        display_text: "Payment to Vendor A",
        amount: 1234.56,
        date: "2024-10-15",
        metadata: { account: "Checking", type: "debit" }
      }
    ],
    source_2: [
      {
        item_id: "ledger_001",
        display_text: "Accounts Payable - Vendor A",
        amount: 1234.56,
        date: "2024-10-15",
        metadata: { category: "Operating Expenses" }
      }
    ],
    total_source_1: 100,
    total_source_2: 98,
    unmatched_source_1: 18,
    unmatched_source_2: 15
  }}
  suggestedMatches={[
    {
      candidate_id: "candidate_001",
      source_1_item: { item_id: "bank_txn_001", display_text: "Payment to Vendor A", amount: 1234.56, date: "2024-10-15" },
      source_2_item: { item_id: "ledger_001", display_text: "Accounts Payable - Vendor A", amount: 1234.56, date: "2024-10-15" },
      similarity_score: 0.98,
      feature_scores: {
        amount_score: 1.0,
        date_score: 1.0,
        description_score: 0.95
      },
      decision: "auto_link",
      match_details: {}
    }
  ]}
  matchedItems={[]}
  config={{
    thresholds: { auto_link: 0.95, auto_suggest: 0.70, manual: 0.50 },
    tolerances: { amount_percent: 0.05, date_days: 3 },
    weights: { amount: 0.5, date: 0.3, description: 0.2 },
    blocking: { enabled: true, fields: ["date", "amount_bucket"] }
  }}
  onAcceptMatch={async (candidateId) => console.log("Accepted:", candidateId)}
  onRejectMatch={async (candidateId) => console.log("Rejected:", candidateId)}
  onManualMatch={async (s1, s2) => console.log("Manual match:", s1, s2)}
  onUnmatch={async (matchId) => console.log("Unmatched:", matchId)}
  onUpdateConfig={async (config) => console.log("Updated config:", config)}
  onRerunReconciliation={async () => console.log("Re-running reconciliation")}
  onExportReport={async () => new Blob()}
/>
```

### Healthcare: Claim-Payment Matching

```tsx
<ReconciliationDashboard
  unmatchedItems={{
    source_1: [
      {
        item_id: "claim_001",
        display_text: "Dr. Smith - Annual Checkup",
        amount: 350.00,
        date: "2024-10-10",
        metadata: { claim_id: "CLM12345", patient: "John Doe", cpt_code: "99213" }
      }
    ],
    source_2: [
      {
        item_id: "payment_001",
        display_text: "Insurance Payment - CLM12345",
        amount: 280.00,
        date: "2024-10-15",
        metadata: { payment_id: "PAY98765", coverage_rate: 0.80 }
      }
    ],
    total_source_1: 50,
    total_source_2: 45,
    unmatched_source_1: 8,
    unmatched_source_2: 5
  }}
  config={{
    thresholds: { auto_link: 0.90, auto_suggest: 0.70, manual: 0.50 },
    tolerances: { amount_percent: 0.20, date_days: 7 },  // Higher tolerance for healthcare
    weights: { amount: 0.40, date: 0.20, claim_id: 0.30, patient: 0.10 },
    blocking: { enabled: true, fields: ["claim_id"] }
  }}
  // ... other props
/>
```

---

## Related Components

- **ReconciliationStore** (OL) - Data persistence for matches
- **ReconciliationEngine** (OL) - Generates match candidates
- **MatchReviewDialog** (IL) - Detailed match review UI
- **ManualMatchDialog** (IL) - Manual match creation UI

---

**Status:** ✅ Ready for implementation
