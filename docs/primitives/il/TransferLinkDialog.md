# TransferLinkDialog (IL Component)

## Definition

**TransferLinkDialog** is a modal dialog component for manually creating relationship links between transactions. It provides transaction search, relationship type selection, validation, and preview capabilities to safely establish connections like transfers, reimbursements, FX conversions, and corrections.

**Problem it solves:**
- No standardized UI for manually linking related transactions
- Users unsure how to find and connect transactions across accounts/time
- No preview of link before creation (risk of incorrect links)
- Duplicate links created accidentally
- Self-links (transaction to itself) not prevented
- No validation of user ownership before linking
- Unclear relationship semantics (transfer vs reimbursement vs correction)
- Cannot annotate non-obvious relationships with notes

**Solution:**
- Searchable transaction list with filters (date, account, amount, text)
- Clear relationship type selector with semantic labels
- Preview link before creation with both transactions visible
- Validation prevents duplicate links, self-links, and unauthorized links
- Required notes field for "other" relationship type
- Loading and error states for async operations
- Keyboard navigation (Esc to close, Tab to navigate, Enter to search)
- Multi-step workflow: Search → Select → Preview → Create

---

## Interface Contract

```typescript
interface TransferLinkDialogProps {
  // Current transaction context
  currentTransaction: Transaction;
  isOpen: boolean;

  // Callbacks
  onClose: () => void;
  onCreate: (linkData: TransactionLinkCreate) => Promise<void>;

  // Search function (provided by parent)
  onSearchTransactions: (query: SearchQuery) => Promise<Transaction[]>;

  // Existing links (for duplicate detection)
  existingLinks?: TransactionLink[];

  // UI states
  loading?: boolean;
  error?: string | null;

  // Display options
  theme?: "light" | "dark";
  maxSearchResults?: number;  // Default: 50
}

interface Transaction {
  transaction_id: string;
  amount: number;  // Negative for expenses, positive for income
  currency: string;
  date: string;  // ISO date
  description: string;
  account_name: string;
  counterparty_name?: string;
  category?: string;
}

interface TransactionLink {
  link_id: string;
  transaction_a_id: string;
  transaction_b_id: string;
  relationship_type: RelationshipType;
  notes?: string;
  created_at: string;
}

interface TransactionLinkCreate {
  transaction_a_id: string;  // Current transaction
  transaction_b_id: string;  // Selected transaction
  relationship_type: RelationshipType;
  notes?: string;  // Required if relationship_type is "other"
}

type RelationshipType =
  | "transfer"           // Money moved between own accounts
  | "fx_conversion"      // Currency conversion (same person, different currency)
  | "reimbursement"      // Expense reimbursed by employer/friend
  | "split"              // Transaction split across multiple records
  | "correction"         // Correcting a duplicate or error
  | "other";             // Other relationship (requires notes)

interface SearchQuery {
  text?: string;                    // Search description/counterparty
  date_from?: string;               // ISO date
  date_to?: string;                 // ISO date
  account_ids?: string[];           // Filter by accounts
  amount_min?: number;              // Filter by amount range
  amount_max?: number;              // Filter by amount range
  exclude_transaction_id?: string;  // Exclude current transaction
}

interface RelationshipTypeOption {
  type: RelationshipType;
  label: string;
  description: string;
  icon: string;
  requiresNotes: boolean;
}
```

---

## Component Structure

```tsx
import React, { useState, useEffect, useMemo } from 'react';

export const TransferLinkDialog: React.FC<TransferLinkDialogProps> = ({
  currentTransaction,
  isOpen,
  onClose,
  onCreate,
  onSearchTransactions,
  existingLinks = [],
  loading = false,
  error = null,
  theme = "light",
  maxSearchResults = 50
}) => {
  // State management
  const [step, setStep] = useState<"search" | "preview">("search");
  const [searchQuery, setSearchQuery] = useState<SearchQuery>({
    exclude_transaction_id: currentTransaction.transaction_id
  });
  const [searchText, setSearchText] = useState("");
  const [searching, setSearching] = useState(false);
  const [searchResults, setSearchResults] = useState<Transaction[]>([]);
  const [selectedTransaction, setSelectedTransaction] = useState<Transaction | null>(null);
  const [relationshipType, setRelationshipType] = useState<RelationshipType>("transfer");
  const [notes, setNotes] = useState("");
  const [creating, setCreating] = useState(false);
  const [validationErrors, setValidationErrors] = useState<string[]>([]);

  // Relationship type options
  const relationshipTypes: RelationshipTypeOption[] = [
    {
      type: "transfer",
      label: "Transfer",
      description: "Money moved between your own accounts",
      icon: "🔄",
      requiresNotes: false
    },
    {
      type: "fx_conversion",
      label: "FX Conversion",
      description: "Currency exchange between accounts",
      icon: "💱",
      requiresNotes: false
    },
    {
      type: "reimbursement",
      label: "Reimbursement",
      description: "Expense reimbursed by employer or friend",
      icon: "💰",
      requiresNotes: false
    },
    {
      type: "split",
      label: "Split Transaction",
      description: "Transaction split across multiple records",
      icon: "✂️",
      requiresNotes: false
    },
    {
      type: "correction",
      label: "Correction",
      description: "Correcting a duplicate or error entry",
      icon: "🔧",
      requiresNotes: false
    },
    {
      type: "other",
      label: "Other",
      description: "Custom relationship (explain in notes)",
      icon: "📝",
      requiresNotes: true
    }
  ];

  // Reset state when dialog opens
  useEffect(() => {
    if (isOpen) {
      setStep("search");
      setSearchQuery({
        exclude_transaction_id: currentTransaction.transaction_id
      });
      setSearchText("");
      setSearchResults([]);
      setSelectedTransaction(null);
      setRelationshipType("transfer");
      setNotes("");
      setValidationErrors([]);
    }
  }, [isOpen, currentTransaction.transaction_id]);

  // Validate link before creation
  const validateLink = (targetTransaction: Transaction): string[] => {
    const errors: string[] = [];

    // Prevent self-linking
    if (targetTransaction.transaction_id === currentTransaction.transaction_id) {
      errors.push("Cannot link a transaction to itself");
    }

    // Check for duplicate links
    const isDuplicate = existingLinks.some(link =>
      (link.transaction_a_id === currentTransaction.transaction_id &&
       link.transaction_b_id === targetTransaction.transaction_id) ||
      (link.transaction_b_id === currentTransaction.transaction_id &&
       link.transaction_a_id === targetTransaction.transaction_id)
    );

    if (isDuplicate) {
      errors.push("A link already exists between these transactions");
    }

    // Validate notes for "other" type
    if (relationshipType === "other" && !notes.trim()) {
      errors.push("Notes are required when relationship type is 'Other'");
    }

    // Validate amount consistency for transfers
    if (relationshipType === "transfer") {
      const amountA = Math.abs(currentTransaction.amount);
      const amountB = Math.abs(targetTransaction.amount);
      const variance = Math.abs(amountA - amountB) / Math.max(amountA, amountB);

      if (variance > 0.1) {  // Allow 10% variance for transfer fees
        errors.push(
          `Transfer amounts differ significantly: ` +
          `${formatCurrency(amountA, currentTransaction.currency)} vs ` +
          `${formatCurrency(amountB, targetTransaction.currency)} ` +
          `(${Math.round(variance * 100)}% difference)`
        );
      }
    }

    return errors;
  };

  // Handle search
  const handleSearch = async () => {
    setSearching(true);
    setSearchResults([]);
    setValidationErrors([]);

    try {
      const query: SearchQuery = {
        ...searchQuery,
        text: searchText.trim() || undefined,
        exclude_transaction_id: currentTransaction.transaction_id
      };

      const results = await onSearchTransactions(query);
      setSearchResults(results.slice(0, maxSearchResults));

      if (results.length === 0) {
        setValidationErrors(["No transactions found matching your search"]);
      }
    } catch (err) {
      setValidationErrors([
        err instanceof Error ? err.message : "Search failed"
      ]);
    } finally {
      setSearching(false);
    }
  };

  // Handle transaction selection
  const handleSelectTransaction = (transaction: Transaction) => {
    setSelectedTransaction(transaction);
    setStep("preview");
    setValidationErrors([]);
  };

  // Handle link creation
  const handleCreateLink = async () => {
    if (!selectedTransaction) return;

    // Validate
    const errors = validateLink(selectedTransaction);
    if (errors.length > 0) {
      setValidationErrors(errors);
      return;
    }

    setCreating(true);
    setValidationErrors([]);

    try {
      const linkData: TransactionLinkCreate = {
        transaction_a_id: currentTransaction.transaction_id,
        transaction_b_id: selectedTransaction.transaction_id,
        relationship_type: relationshipType,
        notes: notes.trim() || undefined
      };

      await onCreate(linkData);
      onClose();
    } catch (err) {
      setValidationErrors([
        err instanceof Error ? err.message : "Failed to create link"
      ]);
    } finally {
      setCreating(false);
    }
  };

  // Handle keyboard shortcuts
  useEffect(() => {
    if (!isOpen) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      } else if (e.key === 'Enter' && step === "search" && !searching) {
        e.preventDefault();
        handleSearch();
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, step, searching, onClose]);

  // Format currency
  const formatCurrency = (amount: number, currency: string): string => {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency,
      minimumFractionDigits: 2
    }).format(Math.abs(amount));
  };

  // Format date
  const formatDate = (dateStr: string): string => {
    const date = new Date(dateStr);
    return date.toLocaleDateString('en-US', {
      month: 'short',
      day: 'numeric',
      year: 'numeric'
    });
  };

  // Get selected relationship type details
  const selectedRelationType = useMemo(() =>
    relationshipTypes.find(rt => rt.type === relationshipType),
    [relationshipType]
  );

  if (!isOpen) return null;

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div
        className="modal-content transfer-link-dialog"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="modal-header">
          <div className="header-content">
            <h3>Link Transaction</h3>
            <div className="current-transaction-context">
              <div className="context-row">
                <span className="context-label">Current:</span>
                <span className="context-value">
                  {currentTransaction.description}
                </span>
              </div>
              <div className="context-meta">
                {formatCurrency(currentTransaction.amount, currentTransaction.currency)} •{' '}
                {formatDate(currentTransaction.date)} •{' '}
                {currentTransaction.account_name}
              </div>
            </div>
          </div>
          <button
            className="close-button"
            onClick={onClose}
            aria-label="Close dialog"
          >
            ×
          </button>
        </div>

        {/* Body */}
        <div className="modal-body">
          {step === "search" && (
            <SearchStep
              searchText={searchText}
              onSearchTextChange={setSearchText}
              searchQuery={searchQuery}
              onSearchQueryChange={setSearchQuery}
              onSearch={handleSearch}
              searching={searching}
              searchResults={searchResults}
              onSelectTransaction={handleSelectTransaction}
              validationErrors={validationErrors}
              formatCurrency={formatCurrency}
              formatDate={formatDate}
              theme={theme}
            />
          )}

          {step === "preview" && selectedTransaction && (
            <PreviewStep
              currentTransaction={currentTransaction}
              selectedTransaction={selectedTransaction}
              relationshipType={relationshipType}
              relationshipTypes={relationshipTypes}
              onRelationshipTypeChange={setRelationshipType}
              notes={notes}
              onNotesChange={setNotes}
              validationErrors={validationErrors}
              formatCurrency={formatCurrency}
              formatDate={formatDate}
              theme={theme}
            />
          )}
        </div>

        {/* Footer */}
        <div className="modal-footer">
          {step === "search" && (
            <>
              <button
                className="cancel-button"
                onClick={onClose}
                disabled={searching}
              >
                Cancel
              </button>
              <button
                className="search-button"
                onClick={handleSearch}
                disabled={searching || !searchText.trim()}
              >
                {searching ? "Searching..." : "Search Transactions"}
              </button>
            </>
          )}

          {step === "preview" && (
            <>
              <button
                className="back-button"
                onClick={() => setStep("search")}
                disabled={creating}
              >
                Back
              </button>
              <button
                className="cancel-button"
                onClick={onClose}
                disabled={creating}
              >
                Cancel
              </button>
              <button
                className="create-button"
                onClick={handleCreateLink}
                disabled={
                  creating ||
                  (relationshipType === "other" && !notes.trim()) ||
                  validationErrors.length > 0
                }
              >
                {creating ? "Creating Link..." : "Create Link"}
              </button>
            </>
          )}
        </div>
      </div>
    </div>
  );
};

// Search Step Component
const SearchStep: React.FC<{
  searchText: string;
  onSearchTextChange: (value: string) => void;
  searchQuery: SearchQuery;
  onSearchQueryChange: (query: SearchQuery) => void;
  onSearch: () => void;
  searching: boolean;
  searchResults: Transaction[];
  onSelectTransaction: (transaction: Transaction) => void;
  validationErrors: string[];
  formatCurrency: (amount: number, currency: string) => string;
  formatDate: (date: string) => string;
  theme: "light" | "dark";
}> = ({
  searchText,
  onSearchTextChange,
  searchQuery,
  onSearchQueryChange,
  onSearch,
  searching,
  searchResults,
  onSelectTransaction,
  validationErrors,
  formatCurrency,
  formatDate,
  theme
}) => {
  return (
    <>
      {/* Search Input */}
      <div className="search-section">
        <div className="search-input-group">
          <input
            type="text"
            className="search-input"
            placeholder="Search by description, counterparty, or account..."
            value={searchText}
            onChange={(e) => onSearchTextChange(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === 'Enter' && !searching) {
                e.preventDefault();
                onSearch();
              }
            }}
            autoFocus
          />
          <button
            className="search-icon-button"
            onClick={onSearch}
            disabled={searching || !searchText.trim()}
            aria-label="Search"
          >
            🔍
          </button>
        </div>

        <div className="search-hint">
          Search for the transaction you want to link. Press Enter to search.
        </div>
      </div>

      {/* Filters */}
      <div className="filters-section">
        <details className="filters-collapsible">
          <summary className="filters-toggle">
            Advanced Filters
          </summary>
          <div className="filters-content">
            {/* Date Range */}
            <div className="filter-group">
              <label className="filter-label">Date Range</label>
              <div className="date-range-inputs">
                <input
                  type="date"
                  className="filter-input"
                  value={searchQuery.date_from || ""}
                  onChange={(e) =>
                    onSearchQueryChange({
                      ...searchQuery,
                      date_from: e.target.value || undefined
                    })
                  }
                  placeholder="From"
                />
                <span className="date-separator">to</span>
                <input
                  type="date"
                  className="filter-input"
                  value={searchQuery.date_to || ""}
                  onChange={(e) =>
                    onSearchQueryChange({
                      ...searchQuery,
                      date_to: e.target.value || undefined
                    })
                  }
                  placeholder="To"
                />
              </div>
            </div>

            {/* Amount Range */}
            <div className="filter-group">
              <label className="filter-label">Amount Range</label>
              <div className="amount-range-inputs">
                <input
                  type="number"
                  className="filter-input"
                  value={searchQuery.amount_min || ""}
                  onChange={(e) =>
                    onSearchQueryChange({
                      ...searchQuery,
                      amount_min: e.target.value ? parseFloat(e.target.value) : undefined
                    })
                  }
                  placeholder="Min"
                  step="0.01"
                />
                <span className="amount-separator">to</span>
                <input
                  type="number"
                  className="filter-input"
                  value={searchQuery.amount_max || ""}
                  onChange={(e) =>
                    onSearchQueryChange({
                      ...searchQuery,
                      amount_max: e.target.value ? parseFloat(e.target.value) : undefined
                    })
                  }
                  placeholder="Max"
                  step="0.01"
                />
              </div>
            </div>
          </div>
        </details>
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

      {/* Loading State */}
      {searching && (
        <div className="searching-state">
          <div className="spinner"></div>
          <div className="searching-text">Searching transactions...</div>
        </div>
      )}

      {/* Search Results */}
      {!searching && searchResults.length > 0 && (
        <div className="search-results">
          <div className="results-header">
            Found {searchResults.length} transaction{searchResults.length > 1 ? 's' : ''}
          </div>
          <div className="results-list">
            {searchResults.map(transaction => (
              <div
                key={transaction.transaction_id}
                className="result-item"
                onClick={() => onSelectTransaction(transaction)}
              >
                <div className="result-content">
                  <div className="result-main">
                    <div className="result-description">
                      {transaction.description}
                    </div>
                    <div className={`result-amount ${transaction.amount < 0 ? 'expense' : 'income'}`}>
                      {transaction.amount < 0 ? '-' : '+'}
                      {formatCurrency(transaction.amount, transaction.currency)}
                    </div>
                  </div>
                  <div className="result-meta">
                    <span className="result-date">{formatDate(transaction.date)}</span>
                    <span className="result-separator">•</span>
                    <span className="result-account">{transaction.account_name}</span>
                    {transaction.counterparty_name && (
                      <>
                        <span className="result-separator">•</span>
                        <span className="result-counterparty">{transaction.counterparty_name}</span>
                      </>
                    )}
                  </div>
                </div>
                <div className="result-action">
                  <span className="select-icon">→</span>
                </div>
              </div>
            ))}
          </div>
        </div>
      )}

      {/* Empty State */}
      {!searching && searchResults.length === 0 && searchText && (
        <div className="empty-state">
          <div className="empty-icon">🔍</div>
          <div className="empty-text">
            No transactions found matching "{searchText}"
          </div>
          <div className="empty-hint">
            Try adjusting your search terms or filters
          </div>
        </div>
      )}
    </>
  );
};

// Preview Step Component
const PreviewStep: React.FC<{
  currentTransaction: Transaction;
  selectedTransaction: Transaction;
  relationshipType: RelationshipType;
  relationshipTypes: RelationshipTypeOption[];
  onRelationshipTypeChange: (type: RelationshipType) => void;
  notes: string;
  onNotesChange: (value: string) => void;
  validationErrors: string[];
  formatCurrency: (amount: number, currency: string) => string;
  formatDate: (date: string) => string;
  theme: "light" | "dark";
}> = ({
  currentTransaction,
  selectedTransaction,
  relationshipType,
  relationshipTypes,
  onRelationshipTypeChange,
  notes,
  onNotesChange,
  validationErrors,
  formatCurrency,
  formatDate,
  theme
}) => {
  const selectedType = relationshipTypes.find(rt => rt.type === relationshipType);

  return (
    <>
      {/* Preview Section */}
      <div className="preview-section">
        <div className="preview-header">Preview Link</div>

        <div className="preview-transactions">
          {/* Transaction A (Current) */}
          <div className="preview-transaction current">
            <div className="preview-transaction-badge">Current Transaction</div>
            <div className="preview-transaction-content">
              <div className="preview-transaction-description">
                {currentTransaction.description}
              </div>
              <div className="preview-transaction-details">
                <div className="preview-detail-row">
                  <span className="detail-label">Amount:</span>
                  <span className={`detail-value ${currentTransaction.amount < 0 ? 'expense' : 'income'}`}>
                    {currentTransaction.amount < 0 ? '-' : '+'}
                    {formatCurrency(currentTransaction.amount, currentTransaction.currency)}
                  </span>
                </div>
                <div className="preview-detail-row">
                  <span className="detail-label">Date:</span>
                  <span className="detail-value">{formatDate(currentTransaction.date)}</span>
                </div>
                <div className="preview-detail-row">
                  <span className="detail-label">Account:</span>
                  <span className="detail-value">{currentTransaction.account_name}</span>
                </div>
              </div>
            </div>
          </div>

          {/* Link Arrow */}
          <div className="link-arrow">
            <div className="arrow-icon">
              {selectedType?.icon || "🔗"}
            </div>
            <div className="arrow-label">{selectedType?.label || "Link"}</div>
          </div>

          {/* Transaction B (Selected) */}
          <div className="preview-transaction selected">
            <div className="preview-transaction-badge">Selected Transaction</div>
            <div className="preview-transaction-content">
              <div className="preview-transaction-description">
                {selectedTransaction.description}
              </div>
              <div className="preview-transaction-details">
                <div className="preview-detail-row">
                  <span className="detail-label">Amount:</span>
                  <span className={`detail-value ${selectedTransaction.amount < 0 ? 'expense' : 'income'}`}>
                    {selectedTransaction.amount < 0 ? '-' : '+'}
                    {formatCurrency(selectedTransaction.amount, selectedTransaction.currency)}
                  </span>
                </div>
                <div className="preview-detail-row">
                  <span className="detail-label">Date:</span>
                  <span className="detail-value">{formatDate(selectedTransaction.date)}</span>
                </div>
                <div className="preview-detail-row">
                  <span className="detail-label">Account:</span>
                  <span className="detail-value">{selectedTransaction.account_name}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      {/* Relationship Type Selector */}
      <div className="relationship-section">
        <div className="relationship-header">Relationship Type</div>
        <div className="relationship-types">
          {relationshipTypes.map(rt => (
            <label
              key={rt.type}
              className={`relationship-type-option ${relationshipType === rt.type ? 'selected' : ''}`}
            >
              <input
                type="radio"
                name="relationship-type"
                value={rt.type}
                checked={relationshipType === rt.type}
                onChange={() => onRelationshipTypeChange(rt.type)}
                className="relationship-radio"
              />
              <div className="relationship-content">
                <div className="relationship-icon">{rt.icon}</div>
                <div className="relationship-details">
                  <div className="relationship-label">{rt.label}</div>
                  <div className="relationship-description">{rt.description}</div>
                </div>
              </div>
            </label>
          ))}
        </div>
      </div>

      {/* Notes Field */}
      <div className="notes-section">
        <label className="notes-label" htmlFor="link-notes">
          Notes {relationshipType === "other" && <span className="required-marker">*</span>}
        </label>
        <textarea
          id="link-notes"
          className="notes-textarea"
          placeholder={
            relationshipType === "other"
              ? "Explain the relationship between these transactions (required)"
              : "Add optional notes about this link..."
          }
          value={notes}
          onChange={(e) => onNotesChange(e.target.value)}
          rows={3}
          required={relationshipType === "other"}
        />
        {relationshipType === "other" && !notes.trim() && (
          <div className="notes-hint required">
            Notes are required for "Other" relationship type
          </div>
        )}
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
    </>
  );
};
```

---

## Visual Wireframes

### Step 1: Search (Initial State)
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
│ -$45.00 USD • Nov 1, 2024 • Chase Checking                │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ ┌──────────────────────────────────────────────────┐[🔍]  │
│ │ Search by description, counterparty, or account... │     │
│ └──────────────────────────────────────────────────┘      │
│ Search for the transaction you want to link.              │
│                                                            │
│ ▸ Advanced Filters                                         │
│                                                            │
│                                                            │
│                              [Cancel] [Search Transactions]│
└────────────────────────────────────────────────────────────┘
```

### Step 1: Search (With Filters Expanded)
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
│ -$45.00 USD • Nov 1, 2024 • Chase Checking                │
├────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────────┐[🔍]  │
│ │ expense reimbursement                            │      │
│ └──────────────────────────────────────────────────┘      │
│                                                            │
│ ▾ Advanced Filters                                         │
│   Date Range                                               │
│   [2024-10-01] to [2024-11-30]                            │
│                                                            │
│   Amount Range                                             │
│   [40.00] to [50.00]                                      │
│                                                            │
│                              [Cancel] [Search Transactions]│
└────────────────────────────────────────────────────────────┘
```

### Step 1: Search Results
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
├────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────────┐[🔍]  │
│ │ expense reimbursement                            │      │
│ └──────────────────────────────────────────────────┘      │
│                                                            │
│ Found 3 transactions                                       │
│ ┌────────────────────────────────────────────────────────┐│
│ │ Expense Reimbursement - Oct 2024          +$145.23 USD │→
│ │ Nov 5, 2024 • Wells Fargo Checking • Acme Corp         ││
│ ├────────────────────────────────────────────────────────┤│
│ │ Direct Deposit - Expense Reimbursement   +$45.00 USD  │→
│ │ Nov 3, 2024 • Chase Checking • Acme Corp               ││
│ ├────────────────────────────────────────────────────────┤│
│ │ ACH Credit - Expense Reimbursement       +$322.50 USD │→
│ │ Oct 15, 2024 • Chase Checking • Acme Corp              ││
│ └────────────────────────────────────────────────────────┘│
│                              [Cancel] [Search Transactions]│
└────────────────────────────────────────────────────────────┘
```

### Step 1: Searching State
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│                       ⏳                                    │
│                Searching transactions...                   │
│                                                            │
│                              [Cancel] [Searching...]       │
└────────────────────────────────────────────────────────────┘
```

### Step 1: No Results
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
├────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────────┐[🔍]  │
│ │ nonexistent transaction                          │      │
│ └──────────────────────────────────────────────────┘      │
│                                                            │
│                       🔍                                   │
│         No transactions found matching                     │
│             "nonexistent transaction"                      │
│                                                            │
│    Try adjusting your search terms or filters             │
│                                                            │
│                              [Cancel] [Search Transactions]│
└────────────────────────────────────────────────────────────┘
```

### Step 2: Preview (Transfer Type)
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
├────────────────────────────────────────────────────────────┤
│ Preview Link                                               │
│                                                            │
│ ┌────────────────────────────────────────────────────────┐│
│ │ [Current Transaction]                                  ││
│ │ Uber Ride to Airport                                   ││
│ │ Amount: -$45.00 USD                                    ││
│ │ Date:   Nov 1, 2024                                    ││
│ │ Account: Chase Checking                                ││
│ └────────────────────────────────────────────────────────┘│
│                                                            │
│                         🔄                                 │
│                      Transfer                              │
│                                                            │
│ ┌────────────────────────────────────────────────────────┐│
│ │ [Selected Transaction]                                 ││
│ │ Direct Deposit - Expense Reimbursement                 ││
│ │ Amount: +$45.00 USD                                    ││
│ │ Date:   Nov 3, 2024                                    ││
│ │ Account: Chase Checking                                ││
│ └────────────────────────────────────────────────────────┘│
│                                                            │
│ Relationship Type                                          │
│ (•) 🔄 Transfer                                            │
│     Money moved between your own accounts                  │
│ ( ) 💱 FX Conversion                                       │
│     Currency exchange between accounts                     │
│ ( ) 💰 Reimbursement                                       │
│     Expense reimbursed by employer or friend               │
│ ( ) ✂️ Split Transaction                                   │
│ ( ) 🔧 Correction                                          │
│ ( ) 📝 Other                                               │
│                                                            │
│ Notes                                                      │
│ ┌────────────────────────────────────────────────────────┐│
│ │ Add optional notes about this link...                  ││
│ │                                                        ││
│ └────────────────────────────────────────────────────────┘│
│                                                            │
│                        [Back] [Cancel] [Create Link]      │
└────────────────────────────────────────────────────────────┘
```

### Step 2: Preview (Other Type - Notes Required)
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
├────────────────────────────────────────────────────────────┤
│ [Preview section...]                                       │
│                                                            │
│ Relationship Type                                          │
│ ( ) 🔄 Transfer                                            │
│ ( ) 💱 FX Conversion                                       │
│ ( ) 💰 Reimbursement                                       │
│ ( ) ✂️ Split Transaction                                   │
│ ( ) 🔧 Correction                                          │
│ (•) 📝 Other                                               │
│     Custom relationship (explain in notes)                 │
│                                                            │
│ Notes *                                                    │
│ ┌────────────────────────────────────────────────────────┐│
│ │ Explain the relationship between these                 ││
│ │ transactions (required)                                ││
│ └────────────────────────────────────────────────────────┘│
│ Notes are required for "Other" relationship type           │
│                                                            │
│                        [Back] [Cancel] [Create Link]      │
│                                                   (disabled)│
└────────────────────────────────────────────────────────────┘
```

### Step 2: Preview with Validation Error
```
┌────────────────────────────────────────────────────────────┐
│ Link Transaction                                        [×]│
│ Current: Uber Ride to Airport                             │
├────────────────────────────────────────────────────────────┤
│ [Preview section...]                                       │
│                                                            │
│ ⚠️ Transfer amounts differ significantly: $45.00 USD vs   │
│    $145.23 USD (69% difference)                           │
│                                                            │
│ Relationship Type                                          │
│ (•) 🔄 Transfer                                            │
│                                                            │
│ Notes                                                      │
│ ┌────────────────────────────────────────────────────────┐│
│ │ This is a partial reimbursement for multiple expenses  ││
│ └────────────────────────────────────────────────────────┘│
│                                                            │
│                        [Back] [Cancel] [Create Link]      │
└────────────────────────────────────────────────────────────┘
```

---

## Styling

```css
.transfer-link-dialog {
  max-width: 700px;
  width: 95%;
  max-height: 90vh;
}

/* Header */
.modal-header .header-content {
  flex: 1;
}

.current-transaction-context {
  margin-top: 12px;
  padding: 12px;
  background: var(--context-bg);
  border-radius: 6px;
}

.context-row {
  display: flex;
  gap: 8px;
  margin-bottom: 4px;
}

.context-label {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-muted);
}

.context-value {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-color);
}

.context-meta {
  font-size: 12px;
  color: var(--text-muted);
}

/* Modal Body */
.modal-body {
  padding: 24px;
  max-height: calc(90vh - 200px);
  overflow-y: auto;
}

/* Search Section */
.search-section {
  margin-bottom: 20px;
}

.search-input-group {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}

.search-input {
  flex: 1;
  padding: 12px 16px;
  border: 2px solid var(--input-border);
  border-radius: 8px;
  font-size: 14px;
  background: var(--input-bg);
  color: var(--text-color);
  transition: border-color 0.2s;
}

.search-input:focus {
  outline: none;
  border-color: var(--primary-color);
}

.search-icon-button {
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 20px;
  cursor: pointer;
  transition: all 0.2s;
}

.search-icon-button:hover:not(:disabled) {
  background: var(--primary-hover);
}

.search-icon-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.search-hint {
  font-size: 13px;
  color: var(--text-muted);
}

/* Filters Section */
.filters-section {
  margin-bottom: 20px;
}

.filters-collapsible {
  border: 1px solid var(--divider-color);
  border-radius: 8px;
  background: var(--filters-bg);
}

.filters-toggle {
  padding: 12px 16px;
  font-size: 14px;
  font-weight: 600;
  color: var(--text-color);
  cursor: pointer;
  list-style: none;
  user-select: none;
}

.filters-toggle::-webkit-details-marker {
  display: none;
}

.filters-content {
  padding: 16px;
  border-top: 1px solid var(--divider-color);
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.filter-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.filter-label {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-muted);
}

.filter-input {
  padding: 8px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
  background: var(--input-bg);
  color: var(--text-color);
}

.date-range-inputs,
.amount-range-inputs {
  display: flex;
  align-items: center;
  gap: 12px;
}

.date-range-inputs input,
.amount-range-inputs input {
  flex: 1;
}

.date-separator,
.amount-separator {
  font-size: 13px;
  color: var(--text-muted);
}

/* Validation Errors */
.validation-errors {
  margin-bottom: 16px;
  padding: 12px 16px;
  background: var(--error-bg);
  border: 1px solid var(--error-border);
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.error-item {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  font-size: 13px;
  color: var(--error-color);
}

.error-icon {
  font-size: 16px;
  flex-shrink: 0;
}

.error-text {
  flex: 1;
  line-height: 1.5;
}

/* Searching State */
.searching-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  padding: 48px 24px;
}

.spinner {
  width: 48px;
  height: 48px;
  border: 4px solid var(--spinner-bg);
  border-top-color: var(--primary-color);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.searching-text {
  font-size: 14px;
  color: var(--text-muted);
}

/* Search Results */
.search-results {
  margin-bottom: 16px;
}

.results-header {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-muted);
  margin-bottom: 12px;
  padding: 0 4px;
}

.results-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 400px;
  overflow-y: auto;
}

.result-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: var(--result-bg);
  border: 2px solid var(--result-border);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.result-item:hover {
  border-color: var(--primary-color);
  background: var(--result-hover-bg);
}

.result-content {
  flex: 1;
  min-width: 0;
}

.result-main {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  gap: 16px;
  margin-bottom: 6px;
}

.result-description {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-color);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.result-amount {
  font-size: 14px;
  font-weight: 700;
  flex-shrink: 0;
}

.result-amount.expense {
  color: var(--expense-color);
}

.result-amount.income {
  color: var(--income-color);
}

.result-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
  font-size: 12px;
  color: var(--text-muted);
}

.result-separator {
  color: var(--separator-color);
}

.result-action {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  background: var(--action-bg);
  border-radius: 50%;
  flex-shrink: 0;
  transition: all 0.2s;
}

.result-item:hover .result-action {
  background: var(--primary-color);
  color: white;
}

.select-icon {
  font-size: 18px;
}

/* Empty State */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 48px 24px;
  text-align: center;
}

.empty-icon {
  font-size: 64px;
  margin-bottom: 16px;
  opacity: 0.5;
}

.empty-text {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 8px;
}

.empty-hint {
  font-size: 13px;
  color: var(--text-muted);
}

/* Preview Section */
.preview-section {
  margin-bottom: 24px;
}

.preview-header {
  font-size: 14px;
  font-weight: 600;
  color: var(--heading-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 16px;
}

.preview-transactions {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 20px;
  background: var(--preview-bg);
  border-radius: 12px;
  border: 1px solid var(--preview-border);
}

.preview-transaction {
  padding: 16px;
  background: var(--transaction-bg);
  border: 2px solid var(--transaction-border);
  border-radius: 8px;
}

.preview-transaction.current {
  border-color: var(--primary-color);
}

.preview-transaction.selected {
  border-color: var(--accent-color);
}

.preview-transaction-badge {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--badge-color);
  margin-bottom: 12px;
}

.preview-transaction-description {
  font-size: 15px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 12px;
}

.preview-transaction-details {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.preview-detail-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 13px;
}

.detail-label {
  color: var(--text-muted);
  font-weight: 500;
}

.detail-value {
  color: var(--text-color);
  font-weight: 600;
}

.detail-value.expense {
  color: var(--expense-color);
}

.detail-value.income {
  color: var(--income-color);
}

.link-arrow {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding: 8px 0;
}

.arrow-icon {
  font-size: 32px;
}

.arrow-label {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-muted);
}

/* Relationship Section */
.relationship-section {
  margin-bottom: 24px;
}

.relationship-header {
  font-size: 14px;
  font-weight: 600;
  color: var(--heading-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 12px;
}

.relationship-types {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.relationship-type-option {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: var(--option-bg);
  border: 2px solid var(--option-border);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.relationship-type-option:hover {
  border-color: var(--primary-color);
  background: var(--option-hover-bg);
}

.relationship-type-option.selected {
  border-color: var(--primary-color);
  background: var(--option-selected-bg);
}

.relationship-radio {
  width: 20px;
  height: 20px;
  cursor: pointer;
  flex-shrink: 0;
}

.relationship-content {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
}

.relationship-icon {
  font-size: 24px;
  flex-shrink: 0;
}

.relationship-details {
  flex: 1;
}

.relationship-label {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 2px;
}

.relationship-description {
  font-size: 12px;
  color: var(--text-muted);
  line-height: 1.4;
}

/* Notes Section */
.notes-section {
  margin-bottom: 20px;
}

.notes-label {
  display: block;
  font-size: 14px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 8px;
}

.required-marker {
  color: var(--error-color);
  font-weight: 700;
}

.notes-textarea {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid var(--input-border);
  border-radius: 8px;
  font-size: 14px;
  font-family: inherit;
  background: var(--input-bg);
  color: var(--text-color);
  resize: vertical;
  transition: border-color 0.2s;
}

.notes-textarea:focus {
  outline: none;
  border-color: var(--primary-color);
}

.notes-textarea::placeholder {
  color: var(--placeholder-color);
}

.notes-hint {
  font-size: 12px;
  color: var(--text-muted);
  margin-top: 6px;
}

.notes-hint.required {
  color: var(--error-color);
  font-weight: 500;
}

/* Modal Footer */
.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding: 20px 24px;
  border-top: 1px solid var(--divider-color);
}

.cancel-button,
.back-button {
  padding: 10px 20px;
  background: transparent;
  border: 1px solid var(--button-border);
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  color: var(--text-color);
  transition: all 0.2s;
}

.cancel-button:hover:not(:disabled),
.back-button:hover:not(:disabled) {
  background: var(--button-hover-bg);
  border-color: var(--text-color);
}

.search-button,
.create-button {
  padding: 10px 24px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.search-button:hover:not(:disabled),
.create-button:hover:not(:disabled) {
  background: var(--primary-hover);
}

.search-button:disabled,
.create-button:disabled,
.cancel-button:disabled,
.back-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Theme: Light */
.theme-light {
  --context-bg: #f3f4f6;
  --input-bg: #ffffff;
  --input-border: #d1d5db;
  --filters-bg: #f9fafb;
  --result-bg: #ffffff;
  --result-border: #e5e7eb;
  --result-hover-bg: #f9fafb;
  --action-bg: #f3f4f6;
  --preview-bg: #f9fafb;
  --preview-border: #e5e7eb;
  --transaction-bg: #ffffff;
  --transaction-border: #d1d5db;
  --option-bg: #ffffff;
  --option-border: #e5e7eb;
  --option-hover-bg: #f9fafb;
  --option-selected-bg: #eff6ff;
  --button-border: #d1d5db;
  --button-hover-bg: #f3f4f6;
  --spinner-bg: #e5e7eb;
  --primary-color: #3b82f6;
  --primary-hover: #2563eb;
  --accent-color: #10b981;
  --expense-color: #ef4444;
  --income-color: #10b981;
  --error-bg: #fee2e2;
  --error-border: #fecaca;
  --error-color: #991b1b;
  --text-color: #111827;
  --text-muted: #6b7280;
  --heading-color: #111827;
  --badge-color: #6b7280;
  --placeholder-color: #9ca3af;
  --divider-color: #e5e7eb;
  --separator-color: #d1d5db;
}

/* Theme: Dark */
.theme-dark {
  --context-bg: #1f2937;
  --input-bg: #374151;
  --input-border: #4b5563;
  --filters-bg: #1f2937;
  --result-bg: #1f2937;
  --result-border: #374151;
  --result-hover-bg: #374151;
  --action-bg: #374151;
  --preview-bg: #111827;
  --preview-border: #374151;
  --transaction-bg: #1f2937;
  --transaction-border: #4b5563;
  --option-bg: #1f2937;
  --option-border: #374151;
  --option-hover-bg: #374151;
  --option-selected-bg: #1e3a8a;
  --button-border: #4b5563;
  --button-hover-bg: #374151;
  --spinner-bg: #4b5563;
  --primary-color: #3b82f6;
  --primary-hover: #2563eb;
  --accent-color: #10b981;
  --expense-color: #f87171;
  --income-color: #34d399;
  --error-bg: #7f1d1d;
  --error-border: #991b1b;
  --error-color: #fee2e2;
  --text-color: #f3f4f6;
  --text-muted: #9ca3af;
  --heading-color: #f3f4f6;
  --badge-color: #9ca3af;
  --placeholder-color: #6b7280;
  --divider-color: #374151;
  --separator-color: #4b5563;
}

/* Responsive Design */
@media (max-width: 768px) {
  .transfer-link-dialog {
    max-width: 100%;
    width: 100%;
    height: 100vh;
    max-height: 100vh;
    border-radius: 0;
    margin: 0;
  }

  .modal-body {
    max-height: calc(100vh - 240px);
  }

  .preview-transactions {
    padding: 12px;
  }

  .relationship-types {
    max-height: 300px;
    overflow-y: auto;
  }

  .date-range-inputs,
  .amount-range-inputs {
    flex-direction: column;
  }

  .date-separator,
  .amount-separator {
    display: none;
  }
}
```

---

## Validation Rules

### Self-Link Prevention
```typescript
// Cannot link transaction to itself
if (targetTransaction.transaction_id === currentTransaction.transaction_id) {
  error: "Cannot link a transaction to itself"
}
```

### Duplicate Link Detection
```typescript
// Check if link already exists (bidirectional)
const isDuplicate = existingLinks.some(link =>
  (link.transaction_a_id === currentTx.id && link.transaction_b_id === targetTx.id) ||
  (link.transaction_b_id === currentTx.id && link.transaction_a_id === targetTx.id)
);

if (isDuplicate) {
  error: "A link already exists between these transactions"
}
```

### Notes Required for "Other"
```typescript
if (relationshipType === "other" && !notes.trim()) {
  error: "Notes are required when relationship type is 'Other'"
}
```

### Amount Consistency Warning (Transfer)
```typescript
// For transfers, warn if amounts differ by >10%
if (relationshipType === "transfer") {
  const amountA = Math.abs(currentTransaction.amount);
  const amountB = Math.abs(selectedTransaction.amount);
  const variance = Math.abs(amountA - amountB) / Math.max(amountA, amountB);

  if (variance > 0.1) {  // 10% threshold
    warning: "Transfer amounts differ significantly: $500 vs $550 (10% difference)"
  }
}
```

---

## User Interaction Flows

### Flow 1: Create Transfer Link (Happy Path)
```
User: Clicks "Link to Transaction" button on transaction detail
System:
  1. Opens TransferLinkDialog
  2. Shows current transaction context
  3. Focus on search input

User: Types "deposit savings" and presses Enter
System:
  1. Shows "Searching..." spinner
  2. Searches transactions matching query
  3. Excludes current transaction
  4. Shows 3 results

User: Clicks on "Transfer to Savings - $500.00"
System:
  1. Transitions to preview step
  2. Shows both transactions side by side
  3. Pre-selects "Transfer" relationship type
  4. Shows notes field (optional)

User: Reviews preview and clicks "Create Link"
System:
  1. Validates:
     - Not self-link ✅
     - Not duplicate ✅
     - Amounts similar (±10%) ✅
  2. Creates link in database
  3. Closes dialog
  4. Shows success notification
  5. Updates transaction detail to show link
```

### Flow 2: Create Link with Warnings
```
User: Searches "reimbursement" → Selects transaction
System: Shows preview

User: Selects "Transfer" relationship type
System:
  1. Calculates amount variance
  2. Current: -$45.00, Selected: +$145.23
  3. Variance: 69% (exceeds 10% threshold)
  4. Shows warning:
     "⚠️ Transfer amounts differ significantly: $45.00 vs $145.23 (69% difference)"

User: Changes to "Reimbursement" type
System: Removes warning (reimbursements can have different amounts)

User: Adds note: "Partial reimbursement for Oct expenses"
User: Clicks "Create Link"
System: Creates link successfully
```

### Flow 3: Prevented Duplicate Link
```
User: Searches and selects transaction
System: Shows preview

User: Clicks "Create Link"
System:
  1. Validates link
  2. Detects existing link between transactions
  3. Shows error:
     "⚠️ A link already exists between these transactions"
  4. "Create Link" button remains enabled (allows retry with different transaction)

User: Clicks "Back" → Searches for different transaction
```

### Flow 4: "Other" Type Requires Notes
```
User: Selects transaction in preview
User: Selects "Other" relationship type
System:
  1. Shows notes field with "required" marker (*)
  2. Shows hint: "Notes are required for 'Other' relationship type"
  3. Disables "Create Link" button

User: Types notes: "Split payment for joint expense with roommate"
System:
  1. Enables "Create Link" button

User: Clicks "Create Link"
System: Creates link with notes
```

### Flow 5: Advanced Filters for Targeted Search
```
User: Opens dialog and clicks "▸ Advanced Filters"
System: Expands filter section

User: Sets filters:
  - Date Range: 2024-10-01 to 2024-11-30
  - Amount Range: 40.00 to 50.00

User: Types "uber" and clicks "Search"
System:
  1. Searches with combined criteria:
     - Text: "uber"
     - Date: Last 2 months
     - Amount: $40-$50
  2. Returns 2 narrow results (instead of 50+ without filters)

User: Selects matching transaction
```

### Flow 6: No Results Found
```
User: Types "nonexistent transaction" and searches
System:
  1. Searches database
  2. Finds 0 results
  3. Shows empty state:
     "🔍 No transactions found matching 'nonexistent transaction'"
     "Try adjusting your search terms or filters"

User: Clears search and tries different query
```

---

## Accessibility

```tsx
// Dialog ARIA attributes
<div
  role="dialog"
  aria-labelledby="link-dialog-title"
  aria-modal="true"
  className="modal-overlay"
>
  <h3 id="link-dialog-title">Link Transaction</h3>
</div>

// Search input
<input
  type="text"
  aria-label="Search transactions"
  aria-describedby="search-hint"
/>
<div id="search-hint">
  Search for the transaction you want to link. Press Enter to search.
</div>

// Relationship type radio buttons
<label className="relationship-type-option">
  <input
    type="radio"
    name="relationship-type"
    value="transfer"
    aria-label="Transfer: Money moved between your own accounts"
  />
  <div>Transfer</div>
</label>

// Notes field
<textarea
  id="link-notes"
  aria-label="Link notes"
  aria-required={relationshipType === "other"}
  aria-describedby={relationshipType === "other" ? "notes-required-hint" : undefined}
/>
{relationshipType === "other" && (
  <div id="notes-required-hint" role="alert">
    Notes are required for "Other" relationship type
  </div>
)}

// Validation errors
<div className="validation-errors" role="alert" aria-live="assertive">
  <div className="error-item">
    A link already exists between these transactions
  </div>
</div>
```

**ARIA Roles:**
- Dialog: `role="dialog"`, `aria-modal="true"`, `aria-labelledby`
- Inputs: `aria-label`, `aria-describedby`
- Required fields: `aria-required="true"`
- Errors: `role="alert"`, `aria-live="assertive"`

**Keyboard Support:**
- **Esc**: Close dialog
- **Tab**: Navigate between form fields
- **Enter**: Submit search (when in search input)
- **Arrow keys**: Navigate radio buttons
- **Space**: Select radio button or click button

**Screen Reader:**
- Announces current transaction context on dialog open
- Announces search results count: "Found 3 transactions"
- Announces relationship type changes: "Transfer selected"
- Announces validation errors: "Error: Notes are required for Other relationship type"
- Announces successful link creation: "Link created successfully"

**Focus Management:**
- Auto-focus search input on dialog open
- Return focus to trigger button on dialog close
- Trap focus within dialog (no tabbing outside)

---

## Multi-Domain Examples

### Finance Domain: Expense Reimbursement
```tsx
<TransferLinkDialog
  currentTransaction={{
    transaction_id: "tx_001",
    amount: -45.00,
    currency: "USD",
    date: "2024-11-01",
    description: "Uber Ride to Airport",
    account_name: "Chase Checking",
    counterparty_name: "Uber"
  }}
  isOpen={true}
  onClose={closeDialog}
  onCreate={createLink}
  onSearchTransactions={searchTransactions}
  existingLinks={[]}
/>
// Use case: Link expense to reimbursement payment
// Relationship type: "reimbursement"
```

### Healthcare Domain: Claim Linkage
```tsx
<TransferLinkDialog
  currentTransaction={{
    transaction_id: "claim_001",
    amount: -350.00,
    currency: "USD",
    date: "2024-10-15",
    description: "Dr. Smith - Annual Checkup",
    account_name: "HSA Account",
    counterparty_name: "Smith Medical Group"
  }}
  isOpen={true}
  onClose={closeDialog}
  onCreate={createClaimLink}
  onSearchTransactions={searchClaims}
/>
// Use case: Link claim payment to insurance reimbursement
// Relationship type: "reimbursement"
// Notes: "Insurance claim #12345 - processed on 2024-10-20"
```

### Legal Domain: Court Filing Fee Correction
```tsx
<TransferLinkDialog
  currentTransaction={{
    transaction_id: "filing_001",
    amount: -500.00,
    currency: "USD",
    date: "2024-11-01",
    description: "Court Filing Fee - Case #2024-CV-001",
    account_name: "Trust Account",
    counterparty_name: "Superior Court"
  }}
  isOpen={true}
  onClose={closeDialog}
  onCreate={createFilingLink}
  onSearchTransactions={searchFilings}
/>
// Use case: Link duplicate filing fee payment to correction entry
// Relationship type: "correction"
// Notes: "Duplicate payment refunded - see refund transaction on 2024-11-05"
```

### Research Domain: Grant Budget Transfer
```tsx
<TransferLinkDialog
  currentTransaction={{
    transaction_id: "grant_001",
    amount: -5000.00,
    currency: "USD",
    date: "2024-10-01",
    description: "Equipment Purchase - Grant #NSF-2024-001",
    account_name: "Grant Account A",
    counterparty_name: "Lab Equipment Inc"
  }}
  isOpen={true}
  onClose={closeDialog}
  onCreate={createGrantLink}
  onSearchTransactions={searchGrantTransactions}
/>
// Use case: Link grant expense to budget reallocation
// Relationship type: "transfer"
// Notes: "Budget reallocated from Category 1 to Category 2 per NSF approval"
```

---

## Related Components

**Uses (Dependencies):**
- None (standalone primitive component)

**Used By:**
- DrillDownPanel (transaction detail drawer)
- TransactionTable (bulk link creation)
- AccountReconciliation (matching transactions)

**Similar Patterns:**
- MergeCounterpartiesDialog (merge entities instead of link)
- SeriesSelector (select recurring series)
- AccountSelector (select accounts)

**OL Dependencies:**
- TransactionStore (search, validate ownership)
- LinkStore (create link, check duplicates)
- ValidationService (validate link rules)

---

## Testing Guidance

### Unit Tests
```typescript
describe('TransferLinkDialog', () => {
  it('prevents self-linking', () => {
    const errors = validateLink(currentTx, currentTx);
    expect(errors).toContain("Cannot link a transaction to itself");
  });

  it('detects duplicate links', () => {
    const existingLinks = [
      { transaction_a_id: 'tx_1', transaction_b_id: 'tx_2' }
    ];
    const errors = validateLink(tx_1, tx_2, existingLinks);
    expect(errors).toContain("A link already exists");
  });

  it('requires notes for "other" type', () => {
    const errors = validateLink(tx_1, tx_2, [], 'other', '');
    expect(errors).toContain("Notes are required");
  });

  it('warns for transfer amount mismatch >10%', () => {
    const tx1 = { amount: -500 };
    const tx2 = { amount: 600 };  // 20% difference
    const errors = validateLink(tx1, tx2, [], 'transfer');
    expect(errors).toContain("amounts differ significantly");
  });
});
```

### Integration Tests
```typescript
describe('TransferLinkDialog Integration', () => {
  it('creates transfer link successfully', async () => {
    render(<TransferLinkDialog {...props} />);

    // Search
    fireEvent.change(screen.getByPlaceholderText(/search/i), {
      target: { value: 'transfer' }
    });
    fireEvent.click(screen.getByText(/search transactions/i));
    await waitFor(() => expect(screen.getByText(/found/i)).toBeInTheDocument());

    // Select
    fireEvent.click(screen.getByText(/transfer to savings/i));
    expect(screen.getByText(/preview link/i)).toBeInTheDocument();

    // Create
    fireEvent.click(screen.getByText(/create link/i));
    await waitFor(() => expect(props.onCreate).toHaveBeenCalled());
  });

  it('shows validation error for duplicate link', async () => {
    const existingLinks = [{ transaction_a_id: 'tx_1', transaction_b_id: 'tx_2' }];
    render(<TransferLinkDialog {...props} existingLinks={existingLinks} />);

    // ... select tx_2 ...

    fireEvent.click(screen.getByText(/create link/i));
    await waitFor(() => {
      expect(screen.getByText(/already exists/i)).toBeInTheDocument();
    });
    expect(props.onCreate).not.toHaveBeenCalled();
  });
});
```

### E2E Tests
```typescript
describe('TransferLinkDialog E2E', () => {
  it('completes full link creation workflow', () => {
    cy.visit('/transactions/tx_001');
    cy.contains('Link to Transaction').click();

    // Search
    cy.get('[placeholder*="Search"]').type('reimbursement{enter}');
    cy.contains('Found 3 transactions');

    // Select
    cy.contains('Expense Reimbursement').click();
    cy.contains('Preview Link');

    // Change relationship type
    cy.contains('Reimbursement').click();

    // Add notes
    cy.get('textarea[id="link-notes"]').type('October expenses reimbursed');

    // Create
    cy.contains('Create Link').click();
    cy.contains('Link created successfully');

    // Verify link appears in transaction detail
    cy.contains('Linked to: Expense Reimbursement');
  });
});
```

---

## Summary

TransferLinkDialog provides a comprehensive, validated workflow for manually linking related transactions with:

✅ **Searchable transaction list** with text and advanced filters (date, amount, account)
✅ **6 relationship types** with clear semantic labels and icons
✅ **Preview before creation** showing both transactions side-by-side
✅ **Validation rules** preventing self-links, duplicates, and unauthorized links
✅ **Required notes** for "other" relationship type
✅ **Amount consistency warnings** for transfer relationships (±10% threshold)
✅ **Multi-step workflow** (Search → Preview → Create)
✅ **Loading and error states** for async operations
✅ **Keyboard accessible** (Esc, Tab, Enter, Arrow keys)
✅ **Responsive design** adapts to mobile screens
✅ **Multi-domain support** (finance, healthcare, legal, research)

This dialog is the standard way to manually create transaction relationships throughout the application, ensuring consistent UX and preventing linking errors that could corrupt financial data integrity.
