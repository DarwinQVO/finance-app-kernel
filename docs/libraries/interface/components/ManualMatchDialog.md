# ManualMatchDialog (IL Component)

## Definition

**ManualMatchDialog** is a modal dialog component for manually creating reconciliation matches when auto-detection fails. It provides searchable item selection from both sources, supports one-to-one, one-to-many, and many-to-one cardinality, validates match feasibility, and requires confirmation notes for non-obvious matches.

**Problem it solves:**
- Auto-detection misses matches with large differences (amount, date, description)
- Users cannot manually link items when auto fails
- No UI for creating one-to-many or many-to-one matches (split payments)
- Cannot search across large item lists to find match candidates
- No validation before creating match (prevents impossible matches)
- Missing audit trail for why manual match was created

**Solution:**
- Dual-pane item selector (source 1 on left, source 2 on right)
- Search/filter within each source with multiple criteria
- Multi-select for one-to-many/many-to-one matches
- Real-time validation showing why match is/isn't allowed
- Preview pane showing selected items before confirmation
- Required notes field for manual matches (explains reasoning)
- Cardinality auto-detected based on selections
- Keyboard shortcuts (Tab to switch panes, Enter to search, Escape to cancel)

---

## Interface Contract

```typescript
interface ManualMatchDialogProps {
  // Source data
  sourceItem: MatchableItem;  // Pre-selected item from one source
  availableTargets: MatchableItem[];  // Items from other source

  // Which source is pre-selected
  sourceType: "source_1" | "source_2";

  // Callbacks
  onMatch: (matchData: ManualMatchRequest) => Promise<void>;
  onCancel: () => void;

  // Optional search function (if not provided, uses local filtering)
  onSearch?: (query: SearchQuery, source: "source_1" | "source_2") => Promise<MatchableItem[]>;

  // UI states
  isOpen: boolean;
  loading?: boolean;
  error?: string | null;

  // Display options
  theme?: "light" | "dark";
  allowMultiSelect?: boolean;  // Allow one-to-many/many-to-one (default: true)
  maxTargets?: number;  // Max items to select (default: 10)
}

interface MatchableItem {
  item_id: string;
  display_text: string;
  amount?: number;
  date?: string;
  metadata: Record<string, any>;
}

interface ManualMatchRequest {
  source_1_items: string[];  // Item IDs
  source_2_items: string[];  // Item IDs
  cardinality: "one_to_one" | "one_to_many" | "many_to_one" | "many_to_many";
  notes: string;  // Required for manual matches
}

interface SearchQuery {
  text?: string;
  amount_min?: number;
  amount_max?: number;
  date_from?: string;  // ISO date
  date_to?: string;  // ISO date
}

type DialogState = "idle" | "searching" | "selecting" | "confirming" | "creating" | "error";
```

---

## Component Structure

```tsx
import React, { useState, useEffect, useMemo } from 'react';

export const ManualMatchDialog: React.FC<ManualMatchDialogProps> = ({
  sourceItem,
  availableTargets,
  sourceType,
  onMatch,
  onCancel,
  onSearch,
  isOpen,
  loading = false,
  error = null,
  theme = "light",
  allowMultiSelect = true,
  maxTargets = 10
}) => {
  const [state, setState] = useState<DialogState>("idle");
  const [searchQuery, setSearchQuery] = useState<SearchQuery>({});
  const [searchText, setSearchText] = useState("");
  const [filteredTargets, setFilteredTargets] = useState<MatchableItem[]>(availableTargets);
  const [selectedTargets, setSelectedTargets] = useState<Set<string>>(new Set());
  const [notes, setNotes] = useState("");
  const [validationErrors, setValidationErrors] = useState<string[]>([]);

  // Calculate cardinality based on selections
  const cardinality = useMemo<ManualMatchRequest["cardinality"]>(() => {
    const targetCount = selectedTargets.size;

    if (targetCount === 0) return "one_to_one";  // Default
    if (targetCount === 1) return "one_to_one";
    if (sourceType === "source_1") return "one_to_many";
    return "many_to_one";
  }, [selectedTargets, sourceType]);

  // Calculate totals for preview
  const totals = useMemo(() => {
    const sourceAmount = sourceItem.amount ?? 0;
    const targetAmount = Array.from(selectedTargets).reduce((sum, id) => {
      const target = availableTargets.find(t => t.item_id === id);
      return sum + (target?.amount ?? 0);
    }, 0);

    return {
      source: sourceAmount,
      target: targetAmount,
      diff: Math.abs(sourceAmount - targetAmount)
    };
  }, [sourceItem, selectedTargets, availableTargets]);

  // Reset state when dialog opens
  useEffect(() => {
    if (isOpen) {
      setState("idle");
      setSearchQuery({});
      setSearchText("");
      setFilteredTargets(availableTargets);
      setSelectedTargets(new Set());
      setNotes("");
      setValidationErrors([]);
    }
  }, [isOpen, availableTargets]);

  // Local search/filter
  useEffect(() => {
    if (!searchText && !searchQuery.amount_min && !searchQuery.date_from) {
      setFilteredTargets(availableTargets);
      return;
    }

    let filtered = [...availableTargets];

    // Text search
    if (searchText) {
      const lower = searchText.toLowerCase();
      filtered = filtered.filter(item =>
        item.display_text.toLowerCase().includes(lower)
      );
    }

    // Amount filter
    if (searchQuery.amount_min !== undefined) {
      filtered = filtered.filter(item =>
        item.amount !== undefined && item.amount >= searchQuery.amount_min!
      );
    }
    if (searchQuery.amount_max !== undefined) {
      filtered = filtered.filter(item =>
        item.amount !== undefined && item.amount <= searchQuery.amount_max!
      );
    }

    // Date filter
    if (searchQuery.date_from) {
      filtered = filtered.filter(item =>
        item.date && item.date >= searchQuery.date_from!
      );
    }
    if (searchQuery.date_to) {
      filtered = filtered.filter(item =>
        item.date && item.date <= searchQuery.date_to!
      );
    }

    setFilteredTargets(filtered);
  }, [searchText, searchQuery, availableTargets]);

  // Validate match before creation
  const validateMatch = (): string[] => {
    const errors: string[] = [];

    if (selectedTargets.size === 0) {
      errors.push("Select at least one item to match");
    }

    if (allowMultiSelect && selectedTargets.size > maxTargets) {
      errors.push(`Cannot select more than ${maxTargets} items`);
    }

    if (!notes.trim()) {
      errors.push("Notes are required for manual matches");
    }

    // Amount validation for one-to-one matches
    if (cardinality === "one_to_one" && sourceItem.amount !== undefined) {
      const target = availableTargets.find(t => selectedTargets.has(t.item_id));
      if (target?.amount !== undefined) {
        const diff = Math.abs(sourceItem.amount - target.amount);
        const variance = diff / Math.max(sourceItem.amount, target.amount);

        if (variance > 0.5) {  // Warn if >50% difference
          errors.push(
            `Large amount difference: $${sourceItem.amount.toFixed(2)} vs $${target.amount.toFixed(2)} (${Math.round(variance * 100)}%)`
          );
        }
      }
    }

    // Amount validation for one-to-many/many-to-one
    if (cardinality !== "one_to_one" && totals.diff > 0.01) {
      errors.push(
        `Amounts don't match: source $${totals.source.toFixed(2)} vs targets $${totals.target.toFixed(2)}`
      );
    }

    return errors;
  };

  // Handle target selection
  const handleSelectTarget = (itemId: string) => {
    if (!allowMultiSelect) {
      setSelectedTargets(new Set([itemId]));
      return;
    }

    setSelectedTargets(prev => {
      const next = new Set(prev);
      if (next.has(itemId)) {
        next.delete(itemId);
      } else {
        if (next.size >= maxTargets) {
          alert(`Cannot select more than ${maxTargets} items`);
          return prev;
        }
        next.add(itemId);
      }
      return next;
    });
  };

  // Handle match creation
  const handleCreateMatch = async () => {
    const errors = validateMatch();
    if (errors.length > 0) {
      setValidationErrors(errors);
      return;
    }

    setState("creating");
    setValidationErrors([]);

    try {
      const matchData: ManualMatchRequest = {
        source_1_items: sourceType === "source_1" ? [sourceItem.item_id] : Array.from(selectedTargets),
        source_2_items: sourceType === "source_2" ? [sourceItem.item_id] : Array.from(selectedTargets),
        cardinality,
        notes: notes.trim()
      };

      await onMatch(matchData);
      onCancel();  // Close on success
    } catch (err) {
      setState("error");
      setValidationErrors([
        err instanceof Error ? err.message : "Failed to create match"
      ]);
    }
  };

  // Keyboard shortcuts
  useEffect(() => {
    if (!isOpen) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Escape") {
        onCancel();
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [isOpen, onCancel]);

  if (!isOpen) return null;

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onCancel}>
      <div
        className="modal-content manual-match-dialog"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="modal-header">
          <h3>Create Manual Match</h3>
          <button className="close-button" onClick={onCancel} aria-label="Close">
            ×
          </button>
        </div>

        {/* Body */}
        <div className="modal-body">
          {/* Step 1: Source Item Display */}
          <div className="source-section">
            <div className="section-header">
              Source Item ({sourceType === "source_1" ? "Source 1" : "Source 2"})
            </div>
            <div className="source-item-card">
              <div className="item-text">{sourceItem.display_text}</div>
              {sourceItem.amount !== undefined && (
                <div className="item-amount">${sourceItem.amount.toFixed(2)}</div>
              )}
              {sourceItem.date && (
                <div className="item-date">{sourceItem.date}</div>
              )}
            </div>
          </div>

          {/* Step 2: Target Selection */}
          <div className="target-section">
            <div className="section-header">
              Match to ({sourceType === "source_1" ? "Source 2" : "Source 1"})
              {allowMultiSelect && (
                <span className="select-hint">
                  {selectedTargets.size > 0
                    ? `${selectedTargets.size} selected (max ${maxTargets})`
                    : "Select one or more items"}
                </span>
              )}
            </div>

            {/* Search/Filter */}
            <div className="search-section">
              <input
                type="text"
                className="search-input"
                placeholder="Search items..."
                value={searchText}
                onChange={(e) => setSearchText(e.target.value)}
              />

              <details className="filters-collapsible">
                <summary className="filters-toggle">Filters</summary>
                <div className="filters-content">
                  <div className="filter-group">
                    <label>Amount Range</label>
                    <div className="amount-inputs">
                      <input
                        type="number"
                        placeholder="Min"
                        value={searchQuery.amount_min ?? ""}
                        onChange={(e) =>
                          setSearchQuery(prev => ({
                            ...prev,
                            amount_min: e.target.value ? parseFloat(e.target.value) : undefined
                          }))
                        }
                        step="0.01"
                      />
                      <span>to</span>
                      <input
                        type="number"
                        placeholder="Max"
                        value={searchQuery.amount_max ?? ""}
                        onChange={(e) =>
                          setSearchQuery(prev => ({
                            ...prev,
                            amount_max: e.target.value ? parseFloat(e.target.value) : undefined
                          }))
                        }
                        step="0.01"
                      />
                    </div>
                  </div>

                  <div className="filter-group">
                    <label>Date Range</label>
                    <div className="date-inputs">
                      <input
                        type="date"
                        value={searchQuery.date_from ?? ""}
                        onChange={(e) =>
                          setSearchQuery(prev => ({
                            ...prev,
                            date_from: e.target.value || undefined
                          }))
                        }
                      />
                      <span>to</span>
                      <input
                        type="date"
                        value={searchQuery.date_to ?? ""}
                        onChange={(e) =>
                          setSearchQuery(prev => ({
                            ...prev,
                            date_to: e.target.value || undefined
                          }))
                        }
                      />
                    </div>
                  </div>
                </div>
              </details>
            </div>

            {/* Target Items List */}
            <div className="target-items-list">
              {filteredTargets.length === 0 ? (
                <div className="empty-state">
                  <div className="empty-icon">🔍</div>
                  <div className="empty-text">No items found</div>
                  <div className="empty-hint">Try adjusting your search or filters</div>
                </div>
              ) : (
                filteredTargets.map(item => (
                  <div
                    key={item.item_id}
                    className={`target-item ${selectedTargets.has(item.item_id) ? "selected" : ""}`}
                    onClick={() => handleSelectTarget(item.item_id)}
                  >
                    <input
                      type="checkbox"
                      checked={selectedTargets.has(item.item_id)}
                      onChange={() => {}}  // Handled by div onClick
                    />
                    <div className="item-content">
                      <div className="item-text">{item.display_text}</div>
                      <div className="item-meta">
                        {item.amount !== undefined && (
                          <span className="item-amount">${item.amount.toFixed(2)}</span>
                        )}
                        {item.date && (
                          <>
                            <span className="separator">•</span>
                            <span className="item-date">{item.date}</span>
                          </>
                        )}
                      </div>
                    </div>
                  </div>
                ))
              )}
            </div>
          </div>

          {/* Step 3: Preview */}
          {selectedTargets.size > 0 && (
            <div className="preview-section">
              <div className="section-header">Match Preview</div>

              <div className="preview-content">
                <div className="preview-row">
                  <span className="preview-label">Cardinality:</span>
                  <span className="preview-value">
                    {cardinality.replace(/_/g, "-")}
                  </span>
                </div>

                <div className="preview-row">
                  <span className="preview-label">Source Amount:</span>
                  <span className="preview-value">${totals.source.toFixed(2)}</span>
                </div>

                <div className="preview-row">
                  <span className="preview-label">Target Amount:</span>
                  <span className="preview-value">${totals.target.toFixed(2)}</span>
                </div>

                {totals.diff > 0.01 && (
                  <div className="preview-row warning">
                    <span className="preview-label">Difference:</span>
                    <span className="preview-value">${totals.diff.toFixed(2)}</span>
                  </div>
                )}
              </div>
            </div>
          )}

          {/* Step 4: Notes (Required) */}
          <div className="notes-section">
            <label className="notes-label" htmlFor="match-notes">
              Notes *
            </label>
            <textarea
              id="match-notes"
              className="notes-textarea"
              placeholder="Explain why you're creating this manual match (required)..."
              value={notes}
              onChange={(e) => setNotes(e.target.value)}
              rows={3}
              required
            />
            <div className="notes-hint">
              Required: Explain why auto-detection missed this match
            </div>
          </div>

          {/* Validation Errors */}
          {validationErrors.length > 0 && (
            <div className="validation-errors">
              {validationErrors.map((error, idx) => (
                <div key={idx} className="error-item">
                  <span className="error-icon">⚠️</span>
                  <span className="error-text">{error}</span>
                </div>
              ))}
            </div>
          )}
        </div>

        {/* Footer */}
        <div className="modal-footer">
          <button
            className="cancel-button"
            onClick={onCancel}
            disabled={state === "creating"}
          >
            Cancel
          </button>
          <button
            className="create-button"
            onClick={handleCreateMatch}
            disabled={
              state === "creating" ||
              selectedTargets.size === 0 ||
              !notes.trim()
            }
          >
            {state === "creating" ? "Creating Match..." : "Create Match"}
          </button>
        </div>
      </div>
    </div>
  );
};
```

---

## Visual Wireframes

### Initial State (One-to-One)

```
┌────────────────────────────────────────────────────────────┐
│ Create Manual Match                                    [×] │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Source Item (Source 1)                                     │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Payment to Vendor B                                │    │
│ │ $2,345.67 • Oct 18, 2024                           │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ Match to (Source 2)                   Select one or more items│
│ ┌────────────────────────────────────────────────────┐    │
│ │ Search items...                                    │    │
│ └────────────────────────────────────────────────────┘    │
│ ▸ Filters                                                  │
│                                                            │
│ ┌────────────────────────────────────────────────────┐    │
│ │ □ Wire Transfer Received        $2,345.67 • Oct 18 │    │
│ │ □ Customer Refund               $456.78 • Oct 19   │    │
│ │ □ Monthly Service Charge        $15.00 • Oct 20    │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ Notes *                                                    │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Explain why you're creating this manual match...  │    │
│ │                                                    │    │
│ └────────────────────────────────────────────────────┘    │
│ Required: Explain why auto-detection missed this match    │
│                                                            │
│                              [Cancel]  [Create Match]     │
└────────────────────────────────────────────────────────────┘
```

### One-to-Many Selection with Preview

```
┌────────────────────────────────────────────────────────────┐
│ Create Manual Match                                    [×] │
├────────────────────────────────────────────────────────────┤
│ Source Item (Source 1)                                     │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Bulk Deposit                                       │    │
│ │ $5,000.00 • Oct 20, 2024                           │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ Match to (Source 2)                   2 selected (max 10)  │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Search items...                                    │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ ┌────────────────────────────────────────────────────┐    │
│ │ ✓ Payment Received - Client A   $3,000.00 • Oct 20│    │
│ │ ✓ Payment Received - Client B   $2,000.00 • Oct 20│    │
│ │ □ Refund Processed              $150.00 • Oct 21   │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ Match Preview                                              │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Cardinality:       one-to-many                     │    │
│ │ Source Amount:     $5,000.00                       │    │
│ │ Target Amount:     $5,000.00                       │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│ Notes *                                                    │
│ ┌────────────────────────────────────────────────────┐    │
│ │ Single bank deposit split into two ledger entries │    │
│ │ for different clients                              │    │
│ └────────────────────────────────────────────────────┘    │
│                                                            │
│                              [Cancel]  [Create Match]     │
└────────────────────────────────────────────────────────────┘
```

---

## Multi-Domain Examples

### Finance: Manual Payment Allocation

```tsx
<ManualMatchDialog
  sourceItem={{
    item_id: "bank_txn_005",
    display_text: "Bulk Deposit",
    amount: 5000.00,
    date: "2024-10-20",
    metadata: { account: "Operating Account" }
  }}
  availableTargets={[
    {
      item_id: "ledger_010",
      display_text: "Payment Received - Client A",
      amount: 3000.00,
      date: "2024-10-20",
      metadata: { client: "Client A", invoice: "INV-001" }
    },
    {
      item_id: "ledger_011",
      display_text: "Payment Received - Client B",
      amount: 2000.00,
      date: "2024-10-20",
      metadata: { client: "Client B", invoice: "INV-002" }
    }
  ]}
  sourceType="source_1"
  onMatch={async (data) => console.log("Created match:", data)}
  onCancel={() => console.log("Cancelled")}
  isOpen={true}
/>
```

### E-commerce: Order-Shipment Split

```tsx
<ManualMatchDialog
  sourceItem={{
    item_id: "order_001",
    display_text: "Order #12345 - 10 items",
    amount: 299.99,
    date: "2024-10-15",
    metadata: { customer: "John Doe", total_items: 10 }
  }}
  availableTargets={[
    {
      item_id: "shipment_001",
      display_text: "Shipment #SHIP-001 - 6 items",
      amount: 179.99,
      date: "2024-10-16",
      metadata: { carrier: "UPS", tracking: "1Z999AA1" }
    },
    {
      item_id: "shipment_002",
      display_text: "Shipment #SHIP-002 - 4 items",
      amount: 120.00,
      date: "2024-10-18",
      metadata: { carrier: "UPS", tracking: "1Z999AA2" }
    }
  ]}
  sourceType="source_1"
  // ... other props
/>
```

---

## Related Components

- **ReconciliationDashboard** (IL) - Parent component
- **MatchReviewDialog** (IL) - Review suggested matches
- **ReconciliationStore** (OL) - Data persistence
- **ReconciliationEngine** (OL) - Auto-detection

---

**Status:** ✅ Ready for implementation
