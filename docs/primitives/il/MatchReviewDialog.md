# MatchReviewDialog (IL Component)

## Definition

**MatchReviewDialog** is a modal dialog component for reviewing and accepting/rejecting suggested reconciliation matches. It provides side-by-side comparison of candidate items, confidence score breakdown by feature, visual diff highlighting, and detailed match metadata to help users make informed decisions.

**Problem it solves:**
- No detailed view of why a match was suggested (confidence breakdown hidden)
- Users uncertain whether to accept medium-confidence matches
- Cannot see field-level differences between matched items
- No explanation of how confidence score was calculated
- Missing context about match criteria (thresholds, tolerances used)
- Cannot add notes when accepting/rejecting matches

**Solution:**
- Side-by-side item comparison with field-level diff highlighting
- Confidence score breakdown showing individual feature contributions
- Visual indicators for exact matches vs approximate matches
- Expandable match details (thresholds, blocking key, date/amount tolerances)
- Accept with optional notes, reject with required reason
- Keyboard shortcuts (Enter to accept, Escape to cancel)
- Loading states for async operations
- Match history showing previous accept/reject decisions

---

## Interface Contract

```typescript
interface MatchReviewDialogProps {
  // Match candidate to review
  matchCandidate: MatchCandidate;
  isOpen: boolean;

  // Callbacks
  onAccept: (candidateId: string, notes?: string) => Promise<void>;
  onReject: (candidateId: string, reason: string) => Promise<void>;
  onCancel: () => void;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Display options
  theme?: "light" | "dark";
  showMatchDetails?: boolean;  // Show technical details (thresholds, etc.)
  highlightDifferences?: boolean;  // Highlight field differences
}

interface MatchCandidate {
  candidate_id: string;
  source_1_item: MatchableItem;
  source_2_item: MatchableItem;
  similarity_score: number;  // Overall confidence (0.0 to 1.0)
  feature_scores: Record<string, number>;  // Individual feature scores
  decision: "auto_link" | "suggest" | "manual" | "no_match";
  match_details: MatchDetails;
}

interface MatchableItem {
  item_id: string;
  display_text: string;
  amount?: number;
  date?: string;
  metadata: Record<string, any>;  // Domain-specific fields
}

interface MatchDetails {
  blocking_key?: string;
  thresholds?: {
    auto_link: number;
    auto_suggest: number;
    manual: number;
  };
  tolerances?: {
    amount_diff?: number;
    amount_percent?: number;
    date_diff_days?: number;
  };
  matched_features?: string[];  // Features that matched
  mismatched_features?: string[];  // Features that didn't match
}

type DialogState = "idle" | "comparing" | "accepting" | "rejecting" | "error";
```

---

## Component Structure

```tsx
import React, { useState, useEffect, useMemo } from 'react';

export const MatchReviewDialog: React.FC<MatchReviewDialogProps> = ({
  matchCandidate,
  isOpen,
  onAccept,
  onReject,
  onCancel,
  loading = false,
  error = null,
  theme = "light",
  showMatchDetails = true,
  highlightDifferences = true
}) => {
  const [state, setState] = useState<DialogState>("idle");
  const [acceptNotes, setAcceptNotes] = useState("");
  const [rejectReason, setRejectReason] = useState("");
  const [showFeatureBreakdown, setShowFeatureBreakdown] = useState(true);
  const [showTechnicalDetails, setShowTechnicalDetails] = useState(false);

  // Calculate confidence tier
  const confidenceTier = useMemo(() => {
    const score = matchCandidate.similarity_score;
    const thresholds = matchCandidate.match_details.thresholds;

    if (!thresholds) return "unknown";

    if (score >= thresholds.auto_link) return "high";
    if (score >= thresholds.auto_suggest) return "medium";
    if (score >= thresholds.manual) return "low";
    return "very_low";
  }, [matchCandidate]);

  // Calculate field differences
  const fieldDiffs = useMemo(() => {
    const s1 = matchCandidate.source_1_item;
    const s2 = matchCandidate.source_2_item;

    return {
      amount: {
        match: s1.amount === s2.amount,
        diff: s1.amount !== undefined && s2.amount !== undefined
          ? Math.abs(s1.amount - s2.amount)
          : null
      },
      date: {
        match: s1.date === s2.date,
        diff: s1.date && s2.date
          ? Math.abs(new Date(s1.date).getTime() - new Date(s2.date).getTime()) / (1000 * 60 * 60 * 24)
          : null
      },
      text: {
        match: s1.display_text === s2.display_text
      }
    };
  }, [matchCandidate]);

  // Handle accept
  const handleAccept = async () => {
    setState("accepting");
    try {
      await onAccept(matchCandidate.candidate_id, acceptNotes || undefined);
      onCancel();  // Close dialog on success
    } catch (err) {
      setState("error");
      console.error("Failed to accept match:", err);
    }
  };

  // Handle reject
  const handleReject = async () => {
    if (!rejectReason.trim()) {
      alert("Please provide a reason for rejection");
      return;
    }

    setState("rejecting");
    try {
      await onReject(matchCandidate.candidate_id, rejectReason);
      onCancel();  // Close dialog on success
    } catch (err) {
      setState("error");
      console.error("Failed to reject match:", err);
    }
  };

  // Keyboard shortcuts
  useEffect(() => {
    if (!isOpen) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Escape") {
        onCancel();
      } else if (e.key === "Enter" && e.metaKey) {
        // Cmd+Enter to accept
        handleAccept();
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [isOpen, onCancel, handleAccept]);

  if (!isOpen) return null;

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onCancel}>
      <div
        className="modal-content match-review-dialog"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="modal-header">
          <div className="header-content">
            <h3>Review Match Suggestion</h3>
            <div className="confidence-badge-container">
              <span className={`confidence-badge tier-${confidenceTier}`}>
                {Math.round(matchCandidate.similarity_score * 100)}% Confidence
              </span>
              {confidenceTier === "high" && <span className="tier-label">HIGH</span>}
              {confidenceTier === "medium" && <span className="tier-label">MEDIUM</span>}
              {confidenceTier === "low" && <span className="tier-label">LOW</span>}
            </div>
          </div>
          <button className="close-button" onClick={onCancel} aria-label="Close">
            ×
          </button>
        </div>

        {/* Body */}
        <div className="modal-body">
          {/* Side-by-Side Comparison */}
          <div className="comparison-section">
            <div className="comparison-header">Item Comparison</div>

            <div className="comparison-grid">
              {/* Source 1 */}
              <div className="comparison-item source-1">
                <div className="item-label">Source 1</div>
                <div className="item-content">
                  <div className={`item-text ${highlightDifferences && !fieldDiffs.text.match ? "highlight-diff" : ""}`}>
                    {matchCandidate.source_1_item.display_text}
                  </div>
                  {matchCandidate.source_1_item.amount !== undefined && (
                    <div className={`item-amount ${highlightDifferences && !fieldDiffs.amount.match ? "highlight-diff" : ""}`}>
                      ${matchCandidate.source_1_item.amount.toFixed(2)}
                    </div>
                  )}
                  {matchCandidate.source_1_item.date && (
                    <div className={`item-date ${highlightDifferences && !fieldDiffs.date.match ? "highlight-diff" : ""}`}>
                      {matchCandidate.source_1_item.date}
                    </div>
                  )}
                </div>
              </div>

              {/* Match Indicator */}
              <div className="match-indicator">
                <div className={`match-icon tier-${confidenceTier}`}>↔</div>
                <div className="match-score">
                  {Math.round(matchCandidate.similarity_score * 100)}%
                </div>
              </div>

              {/* Source 2 */}
              <div className="comparison-item source-2">
                <div className="item-label">Source 2</div>
                <div className="item-content">
                  <div className={`item-text ${highlightDifferences && !fieldDiffs.text.match ? "highlight-diff" : ""}`}>
                    {matchCandidate.source_2_item.display_text}
                  </div>
                  {matchCandidate.source_2_item.amount !== undefined && (
                    <div className={`item-amount ${highlightDifferences && !fieldDiffs.amount.match ? "highlight-diff" : ""}`}>
                      ${matchCandidate.source_2_item.amount.toFixed(2)}
                    </div>
                  )}
                  {matchCandidate.source_2_item.date && (
                    <div className={`item-date ${highlightDifferences && !fieldDiffs.date.match ? "highlight-diff" : ""}`}>
                      {matchCandidate.source_2_item.date}
                    </div>
                  )}
                </div>
              </div>
            </div>

            {/* Field Differences Summary */}
            {highlightDifferences && (
              <div className="differences-summary">
                {fieldDiffs.amount.diff !== null && fieldDiffs.amount.diff > 0 && (
                  <div className="diff-item">
                    <span className="diff-icon">⚠️</span>
                    <span className="diff-text">
                      Amount difference: ${fieldDiffs.amount.diff.toFixed(2)}
                    </span>
                  </div>
                )}
                {fieldDiffs.date.diff !== null && fieldDiffs.date.diff > 0 && (
                  <div className="diff-item">
                    <span className="diff-icon">⚠️</span>
                    <span className="diff-text">
                      Date difference: {Math.round(fieldDiffs.date.diff)} days
                    </span>
                  </div>
                )}
                {!fieldDiffs.text.match && (
                  <div className="diff-item">
                    <span className="diff-icon">ℹ️</span>
                    <span className="diff-text">
                      Descriptions differ
                    </span>
                  </div>
                )}
              </div>
            )}
          </div>

          {/* Confidence Breakdown */}
          <div className="confidence-breakdown-section">
            <div className="section-header">
              <span>Confidence Breakdown</span>
              <button
                className="toggle-button"
                onClick={() => setShowFeatureBreakdown(!showFeatureBreakdown)}
              >
                {showFeatureBreakdown ? "Hide ▴" : "Show ▾"}
              </button>
            </div>

            {showFeatureBreakdown && (
              <div className="feature-scores">
                {Object.entries(matchCandidate.feature_scores).map(([feature, score]) => (
                  <div key={feature} className="feature-score-row">
                    <span className="feature-name">
                      {feature.replace(/_/g, " ").replace(/\b\w/g, l => l.toUpperCase())}
                    </span>
                    <div className="feature-score-bar-container">
                      <div className="feature-score-bar">
                        <div
                          className={`feature-score-fill tier-${score >= 0.9 ? "high" : score >= 0.7 ? "medium" : "low"}`}
                          style={{ width: `${score * 100}%` }}
                        />
                      </div>
                      <span className="feature-score-value">
                        {Math.round(score * 100)}%
                      </span>
                    </div>
                  </div>
                ))}
              </div>
            )}
          </div>

          {/* Technical Details (Optional) */}
          {showMatchDetails && (
            <div className="technical-details-section">
              <div className="section-header">
                <span>Technical Details</span>
                <button
                  className="toggle-button"
                  onClick={() => setShowTechnicalDetails(!showTechnicalDetails)}
                >
                  {showTechnicalDetails ? "Hide ▴" : "Show ▾"}
                </button>
              </div>

              {showTechnicalDetails && (
                <div className="technical-details">
                  {matchCandidate.match_details.thresholds && (
                    <div className="detail-group">
                      <div className="detail-label">Thresholds:</div>
                      <div className="detail-value">
                        Auto-link ≥{Math.round(matchCandidate.match_details.thresholds.auto_link * 100)}%,
                        Suggest ≥{Math.round(matchCandidate.match_details.thresholds.auto_suggest * 100)}%,
                        Manual ≥{Math.round(matchCandidate.match_details.thresholds.manual * 100)}%
                      </div>
                    </div>
                  )}

                  {matchCandidate.match_details.tolerances && (
                    <div className="detail-group">
                      <div className="detail-label">Tolerances:</div>
                      <div className="detail-value">
                        {matchCandidate.match_details.tolerances.amount_percent &&
                          `Amount ±${matchCandidate.match_details.tolerances.amount_percent * 100}%, `}
                        {matchCandidate.match_details.tolerances.date_diff_days &&
                          `Date ±${matchCandidate.match_details.tolerances.date_diff_days} days`}
                      </div>
                    </div>
                  )}

                  {matchCandidate.match_details.blocking_key && (
                    <div className="detail-group">
                      <div className="detail-label">Blocking Key:</div>
                      <div className="detail-value">{matchCandidate.match_details.blocking_key}</div>
                    </div>
                  )}
                </div>
              )}
            </div>
          )}

          {/* Accept Notes */}
          <div className="notes-section">
            <label className="notes-label" htmlFor="accept-notes">
              Notes (optional)
            </label>
            <textarea
              id="accept-notes"
              className="notes-textarea"
              placeholder="Add optional notes about this match..."
              value={acceptNotes}
              onChange={(e) => setAcceptNotes(e.target.value)}
              rows={2}
            />
          </div>

          {/* Reject Reason (shown when rejecting) */}
          {state === "rejecting" && (
            <div className="reject-section">
              <label className="reject-label" htmlFor="reject-reason">
                Reason for Rejection *
              </label>
              <textarea
                id="reject-reason"
                className="reject-textarea"
                placeholder="Explain why this match is incorrect..."
                value={rejectReason}
                onChange={(e) => setRejectReason(e.target.value)}
                rows={3}
                required
                autoFocus
              />
            </div>
          )}

          {/* Error Message */}
          {error && (
            <div className="error-message">
              <span className="error-icon">⚠️</span>
              <span className="error-text">{error}</span>
            </div>
          )}
        </div>

        {/* Footer */}
        <div className="modal-footer">
          <button
            className="cancel-button"
            onClick={onCancel}
            disabled={state === "accepting" || state === "rejecting"}
          >
            Cancel
          </button>
          <button
            className="reject-button"
            onClick={handleReject}
            disabled={state === "accepting" || state === "rejecting"}
          >
            {state === "rejecting" ? "Rejecting..." : "Reject Match"}
          </button>
          <button
            className="accept-button"
            onClick={handleAccept}
            disabled={state === "accepting" || state === "rejecting"}
          >
            {state === "accepting" ? "Accepting..." : "Accept Match"}
          </button>
        </div>
      </div>
    </div>
  );
};
```

---

## Visual Wireframes

### Initial State (High Confidence)

```
┌────────────────────────────────────────────────────────────┐
│ Review Match Suggestion                                [×] │
│ [98% Confidence] HIGH                                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Item Comparison                                            │
│ ┌────────────────┐         ┌────────────────┐            │
│ │ Source 1       │    ↔    │ Source 2       │            │
│ │                │   98%   │                │            │
│ │ Payment to     │         │ Bank Transfer  │            │
│ │ Vendor A       │         │ - Vendor A     │            │
│ │                │         │                │            │
│ │ $1,234.56      │         │ $1,234.56      │            │
│ │ Oct 15, 2024   │         │ Oct 15, 2024   │            │
│ └────────────────┘         └────────────────┘            │
│                                                            │
│ Confidence Breakdown                        [Hide ▴]      │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Amount Score      ████████████████████  100%       │    │
│ │ Date Score        ████████████████████  100%       │    │
│ │ Description Score ██████████████████▓░   95%       │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ Technical Details                           [Show ▾]      │
│                                                            │
│ Notes (optional)                                           │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Add optional notes about this match...             │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│              [Cancel]  [Reject Match]  [Accept Match]     │
└────────────────────────────────────────────────────────────┘
```

### Medium Confidence with Differences

```
┌────────────────────────────────────────────────────────────┐
│ Review Match Suggestion                                [×] │
│ [85% Confidence] MEDIUM                                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Item Comparison                                            │
│ ┌────────────────┐         ┌────────────────┐            │
│ │ Source 1       │    ↔    │ Source 2       │            │
│ │                │   85%   │                │            │
│ │ Utility        │         │ Electricity    │            │
│ │ Payment        │         │ Bill Payment   │            │
│ │                │         │                │            │
│ │ $123.45        │         │ $123.00        │            │
│ │ Oct 14, 2024   │         │ Oct 15, 2024   │            │
│ └────────────────┘         └────────────────┘            │
│                                                            │
│ ⚠️ Amount difference: $0.45                               │
│ ⚠️ Date difference: 1 days                                │
│                                                            │
│ Confidence Breakdown                        [Hide ▴]      │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Amount Score      ████████████████░░░░   90%       │    │
│ │ Date Score        ████████████████░░░░   80%       │    │
│ │ Description Score ████████████████████   95%       │    │
│ │ Counterparty Score ████████▓░░░░░░░░░░   65%       │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ Notes (optional)                                           │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Utility provider bills one day in advance         │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│              [Cancel]  [Reject Match]  [Accept Match]     │
└────────────────────────────────────────────────────────────┘
```

### Rejecting State

```
┌────────────────────────────────────────────────────────────┐
│ Review Match Suggestion                                [×] │
│ [62% Confidence] LOW                                       │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Item Comparison                                            │
│ ┌────────────────┐         ┌────────────────┐            │
│ │ Source 1       │    ↔    │ Source 2       │            │
│ │                │   62%   │                │            │
│ │ Misc Expense   │         │ Office         │            │
│ │                │         │ Supplies       │            │
│ │                │         │                │            │
│ │ $45.00         │         │ $52.00         │            │
│ │ Oct 10, 2024   │         │ Oct 12, 2024   │            │
│ └────────────────┘         └────────────────┘            │
│                                                            │
│ ⚠️ Amount difference: $7.00                               │
│ ⚠️ Date difference: 2 days                                │
│ ℹ️ Descriptions differ                                    │
│                                                            │
│ Reason for Rejection *                                     │
│ ┌────────────────────────────────────────────────────┐    │
│ │ These are different transactions - misc expense    │    │
│ │ was for parking, office supplies was for pens      │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│              [Cancel]  [Rejecting...]  [Accept Match]     │
└────────────────────────────────────────────────────────────┘
```

---

## Multi-Domain Examples

### Finance: Bank Reconciliation

```tsx
<MatchReviewDialog
  matchCandidate={{
    candidate_id: "candidate_001",
    source_1_item: {
      item_id: "bank_txn_001",
      display_text: "Payment to Vendor A",
      amount: 1234.56,
      date: "2024-10-15",
      metadata: { account: "Checking", reference: "PAY001" }
    },
    source_2_item: {
      item_id: "ledger_001",
      display_text: "Accounts Payable - Vendor A",
      amount: 1234.56,
      date: "2024-10-15",
      metadata: { category: "Operating Expenses", invoice: "INV-001" }
    },
    similarity_score: 0.98,
    feature_scores: {
      amount_score: 1.0,
      date_score: 1.0,
      description_score: 0.95
    },
    decision: "auto_link",
    match_details: {
      thresholds: { auto_link: 0.95, auto_suggest: 0.70, manual: 0.50 },
      tolerances: { amount_percent: 0.05, date_diff_days: 3 },
      blocking_key: "2024-10-15_1234.56"
    }
  }}
  isOpen={true}
  onAccept={async (id, notes) => console.log("Accepted:", id, notes)}
  onReject={async (id, reason) => console.log("Rejected:", id, reason)}
  onCancel={() => console.log("Cancelled")}
/>
```

### Healthcare: Claim-Payment Review

```tsx
<MatchReviewDialog
  matchCandidate={{
    candidate_id: "candidate_002",
    source_1_item: {
      item_id: "claim_001",
      display_text: "Dr. Smith - Annual Checkup",
      amount: 350.00,
      date: "2024-10-10",
      metadata: { claim_id: "CLM12345", patient: "John Doe", cpt_code: "99213" }
    },
    source_2_item: {
      item_id: "payment_001",
      display_text: "Insurance Payment - CLM12345",
      amount: 280.00,
      date: "2024-10-15",
      metadata: { payment_id: "PAY98765", coverage_rate: 0.80 }
    },
    similarity_score: 0.95,
    feature_scores: {
      amount_score: 0.80,  // 80% match ($280 vs $350)
      date_score: 0.90,
      claim_id_score: 1.0,
      patient_score: 1.0
    },
    decision: "auto_link",
    match_details: {
      thresholds: { auto_link: 0.90, auto_suggest: 0.70, manual: 0.50 },
      tolerances: { amount_percent: 0.20, date_diff_days: 7 }
    }
  }}
  // ... other props
/>
```

---

## Related Components

- **ReconciliationDashboard** (IL) - Parent component showing all matches
- **ManualMatchDialog** (IL) - Manual match creation
- **ReconciliationStore** (OL) - Data persistence
- **MatchScorer** (OL) - Calculates confidence scores

---

**Status:** ✅ Ready for implementation
