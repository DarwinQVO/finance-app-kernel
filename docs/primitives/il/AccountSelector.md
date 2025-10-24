# AccountSelector (IL Component)

## Definition

**AccountSelector** is a reusable dropdown component for selecting an account from a list. It's used throughout the application wherever users need to choose an account (transaction editing, filters, reports, transfers).

**Problem it solves:**
- No consistent pattern for selecting accounts across features
- Account dropdowns lack search/filter capabilities
- No visual grouping by account type or currency
- No quick way to create accounts inline
- Inconsistent display of account metadata (type, currency, balance)

**Solution:**
- Single dropdown component for all account selection scenarios
- Built-in search and filtering
- Visual grouping by type or currency
- Optional "Create Account" button
- Consistent account display with icon, name, type, and balance
- Keyboard navigation support

---

## Interface Contract

```typescript
interface AccountSelectorProps {
  // Data
  accounts: Account[];  // All available accounts
  currentAccountId?: string;  // Currently selected account
  excludeAccountIds?: string[];  // Accounts to hide (e.g., for transfers)

  // Callbacks
  onAccountSelect: (accountId: string) => void;
  onAccountCreate?: () => void;  // Show "Create Account" button

  // UI customization
  placeholder?: string;  // Placeholder text (default: "Select an account...")
  showBalance?: boolean;  // Show account balance in dropdown (default: true)
  showCurrencyBadge?: boolean;  // Show currency badge (default: true)
  showCreateButton?: boolean;  // Show "Create Account" button (default: false)
  groupBy?: "type" | "currency" | "none";  // Group accounts (default: "type")
  filterByType?: AccountType[];  // Only show specific account types
  filterByCurrency?: string[];  // Only show specific currencies

  // UI states
  disabled?: boolean;
  loading?: boolean;
  error?: string | null;

  // Customization
  theme?: "light" | "dark";
  size?: "small" | "medium" | "large";
  compact?: boolean;  // Compact mode (no balance, smaller icons)
}

interface Account {
  account_id: string;
  name: string;
  type: AccountType;
  currency: string;
  balance: number;
  is_archived: boolean;
  icon?: string;
  color?: string;
  institution?: string;
}

type AccountType =
  | "cash" | "checking" | "savings" | "credit_card" | "investment"  // Finance
  | "insurance" | "hsa" | "fsa"  // Healthcare
  | "trust" | "escrow" | "retainer"  // Legal
  | "grant" | "budget"  // Research
  | "inventory" | "materials"  // Manufacturing
  | "revenue" | "ad_revenue";  // Media
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useRef, useEffect } from 'react';

export const AccountSelector: React.FC<AccountSelectorProps> = ({
  accounts,
  currentAccountId,
  excludeAccountIds = [],
  onAccountSelect,
  onAccountCreate,
  placeholder = "Select an account...",
  showBalance = true,
  showCurrencyBadge = true,
  showCreateButton = false,
  groupBy = "type",
  filterByType,
  filterByCurrency,
  disabled = false,
  loading = false,
  error = null,
  theme = "light",
  size = "medium",
  compact = false
}) => {
  const [isOpen, setIsOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState("");
  const [highlightedIndex, setHighlightedIndex] = useState(0);
  const dropdownRef = useRef<HTMLDivElement>(null);

  // Find current account
  const currentAccount = accounts.find(a => a.account_id === currentAccountId);

  // Filter accounts
  const filteredAccounts = useMemo(() => {
    let result = accounts;

    // Exclude specific accounts
    if (excludeAccountIds.length > 0) {
      result = result.filter(a => !excludeAccountIds.includes(a.account_id));
    }

    // Filter out archived accounts
    result = result.filter(a => !a.is_archived);

    // Filter by type
    if (filterByType && filterByType.length > 0) {
      result = result.filter(a => filterByType.includes(a.type));
    }

    // Filter by currency
    if (filterByCurrency && filterByCurrency.length > 0) {
      result = result.filter(a => filterByCurrency.includes(a.currency));
    }

    // Search by name or institution
    if (searchQuery) {
      const query = searchQuery.toLowerCase();
      result = result.filter(a =>
        a.name.toLowerCase().includes(query) ||
        a.institution?.toLowerCase().includes(query)
      );
    }

    return result;
  }, [accounts, excludeAccountIds, filterByType, filterByCurrency, searchQuery]);

  // Group accounts
  const groupedAccounts = useMemo(() => {
    if (groupBy === "none") {
      return { "All Accounts": filteredAccounts };
    }

    const groups: Record<string, Account[]> = {};
    filteredAccounts.forEach(account => {
      const key = groupBy === "type" ? account.type : account.currency;
      if (!groups[key]) groups[key] = [];
      groups[key].push(account);
    });

    // Sort groups alphabetically
    const sortedGroups: Record<string, Account[]> = {};
    Object.keys(groups).sort().forEach(key => {
      sortedGroups[key] = groups[key];
    });

    return sortedGroups;
  }, [filteredAccounts, groupBy]);

  // Flatten accounts for keyboard navigation
  const flattenedAccounts = useMemo(() => {
    return filteredAccounts;
  }, [filteredAccounts]);

  // Close dropdown when clicking outside
  useEffect(() => {
    const handleClickOutside = (event: MouseEvent) => {
      if (dropdownRef.current && !dropdownRef.current.contains(event.target as Node)) {
        setIsOpen(false);
      }
    };

    if (isOpen) {
      document.addEventListener('mousedown', handleClickOutside);
      return () => document.removeEventListener('mousedown', handleClickOutside);
    }
  }, [isOpen]);

  // Keyboard navigation
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (!isOpen) {
      if (e.key === 'Enter' || e.key === ' ') {
        setIsOpen(true);
        e.preventDefault();
      }
      return;
    }

    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setHighlightedIndex(prev =>
          prev < flattenedAccounts.length - 1 ? prev + 1 : prev
        );
        break;

      case 'ArrowUp':
        e.preventDefault();
        setHighlightedIndex(prev => (prev > 0 ? prev - 1 : prev));
        break;

      case 'Enter':
        e.preventDefault();
        if (flattenedAccounts[highlightedIndex]) {
          handleSelect(flattenedAccounts[highlightedIndex].account_id);
        }
        break;

      case 'Escape':
        e.preventDefault();
        setIsOpen(false);
        setSearchQuery("");
        break;
    }
  };

  // Handle account selection
  const handleSelect = (accountId: string) => {
    onAccountSelect(accountId);
    setIsOpen(false);
    setSearchQuery("");
    setHighlightedIndex(0);
  };

  // Loading state
  if (loading) {
    return (
      <div className={`account-selector theme-${theme} size-${size} loading`}>
        <div className="selector-trigger disabled">
          <div className="skeleton skeleton-text"></div>
        </div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`account-selector theme-${theme} size-${size} error`}>
        <div className="selector-trigger disabled">
          <span className="error-text">⚠️ {error}</span>
        </div>
      </div>
    );
  }

  return (
    <div
      ref={dropdownRef}
      className={`account-selector theme-${theme} size-${size} ${compact ? 'compact' : ''} ${disabled ? 'disabled' : ''}`}
      onKeyDown={handleKeyDown}
    >
      {/* Trigger Button */}
      <button
        className={`selector-trigger ${isOpen ? 'open' : ''}`}
        onClick={() => !disabled && setIsOpen(!isOpen)}
        disabled={disabled}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-label="Select account"
      >
        {currentAccount ? (
          <div className="selected-account">
            <div
              className="account-icon-small"
              style={{ background: currentAccount.color }}
            >
              {currentAccount.icon || getDefaultIcon(currentAccount.type)}
            </div>
            <div className="account-info-compact">
              <span className="account-name">{currentAccount.name}</span>
              {showCurrencyBadge && (
                <span className="currency-badge">{currentAccount.currency}</span>
              )}
            </div>
            {showBalance && !compact && (
              <span className="account-balance-small">
                {formatCurrency(currentAccount.balance, currentAccount.currency)}
              </span>
            )}
          </div>
        ) : (
          <span className="placeholder">{placeholder}</span>
        )}
        <span className="dropdown-arrow">▼</span>
      </button>

      {/* Dropdown Menu */}
      {isOpen && (
        <div className="selector-dropdown" role="listbox">
          {/* Search Box */}
          <div className="search-box">
            <input
              type="text"
              placeholder="Search accounts..."
              value={searchQuery}
              onChange={(e) => {
                setSearchQuery(e.target.value);
                setHighlightedIndex(0);
              }}
              autoFocus
              className="search-input"
            />
          </div>

          {/* Create Account Button */}
          {showCreateButton && onAccountCreate && (
            <button
              className="create-account-button"
              onClick={() => {
                onAccountCreate();
                setIsOpen(false);
              }}
            >
              + Create New Account
            </button>
          )}

          {/* Account Groups */}
          {Object.keys(groupedAccounts).length === 0 ? (
            <div className="empty-state">
              <div className="empty-icon">🔍</div>
              <div className="empty-message">
                No accounts found matching "{searchQuery}"
              </div>
            </div>
          ) : (
            <div className="account-groups">
              {Object.entries(groupedAccounts).map(([groupName, groupAccounts]) => (
                <div key={groupName} className="account-group">
                  {groupBy !== "none" && (
                    <div className="group-header">
                      {formatGroupName(groupName, groupBy)}
                    </div>
                  )}

                  {groupAccounts.map((account, index) => {
                    const globalIndex = flattenedAccounts.findIndex(
                      a => a.account_id === account.account_id
                    );
                    const isHighlighted = globalIndex === highlightedIndex;
                    const isSelected = account.account_id === currentAccountId;

                    return (
                      <div
                        key={account.account_id}
                        className={`account-item ${isHighlighted ? 'highlighted' : ''} ${isSelected ? 'selected' : ''}`}
                        onClick={() => handleSelect(account.account_id)}
                        onMouseEnter={() => setHighlightedIndex(globalIndex)}
                        role="option"
                        aria-selected={isSelected}
                      >
                        <div
                          className="account-icon"
                          style={{ background: account.color }}
                        >
                          {account.icon || getDefaultIcon(account.type)}
                        </div>

                        <div className="account-info">
                          <div className="account-name-row">
                            <span className="account-name">{account.name}</span>
                            {showCurrencyBadge && (
                              <span className="currency-badge">
                                {account.currency}
                              </span>
                            )}
                          </div>
                          <div className="account-meta">
                            <span className="account-type">
                              {formatAccountType(account.type)}
                            </span>
                            {account.institution && (
                              <>
                                <span className="separator">•</span>
                                <span className="account-institution">
                                  {account.institution}
                                </span>
                              </>
                            )}
                          </div>
                        </div>

                        {showBalance && !compact && (
                          <div className="account-balance">
                            {formatCurrency(account.balance, account.currency)}
                          </div>
                        )}

                        {isSelected && (
                          <div className="selected-checkmark">✓</div>
                        )}
                      </div>
                    );
                  })}
                </div>
              ))}
            </div>
          )}
        </div>
      )}
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

function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: currency,
    minimumFractionDigits: 0,
    maximumFractionDigits: 0
  }).format(amount);
}
```

---

## Styling

```css
.account-selector {
  position: relative;
  width: 100%;
  max-width: 400px;
}

.account-selector.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Trigger Button */
.selector-trigger {
  width: 100%;
  padding: 10px 14px;
  background: var(--trigger-bg);
  border: 1px solid var(--trigger-border);
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  transition: all 0.2s;
  font-size: 14px;
}

.selector-trigger:hover:not(:disabled) {
  border-color: var(--trigger-border-hover);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.selector-trigger.open {
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.selector-trigger:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.selected-account {
  display: flex;
  align-items: center;
  gap: 10px;
  flex: 1;
  min-width: 0;
}

.account-icon-small {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  flex-shrink: 0;
}

.account-info-compact {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
  min-width: 0;
}

.account-name {
  font-weight: 500;
  color: var(--text-color);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.currency-badge {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  font-weight: 600;
  flex-shrink: 0;
}

.account-balance-small {
  font-size: 13px;
  font-weight: 600;
  color: var(--balance-color);
  margin-left: auto;
  flex-shrink: 0;
}

.placeholder {
  color: var(--placeholder-color);
  font-size: 14px;
}

.dropdown-arrow {
  color: var(--arrow-color);
  font-size: 10px;
  transition: transform 0.2s;
  flex-shrink: 0;
}

.selector-trigger.open .dropdown-arrow {
  transform: rotate(180deg);
}

/* Dropdown Menu */
.selector-dropdown {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: var(--dropdown-bg);
  border: 1px solid var(--dropdown-border);
  border-radius: 8px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.15);
  max-height: 400px;
  overflow-y: auto;
  z-index: 1000;
}

/* Search Box */
.search-box {
  padding: 12px;
  border-bottom: 1px solid var(--divider-color);
  position: sticky;
  top: 0;
  background: var(--dropdown-bg);
  z-index: 1;
}

.search-input {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
  background: var(--input-bg);
  color: var(--text-color);
}

.search-input:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

/* Create Account Button */
.create-account-button {
  width: 100%;
  padding: 12px;
  background: var(--create-button-bg);
  color: var(--create-button-color);
  border: none;
  border-bottom: 1px solid var(--divider-color);
  font-weight: 600;
  cursor: pointer;
  text-align: left;
  transition: background 0.2s;
}

.create-account-button:hover {
  background: var(--create-button-hover-bg);
}

/* Account Groups */
.account-groups {
  padding: 8px 0;
}

.account-group {
  margin-bottom: 8px;
}

.group-header {
  padding: 8px 16px;
  font-size: 11px;
  font-weight: 600;
  color: var(--group-header-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

/* Account Item */
.account-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 16px;
  cursor: pointer;
  transition: background 0.2s;
}

.account-item:hover,
.account-item.highlighted {
  background: var(--item-hover-bg);
}

.account-item.selected {
  background: var(--item-selected-bg);
}

.account-icon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  flex-shrink: 0;
}

.account-info {
  flex: 1;
  min-width: 0;
}

.account-name-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 4px;
}

.account-meta {
  font-size: 12px;
  color: var(--secondary-color);
  display: flex;
  align-items: center;
  gap: 6px;
}

.separator {
  color: var(--separator-color);
}

.account-balance {
  font-size: 14px;
  font-weight: 600;
  color: var(--balance-color);
  flex-shrink: 0;
}

.selected-checkmark {
  font-size: 16px;
  color: var(--primary-color);
  flex-shrink: 0;
}

/* Empty State */
.empty-state {
  text-align: center;
  padding: 40px 20px;
}

.empty-icon {
  font-size: 48px;
  margin-bottom: 12px;
  opacity: 0.5;
}

.empty-message {
  font-size: 14px;
  color: var(--secondary-color);
}

/* Loading State */
.skeleton {
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
  border-radius: 4px;
}

.skeleton-text {
  height: 20px;
  width: 60%;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* Size Variants */
.size-small .selector-trigger { padding: 6px 10px; font-size: 13px; }
.size-small .account-icon-small { width: 24px; height: 24px; font-size: 14px; }
.size-small .account-icon { width: 32px; height: 32px; font-size: 16px; }

.size-medium .selector-trigger { padding: 10px 14px; font-size: 14px; }
.size-medium .account-icon-small { width: 32px; height: 32px; font-size: 16px; }
.size-medium .account-icon { width: 40px; height: 40px; font-size: 20px; }

.size-large .selector-trigger { padding: 12px 16px; font-size: 16px; }
.size-large .account-icon-small { width: 40px; height: 40px; font-size: 20px; }
.size-large .account-icon { width: 48px; height: 48px; font-size: 24px; }

/* Compact Mode */
.compact .account-balance,
.compact .account-balance-small {
  display: none;
}

.compact .account-icon,
.compact .account-icon-small {
  width: 28px;
  height: 28px;
  font-size: 14px;
}

/* Theme: Light */
.theme-light {
  --trigger-bg: #ffffff;
  --trigger-border: #d1d5db;
  --trigger-border-hover: #9ca3af;
  --dropdown-bg: #ffffff;
  --dropdown-border: #d1d5db;
  --input-bg: #ffffff;
  --input-border: #d1d5db;
  --primary-color: #3b82f6;
  --text-color: #111827;
  --placeholder-color: #9ca3af;
  --arrow-color: #6b7280;
  --badge-bg: #dbeafe;
  --badge-color: #1e40af;
  --balance-color: #111827;
  --divider-color: #e5e7eb;
  --create-button-bg: #3b82f6;
  --create-button-color: #ffffff;
  --create-button-hover-bg: #2563eb;
  --group-header-color: #6b7280;
  --item-hover-bg: #f3f4f6;
  --item-selected-bg: #e0e7ff;
  --secondary-color: #6b7280;
  --separator-color: #d1d5db;
}

/* Theme: Dark */
.theme-dark {
  --trigger-bg: #374151;
  --trigger-border: #4b5563;
  --trigger-border-hover: #6b7280;
  --dropdown-bg: #1f2937;
  --dropdown-border: #374151;
  --input-bg: #374151;
  --input-border: #4b5563;
  --primary-color: #3b82f6;
  --text-color: #f3f4f6;
  --placeholder-color: #6b7280;
  --arrow-color: #9ca3af;
  --badge-bg: #1e3a8a;
  --badge-color: #93c5fd;
  --balance-color: #f3f4f6;
  --divider-color: #374151;
  --create-button-bg: #3b82f6;
  --create-button-color: #ffffff;
  --create-button-hover-bg: #2563eb;
  --group-header-color: #9ca3af;
  --item-hover-bg: #374151;
  --item-selected-bg: #1e3a8a;
  --secondary-color: #9ca3af;
  --separator-color: #4b5563;
}

/* Responsive */
@media (max-width: 768px) {
  .account-selector {
    max-width: 100%;
  }

  .selector-dropdown {
    max-height: 60vh;
  }
}
```

---

## Visual Wireframes

### Collapsed State
```
┌────────────────────────────────────────┐
│ [💰] Chase Checking    [USD]    $5,234 ▼│
└────────────────────────────────────────┘
```

### Expanded State (Grouped by Type)
```
┌────────────────────────────────────────┐
│ [💰] Chase Checking    [USD]    $5,234 ▲│
└────────────────────────────────────────┘
  ┌──────────────────────────────────────┐
  │ Search accounts...                   │
  ├──────────────────────────────────────┤
  │ + Create New Account                 │
  ├──────────────────────────────────────┤
  │ CHECKING                             │
  │ [🏦] Chase Checking    [USD]  $5,234 ✓│
  │ [🏦] Wells Fargo       [USD]  $1,890  │
  ├──────────────────────────────────────┤
  │ SAVINGS                              │
  │ [🐖] Emergency Fund    [USD]  $12,450 │
  │ [🐖] Vacation Savings  [USD]  $3,200  │
  ├──────────────────────────────────────┤
  │ CREDIT CARD                          │
  │ [💳] Amex Gold         [USD] -$2,341  │
  └──────────────────────────────────────┘
```

---

## Multi-Domain Applicability

### Finance Domain (Transaction Editing)
```tsx
<AccountSelector
  accounts={userAccounts}
  currentAccountId={transaction.account_id}
  onAccountSelect={(id) => updateTransaction({ account_id: id })}
  showBalance={true}
  showCurrencyBadge={true}
  groupBy="type"
  placeholder="Select account..."
/>
// Use case: User editing transaction, selecting which account it belongs to
```

### Finance Domain (Transfer Between Accounts)
```tsx
<div className="transfer-form">
  <label>From Account</label>
  <AccountSelector
    accounts={userAccounts}
    currentAccountId={transfer.from_account_id}
    excludeAccountIds={transfer.to_account_id ? [transfer.to_account_id] : []}
    onAccountSelect={(id) => setTransfer({ ...transfer, from_account_id: id })}
    groupBy="type"
  />

  <label>To Account</label>
  <AccountSelector
    accounts={userAccounts}
    currentAccountId={transfer.to_account_id}
    excludeAccountIds={transfer.from_account_id ? [transfer.from_account_id] : []}
    onAccountSelect={(id) => setTransfer({ ...transfer, to_account_id: id })}
    groupBy="type"
  />
</div>
// Use case: User creating transfer, can't select same account twice
```

### Healthcare Domain (Insurance Account Selector)
```tsx
<AccountSelector
  accounts={insuranceAccounts}
  currentAccountId={claim.insurance_account_id}
  onAccountSelect={(id) => updateClaim({ insurance_account_id: id })}
  filterByType={["insurance", "hsa", "fsa"]}
  groupBy="type"
  placeholder="Select insurance account..."
  showBalance={false}
/>
// Use case: Filing medical claim, selecting insurance coverage
```

### Legal Domain (Trust Account Selector)
```tsx
<AccountSelector
  accounts={trustAccounts}
  currentAccountId={transaction.trust_account_id}
  onAccountSelect={(id) => recordTransaction({ trust_account_id: id })}
  filterByType={["trust", "escrow", "retainer"]}
  groupBy="type"
  showBalance={true}
  placeholder="Select trust account..."
/>
// Use case: Recording client payment to trust account
```

### Research Domain (Grant Funding Source)
```tsx
<AccountSelector
  accounts={researchAccounts}
  currentAccountId={expense.grant_id}
  onAccountSelect={(id) => updateExpense({ grant_id: id })}
  filterByType={["grant", "budget"]}
  groupBy="currency"
  placeholder="Select funding source..."
  showBalance={true}
/>
// Use case: Recording research expense, selecting grant that covers it
```

### Manufacturing Domain (Inventory Account)
```tsx
<AccountSelector
  accounts={inventoryAccounts}
  currentAccountId={requisition.inventory_account_id}
  onAccountSelect={(id) => updateRequisition({ inventory_account_id: id })}
  filterByType={["inventory", "materials"]}
  groupBy="type"
  placeholder="Select inventory account..."
  showBalance={true}
/>
// Use case: Material requisition, selecting inventory account
```

### Media Domain (Revenue Account Selector)
```tsx
<AccountSelector
  accounts={revenueAccounts}
  currentAccountId={payment.revenue_account_id}
  onAccountSelect={(id) => recordPayment({ revenue_account_id: id })}
  filterByType={["revenue", "ad_revenue"]}
  groupBy="currency"
  placeholder="Select revenue account..."
  showBalance={true}
  showCurrencyBadge={true}
/>
// Use case: Recording platform payment, selecting revenue stream
```

---

## Features

### 1. Search Accounts
```typescript
// Type to search by name or institution
searchQuery = "chase";
// Matches: "Chase Checking", "Chase Savings"
```

### 2. Group by Type or Currency
```typescript
// Group by account type
groupBy = "type";
// Result: Checking (2), Savings (2), Credit Card (1)

// Group by currency
groupBy = "currency";
// Result: USD (4), EUR (1)
```

### 3. Filter by Type
```typescript
// Only show specific account types
filterByType = ["checking", "savings"];
// Hides: Credit Card, Investment accounts
```

### 4. Filter by Currency
```typescript
// Only show USD accounts
filterByCurrency = ["USD"];
// Hides: EUR, MXN accounts
```

### 5. Exclude Accounts
```typescript
// Exclude specific accounts (useful for transfers)
excludeAccountIds = ["account_123"];
// Account_123 won't appear in dropdown
```

### 6. Create Account Inline
```typescript
// Show "Create Account" button in dropdown
showCreateButton = true;
onAccountCreate = () => openCreateAccountModal();
```

### 7. Keyboard Navigation
```typescript
// Arrow Up/Down: Navigate accounts
// Enter: Select highlighted account
// Escape: Close dropdown
```

### 8. Show/Hide Balance
```typescript
// Show balance next to account name
showBalance = true;  // Default

// Hide balance (compact mode)
showBalance = false;
```

---

## Accessibility

```tsx
<div
  className="account-selector"
  role="combobox"
  aria-haspopup="listbox"
  aria-expanded={isOpen}
  aria-label="Select account"
>
  <button
    className="selector-trigger"
    aria-label={currentAccount ? `Selected account: ${currentAccount.name}` : "No account selected"}
  >
    {/* Trigger content */}
  </button>

  <div className="selector-dropdown" role="listbox">
    <div
      className="account-item"
      role="option"
      aria-selected={isSelected}
      aria-label={`${account.name}, ${formatAccountType(account.type)}, Balance: ${formattedBalance}`}
    >
      {/* Account content */}
    </div>
  </div>
</div>
```

---

## Related Components

**Uses:**
- None (primitive UI component)

**Used by:**
- TransactionEditor (select account for transaction)
- TransferForm (select from/to accounts)
- TransactionFilters (filter by account)
- ReportBuilder (select accounts for report)
- BudgetEditor (select account for budget category)

**Similar patterns:**
- EntitySelector (for selecting entities)
- SourceSelector (for selecting research sources)
- CategorySelector (for selecting categories)
- TagSelector (for selecting tags)

---

## Reusability Pattern

**AccountSelector is domain-agnostic.** The same component works across all domains because:

1. **Account types are extensible**: Add new types for any domain
2. **Currency support**: Works with any currency code
3. **Flexible filtering**: Filter by type, currency, or custom criteria
4. **Grouping options**: Group by type or currency to organize accounts
5. **Search built-in**: Find accounts quickly regardless of list size
6. **Keyboard navigation**: Accessible and efficient for power users

**Same selector, different contexts:**
- Finance: Bank accounts, credit cards, investment accounts
- Healthcare: Insurance policies, HSA/FSA accounts
- Legal: Trust accounts, escrow, client retainers
- Research: Grant funds, lab budgets
- Manufacturing: Inventory accounts, materials budgets
- Media: Revenue streams, ad revenue accounts
