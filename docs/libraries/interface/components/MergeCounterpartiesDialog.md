# MergeCounterpartiesDialog (IL Component)

## Definition

**MergeCounterpartiesDialog** is a modal dialog for merging duplicate counterparties into a single canonical entity. It provides a guided workflow with preview, confirmation, and progress indication to safely consolidate counterparty records and their associated transactions.

**Problem it solves:**
- Duplicate counterparties created with slight name variations (e.g., "Amazon", "AMZN", "Amazon.com")
- No safe way to merge counterparties without losing data
- No preview of what will happen after merge
- Users unsure which counterparty to keep as primary
- No way to see transaction impact before merging
- Merge operation is irreversible but users don't know consequences

**Solution:**
- Step-by-step merge wizard with clear progression
- Select primary counterparty (keeps its canonical name)
- Preview combined aliases and transaction counts
- Show which transactions will be reassigned
- Confirmation step with irreversible warning
- Progress indicator during merge operation
- Success/error feedback after merge completes

---

## Interface Contract

```typescript
interface MergeCounterpartiesDialogProps {
  // Data
  counterparties: Counterparty[];  // Counterparties to merge (2+)
  transactionCounts?: Record<string, number>;  // Transaction count per counterparty

  // Callbacks
  onClose: () => void;
  onMerge: (primaryId: string, mergeIds: string[]) => Promise<void>;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Customization
  theme?: "light" | "dark";
}

interface Counterparty {
  counterparty_id: string;
  canonical_name: string;
  aliases: string[];
  type: CounterpartyType;
  notes?: string;
  tags?: string[];
  created_at: string;
  updated_at: string;
}

type CounterpartyType = "merchant" | "person" | "business" | "government";

interface MergePreview {
  primary: Counterparty;
  merging: Counterparty[];
  resulting_canonical_name: string;
  resulting_aliases: string[];  // Combined aliases from all counterparties
  total_transactions: number;  // Sum of all transaction counts
  affected_counterparty_ids: string[];  // IDs that will be removed
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo } from 'react';

export const MergeCounterpartiesDialog: React.FC<MergeCounterpartiesDialogProps> = ({
  counterparties,
  transactionCounts = {},
  onClose,
  onMerge,
  loading = false,
  error = null,
  theme = "light"
}) => {
  const [step, setStep] = useState<"select" | "preview" | "confirm" | "merging" | "complete">("select");
  const [primaryId, setPrimaryId] = useState<string>(counterparties[0]?.counterparty_id || "");
  const [mergeError, setMergeError] = useState<string | null>(error);
  const [mergeProgress, setMergeProgress] = useState(0);

  // Calculate merge preview
  const mergePreview = useMemo((): MergePreview => {
    const primary = counterparties.find(c => c.counterparty_id === primaryId);
    const merging = counterparties.filter(c => c.counterparty_id !== primaryId);

    if (!primary) {
      return {
        primary: counterparties[0],
        merging: counterparties.slice(1),
        resulting_canonical_name: counterparties[0]?.canonical_name || "",
        resulting_aliases: [],
        total_transactions: 0,
        affected_counterparty_ids: []
      };
    }

    // Combine all aliases (excluding canonical names)
    const allAliases = new Set<string>();

    // Add primary's aliases
    primary.aliases.forEach(alias => allAliases.add(alias));

    // Add merging counterparties' canonical names as aliases
    merging.forEach(c => {
      allAliases.add(c.canonical_name);
      c.aliases.forEach(alias => allAliases.add(alias));
    });

    // Remove primary's canonical name from aliases if present
    allAliases.delete(primary.canonical_name);

    // Calculate total transactions
    const totalTransactions = counterparties.reduce(
      (sum, c) => sum + (transactionCounts[c.counterparty_id] || 0),
      0
    );

    return {
      primary,
      merging,
      resulting_canonical_name: primary.canonical_name,
      resulting_aliases: Array.from(allAliases).sort(),
      total_transactions: totalTransactions,
      affected_counterparty_ids: merging.map(c => c.counterparty_id)
    };
  }, [counterparties, primaryId, transactionCounts]);

  // Handle merge execution
  const handleMerge = async () => {
    setStep("merging");
    setMergeError(null);
    setMergeProgress(0);

    try {
      // Simulate progress (in real app, this would come from backend)
      const progressInterval = setInterval(() => {
        setMergeProgress(prev => Math.min(prev + 10, 90));
      }, 200);

      await onMerge(primaryId, mergePreview.affected_counterparty_ids);

      clearInterval(progressInterval);
      setMergeProgress(100);
      setStep("complete");

      // Auto-close after success
      setTimeout(() => {
        onClose();
      }, 2000);
    } catch (err) {
      setMergeError(err instanceof Error ? err.message : "Failed to merge counterparties");
      setStep("confirm");  // Go back to confirm step
    }
  };

  // Render different steps
  const renderStep = () => {
    switch (step) {
      case "select":
        return (
          <SelectPrimaryStep
            counterparties={counterparties}
            transactionCounts={transactionCounts}
            primaryId={primaryId}
            onPrimarySelect={setPrimaryId}
            onNext={() => setStep("preview")}
            onCancel={onClose}
            theme={theme}
          />
        );

      case "preview":
        return (
          <PreviewStep
            preview={mergePreview}
            onBack={() => setStep("select")}
            onNext={() => setStep("confirm")}
            theme={theme}
          />
        );

      case "confirm":
        return (
          <ConfirmStep
            preview={mergePreview}
            error={mergeError}
            onBack={() => setStep("preview")}
            onConfirm={handleMerge}
            onCancel={onClose}
            theme={theme}
          />
        );

      case "merging":
        return (
          <MergingStep
            progress={mergeProgress}
            theme={theme}
          />
        );

      case "complete":
        return (
          <CompleteStep
            preview={mergePreview}
            theme={theme}
          />
        );
    }
  };

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content merge-dialog" onClick={(e) => e.stopPropagation()}>
        {/* Progress Indicator */}
        <div className="merge-progress-bar">
          <div className="progress-steps">
            <div className={`progress-step ${step === "select" ? "active" : "completed"}`}>
              <div className="step-number">1</div>
              <div className="step-label">Select Primary</div>
            </div>
            <div className={`progress-step ${step === "preview" ? "active" : step === "confirm" || step === "merging" || step === "complete" ? "completed" : ""}`}>
              <div className="step-number">2</div>
              <div className="step-label">Preview</div>
            </div>
            <div className={`progress-step ${step === "confirm" || step === "merging" ? "active" : step === "complete" ? "completed" : ""}`}>
              <div className="step-number">3</div>
              <div className="step-label">Confirm</div>
            </div>
          </div>
        </div>

        {/* Step Content */}
        {renderStep()}
      </div>
    </div>
  );
};

// Step 1: Select Primary Counterparty
const SelectPrimaryStep: React.FC<{
  counterparties: Counterparty[];
  transactionCounts: Record<string, number>;
  primaryId: string;
  onPrimarySelect: (id: string) => void;
  onNext: () => void;
  onCancel: () => void;
  theme: "light" | "dark";
}> = ({ counterparties, transactionCounts, primaryId, onPrimarySelect, onNext, onCancel, theme }) => {
  return (
    <>
      <div className="modal-header">
        <h3>Select Primary Counterparty</h3>
        <button className="close-button" onClick={onCancel}>×</button>
      </div>

      <div className="modal-body">
        <div className="instruction-text">
          Choose which counterparty to keep as the primary entry. Its canonical name will be used,
          and all other names will become aliases.
        </div>

        <div className="counterparty-selection-list">
          {counterparties.map(counterparty => {
            const txCount = transactionCounts[counterparty.counterparty_id] || 0;
            const isSelected = counterparty.counterparty_id === primaryId;

            return (
              <div
                key={counterparty.counterparty_id}
                className={`selection-card ${isSelected ? 'selected' : ''}`}
                onClick={() => onPrimarySelect(counterparty.counterparty_id)}
              >
                <div className="selection-radio">
                  <input
                    type="radio"
                    checked={isSelected}
                    onChange={() => onPrimarySelect(counterparty.counterparty_id)}
                    aria-label={`Select ${counterparty.canonical_name} as primary`}
                  />
                </div>

                <div className="counterparty-icon">
                  {getDefaultIcon(counterparty.type)}
                </div>

                <div className="counterparty-details">
                  <div className="counterparty-name">{counterparty.canonical_name}</div>
                  <div className="counterparty-meta">
                    <span className="type-badge">{formatCounterpartyType(counterparty.type)}</span>
                    <span className="separator">•</span>
                    <span className="transaction-count">{txCount} transactions</span>
                  </div>
                  {counterparty.aliases.length > 0 && (
                    <div className="aliases-preview">
                      Aliases: {counterparty.aliases.slice(0, 3).join(', ')}
                      {counterparty.aliases.length > 3 && ` +${counterparty.aliases.length - 3} more`}
                    </div>
                  )}
                </div>

                {isSelected && (
                  <div className="selected-badge">PRIMARY</div>
                )}
              </div>
            );
          })}
        </div>

        <div className="recommendation-box">
          <div className="recommendation-icon">💡</div>
          <div className="recommendation-text">
            <strong>Recommendation:</strong> Select the counterparty with the most transactions
            or the most accurate canonical name.
          </div>
        </div>
      </div>

      <div className="modal-footer">
        <button className="cancel-button" onClick={onCancel}>
          Cancel
        </button>
        <button className="next-button" onClick={onNext} disabled={!primaryId}>
          Next: Preview Merge
        </button>
      </div>
    </>
  );
};

// Step 2: Preview Merge
const PreviewStep: React.FC<{
  preview: MergePreview;
  onBack: () => void;
  onNext: () => void;
  theme: "light" | "dark";
}> = ({ preview, onBack, onNext, theme }) => {
  return (
    <>
      <div className="modal-header">
        <h3>Preview Merge</h3>
      </div>

      <div className="modal-body">
        <div className="preview-section">
          <h4>Result After Merge</h4>

          <div className="result-card">
            <div className="result-row">
              <div className="result-label">Canonical Name:</div>
              <div className="result-value primary-name">{preview.resulting_canonical_name}</div>
            </div>

            <div className="result-row">
              <div className="result-label">Type:</div>
              <div className="result-value">
                <span className="type-badge">{formatCounterpartyType(preview.primary.type)}</span>
              </div>
            </div>

            <div className="result-row">
              <div className="result-label">Aliases:</div>
              <div className="result-value">
                {preview.resulting_aliases.length > 0 ? (
                  <div className="aliases-list">
                    {preview.resulting_aliases.map(alias => (
                      <span key={alias} className="alias-chip">{alias}</span>
                    ))}
                  </div>
                ) : (
                  <span className="no-aliases">No aliases</span>
                )}
              </div>
            </div>

            <div className="result-row">
              <div className="result-label">Total Transactions:</div>
              <div className="result-value total-transactions">
                {preview.total_transactions}
              </div>
            </div>
          </div>
        </div>

        <div className="preview-section">
          <h4>Counterparties Being Merged</h4>

          <div className="merge-flow">
            {/* Primary */}
            <div className="merge-item primary">
              <div className="merge-icon">👑</div>
              <div className="merge-name">{preview.primary.canonical_name}</div>
              <div className="merge-badge">PRIMARY (Kept)</div>
            </div>

            {/* Arrow */}
            <div className="merge-arrow">↓</div>

            {/* Merging */}
            {preview.merging.map(counterparty => (
              <div key={counterparty.counterparty_id} className="merge-item merging">
                <div className="merge-icon">📦</div>
                <div className="merge-name">{counterparty.canonical_name}</div>
                <div className="merge-badge removed">Will be removed</div>
              </div>
            ))}
          </div>
        </div>

        <div className="impact-summary">
          <h4>Impact Summary</h4>
          <ul className="impact-list">
            <li>
              <strong>{preview.merging.length}</strong> counterparty record{preview.merging.length > 1 ? 's' : ''} will be merged into{' '}
              <strong>{preview.primary.canonical_name}</strong>
            </li>
            <li>
              <strong>{preview.total_transactions}</strong> transactions will be associated with the primary counterparty
            </li>
            <li>
              All aliases will be preserved ({preview.resulting_aliases.length} total)
            </li>
            <li>
              Removed counterparties will no longer appear in lists or filters
            </li>
          </ul>
        </div>
      </div>

      <div className="modal-footer">
        <button className="back-button" onClick={onBack}>
          Back
        </button>
        <button className="next-button" onClick={onNext}>
          Next: Confirm Merge
        </button>
      </div>
    </>
  );
};

// Step 3: Confirm Merge
const ConfirmStep: React.FC<{
  preview: MergePreview;
  error: string | null;
  onBack: () => void;
  onConfirm: () => void;
  onCancel: () => void;
  theme: "light" | "dark";
}> = ({ preview, error, onBack, onConfirm, onCancel, theme }) => {
  const [confirmationText, setConfirmationText] = useState("");
  const confirmWord = "MERGE";
  const isConfirmed = confirmationText === confirmWord;

  return (
    <>
      <div className="modal-header">
        <h3>Confirm Merge</h3>
      </div>

      <div className="modal-body">
        {error && (
          <div className="error-box">
            <div className="error-icon">⚠️</div>
            <div className="error-message">{error}</div>
          </div>
        )}

        <div className="warning-box">
          <div className="warning-icon">⚠️</div>
          <div className="warning-content">
            <h4>Warning: This Action is Irreversible</h4>
            <p>
              Once you merge these counterparties, the operation cannot be undone.
              The following counterparties will be permanently removed:
            </p>
            <ul className="removed-list">
              {preview.merging.map(c => (
                <li key={c.counterparty_id}>{c.canonical_name}</li>
              ))}
            </ul>
            <p>
              All their transactions will be reassigned to <strong>{preview.primary.canonical_name}</strong>.
            </p>
          </div>
        </div>

        <div className="confirmation-input-section">
          <label htmlFor="confirmation">
            Type <strong>{confirmWord}</strong> to confirm:
          </label>
          <input
            type="text"
            id="confirmation"
            value={confirmationText}
            onChange={(e) => setConfirmationText(e.target.value.toUpperCase())}
            placeholder={confirmWord}
            className="confirmation-input"
            autoFocus
          />
        </div>

        <div className="final-summary">
          <div className="summary-item">
            <div className="summary-icon">👑</div>
            <div className="summary-text">
              Primary: <strong>{preview.primary.canonical_name}</strong>
            </div>
          </div>
          <div className="summary-item">
            <div className="summary-icon">📦</div>
            <div className="summary-text">
              Merging: <strong>{preview.merging.length} counterparty/counterparties</strong>
            </div>
          </div>
          <div className="summary-item">
            <div className="summary-icon">💼</div>
            <div className="summary-text">
              Total transactions: <strong>{preview.total_transactions}</strong>
            </div>
          </div>
        </div>
      </div>

      <div className="modal-footer">
        <button className="back-button" onClick={onBack}>
          Back
        </button>
        <button className="cancel-button" onClick={onCancel}>
          Cancel
        </button>
        <button
          className="confirm-button danger"
          onClick={onConfirm}
          disabled={!isConfirmed}
        >
          Merge Counterparties
        </button>
      </div>
    </>
  );
};

// Step 4: Merging (Progress)
const MergingStep: React.FC<{
  progress: number;
  theme: "light" | "dark";
}> = ({ progress, theme }) => {
  return (
    <>
      <div className="modal-header">
        <h3>Merging Counterparties...</h3>
      </div>

      <div className="modal-body merging-body">
        <div className="progress-spinner">
          <div className="spinner"></div>
        </div>

        <div className="progress-text">
          Merging counterparties and updating transactions...
        </div>

        <div className="progress-bar-container">
          <div className="progress-bar">
            <div
              className="progress-fill"
              style={{ width: `${progress}%` }}
            ></div>
          </div>
          <div className="progress-percentage">{progress}%</div>
        </div>

        <div className="progress-status">
          {progress < 30 && "Validating counterparties..."}
          {progress >= 30 && progress < 60 && "Merging records..."}
          {progress >= 60 && progress < 90 && "Updating transactions..."}
          {progress >= 90 && "Finalizing merge..."}
        </div>
      </div>
    </>
  );
};

// Step 5: Complete
const CompleteStep: React.FC<{
  preview: MergePreview;
  theme: "light" | "dark";
}> = ({ preview, theme }) => {
  return (
    <>
      <div className="modal-header">
        <h3>Merge Complete!</h3>
      </div>

      <div className="modal-body complete-body">
        <div className="success-icon">✅</div>

        <div className="success-message">
          Successfully merged {preview.merging.length} counterparty/counterparties into{' '}
          <strong>{preview.primary.canonical_name}</strong>
        </div>

        <div className="complete-summary">
          <div className="summary-stat">
            <div className="stat-value">{preview.total_transactions}</div>
            <div className="stat-label">Total Transactions</div>
          </div>
          <div className="summary-stat">
            <div className="stat-value">{preview.resulting_aliases.length}</div>
            <div className="stat-label">Combined Aliases</div>
          </div>
          <div className="summary-stat">
            <div className="stat-value">{preview.merging.length}</div>
            <div className="stat-label">Records Merged</div>
          </div>
        </div>

        <div className="auto-close-notice">
          This dialog will close automatically...
        </div>
      </div>
    </>
  );
};

// Helper functions
function formatCounterpartyType(type: CounterpartyType): string {
  const labels: Record<CounterpartyType, string> = {
    merchant: "Merchant",
    person: "Person",
    business: "Business",
    government: "Government"
  };
  return labels[type] || type;
}

function getDefaultIcon(type: CounterpartyType): string {
  const icons: Record<CounterpartyType, string> = {
    merchant: "🏪",
    person: "👤",
    business: "🏢",
    government: "🏛️"
  };
  return icons[type] || "👥";
}
```

---

## Styling

```css
.merge-dialog {
  max-width: 700px;
  width: 95%;
}

/* Progress Bar at Top */
.merge-progress-bar {
  padding: 20px 24px;
  border-bottom: 1px solid var(--divider-color);
}

.progress-steps {
  display: flex;
  justify-content: space-between;
  align-items: center;
  position: relative;
}

.progress-steps::before {
  content: '';
  position: absolute;
  top: 20px;
  left: 40px;
  right: 40px;
  height: 2px;
  background: var(--progress-line-bg);
  z-index: 0;
}

.progress-step {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  position: relative;
  z-index: 1;
}

.step-number {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: var(--step-inactive-bg);
  color: var(--step-inactive-color);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 16px;
  border: 3px solid var(--modal-bg);
  transition: all 0.3s;
}

.progress-step.active .step-number {
  background: var(--primary-color);
  color: white;
}

.progress-step.completed .step-number {
  background: var(--success-color);
  color: white;
}

.progress-step.completed .step-number::after {
  content: '✓';
}

.step-label {
  font-size: 12px;
  color: var(--secondary-color);
  font-weight: 500;
}

.progress-step.active .step-label {
  color: var(--primary-color);
  font-weight: 600;
}

/* Modal Body */
.modal-body {
  padding: 24px;
  max-height: 60vh;
  overflow-y: auto;
}

.instruction-text {
  font-size: 14px;
  color: var(--secondary-color);
  margin-bottom: 20px;
  line-height: 1.5;
}

/* Selection List */
.counterparty-selection-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 20px;
}

.selection-card {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: var(--card-bg);
  border: 2px solid var(--card-border);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.selection-card:hover {
  border-color: var(--primary-color);
  background: var(--card-hover-bg);
}

.selection-card.selected {
  border-color: var(--primary-color);
  background: var(--card-selected-bg);
}

.selection-radio input {
  width: 20px;
  height: 20px;
  cursor: pointer;
}

.counterparty-icon {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: var(--icon-bg);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  flex-shrink: 0;
}

.counterparty-details {
  flex: 1;
  min-width: 0;
}

.counterparty-name {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 4px;
}

.counterparty-meta {
  font-size: 13px;
  color: var(--secondary-color);
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 6px;
}

.type-badge {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  font-weight: 600;
}

.separator {
  color: var(--separator-color);
}

.aliases-preview {
  font-size: 12px;
  color: var(--tertiary-color);
  font-style: italic;
}

.selected-badge {
  font-size: 11px;
  padding: 4px 10px;
  background: var(--primary-color);
  color: white;
  border-radius: 4px;
  font-weight: 700;
  letter-spacing: 0.5px;
}

/* Recommendation Box */
.recommendation-box {
  display: flex;
  gap: 12px;
  padding: 12px 16px;
  background: var(--info-bg);
  border: 1px solid var(--info-border);
  border-radius: 8px;
  margin-top: 16px;
}

.recommendation-icon {
  font-size: 24px;
  flex-shrink: 0;
}

.recommendation-text {
  font-size: 13px;
  color: var(--text-color);
  line-height: 1.5;
}

/* Preview Section */
.preview-section {
  margin-bottom: 24px;
}

.preview-section h4 {
  font-size: 14px;
  font-weight: 600;
  color: var(--heading-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 12px;
}

.result-card {
  background: var(--result-bg);
  border: 1px solid var(--result-border);
  border-radius: 8px;
  padding: 16px;
}

.result-row {
  display: flex;
  padding: 10px 0;
  border-bottom: 1px solid var(--divider-color);
}

.result-row:last-child {
  border-bottom: none;
}

.result-label {
  width: 140px;
  font-size: 13px;
  font-weight: 600;
  color: var(--secondary-color);
  flex-shrink: 0;
}

.result-value {
  flex: 1;
  font-size: 14px;
  color: var(--text-color);
}

.primary-name {
  font-weight: 700;
  color: var(--primary-color);
  font-size: 16px;
}

.aliases-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.alias-chip {
  font-size: 12px;
  padding: 4px 10px;
  background: var(--chip-bg);
  border: 1px dashed var(--chip-border);
  border-radius: 4px;
  color: var(--text-color);
}

.no-aliases {
  font-style: italic;
  color: var(--tertiary-color);
}

.total-transactions {
  font-weight: 700;
  font-size: 18px;
  color: var(--success-color);
}

/* Merge Flow */
.merge-flow {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 16px;
  background: var(--flow-bg);
  border-radius: 8px;
}

.merge-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: var(--item-bg);
  border-radius: 6px;
}

.merge-item.primary {
  border: 2px solid var(--primary-color);
  background: var(--primary-light-bg);
}

.merge-item.merging {
  border: 2px dashed var(--warning-color);
  opacity: 0.7;
}

.merge-icon {
  font-size: 24px;
}

.merge-name {
  flex: 1;
  font-weight: 600;
  color: var(--text-color);
}

.merge-badge {
  font-size: 11px;
  padding: 3px 8px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  font-weight: 600;
}

.merge-badge.removed {
  background: var(--warning-bg);
  color: var(--warning-color);
}

.merge-arrow {
  text-align: center;
  font-size: 24px;
  color: var(--secondary-color);
  margin: -6px 0;
}

/* Impact Summary */
.impact-summary h4 {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 12px;
}

.impact-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.impact-list li {
  padding: 8px 0;
  padding-left: 24px;
  position: relative;
  font-size: 14px;
  color: var(--text-color);
  line-height: 1.5;
}

.impact-list li::before {
  content: '•';
  position: absolute;
  left: 8px;
  color: var(--primary-color);
  font-weight: bold;
}

/* Warning Box */
.warning-box {
  display: flex;
  gap: 16px;
  padding: 16px;
  background: var(--warning-light-bg);
  border: 2px solid var(--warning-color);
  border-radius: 8px;
  margin-bottom: 24px;
}

.warning-icon {
  font-size: 32px;
  flex-shrink: 0;
}

.warning-content h4 {
  font-size: 16px;
  font-weight: 700;
  color: var(--warning-dark-color);
  margin: 0 0 8px 0;
}

.warning-content p {
  font-size: 14px;
  color: var(--text-color);
  margin: 8px 0;
  line-height: 1.5;
}

.removed-list {
  margin: 12px 0;
  padding-left: 24px;
}

.removed-list li {
  font-weight: 600;
  color: var(--warning-dark-color);
  margin: 4px 0;
}

/* Confirmation Input */
.confirmation-input-section {
  margin-bottom: 24px;
}

.confirmation-input-section label {
  display: block;
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 8px;
  color: var(--text-color);
}

.confirmation-input {
  width: 100%;
  padding: 12px;
  border: 2px solid var(--input-border);
  border-radius: 6px;
  font-size: 16px;
  font-weight: 700;
  text-align: center;
  letter-spacing: 2px;
  background: var(--input-bg);
  color: var(--text-color);
}

.confirmation-input:focus {
  outline: none;
  border-color: var(--primary-color);
}

/* Final Summary */
.final-summary {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 16px;
  background: var(--summary-bg);
  border-radius: 8px;
}

.summary-item {
  display: flex;
  align-items: center;
  gap: 12px;
}

.summary-icon {
  font-size: 24px;
}

.summary-text {
  font-size: 14px;
  color: var(--text-color);
}

/* Merging Body */
.merging-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 24px;
  text-align: center;
}

.progress-spinner {
  margin-bottom: 24px;
}

.spinner {
  width: 60px;
  height: 60px;
  border: 4px solid var(--spinner-bg);
  border-top-color: var(--primary-color);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.progress-text {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 24px;
}

.progress-bar-container {
  width: 100%;
  margin-bottom: 12px;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: var(--progress-bg);
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 8px;
}

.progress-fill {
  height: 100%;
  background: var(--primary-color);
  transition: width 0.3s ease;
}

.progress-percentage {
  font-size: 14px;
  font-weight: 600;
  color: var(--primary-color);
}

.progress-status {
  font-size: 13px;
  color: var(--secondary-color);
  font-style: italic;
}

/* Complete Body */
.complete-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40px 24px;
  text-align: center;
}

.success-icon {
  font-size: 80px;
  margin-bottom: 20px;
}

.success-message {
  font-size: 16px;
  color: var(--text-color);
  margin-bottom: 32px;
  line-height: 1.5;
}

.complete-summary {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px;
  width: 100%;
  margin-bottom: 24px;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.stat-value {
  font-size: 32px;
  font-weight: 700;
  color: var(--success-color);
}

.stat-label {
  font-size: 12px;
  color: var(--secondary-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.auto-close-notice {
  font-size: 13px;
  color: var(--tertiary-color);
  font-style: italic;
}

/* Error Box */
.error-box {
  display: flex;
  gap: 12px;
  padding: 12px 16px;
  background: var(--error-bg);
  border: 1px solid var(--error-border);
  border-radius: 8px;
  margin-bottom: 20px;
}

.error-icon {
  font-size: 24px;
  flex-shrink: 0;
}

.error-message {
  font-size: 14px;
  color: var(--error-color);
  font-weight: 500;
}

/* Modal Footer */
.modal-footer {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
  padding: 16px 24px;
  border-top: 1px solid var(--divider-color);
}

.back-button,
.cancel-button {
  padding: 10px 20px;
  background: transparent;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  color: var(--text-color);
}

.next-button {
  padding: 10px 20px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.next-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.confirm-button {
  padding: 10px 20px;
  background: var(--danger-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.confirm-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Theme: Light */
.theme-light {
  --divider-color: #e5e7eb;
  --progress-line-bg: #e5e7eb;
  --step-inactive-bg: #f3f4f6;
  --step-inactive-color: #9ca3af;
  --success-color: #10b981;
  --primary-color: #3b82f6;
  --modal-bg: #ffffff;
  --card-bg: #ffffff;
  --card-border: #e5e7eb;
  --card-hover-bg: #f9fafb;
  --card-selected-bg: #e0e7ff;
  --icon-bg: #f3f4f6;
  --text-color: #111827;
  --secondary-color: #6b7280;
  --tertiary-color: #9ca3af;
  --heading-color: #111827;
  --badge-bg: #dbeafe;
  --badge-color: #1e40af;
  --separator-color: #d1d5db;
  --info-bg: #dbeafe;
  --info-border: #93c5fd;
  --result-bg: #f9fafb;
  --result-border: #e5e7eb;
  --chip-bg: #f3f4f6;
  --chip-border: #d1d5db;
  --flow-bg: #f9fafb;
  --item-bg: #ffffff;
  --primary-light-bg: #eff6ff;
  --warning-color: #f59e0b;
  --warning-bg: #fef3c7;
  --warning-light-bg: #fffbeb;
  --warning-dark-color: #92400e;
  --input-border: #d1d5db;
  --input-bg: #ffffff;
  --summary-bg: #f9fafb;
  --spinner-bg: #e5e7eb;
  --progress-bg: #e5e7eb;
  --error-bg: #fee;
  --error-border: #fcc;
  --error-color: #c33;
  --danger-color: #ef4444;
}

/* Theme: Dark */
.theme-dark {
  --divider-color: #374151;
  --progress-line-bg: #374151;
  --step-inactive-bg: #4b5563;
  --step-inactive-color: #6b7280;
  --success-color: #10b981;
  --primary-color: #3b82f6;
  --modal-bg: #1f2937;
  --card-bg: #374151;
  --card-border: #4b5563;
  --card-hover-bg: #4b5563;
  --card-selected-bg: #1e3a8a;
  --icon-bg: #4b5563;
  --text-color: #f3f4f6;
  --secondary-color: #9ca3af;
  --tertiary-color: #6b7280;
  --heading-color: #f3f4f6;
  --badge-bg: #1e3a8a;
  --badge-color: #93c5fd;
  --separator-color: #4b5563;
  --info-bg: #1e3a8a;
  --info-border: #2563eb;
  --result-bg: #111827;
  --result-border: #374151;
  --chip-bg: #374151;
  --chip-border: #4b5563;
  --flow-bg: #111827;
  --item-bg: #1f2937;
  --primary-light-bg: #1e3a8a;
  --warning-color: #f59e0b;
  --warning-bg: #78350f;
  --warning-light-bg: #451a03;
  --warning-dark-color: #fef3c7;
  --input-border: #4b5563;
  --input-bg: #374151;
  --summary-bg: #111827;
  --spinner-bg: #4b5563;
  --progress-bg: #374151;
  --error-bg: #7f1d1d;
  --error-border: #991b1b;
  --error-color: #fecaca;
  --danger-color: #ef4444;
}

/* Responsive */
@media (max-width: 768px) {
  .merge-dialog {
    width: 100%;
    max-width: 100%;
    margin: 0;
    border-radius: 0;
  }

  .complete-summary {
    grid-template-columns: 1fr;
  }

  .progress-steps::before {
    display: none;
  }

  .step-label {
    display: none;
  }
}
```

---

## Visual Wireframes

### Step 1: Select Primary
```
┌──────────────────────────────────────────────────────────┐
│ [1] Select Primary → [2] Preview → [3] Confirm           │
├──────────────────────────────────────────────────────────┤
│ Select Primary Counterparty                          [×] │
│                                                          │
│ Choose which counterparty to keep as the primary entry. │
│                                                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ (•) [🏪] Amazon                           [PRIMARY]│ │
│ │          Merchant • 247 transactions               │ │
│ │          Aliases: amzn.com, amazon.com, AWS        │ │
│ └────────────────────────────────────────────────────┘ │
│ ┌────────────────────────────────────────────────────┐ │
│ │ ( ) [🏪] AMZN                                      │ │
│ │          Merchant • 45 transactions                │ │
│ │          Aliases: Amazon Marketplace               │ │
│ └────────────────────────────────────────────────────┘ │
│ ┌────────────────────────────────────────────────────┐ │
│ │ ( ) [🏪] AWS                                       │ │
│ │          Merchant • 12 transactions                │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│ [💡] Recommendation: Select the counterparty with the   │
│      most transactions or most accurate canonical name.  │
│                                                          │
│                              [Cancel] [Next: Preview]  │
└──────────────────────────────────────────────────────────┘
```

### Step 2: Preview
```
┌──────────────────────────────────────────────────────────┐
│ [1] ✓ → [2] Preview → [3] Confirm                       │
├──────────────────────────────────────────────────────────┤
│ Preview Merge                                            │
│                                                          │
│ RESULT AFTER MERGE                                       │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Canonical Name:  Amazon                            │ │
│ │ Type:            [Merchant]                        │ │
│ │ Aliases:         amzn.com, amazon.com, AWS,        │ │
│ │                  AMZN, Amazon Marketplace           │ │
│ │ Total Trans:     304                               │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│ COUNTERPARTIES BEING MERGED                              │
│ ┌────────────────────────────────────────────────────┐ │
│ │ 👑  Amazon                    [PRIMARY (Kept)]     │ │
│ │                    ↓                                │ │
│ │ 📦  AMZN                      [Will be removed]     │ │
│ │ 📦  AWS                       [Will be removed]     │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│                              [Back] [Next: Confirm]    │
└──────────────────────────────────────────────────────────┘
```

### Step 3: Confirm
```
┌──────────────────────────────────────────────────────────┐
│ [1] ✓ → [2] ✓ → [3] Confirm                             │
├──────────────────────────────────────────────────────────┤
│ Confirm Merge                                            │
│                                                          │
│ ⚠️ WARNING: This Action is Irreversible                 │
│   The following counterparties will be permanently       │
│   removed:                                               │
│   • AMZN                                                 │
│   • AWS                                                  │
│                                                          │
│   All their transactions will be reassigned to Amazon.   │
│                                                          │
│ Type MERGE to confirm:                                   │
│ [_____________MERGE_____________]                        │
│                                                          │
│                [Back] [Cancel] [Merge Counterparties]  │
└──────────────────────────────────────────────────────────┘
```

---

## Accessibility

```tsx
<div className="selection-card" role="radio" aria-checked={isSelected}>
  <input
    type="radio"
    aria-label={`Select ${counterparty.canonical_name} as primary`}
  />
</div>

<button
  className="confirm-button"
  aria-label="Merge counterparties (irreversible action)"
  disabled={!isConfirmed}
>
  Merge Counterparties
</button>
```

---

## Related Components

**Uses:**
- None (standalone dialog)

**Used by:**
- CounterpartyManager (bulk merge workflow)
- DuplicateDetection (merge duplicate suggestions)

**Similar patterns:**
- MergeAccountsDialog
- BulkDeleteDialog
- ConfirmationDialog
