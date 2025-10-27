# RelationshipPanel (IL Component)

## Definition

**RelationshipPanel** is a reusable UI component that displays linked transactions in the transaction detail view. It shows relationship types, confidence scores for auto-detected links, and provides actions to navigate to linked records or unlink relationships.

**Problem it solves:**
- Transaction relationships (transfers, FX, reimbursements) need visual representation
- Auto-detected links require confidence scores for user validation
- Users need to unlink incorrect auto-matches
- Navigating between linked transactions requires clear interaction patterns
- Relationship context is lost when viewing transactions in isolation

**Solution:**
- Visual relationship type indicators with domain-specific icons
- Confidence scores for auto-detected relationships
- One-click navigation to linked transactions
- Unlink action with confirmation dialog
- Responsive design (desktop panel, mobile list)
- Multi-domain applicability (finance, healthcare, legal)

---

## Interface Contract

```typescript
interface RelationshipPanelProps {
  // Data
  transactionId: string;  // Current transaction canonical_id
  relationships: TransactionRelationship[];  // Linked transactions
  loading?: boolean;
  error?: string | null;

  // Callbacks
  onNavigate: (targetTransactionId: string) => void;  // Navigate to linked transaction
  onUnlink: (relationshipId: string) => Promise<void>;  // Remove relationship link
  onRefresh?: () => void;  // Reload relationships after unlink

  // UI customization
  showConfidenceScores?: boolean;  // Show confidence % for auto-detected (default: true)
  allowUnlink?: boolean;  // Enable unlink button (default: true)
  maxHeight?: string;  // Max height before scrolling (default: "400px")
  theme?: "light" | "dark";
}

interface TransactionRelationship {
  relationship_id: string;  // Unique ID for this relationship link
  type: RelationshipType;  // transfer | fx | reimbursement | split | correction
  target_transaction: LinkedTransaction;  // The linked transaction
  confidence_score?: number;  // 0-100, only for auto-detected relationships
  detection_method: "auto" | "manual";  // How link was created
  created_at: string;  // ISO 8601 timestamp
  created_by: string;  // User or "system:relationship_detector_v1.0"
}

type RelationshipType =
  | "transfer"       // Money moved between accounts (same owner)
  | "fx"             // Foreign exchange transaction (currency conversion)
  | "reimbursement"  // Expense reimbursement (employee → company)
  | "split"          // Split transaction (shared expense)
  | "correction";    // Correction/reversal of previous transaction

interface LinkedTransaction {
  canonical_id: string;
  transaction_date: string;
  account: string;
  counterparty: string;
  amount: number;
  currency: string;
  description?: string;
}
```

---

## Component Structure

```tsx
import React, { useState } from 'react';

export const RelationshipPanel: React.FC<RelationshipPanelProps> = ({
  transactionId,
  relationships,
  loading = false,
  error = null,
  onNavigate,
  onUnlink,
  onRefresh,
  showConfidenceScores = true,
  allowUnlink = true,
  maxHeight = "400px",
  theme = "light"
}) => {
  const [unlinkingId, setUnlinkingId] = useState<string | null>(null);
  const [confirmUnlinkId, setConfirmUnlinkId] = useState<string | null>(null);

  const handleUnlink = async (relationshipId: string) => {
    setUnlinkingId(relationshipId);
    try {
      await onUnlink(relationshipId);
      setConfirmUnlinkId(null);
      onRefresh?.();
    } catch (error) {
      console.error('Unlink failed:', error);
    } finally {
      setUnlinkingId(null);
    }
  };

  if (loading) {
    return <RelationshipPanelSkeleton />;
  }

  if (error) {
    return (
      <div className="relationship-panel-error">
        <p>Failed to load relationships: {error}</p>
      </div>
    );
  }

  if (relationships.length === 0) {
    return (
      <div className="relationship-panel-empty">
        <p>No linked transactions</p>
        <span className="empty-icon">🔗</span>
      </div>
    );
  }

  return (
    <div
      className={`relationship-panel ${theme}`}
      style={{ maxHeight }}
    >
      <div className="relationship-panel-header">
        <h3>Linked Transactions ({relationships.length})</h3>
      </div>

      <div className="relationship-list">
        {relationships.map((rel) => (
          <RelationshipCard
            key={rel.relationship_id}
            relationship={rel}
            onNavigate={() => onNavigate(rel.target_transaction.canonical_id)}
            onUnlink={allowUnlink ? () => setConfirmUnlinkId(rel.relationship_id) : undefined}
            showConfidenceScore={showConfidenceScores}
            isUnlinking={unlinkingId === rel.relationship_id}
          />
        ))}
      </div>

      {/* Unlink Confirmation Dialog */}
      {confirmUnlinkId && (
        <UnlinkConfirmationDialog
          relationship={relationships.find(r => r.relationship_id === confirmUnlinkId)!}
          onConfirm={() => handleUnlink(confirmUnlinkId)}
          onCancel={() => setConfirmUnlinkId(null)}
        />
      )}
    </div>
  );
};
```

---

## RelationshipCard Component

```tsx
interface RelationshipCardProps {
  relationship: TransactionRelationship;
  onNavigate: () => void;
  onUnlink?: () => void;
  showConfidenceScore: boolean;
  isUnlinking: boolean;
}

const RelationshipCard: React.FC<RelationshipCardProps> = ({
  relationship,
  onNavigate,
  onUnlink,
  showConfidenceScore,
  isUnlinking
}) => {
  const { type, target_transaction, confidence_score, detection_method } = relationship;

  return (
    <div
      className="relationship-card"
      onClick={onNavigate}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => {
        if (e.key === 'Enter' || e.key === ' ') {
          e.preventDefault();
          onNavigate();
        }
      }}
    >
      {/* Relationship Type Header */}
      <div className="relationship-card-header">
        <div className="relationship-type">
          <span className="relationship-icon">
            {getRelationshipIcon(type)}
          </span>
          <span className="relationship-label">
            {getRelationshipLabel(type)}
          </span>
        </div>

        {/* Confidence Score Badge */}
        {showConfidenceScore && detection_method === "auto" && confidence_score !== undefined && (
          <div className={`confidence-badge ${getConfidenceLevel(confidence_score)}`}>
            {confidence_score}%
          </div>
        )}
      </div>

      {/* Transaction Summary */}
      <div className="relationship-card-body">
        <div className="transaction-summary">
          <div className="summary-row">
            <span className="summary-label">Account:</span>
            <span className="summary-value">{target_transaction.account}</span>
          </div>
          <div className="summary-row">
            <span className="summary-label">Counterparty:</span>
            <span className="summary-value">{target_transaction.counterparty}</span>
          </div>
          <div className="summary-row">
            <span className="summary-label">Amount:</span>
            <span className="summary-value">
              {formatAmount(target_transaction.amount, target_transaction.currency)}
            </span>
          </div>
          <div className="summary-row">
            <span className="summary-label">Date:</span>
            <span className="summary-value">
              {formatDate(target_transaction.transaction_date)}
            </span>
          </div>
        </div>

        {/* Description (if available) */}
        {target_transaction.description && (
          <div className="transaction-description">
            {target_transaction.description}
          </div>
        )}
      </div>

      {/* Footer Actions */}
      <div className="relationship-card-footer">
        <span className="navigate-hint">Click to view transaction →</span>

        {onUnlink && (
          <button
            className="unlink-button"
            onClick={(e) => {
              e.stopPropagation();
              onUnlink();
            }}
            disabled={isUnlinking}
            aria-label="Unlink transaction"
          >
            {isUnlinking ? "Unlinking..." : "Unlink"}
          </button>
        )}
      </div>
    </div>
  );
};
```

---

## UnlinkConfirmationDialog Component

```tsx
interface UnlinkConfirmationDialogProps {
  relationship: TransactionRelationship;
  onConfirm: () => void;
  onCancel: () => void;
}

const UnlinkConfirmationDialog: React.FC<UnlinkConfirmationDialogProps> = ({
  relationship,
  onConfirm,
  onCancel
}) => {
  return (
    <div className="unlink-dialog-overlay" onClick={onCancel}>
      <div
        className="unlink-dialog"
        onClick={(e) => e.stopPropagation()}
        role="dialog"
        aria-labelledby="unlink-dialog-title"
        aria-modal="true"
      >
        <div className="unlink-dialog-header">
          <h3 id="unlink-dialog-title">Confirm Unlink</h3>
        </div>

        <div className="unlink-dialog-body">
          <p>
            Are you sure you want to unlink this {getRelationshipLabel(relationship.type).toLowerCase()}?
          </p>
          <div className="unlink-preview">
            <p><strong>Account:</strong> {relationship.target_transaction.account}</p>
            <p><strong>Amount:</strong> {formatAmount(
              relationship.target_transaction.amount,
              relationship.target_transaction.currency
            )}</p>
            <p><strong>Date:</strong> {formatDate(relationship.target_transaction.transaction_date)}</p>
          </div>
          {relationship.detection_method === "auto" && (
            <p className="unlink-warning">
              ⚠️ This link was auto-detected. Unlinking may affect automatic reconciliation.
            </p>
          )}
        </div>

        <div className="unlink-dialog-footer">
          <button
            className="button-secondary"
            onClick={onCancel}
          >
            Cancel
          </button>
          <button
            className="button-danger"
            onClick={onConfirm}
          >
            Unlink
          </button>
        </div>
      </div>
    </div>
  );
};
```

---

## Helper Functions

```tsx
function getRelationshipIcon(type: RelationshipType): string {
  const icons = {
    "transfer": "🔗",
    "fx": "💱",
    "reimbursement": "💰",
    "split": "✂️",
    "correction": "🔄"
  };
  return icons[type] || "🔗";
}

function getRelationshipLabel(type: RelationshipType): string {
  const labels = {
    "transfer": "Transfer",
    "fx": "Foreign Exchange",
    "reimbursement": "Reimbursement",
    "split": "Split Transaction",
    "correction": "Correction"
  };
  return labels[type] || type;
}

function getConfidenceLevel(score: number): string {
  if (score >= 90) return "high";
  if (score >= 70) return "medium";
  return "low";
}

function formatAmount(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: currency || 'USD'
  }).format(amount);
}

function formatDate(dateString: string): string {
  return new Date(dateString).toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: 'numeric'
  });
}
```

---

## Visual Design

### Desktop View (Panel)

```
┌─────────────────────────────────────────┐
│ Linked Transactions (2)                 │
├─────────────────────────────────────────┤
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ 🔗 Transfer              [92%]      │ │
│ │                                     │ │
│ │ Account:      Checking ***1234      │ │
│ │ Counterparty: Internal Transfer     │ │
│ │ Amount:       -$500.00              │ │
│ │ Date:         May 23, 2025          │ │
│ │                                     │ │
│ │ Click to view transaction →  [Unlink]│ │
│ └─────────────────────────────────────┘ │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ 💱 Foreign Exchange      [85%]      │ │
│ │                                     │ │
│ │ Account:      EUR Account ***5678   │ │
│ │ Counterparty: Wise Transfer         │ │
│ │ Amount:       €450.00               │ │
│ │ Date:         May 23, 2025          │ │
│ │                                     │ │
│ │ Click to view transaction →  [Unlink]│ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Mobile View (Stacked Cards)

```
┌───────────────────────────┐
│ Linked Transactions (2)   │
├───────────────────────────┤
│                           │
│ ┌───────────────────────┐ │
│ │ 🔗 Transfer     [92%] │ │
│ │                       │ │
│ │ Checking ***1234      │ │
│ │ -$500.00              │ │
│ │ May 23, 2025          │ │
│ │                       │ │
│ │ View → | Unlink       │ │
│ └───────────────────────┘ │
│                           │
│ ┌───────────────────────┐ │
│ │ 💱 FX           [85%] │ │
│ │                       │ │
│ │ EUR Account ***5678   │ │
│ │ €450.00               │ │
│ │ May 23, 2025          │ │
│ │                       │ │
│ │ View → | Unlink       │ │
│ └───────────────────────┘ │
└───────────────────────────┘
```

### Empty State

```
┌─────────────────────────────┐
│ Linked Transactions (0)     │
├─────────────────────────────┤
│                             │
│         🔗                  │
│                             │
│   No linked transactions    │
│                             │
└─────────────────────────────┘
```

### Loading State

```
┌─────────────────────────────────────────┐
│ Linked Transactions                     │
├─────────────────────────────────────────┤
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ ████████████░░░░░░░░░░░░░░░░░░░░░░░│ │
│ │ ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │
│ │ ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Unlink Confirmation Dialog

```
┌───────────────────────────────────┐
│ Confirm Unlink                    │
├───────────────────────────────────┤
│                                   │
│ Are you sure you want to unlink   │
│ this transfer?                    │
│                                   │
│ Account:      Checking ***1234    │
│ Amount:       -$500.00            │
│ Date:         May 23, 2025        │
│                                   │
│ ⚠️ This link was auto-detected.   │
│ Unlinking may affect automatic    │
│ reconciliation.                   │
│                                   │
├───────────────────────────────────┤
│         [Cancel]  [Unlink]        │
└───────────────────────────────────┘
```

---

## State Management

### States

```tsx
type RelationshipPanelState =
  | "loading"        // Fetching relationships from API
  | "empty"          // No relationships found
  | "single"         // One relationship
  | "multiple"       // Multiple relationships
  | "error"          // Failed to load relationships
  | "unlinking";     // Unlink operation in progress

// State transitions
const stateTransitions = {
  loading: ["empty", "single", "multiple", "error"],
  empty: ["loading", "single", "multiple"],
  single: ["loading", "empty", "multiple", "unlinking"],
  multiple: ["loading", "empty", "single", "unlinking"],
  error: ["loading"],
  unlinking: ["single", "multiple", "empty", "error"]
};
```

### Loading State

```tsx
const RelationshipPanelSkeleton: React.FC = () => {
  return (
    <div className="relationship-panel-skeleton">
      <div className="skeleton-header">
        <div className="skeleton-title"></div>
      </div>
      <div className="skeleton-cards">
        <div className="skeleton-card">
          <div className="skeleton-line"></div>
          <div className="skeleton-line"></div>
          <div className="skeleton-line"></div>
        </div>
      </div>
    </div>
  );
};
```

---

## CSS Styling

```css
/* Panel Container */
.relationship-panel {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow-y: auto;
}

.relationship-panel-header {
  padding: 16px;
  border-bottom: 1px solid #e5e7eb;
}

.relationship-panel-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: #111827;
}

/* Relationship List */
.relationship-list {
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

/* Relationship Card */
.relationship-card {
  background: #f9fafb;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  padding: 12px;
  cursor: pointer;
  transition: all 0.2s;
}

.relationship-card:hover {
  background: #f3f4f6;
  border-color: #3b82f6;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.relationship-card:focus {
  outline: 2px solid #3b82f6;
  outline-offset: 2px;
}

/* Card Header */
.relationship-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.relationship-type {
  display: flex;
  align-items: center;
  gap: 8px;
}

.relationship-icon {
  font-size: 18px;
}

.relationship-label {
  font-size: 14px;
  font-weight: 600;
  color: #374151;
}

/* Confidence Badge */
.confidence-badge {
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 600;
}

.confidence-badge.high {
  background: #d1fae5;
  color: #065f46;
}

.confidence-badge.medium {
  background: #fef3c7;
  color: #92400e;
}

.confidence-badge.low {
  background: #fee2e2;
  color: #991b1b;
}

/* Transaction Summary */
.transaction-summary {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 8px;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
}

.summary-label {
  color: #6b7280;
  font-weight: 500;
}

.summary-value {
  color: #111827;
  font-weight: 400;
}

/* Transaction Description */
.transaction-description {
  font-size: 12px;
  color: #6b7280;
  font-style: italic;
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #e5e7eb;
}

/* Card Footer */
.relationship-card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #e5e7eb;
}

.navigate-hint {
  font-size: 12px;
  color: #3b82f6;
  font-weight: 500;
}

.unlink-button {
  padding: 6px 12px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
  color: #ef4444;
  cursor: pointer;
  transition: all 0.2s;
}

.unlink-button:hover {
  background: #fef2f2;
  border-color: #ef4444;
}

.unlink-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Empty State */
.relationship-panel-empty {
  padding: 48px 24px;
  text-align: center;
  color: #6b7280;
}

.empty-icon {
  font-size: 48px;
  display: block;
  margin-bottom: 12px;
  opacity: 0.5;
}

/* Error State */
.relationship-panel-error {
  padding: 24px;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 6px;
  margin: 12px;
}

.relationship-panel-error p {
  color: #991b1b;
  margin: 0;
}

/* Unlink Confirmation Dialog */
.unlink-dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.unlink-dialog {
  background: white;
  border-radius: 8px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
  max-width: 400px;
  width: 90%;
}

.unlink-dialog-header {
  padding: 16px 20px;
  border-bottom: 1px solid #e5e7eb;
}

.unlink-dialog-header h3 {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
  color: #111827;
}

.unlink-dialog-body {
  padding: 20px;
}

.unlink-dialog-body p {
  margin: 0 0 16px 0;
  color: #374151;
}

.unlink-preview {
  background: #f9fafb;
  border: 1px solid #e5e7eb;
  border-radius: 4px;
  padding: 12px;
  margin-bottom: 16px;
}

.unlink-preview p {
  margin: 6px 0;
  font-size: 13px;
}

.unlink-warning {
  background: #fef3c7;
  border: 1px solid #fcd34d;
  border-radius: 4px;
  padding: 12px;
  font-size: 13px;
  color: #92400e;
}

.unlink-dialog-footer {
  padding: 16px 20px;
  border-top: 1px solid #e5e7eb;
  display: flex;
  gap: 12px;
  justify-content: flex-end;
}

.button-secondary {
  padding: 8px 16px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
  color: #374151;
  cursor: pointer;
  transition: all 0.2s;
}

.button-secondary:hover {
  background: #f9fafb;
  border-color: #d1d5db;
}

.button-danger {
  padding: 8px 16px;
  background: #ef4444;
  border: 1px solid #ef4444;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
  color: white;
  cursor: pointer;
  transition: all 0.2s;
}

.button-danger:hover {
  background: #dc2626;
  border-color: #dc2626;
}

/* Responsive Design */
@media (max-width: 768px) {
  .relationship-panel {
    border-radius: 0;
    border-left: none;
    border-right: none;
  }

  .relationship-card {
    padding: 10px;
  }

  .transaction-summary {
    font-size: 12px;
  }

  .relationship-card-footer {
    flex-direction: column;
    gap: 8px;
    align-items: stretch;
  }

  .unlink-button {
    width: 100%;
  }

  .navigate-hint {
    text-align: center;
  }
}

/* Dark Theme */
.relationship-panel.dark {
  background: #1f2937;
  border-color: #374151;
}

.relationship-panel.dark .relationship-panel-header h3 {
  color: #f9fafb;
}

.relationship-panel.dark .relationship-card {
  background: #111827;
  border-color: #374151;
}

.relationship-panel.dark .relationship-card:hover {
  background: #1f2937;
  border-color: #3b82f6;
}

.relationship-panel.dark .summary-label {
  color: #9ca3af;
}

.relationship-panel.dark .summary-value {
  color: #f9fafb;
}
```

---

## Accessibility

### Keyboard Navigation

**Supported Keys:**
- `Tab` - Focus next relationship card
- `Shift+Tab` - Focus previous relationship card
- `Enter` / `Space` - Navigate to linked transaction
- `Escape` - Close unlink confirmation dialog

**Focus Management:**
```tsx
// Focus trap in unlink dialog
const UnlinkConfirmationDialog: React.FC = ({ onConfirm, onCancel }) => {
  const dialogRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const handleKeydown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onCancel();
      }
    };

    // Trap focus within dialog
    const firstFocusable = dialogRef.current?.querySelector('button');
    firstFocusable?.focus();

    window.addEventListener('keydown', handleKeydown);
    return () => window.removeEventListener('keydown', handleKeydown);
  }, []);

  return (
    <div ref={dialogRef} className="unlink-dialog">
      {/* Dialog content */}
    </div>
  );
};
```

### ARIA Attributes

```tsx
<div
  className="relationship-panel"
  role="region"
  aria-labelledby="relationships-title"
>
  <div className="relationship-panel-header">
    <h3 id="relationships-title">Linked Transactions ({relationships.length})</h3>
  </div>

  <div className="relationship-list" role="list">
    <div
      className="relationship-card"
      role="listitem"
      tabIndex={0}
      aria-label={`${getRelationshipLabel(type)} with ${account}, ${formatAmount(amount, currency)}`}
      onClick={onNavigate}
    >
      {/* Card content */}
    </div>
  </div>
</div>

{/* Unlink button */}
<button
  aria-label={`Unlink ${getRelationshipLabel(type)} transaction`}
  aria-describedby={`unlink-description-${relationshipId}`}
>
  Unlink
</button>
<span id={`unlink-description-${relationshipId}`} className="sr-only">
  This will remove the link between the current transaction and {account}
</span>
```

### Screen Reader Support

**Announcements:**
```tsx
// Announce relationship count
<div role="status" aria-live="polite" className="sr-only">
  {relationships.length} linked transaction{relationships.length !== 1 ? 's' : ''} found
</div>

// Announce unlink success
<div role="status" aria-live="polite" className="sr-only">
  {unlinkSuccess && "Transaction unlinked successfully"}
</div>

// Announce unlink error
<div role="alert" aria-live="assertive" className="sr-only">
  {unlinkError && `Unlink failed: ${unlinkError}`}
</div>
```

**Screen Reader Only Text:**
```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}
```

---

## Multi-Domain Applicability

### Finance Domain: Transfer Linking

```tsx
<RelationshipPanel
  transactionId="txn_12345"
  relationships={[
    {
      relationship_id: "rel_001",
      type: "transfer",
      target_transaction: {
        canonical_id: "txn_67890",
        transaction_date: "2025-05-23",
        account: "Checking ***1234",
        counterparty: "Internal Transfer",
        amount: -500.00,
        currency: "USD",
        description: "Transfer to savings"
      },
      confidence_score: 92,
      detection_method: "auto",
      created_at: "2025-05-23T10:30:00Z",
      created_by: "system:relationship_detector_v1.0"
    }
  ]}
  onNavigate={(id) => router.push(`/transactions/${id}`)}
  onUnlink={async (relId) => {
    await fetch(`/api/relationships/${relId}`, { method: 'DELETE' });
  }}
/>
// Shows: Bank account transfer with 92% confidence score
```

### Finance Domain: Foreign Exchange

```tsx
<RelationshipPanel
  transactionId="txn_fx_001"
  relationships={[
    {
      relationship_id: "rel_fx_001",
      type: "fx",
      target_transaction: {
        canonical_id: "txn_fx_002",
        transaction_date: "2025-05-23",
        account: "EUR Account ***5678",
        counterparty: "Wise Transfer",
        amount: 450.00,
        currency: "EUR",
        description: "USD → EUR conversion"
      },
      confidence_score: 85,
      detection_method: "auto",
      created_at: "2025-05-23T14:15:00Z",
      created_by: "system:fx_detector_v1.0"
    }
  ]}
  onNavigate={(id) => router.push(`/transactions/${id}`)}
  onUnlink={unlinkRelationship}
/>
// Shows: Currency conversion with matching timestamp
```

### Finance Domain: Reimbursement

```tsx
<RelationshipPanel
  transactionId="txn_expense_001"
  relationships={[
    {
      relationship_id: "rel_reimb_001",
      type: "reimbursement",
      target_transaction: {
        canonical_id: "txn_reimb_001",
        transaction_date: "2025-05-30",
        account: "Corporate Card ***9999",
        counterparty: "Acme Corp",
        amount: 125.50,
        currency: "USD",
        description: "Expense reimbursement received"
      },
      confidence_score: 78,
      detection_method: "auto",
      created_at: "2025-05-30T09:00:00Z",
      created_by: "system:reimbursement_detector_v1.0"
    }
  ]}
  onNavigate={(id) => router.push(`/transactions/${id}`)}
  onUnlink={unlinkRelationship}
/>
// Shows: Expense and reimbursement pair
```

### Healthcare Domain: Insurance Claim Relationships

```tsx
interface HealthcareRelationship extends TransactionRelationship {
  type: "claim_payment" | "deductible" | "copay" | "coinsurance";
  target_transaction: {
    canonical_id: string;
    service_date: string;
    provider: string;
    patient: string;
    amount: number;
    currency: string;
    claim_number?: string;
  };
}

<RelationshipPanel
  transactionId="claim_12345"
  relationships={[
    {
      relationship_id: "rel_claim_001",
      type: "claim_payment",
      target_transaction: {
        canonical_id: "payment_67890",
        service_date: "2025-05-15",
        provider: "City Hospital",
        patient: "John Doe",
        amount: 1500.00,
        currency: "USD",
        claim_number: "CLM-2025-001"
      },
      confidence_score: 95,
      detection_method: "auto",
      created_at: "2025-05-20T11:00:00Z",
      created_by: "system:claim_matcher_v2.0"
    }
  ]}
  onNavigate={(id) => router.push(`/claims/${id}`)}
  onUnlink={unlinkClaim}
/>
// Shows: Insurance claim and payment relationship
// Icon: 🏥 (customized for healthcare domain)
```

### Legal Domain: Case Document Relationships

```tsx
interface LegalRelationship extends TransactionRelationship {
  type: "amendment" | "reference" | "precedent" | "superseded";
  target_transaction: {
    canonical_id: string;
    document_date: string;
    document_type: string;
    parties: string;
    case_number?: string;
    description?: string;
  };
}

<RelationshipPanel
  transactionId="doc_contract_001"
  relationships={[
    {
      relationship_id: "rel_amend_001",
      type: "amendment",
      target_transaction: {
        canonical_id: "doc_contract_002",
        document_date: "2025-06-01",
        document_type: "Amendment",
        parties: "Acme Corp vs Beta Inc",
        case_number: "CV-2025-0123",
        description: "First amendment to service agreement"
      },
      confidence_score: null,
      detection_method: "manual",
      created_at: "2025-06-01T16:30:00Z",
      created_by: "lawyer_smith@firm.com"
    }
  ]}
  onNavigate={(id) => router.push(`/documents/${id}`)}
  onUnlink={unlinkDocument}
  showConfidenceScores={false}  // Manual links don't have scores
/>
// Shows: Contract amendment relationship
// Icon: 📄 (customized for legal domain)
```

### Research Domain (RSRCH - Utilitario): Fact Source Relationships

```tsx
interface RSRCHRelationship extends TransactionRelationship {
  type: "fact_source" | "corroboration" | "contradiction" | "enrichment";
  target_transaction: {
    canonical_id: string;
    discovered_at: string;
    claim: string;
    source_type: string;
    source_url?: string;
    subject_entity?: string;
  };
}

<RelationshipPanel
  transactionId="fact_sama_helion_001"
  relationships={[
    {
      relationship_id: "rel_source_001",
      type: "corroboration",
      target_transaction: {
        canonical_id: "fact_sama_helion_002",
        discovered_at: "2024-03-15",
        claim: "Sam Altman invested $375M in Helion Energy",
        source_type: "podcast",
        source_url: "https://lexfridman.com/sama-interview",
        subject_entity: "Sam Altman"
      },
      confidence_score: 92,
      detection_method: "auto",
      created_at: "2025-05-23T14:00:00Z",
      created_by: "system:fact_linker_v2.0"
    }
  ]}
  onNavigate={(id) => router.push(`/facts/${id}`)}
  onUnlink={unlinkFactSource}
/>
// Shows: Fact source corroboration relationship
// Icon: 🔗 (customized for RSRCH domain)
```

### Domain-Agnostic Pattern

**Core Pattern: Linked Record Display Panel**

Common elements across all domains:
1. **Relationship Type Icon** - Visual indicator (🔗, 💱, 🏥, 📄, 📚)
2. **Target Record Summary** - Domain-specific fields (account, provider, parties, authors)
3. **Confidence Score** - For auto-detected relationships (optional)
4. **Navigation Action** - Click to view linked record
5. **Unlink Action** - Remove relationship with confirmation
6. **Empty/Loading States** - Consistent UX across domains

**Customization Points:**
```tsx
// Domain-specific configuration
const domainConfig = {
  finance: {
    relationshipTypes: ["transfer", "fx", "reimbursement", "split", "correction"],
    summaryFields: ["account", "counterparty", "amount", "currency", "date"],
    icon: "🔗"
  },
  healthcare: {
    relationshipTypes: ["claim_payment", "deductible", "copay", "coinsurance"],
    summaryFields: ["provider", "patient", "amount", "service_date", "claim_number"],
    icon: "🏥"
  },
  legal: {
    relationshipTypes: ["amendment", "reference", "precedent", "superseded"],
    summaryFields: ["document_type", "parties", "case_number", "document_date"],
    icon: "📄"
  },
  research: {
    relationshipTypes: ["fact_source", "corroboration", "contradiction", "enrichment"],
    summaryFields: ["claim", "subject_entity", "source_type", "discovered_at", "source_url"],
    icon: "🔗"
  }
};
```

---

## Related Components

### Used By

**Transaction Detail View (Vertical 3.5):**
```tsx
function TransactionDetailView({ transactionId }) {
  const { data: transaction } = useTransaction(transactionId);
  const { data: relationships } = useRelationships(transactionId);

  return (
    <div className="transaction-detail">
      {/* Transaction details */}
      <TransactionHeader transaction={transaction} />
      <TransactionFields transaction={transaction} />

      {/* Relationships panel */}
      {relationships.length > 0 && (
        <RelationshipPanel
          transactionId={transactionId}
          relationships={relationships}
          onNavigate={(id) => router.push(`/transactions/${id}`)}
          onUnlink={unlinkRelationship}
        />
      )}
    </div>
  );
}
```

**DrillDownPanel (IL Component):**
```tsx
// Add relationships tab to drill-down panel
<DrillDownPanel
  drilldownData={data}
  defaultTab="relationships"
  onClose={closePanel}
>
  <RelationshipPanel
    transactionId={data.canonical.canonical_id}
    relationships={data.relationships}
    onNavigate={navigateToTransaction}
    onUnlink={unlinkRelationship}
  />
</DrillDownPanel>
```

### Uses

**No direct dependencies** - RelationshipPanel is a standalone presentation component.

**Optional integrations:**
- **TransactionCard** (IL) - Can display mini transaction preview
- **ConfirmationDialog** (IL) - Reusable dialog for unlink confirmation
- **Toast/Notification** (IL) - Show success/error messages after unlink

---

## Example Usage

### Basic Usage (Single Relationship)

```tsx
import { RelationshipPanel } from '@/components/RelationshipPanel';

function TransactionPage() {
  const [relationships, setRelationships] = useState<TransactionRelationship[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/transactions/${transactionId}/relationships`)
      .then(res => res.json())
      .then(data => {
        setRelationships(data);
        setLoading(false);
      });
  }, [transactionId]);

  const handleNavigate = (targetId: string) => {
    router.push(`/transactions/${targetId}`);
  };

  const handleUnlink = async (relationshipId: string) => {
    await fetch(`/api/relationships/${relationshipId}`, {
      method: 'DELETE'
    });
    // Refresh relationships
    setRelationships(prev => prev.filter(r => r.relationship_id !== relationshipId));
  };

  return (
    <div className="transaction-page">
      <h1>Transaction Details</h1>

      <RelationshipPanel
        transactionId={transactionId}
        relationships={relationships}
        loading={loading}
        onNavigate={handleNavigate}
        onUnlink={handleUnlink}
      />
    </div>
  );
}
```

### Multiple Relationships (Transfer Chain)

```tsx
function TransferChainView() {
  // Example: Checking → Savings → Investment
  const relationships = [
    {
      relationship_id: "rel_001",
      type: "transfer",
      target_transaction: {
        canonical_id: "txn_002",
        transaction_date: "2025-05-23",
        account: "Savings ***5678",
        counterparty: "Internal Transfer",
        amount: -1000.00,
        currency: "USD"
      },
      confidence_score: 95,
      detection_method: "auto",
      created_at: "2025-05-23T10:00:00Z",
      created_by: "system:relationship_detector_v1.0"
    },
    {
      relationship_id: "rel_002",
      type: "transfer",
      target_transaction: {
        canonical_id: "txn_003",
        transaction_date: "2025-05-23",
        account: "Investment ***9999",
        counterparty: "Internal Transfer",
        amount: -500.00,
        currency: "USD"
      },
      confidence_score: 92,
      detection_method: "auto",
      created_at: "2025-05-23T10:01:00Z",
      created_by: "system:relationship_detector_v1.0"
    }
  ];

  return (
    <RelationshipPanel
      transactionId="txn_001"
      relationships={relationships}
      onNavigate={navigateToTransaction}
      onUnlink={unlinkRelationship}
    />
  );
}
// Shows: Two linked transfers in sequence
```

### Manual Relationship (No Confidence Score)

```tsx
function ManualLinkView() {
  const relationships = [
    {
      relationship_id: "rel_manual_001",
      type: "reimbursement",
      target_transaction: {
        canonical_id: "txn_reimb_001",
        transaction_date: "2025-05-30",
        account: "Corporate Card ***9999",
        counterparty: "Acme Corp",
        amount: 125.50,
        currency: "USD",
        description: "Manual reimbursement link"
      },
      confidence_score: null,  // Manual links have no score
      detection_method: "manual",
      created_at: "2025-05-30T15:00:00Z",
      created_by: "eugenio@example.com"
    }
  ];

  return (
    <RelationshipPanel
      transactionId="txn_expense_001"
      relationships={relationships}
      onNavigate={navigateToTransaction}
      onUnlink={unlinkRelationship}
      showConfidenceScores={false}  // Hide scores for manual links
    />
  );
}
// Shows: Manual link without confidence badge
```

### Read-Only Mode (No Unlink)

```tsx
function ReadOnlyRelationshipView() {
  return (
    <RelationshipPanel
      transactionId="txn_001"
      relationships={relationships}
      onNavigate={navigateToTransaction}
      onUnlink={null}  // No unlink callback = read-only
      allowUnlink={false}  // Explicitly disable unlink button
    />
  );
}
// Shows: Relationships without unlink button (view-only)
```

### With Error Handling

```tsx
function RobustRelationshipView() {
  const [error, setError] = useState<string | null>(null);

  const handleUnlink = async (relationshipId: string) => {
    try {
      const response = await fetch(`/api/relationships/${relationshipId}`, {
        method: 'DELETE'
      });

      if (!response.ok) {
        throw new Error('Unlink failed');
      }

      // Refresh relationships
      onRefresh();
    } catch (err) {
      setError('Failed to unlink transaction. Please try again.');
    }
  };

  return (
    <>
      {error && (
        <div className="error-banner">
          {error}
          <button onClick={() => setError(null)}>Dismiss</button>
        </div>
      )}

      <RelationshipPanel
        transactionId={transactionId}
        relationships={relationships}
        error={error}
        onNavigate={navigateToTransaction}
        onUnlink={handleUnlink}
      />
    </>
  );
}
```

---

## Testing Guidance

### Unit Tests

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { RelationshipPanel } from './RelationshipPanel';

describe('RelationshipPanel', () => {
  const mockRelationships: TransactionRelationship[] = [
    {
      relationship_id: 'rel_001',
      type: 'transfer',
      target_transaction: {
        canonical_id: 'txn_002',
        transaction_date: '2025-05-23',
        account: 'Checking ***1234',
        counterparty: 'Internal Transfer',
        amount: -500.00,
        currency: 'USD'
      },
      confidence_score: 92,
      detection_method: 'auto',
      created_at: '2025-05-23T10:00:00Z',
      created_by: 'system:relationship_detector_v1.0'
    }
  ];

  test('renders relationship card with correct data', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    expect(screen.getByText('Transfer')).toBeInTheDocument();
    expect(screen.getByText('Checking ***1234')).toBeInTheDocument();
    expect(screen.getByText('-$500.00')).toBeInTheDocument();
    expect(screen.getByText('92%')).toBeInTheDocument();
  });

  test('shows empty state when no relationships', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={[]}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    expect(screen.getByText('No linked transactions')).toBeInTheDocument();
  });

  test('calls onNavigate when card clicked', () => {
    const mockNavigate = jest.fn();
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={mockNavigate}
        onUnlink={jest.fn()}
      />
    );

    const card = screen.getByRole('button', { name: /transfer with checking/i });
    fireEvent.click(card);

    expect(mockNavigate).toHaveBeenCalledWith('txn_002');
  });

  test('shows unlink confirmation dialog', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    const unlinkButton = screen.getByRole('button', { name: /unlink/i });
    fireEvent.click(unlinkButton);

    expect(screen.getByText('Confirm Unlink')).toBeInTheDocument();
    expect(screen.getByText(/are you sure/i)).toBeInTheDocument();
  });

  test('calls onUnlink when confirmed', async () => {
    const mockUnlink = jest.fn().mockResolvedValue(undefined);
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={mockUnlink}
      />
    );

    // Open confirmation dialog
    const unlinkButton = screen.getByRole('button', { name: /unlink transaction/i });
    fireEvent.click(unlinkButton);

    // Confirm unlink
    const confirmButton = screen.getByRole('button', { name: 'Unlink' });
    fireEvent.click(confirmButton);

    await waitFor(() => {
      expect(mockUnlink).toHaveBeenCalledWith('rel_001');
    });
  });

  test('cancels unlink when cancelled', () => {
    const mockUnlink = jest.fn();
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={mockUnlink}
      />
    );

    // Open confirmation dialog
    const unlinkButton = screen.getByRole('button', { name: /unlink transaction/i });
    fireEvent.click(unlinkButton);

    // Cancel
    const cancelButton = screen.getByRole('button', { name: 'Cancel' });
    fireEvent.click(cancelButton);

    expect(mockUnlink).not.toHaveBeenCalled();
    expect(screen.queryByText('Confirm Unlink')).not.toBeInTheDocument();
  });

  test('hides unlink button when allowUnlink=false', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
        allowUnlink={false}
      />
    );

    expect(screen.queryByRole('button', { name: /unlink/i })).not.toBeInTheDocument();
  });

  test('hides confidence score when showConfidenceScores=false', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
        showConfidenceScores={false}
      />
    );

    expect(screen.queryByText('92%')).not.toBeInTheDocument();
  });

  test('displays loading skeleton', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={[]}
        loading={true}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    expect(screen.getByTestId('relationship-panel-skeleton')).toBeInTheDocument();
  });

  test('displays error message', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={[]}
        error="Failed to load relationships"
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    expect(screen.getByText(/failed to load relationships/i)).toBeInTheDocument();
  });
});
```

### Integration Tests

```tsx
describe('RelationshipPanel Integration', () => {
  test('full unlink flow', async () => {
    const mockFetch = jest.fn().mockResolvedValue({ ok: true });
    global.fetch = mockFetch;

    const { rerender } = render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={async (id) => {
          await fetch(`/api/relationships/${id}`, { method: 'DELETE' });
        }}
      />
    );

    // Click unlink
    const unlinkButton = screen.getByRole('button', { name: /unlink transaction/i });
    fireEvent.click(unlinkButton);

    // Confirm
    const confirmButton = screen.getByRole('button', { name: 'Unlink' });
    fireEvent.click(confirmButton);

    // Verify API call
    await waitFor(() => {
      expect(mockFetch).toHaveBeenCalledWith(
        '/api/relationships/rel_001',
        { method: 'DELETE' }
      );
    });

    // Rerender with empty relationships
    rerender(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={[]}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    // Verify empty state
    expect(screen.getByText('No linked transactions')).toBeInTheDocument();
  });

  test('keyboard navigation', () => {
    render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    const card = screen.getByRole('button', { name: /transfer with checking/i });
    card.focus();

    fireEvent.keyDown(card, { key: 'Enter' });
    // Verify navigation called
  });
});
```

### Accessibility Tests

```tsx
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('RelationshipPanel Accessibility', () => {
  test('has no accessibility violations', async () => {
    const { container } = render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  test('unlink dialog has no violations', async () => {
    const { container } = render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    // Open dialog
    const unlinkButton = screen.getByRole('button', { name: /unlink transaction/i });
    fireEvent.click(unlinkButton);

    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

### Visual Regression Tests

```tsx
import { render } from '@testing-library/react';
import { toMatchImageSnapshot } from 'jest-image-snapshot';

expect.extend({ toMatchImageSnapshot });

describe('RelationshipPanel Visual Regression', () => {
  test('renders correctly with single relationship', () => {
    const { container } = render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={[mockRelationships[0]]}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    expect(container).toMatchImageSnapshot();
  });

  test('renders correctly with multiple relationships', () => {
    const { container } = render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={mockRelationships}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    expect(container).toMatchImageSnapshot();
  });

  test('renders empty state correctly', () => {
    const { container } = render(
      <RelationshipPanel
        transactionId="txn_001"
        relationships={[]}
        onNavigate={jest.fn()}
        onUnlink={jest.fn()}
      />
    );

    expect(container).toMatchImageSnapshot();
  });
});
```

---

## Performance Considerations

### Optimization Strategies

```tsx
// Memoize relationship cards to prevent unnecessary re-renders
const RelationshipCard = React.memo<RelationshipCardProps>(({ relationship, ...props }) => {
  // Component implementation
}, (prevProps, nextProps) => {
  return (
    prevProps.relationship.relationship_id === nextProps.relationship.relationship_id &&
    prevProps.isUnlinking === nextProps.isUnlinking
  );
});

// Virtualize long lists of relationships (100+)
import { FixedSizeList } from 'react-window';

const VirtualizedRelationshipList: React.FC = ({ relationships }) => {
  return (
    <FixedSizeList
      height={400}
      itemCount={relationships.length}
      itemSize={120}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <RelationshipCard relationship={relationships[index]} />
        </div>
      )}
    </FixedSizeList>
  );
};
```

### Lazy Loading

```tsx
// Load relationships on demand (not immediately)
const useLazyRelationships = (transactionId: string) => {
  const [relationships, setRelationships] = useState<TransactionRelationship[]>([]);
  const [loading, setLoading] = useState(false);
  const [loaded, setLoaded] = useState(false);

  const loadRelationships = useCallback(async () => {
    if (loaded) return;

    setLoading(true);
    try {
      const response = await fetch(`/api/transactions/${transactionId}/relationships`);
      const data = await response.json();
      setRelationships(data);
      setLoaded(true);
    } finally {
      setLoading(false);
    }
  }, [transactionId, loaded]);

  return { relationships, loading, loadRelationships };
};

// Usage: Load relationships when panel expanded
function TransactionDetailView() {
  const [panelExpanded, setPanelExpanded] = useState(false);
  const { relationships, loading, loadRelationships } = useLazyRelationships(transactionId);

  useEffect(() => {
    if (panelExpanded) {
      loadRelationships();
    }
  }, [panelExpanded]);

  return (
    <Collapsible
      trigger="Linked Transactions"
      onOpen={() => setPanelExpanded(true)}
    >
      <RelationshipPanel
        transactionId={transactionId}
        relationships={relationships}
        loading={loading}
        onNavigate={navigateToTransaction}
        onUnlink={unlinkRelationship}
      />
    </Collapsible>
  );
}
```

---

## API Integration

### Backend Endpoint Example

```typescript
// GET /api/transactions/:id/relationships
// Returns array of linked transactions

interface GetRelationshipsResponse {
  relationships: TransactionRelationship[];
  total_count: number;
}

app.get('/api/transactions/:id/relationships', async (req, res) => {
  const { id } = req.params;

  const relationships = await db.query(`
    SELECT
      r.relationship_id,
      r.type,
      r.confidence_score,
      r.detection_method,
      r.created_at,
      r.created_by,
      t.canonical_id,
      t.transaction_date,
      t.account,
      t.counterparty,
      t.amount,
      t.currency,
      t.description
    FROM transaction_relationships r
    JOIN transactions t ON r.target_transaction_id = t.canonical_id
    WHERE r.source_transaction_id = $1
    ORDER BY r.created_at DESC
  `, [id]);

  res.json({
    relationships: relationships.rows,
    total_count: relationships.rowCount
  });
});

// DELETE /api/relationships/:id
// Removes relationship link

app.delete('/api/relationships/:id', async (req, res) => {
  const { id } = req.params;

  await db.query(`
    DELETE FROM transaction_relationships
    WHERE relationship_id = $1
  `, [id]);

  res.json({ success: true });
});
```

---

## File Locations

**Component:**
- `/Users/darwinborges/Description/docs/primitives/il/RelationshipPanel.md` (this file)

**Implementation (future):**
- `apps/web/components/RelationshipPanel.tsx`
- `apps/web/components/RelationshipPanel.test.tsx`
- `apps/web/components/RelationshipPanel.module.css`

**Related:**
- `apps/web/components/DrillDownPanel.tsx` (uses RelationshipPanel)
- `apps/web/components/TransactionDetailView.tsx` (uses RelationshipPanel)
- `apps/web/api/relationships/[id]/route.ts` (backend API)
