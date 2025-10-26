# AccountManager (IL Component)

## Definition

**AccountManager** is a full CRUD UI component for managing accounts (create, edit, archive, list). It provides a complete interface for account lifecycle management with search, filtering, and sorting capabilities.

**Problem it solves:**
- No standardized UI for managing accounts across different domains
- Inconsistent account creation/editing workflows
- No unified pattern for filtering accounts by type, currency, or status
- Account archival vs deletion confusion
- Scattered account management across multiple screens

**Solution:**
- Single component for all account CRUD operations
- Consistent list view with inline actions
- Modal-based create/edit forms
- Soft delete via archive (preserves historical data)
- Search, filter, and sort built-in
- Responsive design (table on desktop, cards on mobile)

---

## Interface Contract

```typescript
interface AccountManagerProps {
  // Data
  accounts: Account[];  // All accounts (active + archived)
  currentUserId: string;  // For ownership checks

  // Callbacks
  onAccountCreate: (account: CreateAccountInput) => Promise<Account>;
  onAccountUpdate: (accountId: string, updates: Partial<Account>) => Promise<Account>;
  onAccountArchive: (accountId: string) => Promise<void>;
  onAccountRestore: (accountId: string) => Promise<void>;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Filters & Display
  initialFilter?: AccountFilter;  // Pre-applied filter
  showArchived?: boolean;  // Show archived accounts (default: false)
  allowCreate?: boolean;  // Show "New Account" button (default: true)
  allowEdit?: boolean;  // Allow editing accounts (default: true)
  allowArchive?: boolean;  // Allow archiving accounts (default: true)

  // Customization
  theme?: "light" | "dark";
  compact?: boolean;  // Compact view for smaller spaces
  groupBy?: "type" | "currency" | "none";  // Group accounts in list
}

interface Account {
  account_id: string;
  name: string;
  type: AccountType;
  currency: string;  // ISO 4217 code (USD, EUR, MXN, etc.)
  balance: number;  // Current balance
  is_archived: boolean;
  created_at: string;
  updated_at: string;
  owner_id: string;

  // Optional metadata
  description?: string;
  institution?: string;  // Bank, broker, hospital, etc.
  account_number?: string;  // Last 4 digits
  icon?: string;  // Emoji or icon identifier
  color?: string;  // Hex color for UI
}

type AccountType =
  | "cash" | "checking" | "savings" | "credit_card" | "investment"  // Finance
  | "insurance" | "hsa" | "fsa"  // Healthcare
  | "trust" | "escrow" | "retainer"  // Legal
  | "grant" | "budget"  // Research
  | "inventory" | "materials"  // Manufacturing
  | "revenue" | "ad_revenue";  // Media

interface CreateAccountInput {
  name: string;
  type: AccountType;
  currency: string;
  initial_balance?: number;
  description?: string;
  institution?: string;
  icon?: string;
  color?: string;
}

interface AccountFilter {
  searchQuery?: string;  // Search by name or institution
  types?: AccountType[];  // Filter by account type
  currencies?: string[];  // Filter by currency
  showArchived?: boolean;  // Include archived accounts
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo } from 'react';

export const AccountManager: React.FC<AccountManagerProps> = ({
  accounts,
  currentUserId,
  onAccountCreate,
  onAccountUpdate,
  onAccountArchive,
  onAccountRestore,
  loading = false,
  error = null,
  initialFilter = {},
  showArchived = false,
  allowCreate = true,
  allowEdit = true,
  allowArchive = true,
  theme = "light",
  compact = false,
  groupBy = "none"
}) => {
  const [filter, setFilter] = useState<AccountFilter>({
    showArchived,
    ...initialFilter
  });
  const [sortBy, setSortBy] = useState<"name" | "type" | "balance" | "updated_at">("name");
  const [sortDirection, setSortDirection] = useState<"asc" | "desc">("asc");
  const [showCreateModal, setShowCreateModal] = useState(false);
  const [editingAccount, setEditingAccount] = useState<Account | null>(null);
  const [archiveConfirm, setArchiveConfirm] = useState<Account | null>(null);

  // Filter accounts
  const filteredAccounts = useMemo(() => {
    let result = accounts;

    // Filter by archived status
    if (!filter.showArchived) {
      result = result.filter(a => !a.is_archived);
    }

    // Search by name or institution
    if (filter.searchQuery) {
      const query = filter.searchQuery.toLowerCase();
      result = result.filter(a =>
        a.name.toLowerCase().includes(query) ||
        a.institution?.toLowerCase().includes(query)
      );
    }

    // Filter by type
    if (filter.types && filter.types.length > 0) {
      result = result.filter(a => filter.types!.includes(a.type));
    }

    // Filter by currency
    if (filter.currencies && filter.currencies.length > 0) {
      result = result.filter(a => filter.currencies!.includes(a.currency));
    }

    return result;
  }, [accounts, filter]);

  // Sort accounts
  const sortedAccounts = useMemo(() => {
    const sorted = [...filteredAccounts];
    sorted.sort((a, b) => {
      let comparison = 0;

      switch (sortBy) {
        case "name":
          comparison = a.name.localeCompare(b.name);
          break;
        case "type":
          comparison = a.type.localeCompare(b.type);
          break;
        case "balance":
          comparison = a.balance - b.balance;
          break;
        case "updated_at":
          comparison = new Date(a.updated_at).getTime() - new Date(b.updated_at).getTime();
          break;
      }

      return sortDirection === "asc" ? comparison : -comparison;
    });

    return sorted;
  }, [filteredAccounts, sortBy, sortDirection]);

  // Group accounts
  const groupedAccounts = useMemo(() => {
    if (groupBy === "none") {
      return { "All Accounts": sortedAccounts };
    }

    const groups: Record<string, Account[]> = {};
    sortedAccounts.forEach(account => {
      const key = groupBy === "type" ? account.type : account.currency;
      if (!groups[key]) groups[key] = [];
      groups[key].push(account);
    });

    return groups;
  }, [sortedAccounts, groupBy]);

  // Loading state
  if (loading) {
    return (
      <div className={`account-manager theme-${theme} loading`}>
        <div className="skeleton skeleton-header"></div>
        <div className="skeleton skeleton-list"></div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`account-manager theme-${theme} error`}>
        <div className="error-icon">⚠️</div>
        <div className="error-message">{error}</div>
      </div>
    );
  }

  return (
    <div className={`account-manager theme-${theme} ${compact ? 'compact' : ''}`}>
      {/* Header */}
      <div className="manager-header">
        <h2>Account Management</h2>
        {allowCreate && (
          <button
            className="create-button"
            onClick={() => setShowCreateModal(true)}
          >
            + New Account
          </button>
        )}
      </div>

      {/* Search & Filters */}
      <div className="manager-filters">
        <input
          type="text"
          placeholder="Search accounts..."
          value={filter.searchQuery || ""}
          onChange={(e) => setFilter({ ...filter, searchQuery: e.target.value })}
          className="search-input"
        />

        <select
          value={groupBy}
          onChange={(e) => setGroupBy(e.target.value as any)}
          className="group-by-select"
        >
          <option value="none">No Grouping</option>
          <option value="type">Group by Type</option>
          <option value="currency">Group by Currency</option>
        </select>

        <select
          value={sortBy}
          onChange={(e) => setSortBy(e.target.value as any)}
          className="sort-select"
        >
          <option value="name">Sort by Name</option>
          <option value="type">Sort by Type</option>
          <option value="balance">Sort by Balance</option>
          <option value="updated_at">Sort by Last Updated</option>
        </select>

        <button
          className="sort-direction-button"
          onClick={() => setSortDirection(sortDirection === "asc" ? "desc" : "asc")}
          title={`Sort ${sortDirection === "asc" ? "descending" : "ascending"}`}
        >
          {sortDirection === "asc" ? "↑" : "↓"}
        </button>

        <label className="show-archived-toggle">
          <input
            type="checkbox"
            checked={filter.showArchived}
            onChange={(e) => setFilter({ ...filter, showArchived: e.target.checked })}
          />
          Show Archived
        </label>
      </div>

      {/* Account List */}
      {Object.keys(groupedAccounts).length === 0 ? (
        <div className="empty-state">
          <div className="empty-icon">🏦</div>
          <div className="empty-message">
            {filter.searchQuery
              ? `No accounts found matching "${filter.searchQuery}"`
              : "No accounts yet. Create your first account to get started."}
          </div>
          {allowCreate && !filter.searchQuery && (
            <button
              className="create-button-secondary"
              onClick={() => setShowCreateModal(true)}
            >
              Create Account
            </button>
          )}
        </div>
      ) : (
        <div className="account-groups">
          {Object.entries(groupedAccounts).map(([groupName, groupAccounts]) => (
            <div key={groupName} className="account-group">
              {groupBy !== "none" && (
                <div className="group-header">
                  {formatGroupName(groupName, groupBy)}
                  <span className="group-count">({groupAccounts.length})</span>
                </div>
              )}

              <div className="account-list">
                {groupAccounts.map(account => (
                  <AccountRow
                    key={account.account_id}
                    account={account}
                    onEdit={allowEdit ? () => setEditingAccount(account) : undefined}
                    onArchive={allowArchive ? () => setArchiveConfirm(account) : undefined}
                    onRestore={allowArchive && account.is_archived ? () => onAccountRestore(account.account_id) : undefined}
                    compact={compact}
                  />
                ))}
              </div>
            </div>
          ))}
        </div>
      )}

      {/* Create Account Modal */}
      {showCreateModal && (
        <AccountCreateModal
          onClose={() => setShowCreateModal(false)}
          onCreate={async (data) => {
            await onAccountCreate(data);
            setShowCreateModal(false);
          }}
          theme={theme}
        />
      )}

      {/* Edit Account Modal */}
      {editingAccount && (
        <AccountEditModal
          account={editingAccount}
          onClose={() => setEditingAccount(null)}
          onUpdate={async (updates) => {
            await onAccountUpdate(editingAccount.account_id, updates);
            setEditingAccount(null);
          }}
          theme={theme}
        />
      )}

      {/* Archive Confirmation Modal */}
      {archiveConfirm && (
        <ArchiveConfirmModal
          account={archiveConfirm}
          onClose={() => setArchiveConfirm(null)}
          onConfirm={async () => {
            await onAccountArchive(archiveConfirm.account_id);
            setArchiveConfirm(null);
          }}
          theme={theme}
        />
      )}
    </div>
  );
};

// Individual account row component
const AccountRow: React.FC<{
  account: Account;
  onEdit?: () => void;
  onArchive?: () => void;
  onRestore?: () => void;
  compact: boolean;
}> = ({ account, onEdit, onArchive, onRestore, compact }) => {
  const formattedBalance = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: account.currency,
    minimumFractionDigits: 2
  }).format(account.balance);

  return (
    <div className={`account-row ${account.is_archived ? 'archived' : ''} ${compact ? 'compact' : ''}`}>
      <div className="account-icon" style={{ background: account.color }}>
        {account.icon || getDefaultIcon(account.type)}
      </div>

      <div className="account-info">
        <div className="account-name">
          {account.name}
          {account.is_archived && <span className="archived-badge">Archived</span>}
        </div>
        <div className="account-meta">
          <span className="account-type">{formatAccountType(account.type)}</span>
          {account.institution && (
            <>
              <span className="separator">•</span>
              <span className="account-institution">{account.institution}</span>
            </>
          )}
          {account.account_number && (
            <>
              <span className="separator">•</span>
              <span className="account-number">****{account.account_number}</span>
            </>
          )}
        </div>
      </div>

      <div className="account-balance">
        <div className="balance-amount">{formattedBalance}</div>
        <div className="balance-currency">{account.currency}</div>
      </div>

      <div className="account-actions">
        {onEdit && !account.is_archived && (
          <button className="action-button edit" onClick={onEdit} title="Edit account">
            ✏️
          </button>
        )}
        {onArchive && !account.is_archived && (
          <button className="action-button archive" onClick={onArchive} title="Archive account">
            📦
          </button>
        )}
        {onRestore && account.is_archived && (
          <button className="action-button restore" onClick={onRestore} title="Restore account">
            ↺
          </button>
        )}
      </div>
    </div>
  );
};

// Helper functions
function formatGroupName(key: string, groupBy: "type" | "currency"): string {
  if (groupBy === "type") {
    return formatAccountType(key as AccountType);
  }
  return key;  // Currency code
}

function formatAccountType(type: AccountType): string {
  const labels: Record<AccountType, string> = {
    cash: "Cash",
    checking: "Checking",
    savings: "Savings",
    credit_card: "Credit Card",
    investment: "Investment",
    insurance: "Insurance",
    hsa: "HSA",
    fsa: "FSA",
    trust: "Trust Account",
    escrow: "Escrow",
    retainer: "Retainer",
    grant: "Grant",
    budget: "Budget",
    inventory: "Inventory",
    materials: "Materials",
    revenue: "Revenue",
    ad_revenue: "Ad Revenue"
  };
  return labels[type] || type;
}

function getDefaultIcon(type: AccountType): string {
  const icons: Record<AccountType, string> = {
    cash: "💵",
    checking: "🏦",
    savings: "🐖",
    credit_card: "💳",
    investment: "📈",
    insurance: "🏥",
    hsa: "🏥",
    fsa: "🏥",
    trust: "⚖️",
    escrow: "🏠",
    retainer: "💼",
    grant: "🎓",
    budget: "📊",
    inventory: "📦",
    materials: "🔧",
    revenue: "💰",
    ad_revenue: "📺"
  };
  return icons[type] || "🏦";
}
```

---

## Create Account Modal

```tsx
const AccountCreateModal: React.FC<{
  onClose: () => void;
  onCreate: (data: CreateAccountInput) => Promise<void>;
  theme: "light" | "dark";
}> = ({ onClose, onCreate, theme }) => {
  const [formData, setFormData] = useState<CreateAccountInput>({
    name: "",
    type: "checking",
    currency: "USD",
    initial_balance: 0,
    description: "",
    institution: "",
    icon: "",
    color: "#3b82f6"
  });
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);
    setError(null);

    try {
      // Validation
      if (!formData.name.trim()) {
        throw new Error("Account name is required");
      }

      await onCreate(formData);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to create account");
      setSubmitting(false);
    }
  };

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Create New Account</h3>
          <button className="close-button" onClick={onClose}>×</button>
        </div>

        <form onSubmit={handleSubmit} className="account-form">
          {error && <div className="form-error">{error}</div>}

          <div className="form-group">
            <label htmlFor="name">Account Name *</label>
            <input
              type="text"
              id="name"
              value={formData.name}
              onChange={(e) => setFormData({ ...formData, name: e.target.value })}
              placeholder="e.g., Chase Checking, Emergency Fund"
              required
              autoFocus
            />
          </div>

          <div className="form-row">
            <div className="form-group">
              <label htmlFor="type">Account Type *</label>
              <select
                id="type"
                value={formData.type}
                onChange={(e) => setFormData({ ...formData, type: e.target.value as AccountType })}
                required
              >
                <optgroup label="Finance">
                  <option value="cash">Cash</option>
                  <option value="checking">Checking</option>
                  <option value="savings">Savings</option>
                  <option value="credit_card">Credit Card</option>
                  <option value="investment">Investment</option>
                </optgroup>
                <optgroup label="Healthcare">
                  <option value="insurance">Insurance</option>
                  <option value="hsa">HSA</option>
                  <option value="fsa">FSA</option>
                </optgroup>
                <optgroup label="Legal">
                  <option value="trust">Trust Account</option>
                  <option value="escrow">Escrow</option>
                  <option value="retainer">Retainer</option>
                </optgroup>
                <optgroup label="Other">
                  <option value="grant">Grant</option>
                  <option value="budget">Budget</option>
                </optgroup>
              </select>
            </div>

            <div className="form-group">
              <label htmlFor="currency">Currency *</label>
              <select
                id="currency"
                value={formData.currency}
                onChange={(e) => setFormData({ ...formData, currency: e.target.value })}
                required
              >
                <option value="USD">USD - US Dollar</option>
                <option value="EUR">EUR - Euro</option>
                <option value="GBP">GBP - British Pound</option>
                <option value="MXN">MXN - Mexican Peso</option>
                <option value="CAD">CAD - Canadian Dollar</option>
              </select>
            </div>
          </div>

          <div className="form-group">
            <label htmlFor="initial_balance">Initial Balance</label>
            <input
              type="number"
              id="initial_balance"
              value={formData.initial_balance}
              onChange={(e) => setFormData({ ...formData, initial_balance: parseFloat(e.target.value) || 0 })}
              step="0.01"
            />
          </div>

          <div className="form-group">
            <label htmlFor="institution">Institution (Optional)</label>
            <input
              type="text"
              id="institution"
              value={formData.institution}
              onChange={(e) => setFormData({ ...formData, institution: e.target.value })}
              placeholder="e.g., Chase Bank, Fidelity"
            />
          </div>

          <div className="form-group">
            <label htmlFor="description">Description (Optional)</label>
            <textarea
              id="description"
              value={formData.description}
              onChange={(e) => setFormData({ ...formData, description: e.target.value })}
              rows={3}
              placeholder="Add notes about this account..."
            />
          </div>

          <div className="form-row">
            <div className="form-group">
              <label htmlFor="icon">Icon (Emoji)</label>
              <input
                type="text"
                id="icon"
                value={formData.icon}
                onChange={(e) => setFormData({ ...formData, icon: e.target.value })}
                placeholder="e.g., 💰"
                maxLength={2}
              />
            </div>

            <div className="form-group">
              <label htmlFor="color">Color</label>
              <input
                type="color"
                id="color"
                value={formData.color}
                onChange={(e) => setFormData({ ...formData, color: e.target.value })}
              />
            </div>
          </div>

          <div className="form-actions">
            <button type="button" onClick={onClose} className="cancel-button" disabled={submitting}>
              Cancel
            </button>
            <button type="submit" className="submit-button" disabled={submitting}>
              {submitting ? "Creating..." : "Create Account"}
            </button>
          </div>
        </form>
      </div>
    </div>
  );
};
```

---

## Styling

```css
.account-manager {
  background: var(--manager-bg);
  border: 1px solid var(--manager-border);
  border-radius: 8px;
  padding: 24px;
}

.manager-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.manager-header h2 {
  font-size: 24px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0;
}

.create-button {
  padding: 10px 20px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.create-button:hover {
  background: var(--primary-color-hover);
}

.manager-filters {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.search-input {
  flex: 1;
  min-width: 200px;
  padding: 8px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
}

.group-by-select,
.sort-select {
  padding: 8px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
  background: var(--input-bg);
  cursor: pointer;
}

.sort-direction-button {
  padding: 8px 16px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  background: var(--input-bg);
  cursor: pointer;
  font-size: 18px;
  transition: background 0.2s;
}

.sort-direction-button:hover {
  background: var(--input-hover-bg);
}

.show-archived-toggle {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  cursor: pointer;
}

.account-group {
  margin-bottom: 24px;
}

.group-header {
  font-size: 14px;
  font-weight: 600;
  color: var(--group-header-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 12px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.group-count {
  color: var(--secondary-color);
  font-weight: 400;
}

.account-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.account-row {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: var(--row-bg);
  border: 1px solid var(--row-border);
  border-radius: 8px;
  transition: all 0.2s;
}

.account-row:hover {
  background: var(--row-hover-bg);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.account-row.archived {
  opacity: 0.6;
}

.account-row.compact {
  padding: 12px;
  gap: 12px;
}

.account-icon {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  flex-shrink: 0;
}

.account-info {
  flex: 1;
  min-width: 0;
}

.account-name {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 4px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.archived-badge {
  font-size: 11px;
  padding: 2px 8px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  text-transform: uppercase;
  font-weight: 500;
}

.account-meta {
  font-size: 13px;
  color: var(--secondary-color);
  display: flex;
  align-items: center;
  gap: 6px;
}

.separator {
  color: var(--separator-color);
}

.account-balance {
  text-align: right;
}

.balance-amount {
  font-size: 18px;
  font-weight: 700;
  color: var(--balance-color);
  margin-bottom: 2px;
}

.balance-currency {
  font-size: 12px;
  color: var(--secondary-color);
}

.account-actions {
  display: flex;
  gap: 8px;
}

.action-button {
  padding: 8px 12px;
  background: none;
  border: 1px solid var(--action-border);
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.2s;
}

.action-button:hover {
  background: var(--action-hover-bg);
  transform: scale(1.1);
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 60px 20px;
}

.empty-icon {
  font-size: 64px;
  margin-bottom: 16px;
}

.empty-message {
  font-size: 16px;
  color: var(--secondary-color);
  margin-bottom: 24px;
}

.create-button-secondary {
  padding: 12px 24px;
  background: var(--secondary-button-bg);
  color: var(--secondary-button-color);
  border: 1px solid var(--secondary-button-border);
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.create-button-secondary:hover {
  background: var(--secondary-button-hover-bg);
}

/* Modal styles */
.modal-overlay {
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

.modal-content {
  background: var(--modal-bg);
  border-radius: 12px;
  padding: 24px;
  max-width: 600px;
  width: 90%;
  max-height: 90vh;
  overflow-y: auto;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.modal-header h3 {
  font-size: 20px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0;
}

.close-button {
  background: none;
  border: none;
  font-size: 28px;
  cursor: pointer;
  color: var(--secondary-color);
  line-height: 1;
  padding: 0;
}

.account-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.form-error {
  padding: 12px;
  background: #fee;
  border: 1px solid #fcc;
  border-radius: 6px;
  color: #c33;
  font-size: 14px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.form-group label {
  font-size: 14px;
  font-weight: 500;
  color: var(--label-color);
}

.form-group input,
.form-group select,
.form-group textarea {
  padding: 10px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
  background: var(--input-bg);
  color: var(--text-color);
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.form-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
  margin-top: 8px;
}

.cancel-button {
  padding: 10px 20px;
  background: transparent;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
}

.submit-button {
  padding: 10px 20px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.submit-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Theme: Light */
.theme-light {
  --manager-bg: #ffffff;
  --manager-border: #e5e7eb;
  --heading-color: #111827;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --input-bg: #ffffff;
  --input-border: #d1d5db;
  --input-hover-bg: #f3f4f6;
  --group-header-color: #6b7280;
  --row-bg: #ffffff;
  --row-border: #e5e7eb;
  --row-hover-bg: #f9fafb;
  --text-color: #111827;
  --secondary-color: #6b7280;
  --balance-color: #111827;
  --badge-bg: #fef3c7;
  --badge-color: #92400e;
  --separator-color: #d1d5db;
  --action-border: #e5e7eb;
  --action-hover-bg: #f3f4f6;
  --secondary-button-bg: #ffffff;
  --secondary-button-color: #374151;
  --secondary-button-border: #d1d5db;
  --secondary-button-hover-bg: #f3f4f6;
  --modal-bg: #ffffff;
  --label-color: #374151;
}

/* Theme: Dark */
.theme-dark {
  --manager-bg: #1f2937;
  --manager-border: #374151;
  --heading-color: #f3f4f6;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --input-bg: #374151;
  --input-border: #4b5563;
  --input-hover-bg: #4b5563;
  --group-header-color: #9ca3af;
  --row-bg: #374151;
  --row-border: #4b5563;
  --row-hover-bg: #4b5563;
  --text-color: #f3f4f6;
  --secondary-color: #9ca3af;
  --balance-color: #f3f4f6;
  --badge-bg: #78350f;
  --badge-color: #fef3c7;
  --separator-color: #4b5563;
  --action-border: #4b5563;
  --action-hover-bg: #1f2937;
  --secondary-button-bg: #374151;
  --secondary-button-color: #f3f4f6;
  --secondary-button-border: #4b5563;
  --secondary-button-hover-bg: #4b5563;
  --modal-bg: #1f2937;
  --label-color: #d1d5db;
}

/* Responsive */
@media (max-width: 768px) {
  .account-row {
    flex-direction: column;
    align-items: flex-start;
  }

  .account-balance {
    text-align: left;
  }

  .form-row {
    grid-template-columns: 1fr;
  }
}
```

---

## Multi-Domain Applicability

### Finance Domain
```tsx
<AccountManager
  accounts={financeAccounts}
  currentUserId="user_123"
  onAccountCreate={async (data) => {
    return await createAccount(data);
  }}
  onAccountUpdate={async (id, updates) => {
    return await updateAccount(id, updates);
  }}
  onAccountArchive={async (id) => {
    await archiveAccount(id);
  }}
  onAccountRestore={async (id) => {
    await restoreAccount(id);
  }}
  initialFilter={{ types: ["checking", "savings", "credit_card"] }}
  groupBy="type"
/>
// Accounts: Chase Checking, Emergency Savings, Amex Credit Card
```

### Healthcare Domain
```tsx
<AccountManager
  accounts={healthcareAccounts}
  currentUserId="patient_456"
  onAccountCreate={createHealthAccount}
  onAccountUpdate={updateHealthAccount}
  onAccountArchive={archiveHealthAccount}
  onAccountRestore={restoreHealthAccount}
  initialFilter={{ types: ["insurance", "hsa", "fsa"] }}
  groupBy="type"
/>
// Accounts: Blue Cross Insurance, HSA (Chase), FSA (Work)
```

### Legal Domain
```tsx
<AccountManager
  accounts={legalAccounts}
  currentUserId="attorney_789"
  onAccountCreate={createLegalAccount}
  onAccountUpdate={updateLegalAccount}
  onAccountArchive={archiveLegalAccount}
  onAccountRestore={restoreLegalAccount}
  initialFilter={{ types: ["trust", "escrow", "retainer"] }}
  groupBy="type"
/>
// Accounts: Smith Family Trust, Property Escrow, Client Retainer Fund
```

### Research Domain (RSRCH - Utilitario)
```tsx
<AccountManager
  accounts={rsrchAccounts}
  currentUserId="rsrch_analyst_101"
  onAccountCreate={createRSRCHAccount}
  onAccountUpdate={updateRSRCHAccount}
  onAccountArchive={archiveRSRCHAccount}
  onAccountRestore={restoreRSRCHAccount}
  initialFilter={{ types: ["research_fund", "scraping_budget"] }}
  groupBy="currency"
/>
// Accounts: VC Research Fund 2025, Web Scraping Budget, API Costs Account
```

### Manufacturing Domain
```tsx
<AccountManager
  accounts={manufacturingAccounts}
  currentUserId="manager_202"
  onAccountCreate={createMfgAccount}
  onAccountUpdate={updateMfgAccount}
  onAccountArchive={archiveMfgAccount}
  onAccountRestore={restoreMfgAccount}
  initialFilter={{ types: ["inventory", "materials"] }}
  groupBy="type"
/>
// Accounts: Raw Materials Inventory, Finished Goods, Tool Budget
```

### Media Domain
```tsx
<AccountManager
  accounts={mediaAccounts}
  currentUserId="creator_303"
  onAccountCreate={createMediaAccount}
  onAccountUpdate={updateMediaAccount}
  onAccountArchive={archiveMediaAccount}
  onAccountRestore={restoreMediaAccount}
  initialFilter={{ types: ["revenue", "ad_revenue"] }}
  groupBy="currency"
/>
// Accounts: YouTube Revenue (USD), TikTok Creator Fund (USD), Patreon (EUR)
```

---

## Features

### 1. Search Accounts
```typescript
// Search by name or institution
filter.searchQuery = "chase";
// Matches: "Chase Checking", "Chase Savings"
```

### 2. Filter by Type
```typescript
// Show only specific account types
filter.types = ["checking", "savings"];
```

### 3. Filter by Currency
```typescript
// Show only USD accounts
filter.currencies = ["USD"];
```

### 4. Sort Accounts
```typescript
// Sort by name, type, balance, or last updated
sortBy = "balance";
sortDirection = "desc";  // Highest balance first
```

### 5. Group Accounts
```typescript
// Group by type or currency for better organization
groupBy = "type";
// Result: Checking (3), Savings (2), Credit Card (1)
```

### 6. Archive vs Delete
```typescript
// Soft delete: account is hidden but preserved
onAccountArchive(accountId);

// Restore archived account
onAccountRestore(accountId);

// No hard delete - preserves historical data integrity
```

### 7. Responsive Design
```typescript
// Desktop: Table layout
// Mobile: Card layout (stacks vertically)
```

---

## Accessibility

```tsx
<div
  className="account-row"
  role="listitem"
  aria-label={`${account.name}, ${formatAccountType(account.type)}, Balance: ${formattedBalance}`}
  tabIndex={0}
>
  {/* Account content */}
</div>

<button
  className="action-button edit"
  aria-label={`Edit ${account.name}`}
  onClick={onEdit}
>
  ✏️
</button>

<input
  type="text"
  aria-label="Search accounts by name or institution"
  placeholder="Search accounts..."
/>
```

---

## Related Components

**Uses:**
- None (primitive UI component)

**Used by:**
- Account settings page
- Financial dashboard
- Transaction import wizard
- Any feature requiring account management

**Similar patterns:**
- EntityManager (for managing entities)
- SourceManager (for managing research sources)
- ContactManager (for managing contacts)
