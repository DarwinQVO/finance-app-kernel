# CounterpartySelector (IL Component)

## Definition

**CounterpartySelector** is a reusable dropdown component for selecting a counterparty from a list. It's used throughout the application wherever users need to choose a counterparty (transaction editing, filters, reports, payments).

**Problem it solves:**
- No consistent pattern for selecting counterparties across features
- Counterparty dropdowns lack search capabilities (especially for aliases)
- No visual grouping by counterparty type
- No quick way to create counterparties inline
- Inconsistent display of counterparty metadata (type, transaction count)
- Difficult to find counterparties when there are many entries

**Solution:**
- Single dropdown component for all counterparty selection scenarios
- Built-in search across canonical names AND aliases
- Visual grouping by counterparty type
- Optional "Create Counterparty" button for inline creation
- Consistent counterparty display with icon, name, type, and transaction count
- Keyboard navigation support
- Smart matching (fuzzy search on aliases)

---

## Interface Contract

```typescript
interface CounterpartySelectorProps {
  // Data
  counterparties: Counterparty[];  // All available counterparties
  currentCounterpartyId?: string;  // Currently selected counterparty
  excludeCounterpartyIds?: string[];  // Counterparties to hide
  transactionCounts?: Record<string, number>;  // Transaction count per counterparty

  // Callbacks
  onCounterpartySelect: (counterpartyId: string) => void;
  onCounterpartyCreate?: () => void;  // Show "Create Counterparty" button

  // UI customization
  placeholder?: string;  // Placeholder text (default: "Select a counterparty...")
  showTransactionCount?: boolean;  // Show transaction count (default: true)
  showTypeBadge?: boolean;  // Show type badge (default: true)
  showCreateButton?: boolean;  // Show "Create Counterparty" button (default: false)
  groupBy?: "type" | "none";  // Group counterparties (default: "type")
  filterByType?: CounterpartyType[];  // Only show specific types

  // UI states
  disabled?: boolean;
  loading?: boolean;
  error?: string | null;

  // Customization
  theme?: "light" | "dark";
  size?: "small" | "medium" | "large";
  compact?: boolean;  // Compact mode (no transaction count, smaller icons)
}

interface Counterparty {
  counterparty_id: string;
  canonical_name: string;
  aliases: string[];
  type: CounterpartyType;
  notes?: string;
  tags?: string[];
}

type CounterpartyType =
  | "merchant"      // Stores, vendors, service providers
  | "person"        // Individuals (friends, family, employees)
  | "business"      // Companies, corporations, LLCs
  | "government";   // Tax agencies, city/state departments
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useRef, useEffect } from 'react';

export const CounterpartySelector: React.FC<CounterpartySelectorProps> = ({
  counterparties,
  currentCounterpartyId,
  excludeCounterpartyIds = [],
  transactionCounts = {},
  onCounterpartySelect,
  onCounterpartyCreate,
  placeholder = "Select a counterparty...",
  showTransactionCount = true,
  showTypeBadge = true,
  showCreateButton = false,
  groupBy = "type",
  filterByType,
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

  // Find current counterparty
  const currentCounterparty = counterparties.find(
    c => c.counterparty_id === currentCounterpartyId
  );

  // Filter counterparties
  const filteredCounterparties = useMemo(() => {
    let result = counterparties;

    // Exclude specific counterparties
    if (excludeCounterpartyIds.length > 0) {
      result = result.filter(c => !excludeCounterpartyIds.includes(c.counterparty_id));
    }

    // Filter by type
    if (filterByType && filterByType.length > 0) {
      result = result.filter(c => filterByType.includes(c.type));
    }

    // Search by canonical name or aliases
    if (searchQuery) {
      const query = searchQuery.toLowerCase();
      result = result.filter(c =>
        c.canonical_name.toLowerCase().includes(query) ||
        c.aliases?.some(alias => alias.toLowerCase().includes(query))
      );
    }

    return result;
  }, [counterparties, excludeCounterpartyIds, filterByType, searchQuery]);

  // Sort by transaction count (most used first)
  const sortedCounterparties = useMemo(() => {
    return [...filteredCounterparties].sort((a, b) => {
      const countA = transactionCounts[a.counterparty_id] || 0;
      const countB = transactionCounts[b.counterparty_id] || 0;
      return countB - countA;  // Descending order
    });
  }, [filteredCounterparties, transactionCounts]);

  // Group counterparties
  const groupedCounterparties = useMemo(() => {
    if (groupBy === "none") {
      return { "All Counterparties": sortedCounterparties };
    }

    const groups: Record<string, Counterparty[]> = {};
    sortedCounterparties.forEach(counterparty => {
      const key = counterparty.type;
      if (!groups[key]) groups[key] = [];
      groups[key].push(counterparty);
    });

    // Sort groups by priority: person, merchant, business, government
    const sortedGroups: Record<string, Counterparty[]> = {};
    const order: CounterpartyType[] = ["person", "merchant", "business", "government"];
    order.forEach(type => {
      if (groups[type]) {
        sortedGroups[type] = groups[type];
      }
    });

    return sortedGroups;
  }, [sortedCounterparties, groupBy]);

  // Flatten counterparties for keyboard navigation
  const flattenedCounterparties = useMemo(() => {
    return sortedCounterparties;
  }, [sortedCounterparties]);

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
          prev < flattenedCounterparties.length - 1 ? prev + 1 : prev
        );
        break;

      case 'ArrowUp':
        e.preventDefault();
        setHighlightedIndex(prev => (prev > 0 ? prev - 1 : prev));
        break;

      case 'Enter':
        e.preventDefault();
        if (flattenedCounterparties[highlightedIndex]) {
          handleSelect(flattenedCounterparties[highlightedIndex].counterparty_id);
        }
        break;

      case 'Escape':
        e.preventDefault();
        setIsOpen(false);
        setSearchQuery("");
        break;
    }
  };

  // Handle counterparty selection
  const handleSelect = (counterpartyId: string) => {
    onCounterpartySelect(counterpartyId);
    setIsOpen(false);
    setSearchQuery("");
    setHighlightedIndex(0);
  };

  // Loading state
  if (loading) {
    return (
      <div className={`counterparty-selector theme-${theme} size-${size} loading`}>
        <div className="selector-trigger disabled">
          <div className="skeleton skeleton-text"></div>
        </div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`counterparty-selector theme-${theme} size-${size} error`}>
        <div className="selector-trigger disabled">
          <span className="error-text">⚠️ {error}</span>
        </div>
      </div>
    );
  }

  return (
    <div
      ref={dropdownRef}
      className={`counterparty-selector theme-${theme} size-${size} ${compact ? 'compact' : ''} ${disabled ? 'disabled' : ''}`}
      onKeyDown={handleKeyDown}
    >
      {/* Trigger Button */}
      <button
        className={`selector-trigger ${isOpen ? 'open' : ''}`}
        onClick={() => !disabled && setIsOpen(!isOpen)}
        disabled={disabled}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-label="Select counterparty"
      >
        {currentCounterparty ? (
          <div className="selected-counterparty">
            <div className="counterparty-icon-small">
              {getDefaultIcon(currentCounterparty.type)}
            </div>
            <div className="counterparty-info-compact">
              <span className="counterparty-name">{currentCounterparty.canonical_name}</span>
              {showTypeBadge && (
                <span className="type-badge">
                  {formatCounterpartyType(currentCounterparty.type)}
                </span>
              )}
            </div>
            {showTransactionCount && !compact && (
              <span className="transaction-count-small">
                {transactionCounts[currentCounterparty.counterparty_id] || 0} txns
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
              placeholder="Search by name or alias..."
              value={searchQuery}
              onChange={(e) => {
                setSearchQuery(e.target.value);
                setHighlightedIndex(0);
              }}
              autoFocus
              className="search-input"
            />
          </div>

          {/* Create Counterparty Button */}
          {showCreateButton && onCounterpartyCreate && (
            <button
              className="create-counterparty-button"
              onClick={() => {
                onCounterpartyCreate();
                setIsOpen(false);
              }}
            >
              + Create New Counterparty
            </button>
          )}

          {/* Counterparty Groups */}
          {Object.keys(groupedCounterparties).length === 0 ? (
            <div className="empty-state">
              <div className="empty-icon">🔍</div>
              <div className="empty-message">
                No counterparties found matching "{searchQuery}"
              </div>
            </div>
          ) : (
            <div className="counterparty-groups">
              {Object.entries(groupedCounterparties).map(([groupName, groupCounterparties]) => (
                <div key={groupName} className="counterparty-group">
                  {groupBy !== "none" && (
                    <div className="group-header">
                      {formatCounterpartyType(groupName as CounterpartyType)}
                    </div>
                  )}

                  {groupCounterparties.map((counterparty, index) => {
                    const globalIndex = flattenedCounterparties.findIndex(
                      c => c.counterparty_id === counterparty.counterparty_id
                    );
                    const isHighlighted = globalIndex === highlightedIndex;
                    const isSelected = counterparty.counterparty_id === currentCounterpartyId;
                    const txCount = transactionCounts[counterparty.counterparty_id] || 0;

                    return (
                      <div
                        key={counterparty.counterparty_id}
                        className={`counterparty-item ${isHighlighted ? 'highlighted' : ''} ${isSelected ? 'selected' : ''}`}
                        onClick={() => handleSelect(counterparty.counterparty_id)}
                        onMouseEnter={() => setHighlightedIndex(globalIndex)}
                        role="option"
                        aria-selected={isSelected}
                      >
                        <div className="counterparty-icon">
                          {getDefaultIcon(counterparty.type)}
                        </div>

                        <div className="counterparty-info">
                          <div className="counterparty-name-row">
                            <span className="counterparty-name">
                              {highlightMatch(counterparty.canonical_name, searchQuery)}
                            </span>
                            {showTypeBadge && (
                              <span className="type-badge">
                                {formatCounterpartyType(counterparty.type)}
                              </span>
                            )}
                          </div>
                          {counterparty.aliases.length > 0 && (
                            <div className="counterparty-meta">
                              <span className="aliases-preview">
                                {highlightMatch(
                                  counterparty.aliases.slice(0, 2).join(', '),
                                  searchQuery
                                )}
                                {counterparty.aliases.length > 2 && (
                                  <span className="alias-more">
                                    {' '}+{counterparty.aliases.length - 2} more
                                  </span>
                                )}
                              </span>
                            </div>
                          )}
                        </div>

                        {showTransactionCount && !compact && (
                          <div className="transaction-count">
                            {txCount}
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

// Highlight matching text
function highlightMatch(text: string, query: string): React.ReactNode {
  if (!query) return text;

  const index = text.toLowerCase().indexOf(query.toLowerCase());
  if (index === -1) return text;

  return (
    <>
      {text.slice(0, index)}
      <mark className="search-highlight">{text.slice(index, index + query.length)}</mark>
      {text.slice(index + query.length)}
    </>
  );
}
```

---

## Styling

```css
.counterparty-selector {
  position: relative;
  width: 100%;
  max-width: 400px;
}

.counterparty-selector.disabled {
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

.selected-counterparty {
  display: flex;
  align-items: center;
  gap: 10px;
  flex: 1;
  min-width: 0;
}

.counterparty-icon-small {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--icon-bg);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  flex-shrink: 0;
}

.counterparty-info-compact {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
  min-width: 0;
}

.counterparty-name {
  font-weight: 500;
  color: var(--text-color);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.type-badge {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  font-weight: 600;
  flex-shrink: 0;
}

.transaction-count-small {
  font-size: 12px;
  font-weight: 600;
  color: var(--count-color);
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

/* Create Counterparty Button */
.create-counterparty-button {
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

.create-counterparty-button:hover {
  background: var(--create-button-hover-bg);
}

/* Counterparty Groups */
.counterparty-groups {
  padding: 8px 0;
}

.counterparty-group {
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

/* Counterparty Item */
.counterparty-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 16px;
  cursor: pointer;
  transition: background 0.2s;
}

.counterparty-item:hover,
.counterparty-item.highlighted {
  background: var(--item-hover-bg);
}

.counterparty-item.selected {
  background: var(--item-selected-bg);
}

.counterparty-icon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: var(--icon-bg);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  flex-shrink: 0;
}

.counterparty-info {
  flex: 1;
  min-width: 0;
}

.counterparty-name-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 4px;
}

.counterparty-meta {
  font-size: 12px;
  color: var(--secondary-color);
  display: flex;
  align-items: center;
  gap: 6px;
}

.aliases-preview {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.alias-more {
  font-style: italic;
  color: var(--tertiary-color);
}

.transaction-count {
  font-size: 13px;
  font-weight: 600;
  color: var(--count-color);
  padding: 4px 10px;
  background: var(--count-bg);
  border-radius: 12px;
  flex-shrink: 0;
}

.selected-checkmark {
  font-size: 16px;
  color: var(--primary-color);
  flex-shrink: 0;
}

/* Search Highlighting */
.search-highlight {
  background: var(--highlight-bg);
  color: var(--highlight-color);
  font-weight: 600;
  padding: 0 2px;
  border-radius: 2px;
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
.size-small .counterparty-icon-small { width: 24px; height: 24px; font-size: 14px; }
.size-small .counterparty-icon { width: 32px; height: 32px; font-size: 16px; }

.size-medium .selector-trigger { padding: 10px 14px; font-size: 14px; }
.size-medium .counterparty-icon-small { width: 32px; height: 32px; font-size: 16px; }
.size-medium .counterparty-icon { width: 40px; height: 40px; font-size: 20px; }

.size-large .selector-trigger { padding: 12px 16px; font-size: 16px; }
.size-large .counterparty-icon-small { width: 40px; height: 40px; font-size: 20px; }
.size-large .counterparty-icon { width: 48px; height: 48px; font-size: 24px; }

/* Compact Mode */
.compact .transaction-count,
.compact .transaction-count-small {
  display: none;
}

.compact .counterparty-icon,
.compact .counterparty-icon-small {
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
  --count-color: #6b7280;
  --count-bg: #f3f4f6;
  --divider-color: #e5e7eb;
  --create-button-bg: #3b82f6;
  --create-button-color: #ffffff;
  --create-button-hover-bg: #2563eb;
  --group-header-color: #6b7280;
  --item-hover-bg: #f3f4f6;
  --item-selected-bg: #e0e7ff;
  --secondary-color: #6b7280;
  --tertiary-color: #9ca3af;
  --icon-bg: #f3f4f6;
  --highlight-bg: #fef3c7;
  --highlight-color: #92400e;
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
  --count-color: #9ca3af;
  --count-bg: #374151;
  --divider-color: #374151;
  --create-button-bg: #3b82f6;
  --create-button-color: #ffffff;
  --create-button-hover-bg: #2563eb;
  --group-header-color: #9ca3af;
  --item-hover-bg: #374151;
  --item-selected-bg: #1e3a8a;
  --secondary-color: #9ca3af;
  --tertiary-color: #6b7280;
  --icon-bg: #4b5563;
  --highlight-bg: #78350f;
  --highlight-color: #fef3c7;
}

/* Responsive */
@media (max-width: 768px) {
  .counterparty-selector {
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
│ [🏪] Amazon    [Merchant]    247 txns ▼│
└────────────────────────────────────────┘
```

### Expanded State (Grouped by Type)
```
┌────────────────────────────────────────┐
│ [🏪] Amazon    [Merchant]    247 txns ▲│
└────────────────────────────────────────┘
  ┌──────────────────────────────────────┐
  │ Search by name or alias...           │
  ├──────────────────────────────────────┤
  │ + Create New Counterparty            │
  ├──────────────────────────────────────┤
  │ PERSON                               │
  │ [👤] John Smith     [Person]      89 ✓│
  │      johnsmith@email.com             │
  │ [👤] Jane Doe       [Person]      34  │
  │      J. Doe, Jane D                  │
  ├──────────────────────────────────────┤
  │ MERCHANT                             │
  │ [🏪] Amazon         [Merchant]   247  │
  │      amzn.com, amazon.com, AWS       │
  │ [🏪] Starbucks      [Merchant]   156  │
  │      Starbucks #1234                 │
  ├──────────────────────────────────────┤
  │ BUSINESS                             │
  │ [🏢] Acme Corp      [Business]    45  │
  │      Acme Corporation, Acme Inc      │
  └──────────────────────────────────────┘
```

### Search with Highlighting
```
┌────────────────────────────────────────┐
│ Search: "amaz"                         │
├────────────────────────────────────────┤
│ MERCHANT                               │
│ [🏪] Amazon         [Merchant]   247   │
│      amzn.com, amazon.com, AWS         │
│      ^^^^          ^^^^^                │
│     (highlighted in yellow)            │
└────────────────────────────────────────┘
```

---

## Multi-Domain Applicability

### Finance Domain (Transaction Editing)
```tsx
<CounterpartySelector
  counterparties={allCounterparties}
  currentCounterpartyId={transaction.counterparty_id}
  transactionCounts={txCounts}
  onCounterpartySelect={(id) => updateTransaction({ counterparty_id: id })}
  showTransactionCount={true}
  showTypeBadge={true}
  groupBy="type"
  placeholder="Select recipient or sender..."
/>
// Use case: User editing transaction, selecting who they paid or received from
```

### Finance Domain (Filter Transactions)
```tsx
<CounterpartySelector
  counterparties={allCounterparties}
  currentCounterpartyId={filter.counterparty_id}
  transactionCounts={txCounts}
  onCounterpartySelect={(id) => setFilter({ ...filter, counterparty_id: id })}
  showTransactionCount={true}
  groupBy="type"
  placeholder="Filter by counterparty..."
  showCreateButton={false}
/>
// Use case: Filtering transaction list by specific counterparty
```

### Healthcare Domain (Insurance Provider Selector)
```tsx
<CounterpartySelector
  counterparties={healthcareCounterparties}
  currentCounterpartyId={claim.provider_id}
  transactionCounts={claimCounts}
  onCounterpartySelect={(id) => updateClaim({ provider_id: id })}
  filterByType={["merchant", "business"]}  // Only providers
  groupBy="type"
  placeholder="Select healthcare provider..."
  showTransactionCount={false}
/>
// Use case: Filing medical claim, selecting doctor or hospital
```

### Legal Domain (Client Selector)
```tsx
<CounterpartySelector
  counterparties={legalCounterparties}
  currentCounterpartyId={case.client_id}
  transactionCounts={caseCounts}
  onCounterpartySelect={(id) => updateCase({ client_id: id })}
  filterByType={["person", "business"]}  // Only clients
  groupBy="type"
  placeholder="Select client..."
  showTransactionCount={true}
  showCreateButton={true}
  onCounterpartyCreate={() => openCreateClientModal()}
/>
// Use case: Assigning case to client
```

### Research Domain (RSRCH - Utilitario) - Source Provider Selector
```tsx
<CounterpartySelector
  counterparties={rsrchCounterparties}
  currentCounterpartyId={expense.source_provider_id}
  transactionCounts={sourceProviderCounts}
  onCounterpartySelect={(id) => updateExpense({ source_provider_id: id })}
  filterByType={["api_provider", "media"]}  // Only source providers
  groupBy="type"
  placeholder="Select source provider..."
  showTransactionCount={true}
/>
// Use case: Recording web scraping expense, selecting API provider (Twitter API, etc.)
```

### Manufacturing Domain (Supplier Selector)
```tsx
<CounterpartySelector
  counterparties={manufacturingCounterparties}
  currentCounterpartyId={order.supplier_id}
  transactionCounts={orderCounts}
  onCounterpartySelect={(id) => updateOrder({ supplier_id: id })}
  filterByType={["merchant", "business"]}  // Only suppliers
  groupBy="type"
  placeholder="Select supplier..."
  showTransactionCount={true}
/>
// Use case: Creating purchase order, selecting supplier
```

### Media Domain (Sponsor Selector)
```tsx
<CounterpartySelector
  counterparties={mediaCounterparties}
  currentCounterpartyId={sponsorship.sponsor_id}
  transactionCounts={sponsorshipCounts}
  onCounterpartySelect={(id) => updateSponsorship({ sponsor_id: id })}
  filterByType={["business"]}  // Only companies
  groupBy="type"
  placeholder="Select sponsor..."
  showTransactionCount={true}
  showCreateButton={true}
/>
// Use case: Recording sponsorship deal, selecting brand
```

---

## Features

### 1. Search Canonical Names + Aliases
```typescript
// Search works across both canonical name and all aliases
searchQuery = "amaz";
// Matches:
// - "Amazon" (canonical name)
// - "amzn.com" (alias)
// - "amazon.com" (alias)
```

### 2. Group by Type
```typescript
// Group counterparties by type for better organization
groupBy = "type";
// Result: Person (12), Merchant (45), Business (23), Government (5)
```

### 3. Filter by Type
```typescript
// Only show specific counterparty types
filterByType = ["merchant", "business"];
// Hides: Person, Government
```

### 4. Exclude Counterparties
```typescript
// Exclude specific counterparties
excludeCounterpartyIds = ["counterparty_123"];
// Counterparty_123 won't appear in dropdown
```

### 5. Create Counterparty Inline
```typescript
// Show "Create Counterparty" button in dropdown
showCreateButton = true;
onCounterpartyCreate = () => openCreateCounterpartyModal();
```

### 6. Keyboard Navigation
```typescript
// Arrow Up/Down: Navigate counterparties
// Enter: Select highlighted counterparty
// Escape: Close dropdown
```

### 7. Show Transaction Count
```typescript
// Show number of transactions next to each counterparty
showTransactionCount = true;
// Helps users pick the right counterparty (most used = likely correct)
```

### 8. Smart Sorting
```typescript
// Automatically sorts by transaction count (most used first)
// Makes frequently used counterparties easier to find
```

---

## Accessibility

```tsx
<div
  className="counterparty-selector"
  role="combobox"
  aria-haspopup="listbox"
  aria-expanded={isOpen}
  aria-label="Select counterparty"
>
  <button
    className="selector-trigger"
    aria-label={
      currentCounterparty
        ? `Selected counterparty: ${currentCounterparty.canonical_name}`
        : "No counterparty selected"
    }
  >
    {/* Trigger content */}
  </button>

  <div className="selector-dropdown" role="listbox">
    <div
      className="counterparty-item"
      role="option"
      aria-selected={isSelected}
      aria-label={`${counterparty.canonical_name}, ${formatCounterpartyType(counterparty.type)}, ${txCount} transactions`}
    >
      {/* Counterparty content */}
    </div>
  </div>
</div>
```

---

## Related Components

**Uses:**
- None (primitive UI component)

**Used by:**
- TransactionEditor (select counterparty for transaction)
- TransactionFilters (filter by counterparty)
- ReportBuilder (select counterparties for report)
- PaymentForm (select payment recipient)
- InvoiceEditor (select client or vendor)

**Similar patterns:**
- AccountSelector (for selecting accounts)
- EntitySelector (for selecting entities)
- CategorySelector (for selecting categories)
- TagSelector (for selecting tags)

---

## Reusability Pattern

**CounterpartySelector is domain-agnostic.** The same component works across all domains because:

1. **Counterparty types are extensible**: Add new types for any domain
2. **Flexible filtering**: Filter by type or custom criteria
3. **Alias support**: Handle multiple names for same entity
4. **Search built-in**: Find counterparties quickly regardless of list size
5. **Transaction count**: Shows usage frequency to help disambiguation
6. **Keyboard navigation**: Accessible and efficient for power users

**Same selector, different contexts:**
- Finance: Merchants, people, businesses (payment recipients/senders)
- Healthcare: Doctors, hospitals, insurers (medical providers)
- Legal: Clients, opposing parties, government agencies
- Research: Grant agencies, universities, suppliers
- Manufacturing: Suppliers, distributors, customers
- Media: Sponsors, platforms, collaborators
