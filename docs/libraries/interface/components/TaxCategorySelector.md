# TaxCategorySelector (IL Component)

## Definition

**TaxCategorySelector** is a dropdown component for selecting tax categories with hierarchical display, search, jurisdiction filtering, and auto-suggestion integration. It provides a consistent interface for classifying transactions according to regulatory frameworks (tax law, healthcare compliance, legal requirements).

**Problem it solves:**
- No standardized UI for selecting regulatory categories across different domains
- Hierarchical taxonomies are hard to navigate in flat dropdowns
- No visual distinction between parent and child categories
- Users can't quickly find categories using search
- Compliance levels (deduction rates, reimbursement rates) are not displayed
- Auto-suggestions are disconnected from manual selection workflow
- No way to filter categories by jurisdiction when multi-framework support exists

**Solution:**
- Hierarchical dropdown with visual indentation for parent-child relationships
- Full-text search across category names and descriptions
- Jurisdiction filter for multi-framework scenarios (USA + Mexico, HIPAA + Medicare)
- Deduction rate badges (100%, 50%, 0%) displayed inline
- Auto-suggestion integration with confidence scores
- Keyboard navigation (↑↓ select, Enter confirm, Esc close)
- Group by taxonomy (Schedule C, SAT, CPT, ICD-10)
- Responsive design adapts to mobile

---

## Interface Contract

```typescript
interface TaxCategorySelectorProps {
  // Data
  categories: TaxCategory[];  // All available categories for jurisdiction
  selectedCategoryId: string | null;  // Currently selected category
  jurisdiction: string;  // "usa_federal", "mexico_federal", etc.
  availableJurisdictions?: string[];  // For multi-jurisdiction filtering

  // Auto-suggestions
  suggestions?: CategorySuggestion[];  // Auto-suggested categories with confidence
  showSuggestions?: boolean;  // Show suggestions section (default: true)

  // Callbacks
  onSelect: (categoryId: string) => void;
  onClear?: () => void;  // Clear selection
  onJurisdictionChange?: (jurisdiction: string) => void;

  // UI states
  loading?: boolean;
  error?: string | null;
  disabled?: boolean;

  // Display options
  placeholder?: string;  // Default: "Select tax category..."
  groupByTaxonomy?: boolean;  // Group categories by taxonomy (default: true)
  showDeductionRate?: boolean;  // Show deduction rate badges (default: true)
  showConfidenceScores?: boolean;  // Show confidence for suggestions (default: true)
  compact?: boolean;  // Compact view for inline usage

  // Customization
  theme?: "light" | "dark";
  maxHeight?: number;  // Max height of dropdown (default: 400px)
}

interface TaxCategory {
  category_id: string;
  name: string;
  jurisdiction: string;  // "usa_federal", "mexico_federal", etc.
  taxonomy: string;  // "schedule_c", "sat", "cpt", "icd10", etc.
  code: string;  // "line_8", "84111506", "99213", "E11.9", etc.
  parent_id: string | null;  // For hierarchical display
  deduction_rate: number;  // 0.0 to 1.0 (100%, 50%, 0%)
  is_system: boolean;  // System vs user-created category
  description?: string;
  depth?: number;  // Tree depth for indentation (0 = root, 1 = child, etc.)
}

interface CategorySuggestion {
  category_id: string;
  category_name: string;
  confidence: number;  // 0.0 to 1.0
  reason: string;  // "Based on 10 similar Uber transactions"
  deduction_rate: number;
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo, useRef, useEffect } from 'react';

export const TaxCategorySelector: React.FC<TaxCategorySelectorProps> = ({
  categories,
  selectedCategoryId,
  jurisdiction,
  availableJurisdictions = [jurisdiction],
  suggestions = [],
  showSuggestions = true,
  onSelect,
  onClear,
  onJurisdictionChange,
  loading = false,
  error = null,
  disabled = false,
  placeholder = "Select tax category...",
  groupByTaxonomy = true,
  showDeductionRate = true,
  showConfidenceScores = true,
  compact = false,
  theme = "light",
  maxHeight = 400
}) => {
  const [isOpen, setIsOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState("");
  const [selectedJurisdiction, setSelectedJurisdiction] = useState(jurisdiction);
  const [highlightedIndex, setHighlightedIndex] = useState(0);
  const dropdownRef = useRef<HTMLDivElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  // Close dropdown when clicking outside
  useEffect(() => {
    const handleClickOutside = (event: MouseEvent) => {
      if (dropdownRef.current && !dropdownRef.current.contains(event.target as Node)) {
        setIsOpen(false);
        setSearchQuery("");
      }
    };

    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, []);

  // Build category tree (depth-first traversal)
  const categoryTree = useMemo(() => {
    const buildTree = (parentId: string | null, depth: number = 0): TaxCategory[] => {
      const children = categories
        .filter(c => c.parent_id === parentId && c.jurisdiction === selectedJurisdiction)
        .map(c => ({
          ...c,
          depth
        }));

      return children.flatMap(child => [
        child,
        ...buildTree(child.category_id, depth + 1)
      ]);
    };

    return buildTree(null);
  }, [categories, selectedJurisdiction]);

  // Filter categories by search query
  const filteredCategories = useMemo(() => {
    if (!searchQuery.trim()) return categoryTree;

    const query = searchQuery.toLowerCase();
    return categoryTree.filter(c =>
      c.name.toLowerCase().includes(query) ||
      c.description?.toLowerCase().includes(query) ||
      c.code.toLowerCase().includes(query)
    );
  }, [categoryTree, searchQuery]);

  // Group categories by taxonomy
  const groupedCategories = useMemo(() => {
    if (!groupByTaxonomy) {
      return { "All Categories": filteredCategories };
    }

    const groups: Record<string, TaxCategory[]> = {};
    filteredCategories.forEach(category => {
      const taxonomyKey = formatTaxonomyName(category.taxonomy);
      if (!groups[taxonomyKey]) groups[taxonomyKey] = [];
      groups[taxonomyKey].push(category);
    });

    return groups;
  }, [filteredCategories, groupByTaxonomy]);

  // Flatten grouped categories for keyboard navigation
  const flattenedCategories = useMemo(() => {
    return Object.values(groupedCategories).flat();
  }, [groupedCategories]);

  // Find selected category
  const selectedCategory = useMemo(() => {
    return categories.find(c => c.category_id === selectedCategoryId);
  }, [categories, selectedCategoryId]);

  // Filter suggestions by jurisdiction
  const filteredSuggestions = useMemo(() => {
    if (!showSuggestions) return [];
    return suggestions.filter(s => {
      const category = categories.find(c => c.category_id === s.category_id);
      return category?.jurisdiction === selectedJurisdiction;
    });
  }, [suggestions, categories, selectedJurisdiction, showSuggestions]);

  // Handle keyboard navigation
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (!isOpen) {
      if (e.key === "Enter" || e.key === " ") {
        e.preventDefault();
        setIsOpen(true);
      }
      return;
    }

    switch (e.key) {
      case "ArrowDown":
        e.preventDefault();
        setHighlightedIndex(prev =>
          Math.min(prev + 1, flattenedCategories.length - 1)
        );
        break;

      case "ArrowUp":
        e.preventDefault();
        setHighlightedIndex(prev => Math.max(prev - 1, 0));
        break;

      case "Enter":
        e.preventDefault();
        if (flattenedCategories[highlightedIndex]) {
          handleSelect(flattenedCategories[highlightedIndex].category_id);
        }
        break;

      case "Escape":
        e.preventDefault();
        setIsOpen(false);
        setSearchQuery("");
        break;

      case "Tab":
        setIsOpen(false);
        setSearchQuery("");
        break;
    }
  };

  // Handle category selection
  const handleSelect = (categoryId: string) => {
    onSelect(categoryId);
    setIsOpen(false);
    setSearchQuery("");
    setHighlightedIndex(0);
  };

  // Handle jurisdiction change
  const handleJurisdictionChange = (newJurisdiction: string) => {
    setSelectedJurisdiction(newJurisdiction);
    onJurisdictionChange?.(newJurisdiction);
    setSearchQuery("");
  };

  // Loading state
  if (loading) {
    return (
      <div className={`tax-category-selector theme-${theme} loading ${compact ? 'compact' : ''}`}>
        <div className="selector-trigger skeleton">
          <div className="skeleton-text"></div>
        </div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`tax-category-selector theme-${theme} error ${compact ? 'compact' : ''}`}>
        <div className="error-message">
          <span className="error-icon">⚠️</span>
          {error}
        </div>
      </div>
    );
  }

  return (
    <div
      ref={dropdownRef}
      className={`tax-category-selector theme-${theme} ${compact ? 'compact' : ''} ${isOpen ? 'open' : ''}`}
      onKeyDown={handleKeyDown}
    >
      {/* Trigger Button */}
      <button
        ref={inputRef}
        className={`selector-trigger ${selectedCategory ? 'has-value' : ''}`}
        onClick={() => !disabled && setIsOpen(!isOpen)}
        disabled={disabled}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-label="Select tax category"
      >
        {selectedCategory ? (
          <div className="selected-category">
            <span className="category-name">{selectedCategory.name}</span>
            {showDeductionRate && (
              <DeductionRateBadge rate={selectedCategory.deduction_rate} />
            )}
          </div>
        ) : (
          <span className="placeholder">{placeholder}</span>
        )}

        <div className="trigger-actions">
          {selectedCategory && onClear && (
            <button
              className="clear-button"
              onClick={(e) => {
                e.stopPropagation();
                onClear();
              }}
              aria-label="Clear selection"
            >
              ×
            </button>
          )}
          <span className="dropdown-arrow">{isOpen ? "▲" : "▼"}</span>
        </div>
      </button>

      {/* Dropdown Panel */}
      {isOpen && (
        <div className="dropdown-panel" style={{ maxHeight: `${maxHeight}px` }}>
          {/* Jurisdiction Filter (if multi-jurisdiction) */}
          {availableJurisdictions.length > 1 && (
            <div className="jurisdiction-filter">
              <label>Jurisdiction:</label>
              <select
                value={selectedJurisdiction}
                onChange={(e) => handleJurisdictionChange(e.target.value)}
                className="jurisdiction-select"
              >
                {availableJurisdictions.map(j => (
                  <option key={j} value={j}>
                    {formatJurisdictionName(j)}
                  </option>
                ))}
              </select>
            </div>
          )}

          {/* Search Input */}
          <div className="search-container">
            <input
              type="text"
              className="search-input"
              placeholder="Search categories..."
              value={searchQuery}
              onChange={(e) => setSearchQuery(e.target.value)}
              autoFocus
            />
            {searchQuery && (
              <button
                className="clear-search"
                onClick={() => setSearchQuery("")}
                aria-label="Clear search"
              >
                ×
              </button>
            )}
          </div>

          {/* Auto-Suggestions Section */}
          {filteredSuggestions.length > 0 && !searchQuery && (
            <div className="suggestions-section">
              <div className="section-header">
                <span className="section-icon">💡</span>
                <span className="section-title">Suggested</span>
              </div>
              <div className="suggestion-list">
                {filteredSuggestions.map((suggestion, index) => (
                  <button
                    key={suggestion.category_id}
                    className={`suggestion-item ${highlightedIndex === index ? 'highlighted' : ''}`}
                    onClick={() => handleSelect(suggestion.category_id)}
                    onMouseEnter={() => setHighlightedIndex(index)}
                  >
                    <div className="suggestion-main">
                      <span className="suggestion-name">{suggestion.category_name}</span>
                      {showDeductionRate && (
                        <DeductionRateBadge rate={suggestion.deduction_rate} />
                      )}
                    </div>
                    {showConfidenceScores && (
                      <div className="suggestion-meta">
                        <span className="confidence-badge">
                          {Math.round(suggestion.confidence * 100)}% confident
                        </span>
                        <span className="suggestion-reason">{suggestion.reason}</span>
                      </div>
                    )}
                  </button>
                ))}
              </div>
              <div className="section-divider"></div>
            </div>
          )}

          {/* Category List */}
          {Object.keys(groupedCategories).length === 0 ? (
            <div className="empty-state">
              <div className="empty-icon">🔍</div>
              <div className="empty-message">
                {searchQuery
                  ? `No categories found matching "${searchQuery}"`
                  : "No categories available"}
              </div>
            </div>
          ) : (
            <div className="category-groups" role="listbox">
              {Object.entries(groupedCategories).map(([taxonomyName, taxonomyCategories]) => (
                <div key={taxonomyName} className="category-group">
                  {groupByTaxonomy && (
                    <div className="taxonomy-header">{taxonomyName}</div>
                  )}
                  <div className="category-list">
                    {taxonomyCategories.map((category, index) => {
                      const globalIndex = flattenedCategories.indexOf(category);
                      return (
                        <CategoryItem
                          key={category.category_id}
                          category={category}
                          isSelected={category.category_id === selectedCategoryId}
                          isHighlighted={highlightedIndex === globalIndex}
                          showDeductionRate={showDeductionRate}
                          onSelect={handleSelect}
                          onMouseEnter={() => setHighlightedIndex(globalIndex)}
                        />
                      );
                    })}
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      )}
    </div>
  );
};

// Category Item Component
const CategoryItem: React.FC<{
  category: TaxCategory;
  isSelected: boolean;
  isHighlighted: boolean;
  showDeductionRate: boolean;
  onSelect: (categoryId: string) => void;
  onMouseEnter: () => void;
}> = ({ category, isSelected, isHighlighted, showDeductionRate, onSelect, onMouseEnter }) => {
  return (
    <button
      className={`category-item ${isSelected ? 'selected' : ''} ${isHighlighted ? 'highlighted' : ''}`}
      style={{ paddingLeft: `${12 + (category.depth || 0) * 20}px` }}
      onClick={() => onSelect(category.category_id)}
      onMouseEnter={onMouseEnter}
      role="option"
      aria-selected={isSelected}
    >
      <div className="category-main">
        {category.depth! > 0 && <span className="indent-marker">└─</span>}
        <span className="category-name">{category.name}</span>
        {!category.is_system && <span className="custom-badge">Custom</span>}
      </div>

      <div className="category-meta">
        {category.code && (
          <span className="category-code">{category.code}</span>
        )}
        {showDeductionRate && (
          <DeductionRateBadge rate={category.deduction_rate} />
        )}
      </div>

      {category.description && (
        <div className="category-description">{category.description}</div>
      )}
    </button>
  );
};

// Deduction Rate Badge Component
const DeductionRateBadge: React.FC<{ rate: number }> = ({ rate }) => {
  const percentage = Math.round(rate * 100);
  const badgeClass =
    percentage === 100 ? 'deduction-full' :
    percentage > 0 ? 'deduction-partial' :
    'deduction-none';

  return (
    <span className={`deduction-badge ${badgeClass}`}>
      {percentage}% deductible
    </span>
  );
};

// Helper functions
function formatTaxonomyName(taxonomy: string): string {
  const labels: Record<string, string> = {
    schedule_c: "Schedule C (USA)",
    sat: "SAT (Mexico)",
    cpt: "CPT Codes",
    icd10: "ICD-10 Codes",
    case_law: "Case Law Categories",
    nsf: "NSF Categories",
    nih: "NIH Categories"
  };
  return labels[taxonomy] || taxonomy;
}

function formatJurisdictionName(jurisdiction: string): string {
  const labels: Record<string, string> = {
    usa_federal: "USA Federal",
    mexico_federal: "Mexico Federal",
    hipaa: "HIPAA",
    medicare: "Medicare",
    nsf: "NSF",
    nih: "NIH"
  };
  return labels[jurisdiction] || jurisdiction;
}
```

---

## User Interactions

### 1. Open Dropdown
```
User: Clicks trigger button
System:
  1. Opens dropdown panel
  2. Auto-focuses search input
  3. Shows suggestions (if available)
  4. Highlights first item
```

### 2. Search Categories
```
User: Types "travel" in search
System:
  1. Filters categories in real-time
  2. Highlights matches in name/description
  3. Maintains hierarchical structure
  4. Shows "No results" if no matches
```

### 3. Accept Auto-Suggestion
```
User: Clicks suggested "Travel" category (95% confidence)
System:
  1. Applies category selection
  2. Closes dropdown
  3. Displays selected category with deduction rate badge
  4. Triggers onSelect callback
```

### 4. Navigate with Keyboard
```
User: Presses ↓ arrow key
System:
  1. Highlights next category in list
  2. Skips group headers
  3. Scrolls to keep highlighted item visible

User: Presses Enter
System:
  1. Selects highlighted category
  2. Closes dropdown
```

### 5. Change Jurisdiction
```
User: Selects "Mexico Federal" from jurisdiction filter
System:
  1. Filters categories to Mexico SAT taxonomy
  2. Clears search query
  3. Updates suggestions (if available)
  4. Triggers onJurisdictionChange callback
```

### 6. Clear Selection
```
User: Clicks × button in trigger
System:
  1. Clears selected category
  2. Resets to placeholder text
  3. Triggers onClear callback
```

---

## Wireframes (ASCII Art)

### Closed State (No Selection)
```
┌──────────────────────────────────────────────┐
│ Select tax category...                    ▼ │
└──────────────────────────────────────────────┘
```

### Closed State (With Selection)
```
┌──────────────────────────────────────────────┐
│ Travel  [100% deductible]              × ▼  │
└──────────────────────────────────────────────┘
```

### Open State (With Suggestions)
```
┌──────────────────────────────────────────────┐
│ Travel  [100% deductible]              × ▲  │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ Jurisdiction: [USA Federal ▼]               │
│ ┌──────────────────────────────────────────┐ │
│ │ 🔍 Search categories...                  │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ 💡 Suggested                                 │
│ ┌──────────────────────────────────────────┐ │
│ │ Travel  [100% deductible]                │ │
│ │ 95% confident • Based on 10 Uber txns    │ │
│ └──────────────────────────────────────────┘ │
│ ──────────────────────────────────────────── │
│                                              │
│ Schedule C (USA)                             │
│ ┌──────────────────────────────────────────┐ │
│ │ Advertising             line_8  [100%]   │ │
│ │ Office Expenses         line_18 [100%]   │ │
│ │ └─ Software (Custom)            [100%]   │ │
│ │ Meals (Business)        line_24b [50%]   │ │
│ │ Travel                  line_24a [100%]  │ │← Highlighted
│ └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

### Multi-Jurisdiction with Search
```
┌──────────────────────────────────────────────┐
│ Select tax category...                    ▼ │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ Jurisdiction: [Mexico Federal ▼]            │
│ ┌──────────────────────────────────────────┐ │
│ │ 🔍 publicidad                         × │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ SAT (Mexico)                                 │
│ ┌──────────────────────────────────────────┐ │
│ │ Gastos de Gestión       84111500 [100%]  │ │
│ │ └─ Publicidad           84111506 [100%]  │ │← Match
│ │ └─ Publicidad en Redes (Custom)  [100%]  │ │← Match
│ └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

### Empty State (No Results)
```
┌──────────────────────────────────────────────┐
│ Select tax category...                    ▼ │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ Jurisdiction: [USA Federal ▼]               │
│ ┌──────────────────────────────────────────┐ │
│ │ 🔍 xyz123                             × │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│          🔍                                   │
│   No categories found matching "xyz123"      │
│                                              │
└──────────────────────────────────────────────┘
```

---

## Styling

```css
.tax-category-selector {
  position: relative;
  width: 100%;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.tax-category-selector.compact {
  font-size: 14px;
}

/* Trigger Button */
.selector-trigger {
  width: 100%;
  min-height: 44px;
  padding: 10px 16px;
  background: var(--input-bg);
  border: 2px solid var(--input-border);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 15px;
}

.selector-trigger:hover:not(:disabled) {
  border-color: var(--input-border-hover);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.selector-trigger:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.selector-trigger:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.selector-trigger.has-value {
  border-color: var(--input-border-filled);
}

.selected-category {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
}

.category-name {
  font-weight: 600;
  color: var(--text-color);
}

.placeholder {
  color: var(--text-muted);
}

.trigger-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: auto;
}

.clear-button {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  background: transparent;
  border: none;
  border-radius: 50%;
  color: var(--text-muted);
  cursor: pointer;
  font-size: 18px;
  line-height: 1;
  transition: all 0.2s;
}

.clear-button:hover {
  background: var(--hover-bg);
  color: var(--text-color);
}

.dropdown-arrow {
  color: var(--text-muted);
  font-size: 12px;
  transition: transform 0.2s;
}

.tax-category-selector.open .dropdown-arrow {
  transform: rotate(180deg);
}

/* Dropdown Panel */
.dropdown-panel {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: var(--panel-bg);
  border: 1px solid var(--panel-border);
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  overflow-y: auto;
  z-index: 1000;
  animation: slideDown 0.2s ease-out;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Jurisdiction Filter */
.jurisdiction-filter {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border-bottom: 1px solid var(--divider-color);
  background: var(--section-bg);
}

.jurisdiction-filter label {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-muted);
}

.jurisdiction-select {
  padding: 6px 10px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 13px;
  background: var(--input-bg);
}

/* Search Container */
.search-container {
  position: relative;
  padding: 12px 16px;
  border-bottom: 1px solid var(--divider-color);
}

.search-input {
  width: 100%;
  padding: 8px 32px 8px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
  background: var(--input-bg);
}

.search-input:focus {
  outline: none;
  border-color: var(--primary-color);
}

.clear-search {
  position: absolute;
  right: 24px;
  top: 50%;
  transform: translateY(-50%);
  width: 20px;
  height: 20px;
  background: transparent;
  border: none;
  color: var(--text-muted);
  cursor: pointer;
  font-size: 18px;
  line-height: 1;
}

/* Suggestions Section */
.suggestions-section {
  padding: 12px 0;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 16px 8px;
  font-size: 13px;
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.section-icon {
  font-size: 14px;
}

.suggestion-list {
  display: flex;
  flex-direction: column;
}

.suggestion-item {
  width: 100%;
  padding: 10px 16px;
  background: transparent;
  border: none;
  text-align: left;
  cursor: pointer;
  transition: background 0.15s;
}

.suggestion-item:hover,
.suggestion-item.highlighted {
  background: var(--hover-bg);
}

.suggestion-main {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 4px;
}

.suggestion-name {
  font-weight: 600;
  color: var(--text-color);
  font-size: 14px;
}

.suggestion-meta {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.confidence-badge {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--success-bg);
  color: var(--success-color);
  border-radius: 4px;
  font-weight: 600;
}

.suggestion-reason {
  font-size: 12px;
  color: var(--text-muted);
}

.section-divider {
  height: 1px;
  background: var(--divider-color);
  margin: 12px 0;
}

/* Category Groups */
.category-groups {
  padding: 8px 0;
}

.category-group {
  margin-bottom: 12px;
}

.taxonomy-header {
  padding: 8px 16px;
  font-size: 12px;
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  background: var(--section-bg);
}

.category-list {
  display: flex;
  flex-direction: column;
}

/* Category Item */
.category-item {
  width: 100%;
  padding: 10px 16px;
  background: transparent;
  border: none;
  text-align: left;
  cursor: pointer;
  transition: background 0.15s;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.category-item:hover,
.category-item.highlighted {
  background: var(--hover-bg);
}

.category-item.selected {
  background: var(--selected-bg);
  border-left: 3px solid var(--primary-color);
}

.category-main {
  display: flex;
  align-items: center;
  gap: 8px;
}

.indent-marker {
  color: var(--text-muted);
  font-size: 12px;
  font-weight: 400;
}

.category-name {
  font-weight: 600;
  color: var(--text-color);
  font-size: 14px;
  flex: 1;
}

.custom-badge {
  font-size: 10px;
  padding: 2px 6px;
  background: var(--info-bg);
  color: var(--info-color);
  border-radius: 4px;
  font-weight: 600;
  text-transform: uppercase;
}

.category-meta {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-left: auto;
}

.category-code {
  font-size: 12px;
  color: var(--text-muted);
  font-family: monospace;
}

.deduction-badge {
  font-size: 11px;
  padding: 3px 8px;
  border-radius: 4px;
  font-weight: 600;
  white-space: nowrap;
}

.deduction-badge.deduction-full {
  background: var(--success-bg);
  color: var(--success-color);
}

.deduction-badge.deduction-partial {
  background: var(--warning-bg);
  color: var(--warning-color);
}

.deduction-badge.deduction-none {
  background: var(--error-bg);
  color: var(--error-color);
}

.category-description {
  font-size: 12px;
  color: var(--text-muted);
  margin-top: 4px;
  line-height: 1.4;
}

/* Empty State */
.empty-state {
  padding: 40px 20px;
  text-align: center;
}

.empty-icon {
  font-size: 48px;
  margin-bottom: 12px;
  opacity: 0.5;
}

.empty-message {
  font-size: 14px;
  color: var(--text-muted);
}

/* Loading State */
.tax-category-selector.loading .skeleton {
  background: var(--skeleton-bg);
  animation: pulse 1.5s infinite;
}

.skeleton-text {
  height: 20px;
  width: 200px;
  background: var(--skeleton-shimmer);
  border-radius: 4px;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

/* Error State */
.tax-category-selector.error .error-message {
  padding: 12px 16px;
  background: var(--error-bg);
  color: var(--error-color);
  border: 1px solid var(--error-border);
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.error-icon {
  font-size: 18px;
}

/* Theme Variables */
.tax-category-selector.theme-light {
  --input-bg: #ffffff;
  --input-border: #d1d5db;
  --input-border-hover: #9ca3af;
  --input-border-filled: #3b82f6;
  --panel-bg: #ffffff;
  --panel-border: #e5e7eb;
  --section-bg: #f9fafb;
  --text-color: #111827;
  --text-muted: #6b7280;
  --hover-bg: #f3f4f6;
  --selected-bg: #eff6ff;
  --divider-color: #e5e7eb;
  --primary-color: #3b82f6;
  --success-bg: #d1fae5;
  --success-color: #065f46;
  --warning-bg: #fef3c7;
  --warning-color: #92400e;
  --error-bg: #fee2e2;
  --error-color: #991b1b;
  --info-bg: #dbeafe;
  --info-color: #1e40af;
}

.tax-category-selector.theme-dark {
  --input-bg: #1f2937;
  --input-border: #374151;
  --input-border-hover: #4b5563;
  --input-border-filled: #3b82f6;
  --panel-bg: #1f2937;
  --panel-border: #374151;
  --section-bg: #111827;
  --text-color: #f9fafb;
  --text-muted: #9ca3af;
  --hover-bg: #374151;
  --selected-bg: #1e3a8a;
  --divider-color: #374151;
  --primary-color: #3b82f6;
  --success-bg: #064e3b;
  --success-color: #d1fae5;
  --warning-bg: #78350f;
  --warning-color: #fef3c7;
  --error-bg: #7f1d1d;
  --error-color: #fee2e2;
  --info-bg: #1e3a8a;
  --info-color: #dbeafe;
}

/* Responsive Design */
@media (max-width: 768px) {
  .dropdown-panel {
    position: fixed;
    top: auto;
    bottom: 0;
    left: 0;
    right: 0;
    max-height: 70vh;
    border-radius: 16px 16px 0 0;
    animation: slideUp 0.3s ease-out;
  }

  @keyframes slideUp {
    from {
      transform: translateY(100%);
    }
    to {
      transform: translateY(0);
    }
  }
}
```

---

## Accessibility

```tsx
// ARIA attributes for screen readers
<button
  role="combobox"
  aria-haspopup="listbox"
  aria-expanded={isOpen}
  aria-controls="category-listbox"
  aria-label="Select tax category"
  aria-activedescendant={highlightedCategoryId}
>
  {/* Trigger content */}
</button>

<div
  role="listbox"
  id="category-listbox"
  aria-label="Tax categories"
>
  <button
    role="option"
    aria-selected={isSelected}
    id={`category-${category.category_id}`}
  >
    {/* Category content */}
  </button>
</div>

// Keyboard shortcuts
// ↑↓ arrows: Navigate categories
// Enter: Select highlighted category
// Esc: Close dropdown
// Tab: Close and move to next field
// Type to search: Real-time filtering

// Screen reader announcements
aria-live="polite" aria-atomic="true"
// "Travel category selected, 100% deductible"
// "3 categories found matching search"
// "Suggested: Travel, 95% confident based on 10 similar transactions"
```

---

## Multi-Domain Examples

### Finance Domain (Tax Categories)
```tsx
<TaxCategorySelector
  categories={taxCategories}
  selectedCategoryId={transaction.tax_classification?.category_id}
  jurisdiction="usa_federal"
  availableJurisdictions={["usa_federal", "mexico_federal"]}
  suggestions={autoSuggestions}
  onSelect={(categoryId) => {
    classifyTransaction(transaction.id, categoryId);
  }}
  onClear={() => {
    clearTaxClassification(transaction.id);
  }}
  onJurisdictionChange={(jurisdiction) => {
    setActiveJurisdiction(jurisdiction);
  }}
  showSuggestions={true}
  showDeductionRate={true}
  groupByTaxonomy={true}
  placeholder="Select tax category (Schedule C)..."
/>
// Categories: Advertising (100%), Office Expenses (100%), Meals (50%), Travel (100%)
```

### Healthcare Domain (Procedure Codes)
```tsx
<TaxCategorySelector
  categories={cptCodes}
  selectedCategoryId={claim.procedure_code_id}
  jurisdiction="hipaa"
  availableJurisdictions={["hipaa", "medicare"]}
  suggestions={procedureSuggestions}
  onSelect={(codeId) => {
    assignProcedureCode(claim.id, codeId);
  }}
  showSuggestions={true}
  showDeductionRate={true}  // Shows reimbursement eligibility
  groupByTaxonomy={true}
  placeholder="Select CPT code..."
/>
// Categories: Office Visit (99213, 100% reimbursable), Lab Test (80050, 80% reimbursable)
```

### Legal Domain (Case Categories)
```tsx
<TaxCategorySelector
  categories={caseLawCategories}
  selectedCategoryId={case.category_id}
  jurisdiction="ca_state"
  availableJurisdictions={["ca_state", "federal"]}
  suggestions={categorySuggestions}
  onSelect={(categoryId) => {
    assignCaseCategory(case.id, categoryId);
  }}
  showSuggestions={false}  // No auto-suggestions for legal
  showDeductionRate={true}  // Shows court fee schedule
  groupByTaxonomy={true}
  placeholder="Select case category..."
/>
// Categories: Civil Litigation (filing fee: $435), Family Law (filing fee: $435)
```

### Research Domain (Grant Expense Categories)
```tsx
<TaxCategorySelector
  categories={grantCategories}
  selectedCategoryId={expense.grant_category_id}
  jurisdiction="nsf"
  availableJurisdictions={["nsf", "nih"]}
  suggestions={expenseSuggestions}
  onSelect={(categoryId) => {
    categorizeGrantExpense(expense.id, categoryId);
  }}
  showSuggestions={true}
  showDeductionRate={true}  // Shows cost-share requirement
  groupByTaxonomy={true}
  placeholder="Select grant expense category..."
/>
// Categories: Equipment (100% reimbursable), Travel (100%), Indirect Costs (50% cost-share)
```

---

## Features

### 1. Hierarchical Display
```typescript
// Parent categories with indented children
- Office Expenses (parent, depth=0)
  └─ Software (child, depth=1, custom)
  └─ Supplies (child, depth=1)

// Visual indentation: paddingLeft = 12 + depth * 20
// depth=0: 12px
// depth=1: 32px
// depth=2: 52px
```

### 2. Full-Text Search
```typescript
// Search across multiple fields
searchQuery = "travel";

// Matches:
// - Category name: "Travel"
// - Description: "Travel expenses for business"
// - Code: "line_24a"

// Real-time filtering as user types
```

### 3. Auto-Suggestion Integration
```typescript
// Display ML-powered suggestions at top
suggestions = [
  {
    category_id: "tax_cat_usa_schedule_c_travel",
    category_name: "Travel",
    confidence: 0.95,
    reason: "Based on 10 similar Uber transactions",
    deduction_rate: 1.0
  }
];

// User can accept suggestion or browse all categories
```

### 4. Deduction Rate Badges
```typescript
// Color-coded badges for compliance levels
rate = 1.0 → Green badge "100% deductible"
rate = 0.5 → Yellow badge "50% deductible"
rate = 0.0 → Red badge "0% deductible"
```

### 5. Multi-Jurisdiction Support
```typescript
// Switch between regulatory frameworks
availableJurisdictions = ["usa_federal", "mexico_federal"];

// Filters categories by selected jurisdiction
// Useful for cross-border businesses
```

### 6. Keyboard Navigation
```typescript
// Power user keyboard shortcuts
↑↓ arrows: Navigate categories
Enter: Select highlighted category
Esc: Close dropdown
Tab: Close and move to next field
Type: Real-time search filtering
```

### 7. Responsive Design
```typescript
// Desktop: Dropdown below trigger (max-height: 400px)
// Mobile: Bottom sheet (max-height: 70vh, slides up from bottom)
```

---

## Related Components

**Uses:**
- None (primitive UI component)

**Used by:**
- TransactionList (for inline classification)
- DrillDownPanel (transaction detail drawer)
- TaxCategoryManager (for editing category hierarchy)
- BulkClassificationDialog (classify multiple transactions)

**Similar patterns:**
- CounterpartySelector (select merchant/person)
- AccountSelector (select account)
- SeriesSelector (select recurring series)

**OL Dependencies:**
- TaxCategoryStore (fetch categories)
- TaxCategoryClassifier (auto-suggestions)
- TaxonomyEngine (build hierarchy tree)
