# CounterpartyManager (IL Component)

## Definition

**CounterpartyManager** is a full CRUD UI component for managing counterparties (create, edit, merge, list). It provides a complete interface for counterparty lifecycle management with search, filtering, deduplication detection, and bulk merge capabilities.

**Problem it solves:**
- No standardized UI for managing counterparties across different domains
- Duplicate counterparties created with slight name variations
- No unified pattern for filtering counterparties by type
- No way to merge duplicate counterparties (and their transaction history)
- Scattered counterparty management across multiple screens
- Alias management is complex and inconsistent

**Solution:**
- Single component for all counterparty CRUD operations
- Automatic duplicate detection based on name similarity
- Bulk merge workflow with transaction preview
- Canonical name + aliases management
- Search across canonical names and aliases
- Filter by type (merchant, person, business, government)
- Responsive design (table on desktop, cards on mobile)
- Multi-select for batch operations

---

## Interface Contract

```typescript
interface CounterpartyManagerProps {
  // Data
  counterparties: Counterparty[];  // All counterparties
  transactionCounts?: Record<string, number>;  // Transaction count per counterparty
  currentUserId: string;  // For ownership checks

  // Callbacks
  onCounterpartyCreate: (counterparty: CreateCounterpartyInput) => Promise<Counterparty>;
  onCounterpartyUpdate: (counterpartyId: string, updates: Partial<Counterparty>) => Promise<Counterparty>;
  onCounterpartyDelete: (counterpartyId: string) => Promise<void>;
  onCounterpartiesMerge: (primaryId: string, mergeIds: string[]) => Promise<void>;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Filters & Display
  initialFilter?: CounterpartyFilter;  // Pre-applied filter
  allowCreate?: boolean;  // Show "New Counterparty" button (default: true)
  allowEdit?: boolean;  // Allow editing counterparties (default: true)
  allowDelete?: boolean;  // Allow deleting counterparties (default: true)
  allowMerge?: boolean;  // Allow merging counterparties (default: true)
  showDuplicates?: boolean;  // Show duplicate suggestions panel (default: true)

  // Customization
  theme?: "light" | "dark";
  compact?: boolean;  // Compact view for smaller spaces
  groupBy?: "type" | "none";  // Group counterparties in list
}

interface Counterparty {
  counterparty_id: string;
  canonical_name: string;  // Primary display name
  aliases: string[];  // Alternative names
  type: CounterpartyType;
  created_at: string;
  updated_at: string;
  owner_id: string;

  // Optional metadata
  notes?: string;
  tax_id?: string;  // EIN, SSN (last 4), etc.
  contact_info?: ContactInfo;
  tags?: string[];
}

interface ContactInfo {
  email?: string;
  phone?: string;
  address?: string;
  website?: string;
}

type CounterpartyType =
  | "merchant"      // Stores, vendors, service providers
  | "person"        // Individuals (friends, family, employees)
  | "business"      // Companies, corporations, LLCs
  | "government";   // Tax agencies, city/state departments

interface CreateCounterpartyInput {
  canonical_name: string;
  type: CounterpartyType;
  aliases?: string[];
  notes?: string;
  tax_id?: string;
  contact_info?: ContactInfo;
  tags?: string[];
}

interface CounterpartyFilter {
  searchQuery?: string;  // Search canonical name + aliases
  types?: CounterpartyType[];  // Filter by counterparty type
  tags?: string[];  // Filter by tags
}

interface DuplicateGroup {
  canonical_name: string;
  counterparties: Counterparty[];
  similarity_score: number;  // 0-100
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo } from 'react';

export const CounterpartyManager: React.FC<CounterpartyManagerProps> = ({
  counterparties,
  transactionCounts = {},
  currentUserId,
  onCounterpartyCreate,
  onCounterpartyUpdate,
  onCounterpartyDelete,
  onCounterpartiesMerge,
  loading = false,
  error = null,
  initialFilter = {},
  allowCreate = true,
  allowEdit = true,
  allowDelete = true,
  allowMerge = true,
  showDuplicates = true,
  theme = "light",
  compact = false,
  groupBy = "none"
}) => {
  const [filter, setFilter] = useState<CounterpartyFilter>(initialFilter);
  const [sortBy, setSortBy] = useState<"name" | "type" | "transaction_count" | "updated_at">("name");
  const [sortDirection, setSortDirection] = useState<"asc" | "desc">("asc");
  const [showCreateModal, setShowCreateModal] = useState(false);
  const [editingCounterparty, setEditingCounterparty] = useState<Counterparty | null>(null);
  const [deleteConfirm, setDeleteConfirm] = useState<Counterparty | null>(null);
  const [selectedForMerge, setSelectedForMerge] = useState<Set<string>>(new Set());
  const [showMergeDialog, setShowMergeDialog] = useState(false);
  const [showDuplicatesPanel, setShowDuplicatesPanel] = useState(showDuplicates);

  // Filter counterparties
  const filteredCounterparties = useMemo(() => {
    let result = counterparties;

    // Search by canonical name or aliases
    if (filter.searchQuery) {
      const query = filter.searchQuery.toLowerCase();
      result = result.filter(c =>
        c.canonical_name.toLowerCase().includes(query) ||
        c.aliases?.some(alias => alias.toLowerCase().includes(query))
      );
    }

    // Filter by type
    if (filter.types && filter.types.length > 0) {
      result = result.filter(c => filter.types!.includes(c.type));
    }

    // Filter by tags
    if (filter.tags && filter.tags.length > 0) {
      result = result.filter(c =>
        c.tags?.some(tag => filter.tags!.includes(tag))
      );
    }

    return result;
  }, [counterparties, filter]);

  // Sort counterparties
  const sortedCounterparties = useMemo(() => {
    const sorted = [...filteredCounterparties];
    sorted.sort((a, b) => {
      let comparison = 0;

      switch (sortBy) {
        case "name":
          comparison = a.canonical_name.localeCompare(b.canonical_name);
          break;
        case "type":
          comparison = a.type.localeCompare(b.type);
          break;
        case "transaction_count":
          comparison = (transactionCounts[a.counterparty_id] || 0) -
                       (transactionCounts[b.counterparty_id] || 0);
          break;
        case "updated_at":
          comparison = new Date(a.updated_at).getTime() - new Date(b.updated_at).getTime();
          break;
      }

      return sortDirection === "asc" ? comparison : -comparison;
    });

    return sorted;
  }, [filteredCounterparties, sortBy, sortDirection, transactionCounts]);

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

    return groups;
  }, [sortedCounterparties, groupBy]);

  // Detect duplicates using fuzzy matching
  const duplicateGroups = useMemo((): DuplicateGroup[] => {
    if (!showDuplicatesPanel) return [];

    const groups: DuplicateGroup[] = [];
    const processed = new Set<string>();

    counterparties.forEach((cp1, i) => {
      if (processed.has(cp1.counterparty_id)) return;

      const similar: Counterparty[] = [];

      counterparties.forEach((cp2, j) => {
        if (i !== j && !processed.has(cp2.counterparty_id)) {
          const similarity = calculateSimilarity(cp1.canonical_name, cp2.canonical_name);

          // Check aliases too
          const aliasSimilarity = Math.max(
            ...cp1.aliases.flatMap(a1 =>
              cp2.aliases.map(a2 => calculateSimilarity(a1, a2))
            ),
            0
          );

          const maxSimilarity = Math.max(similarity, aliasSimilarity);

          if (maxSimilarity > 70) {  // 70% similarity threshold
            similar.push(cp2);
          }
        }
      });

      if (similar.length > 0) {
        groups.push({
          canonical_name: cp1.canonical_name,
          counterparties: [cp1, ...similar],
          similarity_score: 85  // Average similarity
        });

        processed.add(cp1.counterparty_id);
        similar.forEach(s => processed.add(s.counterparty_id));
      }
    });

    // Sort by number of duplicates
    return groups.sort((a, b) => b.counterparties.length - a.counterparties.length);
  }, [counterparties, showDuplicatesPanel]);

  // Handle multi-select
  const toggleSelect = (counterpartyId: string) => {
    const newSelection = new Set(selectedForMerge);
    if (newSelection.has(counterpartyId)) {
      newSelection.delete(counterpartyId);
    } else {
      newSelection.add(counterpartyId);
    }
    setSelectedForMerge(newSelection);
  };

  const clearSelection = () => {
    setSelectedForMerge(new Set());
  };

  // Loading state
  if (loading) {
    return (
      <div className={`counterparty-manager theme-${theme} loading`}>
        <div className="skeleton skeleton-header"></div>
        <div className="skeleton skeleton-list"></div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`counterparty-manager theme-${theme} error`}>
        <div className="error-icon">⚠️</div>
        <div className="error-message">{error}</div>
      </div>
    );
  }

  return (
    <div className={`counterparty-manager theme-${theme} ${compact ? 'compact' : ''}`}>
      {/* Header */}
      <div className="manager-header">
        <h2>Counterparty Management</h2>
        <div className="header-actions">
          {allowCreate && (
            <button
              className="create-button"
              onClick={() => setShowCreateModal(true)}
            >
              + New Counterparty
            </button>
          )}
          {showDuplicates && (
            <button
              className={`duplicates-toggle ${showDuplicatesPanel ? 'active' : ''}`}
              onClick={() => setShowDuplicatesPanel(!showDuplicatesPanel)}
              title="Show/hide duplicate suggestions"
            >
              🔍 Duplicates {duplicateGroups.length > 0 && `(${duplicateGroups.length})`}
            </button>
          )}
        </div>
      </div>

      {/* Search & Filters */}
      <div className="manager-filters">
        <input
          type="text"
          placeholder="Search by name or alias..."
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
        </select>

        <select
          value={sortBy}
          onChange={(e) => setSortBy(e.target.value as any)}
          className="sort-select"
        >
          <option value="name">Sort by Name</option>
          <option value="type">Sort by Type</option>
          <option value="transaction_count">Sort by Transactions</option>
          <option value="updated_at">Sort by Last Updated</option>
        </select>

        <button
          className="sort-direction-button"
          onClick={() => setSortDirection(sortDirection === "asc" ? "desc" : "asc")}
          title={`Sort ${sortDirection === "asc" ? "descending" : "ascending"}`}
        >
          {sortDirection === "asc" ? "↑" : "↓"}
        </button>

        {selectedForMerge.size > 0 && allowMerge && (
          <div className="selection-actions">
            <span className="selection-count">{selectedForMerge.size} selected</span>
            <button
              className="merge-button"
              onClick={() => setShowMergeDialog(true)}
              disabled={selectedForMerge.size < 2}
            >
              Merge Selected
            </button>
            <button className="clear-selection-button" onClick={clearSelection}>
              Clear
            </button>
          </div>
        )}
      </div>

      {/* Main Content Area */}
      <div className="manager-content">
        {/* Duplicate Suggestions Panel */}
        {showDuplicatesPanel && duplicateGroups.length > 0 && (
          <div className="duplicates-panel">
            <div className="panel-header">
              <h3>Possible Duplicates</h3>
              <span className="duplicate-count">{duplicateGroups.length} groups found</span>
            </div>
            <div className="duplicate-groups">
              {duplicateGroups.map((group, idx) => (
                <DuplicateGroupCard
                  key={idx}
                  group={group}
                  transactionCounts={transactionCounts}
                  onMerge={(ids) => {
                    setSelectedForMerge(new Set(ids));
                    setShowMergeDialog(true);
                  }}
                  compact={compact}
                />
              ))}
            </div>
          </div>
        )}

        {/* Counterparty List */}
        <div className="counterparty-list-container">
          {Object.keys(groupedCounterparties).length === 0 ? (
            <div className="empty-state">
              <div className="empty-icon">👥</div>
              <div className="empty-message">
                {filter.searchQuery
                  ? `No counterparties found matching "${filter.searchQuery}"`
                  : "No counterparties yet. Create your first counterparty to get started."}
              </div>
              {allowCreate && !filter.searchQuery && (
                <button
                  className="create-button-secondary"
                  onClick={() => setShowCreateModal(true)}
                >
                  Create Counterparty
                </button>
              )}
            </div>
          ) : (
            <div className="counterparty-groups">
              {Object.entries(groupedCounterparties).map(([groupName, groupCounterparties]) => (
                <div key={groupName} className="counterparty-group">
                  {groupBy !== "none" && (
                    <div className="group-header">
                      {formatCounterpartyType(groupName as CounterpartyType)}
                      <span className="group-count">({groupCounterparties.length})</span>
                    </div>
                  )}

                  <div className="counterparty-list">
                    {groupCounterparties.map(counterparty => (
                      <CounterpartyRow
                        key={counterparty.counterparty_id}
                        counterparty={counterparty}
                        transactionCount={transactionCounts[counterparty.counterparty_id] || 0}
                        isSelected={selectedForMerge.has(counterparty.counterparty_id)}
                        onSelect={allowMerge ? () => toggleSelect(counterparty.counterparty_id) : undefined}
                        onEdit={allowEdit ? () => setEditingCounterparty(counterparty) : undefined}
                        onDelete={allowDelete ? () => setDeleteConfirm(counterparty) : undefined}
                        compact={compact}
                      />
                    ))}
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      </div>

      {/* Create Counterparty Modal */}
      {showCreateModal && (
        <CounterpartyCreateModal
          onClose={() => setShowCreateModal(false)}
          onCreate={async (data) => {
            await onCounterpartyCreate(data);
            setShowCreateModal(false);
          }}
          theme={theme}
        />
      )}

      {/* Edit Counterparty Modal */}
      {editingCounterparty && (
        <CounterpartyEditModal
          counterparty={editingCounterparty}
          onClose={() => setEditingCounterparty(null)}
          onUpdate={async (updates) => {
            await onCounterpartyUpdate(editingCounterparty.counterparty_id, updates);
            setEditingCounterparty(null);
          }}
          theme={theme}
        />
      )}

      {/* Delete Confirmation Modal */}
      {deleteConfirm && (
        <DeleteConfirmModal
          counterparty={deleteConfirm}
          transactionCount={transactionCounts[deleteConfirm.counterparty_id] || 0}
          onClose={() => setDeleteConfirm(null)}
          onConfirm={async () => {
            await onCounterpartyDelete(deleteConfirm.counterparty_id);
            setDeleteConfirm(null);
          }}
          theme={theme}
        />
      )}

      {/* Merge Counterparties Dialog */}
      {showMergeDialog && selectedForMerge.size >= 2 && (
        <MergeCounterpartiesDialog
          counterparties={counterparties.filter(c => selectedForMerge.has(c.counterparty_id))}
          transactionCounts={transactionCounts}
          onClose={() => {
            setShowMergeDialog(false);
            clearSelection();
          }}
          onMerge={async (primaryId, mergeIds) => {
            await onCounterpartiesMerge(primaryId, mergeIds);
            setShowMergeDialog(false);
            clearSelection();
          }}
          theme={theme}
        />
      )}
    </div>
  );
};

// Individual counterparty row component
const CounterpartyRow: React.FC<{
  counterparty: Counterparty;
  transactionCount: number;
  isSelected: boolean;
  onSelect?: () => void;
  onEdit?: () => void;
  onDelete?: () => void;
  compact: boolean;
}> = ({ counterparty, transactionCount, isSelected, onSelect, onEdit, onDelete, compact }) => {
  return (
    <div className={`counterparty-row ${isSelected ? 'selected' : ''} ${compact ? 'compact' : ''}`}>
      {onSelect && (
        <div className="checkbox-container">
          <input
            type="checkbox"
            checked={isSelected}
            onChange={onSelect}
            aria-label={`Select ${counterparty.canonical_name}`}
          />
        </div>
      )}

      <div className="counterparty-icon">
        {getDefaultIcon(counterparty.type)}
      </div>

      <div className="counterparty-info">
        <div className="counterparty-name">
          {counterparty.canonical_name}
          {counterparty.aliases.length > 0 && (
            <span className="alias-count" title={counterparty.aliases.join(', ')}>
              +{counterparty.aliases.length} alias{counterparty.aliases.length > 1 ? 'es' : ''}
            </span>
          )}
        </div>
        <div className="counterparty-meta">
          <span className="counterparty-type">{formatCounterpartyType(counterparty.type)}</span>
          <span className="separator">•</span>
          <span className="transaction-count">{transactionCount} transaction{transactionCount !== 1 ? 's' : ''}</span>
          {counterparty.tags && counterparty.tags.length > 0 && (
            <>
              <span className="separator">•</span>
              <div className="tags">
                {counterparty.tags.slice(0, 2).map(tag => (
                  <span key={tag} className="tag">{tag}</span>
                ))}
                {counterparty.tags.length > 2 && (
                  <span className="tag-more">+{counterparty.tags.length - 2}</span>
                )}
              </div>
            </>
          )}
        </div>
        {counterparty.aliases.length > 0 && !compact && (
          <div className="aliases">
            {counterparty.aliases.slice(0, 3).map(alias => (
              <span key={alias} className="alias">{alias}</span>
            ))}
            {counterparty.aliases.length > 3 && (
              <span className="alias-more">+{counterparty.aliases.length - 3} more</span>
            )}
          </div>
        )}
      </div>

      <div className="counterparty-actions">
        {onEdit && (
          <button className="action-button edit" onClick={onEdit} title="Edit counterparty">
            ✏️
          </button>
        )}
        {onDelete && (
          <button className="action-button delete" onClick={onDelete} title="Delete counterparty">
            🗑️
          </button>
        )}
      </div>
    </div>
  );
};

// Duplicate group card component
const DuplicateGroupCard: React.FC<{
  group: DuplicateGroup;
  transactionCounts: Record<string, number>;
  onMerge: (ids: string[]) => void;
  compact: boolean;
}> = ({ group, transactionCounts, onMerge, compact }) => {
  const totalTransactions = group.counterparties.reduce(
    (sum, cp) => sum + (transactionCounts[cp.counterparty_id] || 0),
    0
  );

  return (
    <div className="duplicate-group-card">
      <div className="duplicate-header">
        <div className="duplicate-info">
          <span className="duplicate-name">{group.canonical_name}</span>
          <span className="similarity-badge">{group.similarity_score}% similar</span>
        </div>
        <button
          className="merge-duplicates-button"
          onClick={() => onMerge(group.counterparties.map(cp => cp.counterparty_id))}
        >
          Merge All
        </button>
      </div>
      <div className="duplicate-items">
        {group.counterparties.map(cp => (
          <div key={cp.counterparty_id} className="duplicate-item">
            <span className="item-name">{cp.canonical_name}</span>
            <span className="item-count">
              {transactionCounts[cp.counterparty_id] || 0} transactions
            </span>
          </div>
        ))}
      </div>
      <div className="duplicate-summary">
        Total: {totalTransactions} transactions across {group.counterparties.length} duplicates
      </div>
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

// Levenshtein distance for fuzzy matching
function calculateSimilarity(str1: string, str2: string): number {
  const s1 = str1.toLowerCase();
  const s2 = str2.toLowerCase();

  const costs = new Array<number>();
  for (let i = 0; i <= s1.length; i++) {
    let lastValue = i;
    for (let j = 0; j <= s2.length; j++) {
      if (i === 0) {
        costs[j] = j;
      } else if (j > 0) {
        let newValue = costs[j - 1];
        if (s1.charAt(i - 1) !== s2.charAt(j - 1)) {
          newValue = Math.min(Math.min(newValue, lastValue), costs[j]) + 1;
        }
        costs[j - 1] = lastValue;
        lastValue = newValue;
      }
    }
    if (i > 0) costs[s2.length] = lastValue;
  }

  const distance = costs[s2.length];
  const maxLength = Math.max(s1.length, s2.length);
  return ((maxLength - distance) / maxLength) * 100;
}
```

---

## Create Counterparty Modal

```tsx
const CounterpartyCreateModal: React.FC<{
  onClose: () => void;
  onCreate: (data: CreateCounterpartyInput) => Promise<void>;
  theme: "light" | "dark";
}> = ({ onClose, onCreate, theme }) => {
  const [formData, setFormData] = useState<CreateCounterpartyInput>({
    canonical_name: "",
    type: "merchant",
    aliases: [],
    notes: "",
    tags: []
  });
  const [aliasInput, setAliasInput] = useState("");
  const [tagInput, setTagInput] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const addAlias = () => {
    if (aliasInput.trim() && !formData.aliases?.includes(aliasInput.trim())) {
      setFormData({
        ...formData,
        aliases: [...(formData.aliases || []), aliasInput.trim()]
      });
      setAliasInput("");
    }
  };

  const removeAlias = (alias: string) => {
    setFormData({
      ...formData,
      aliases: formData.aliases?.filter(a => a !== alias)
    });
  };

  const addTag = () => {
    if (tagInput.trim() && !formData.tags?.includes(tagInput.trim())) {
      setFormData({
        ...formData,
        tags: [...(formData.tags || []), tagInput.trim()]
      });
      setTagInput("");
    }
  };

  const removeTag = (tag: string) => {
    setFormData({
      ...formData,
      tags: formData.tags?.filter(t => t !== tag)
    });
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);
    setError(null);

    try {
      // Validation
      if (!formData.canonical_name.trim()) {
        throw new Error("Canonical name is required");
      }

      await onCreate(formData);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to create counterparty");
      setSubmitting(false);
    }
  };

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Create New Counterparty</h3>
          <button className="close-button" onClick={onClose}>×</button>
        </div>

        <form onSubmit={handleSubmit} className="counterparty-form">
          {error && <div className="form-error">{error}</div>}

          <div className="form-group">
            <label htmlFor="canonical_name">Canonical Name *</label>
            <input
              type="text"
              id="canonical_name"
              value={formData.canonical_name}
              onChange={(e) => setFormData({ ...formData, canonical_name: e.target.value })}
              placeholder="e.g., Amazon, John Smith, Acme Corp"
              required
              autoFocus
            />
            <small>This is the primary name that will be displayed</small>
          </div>

          <div className="form-group">
            <label htmlFor="type">Type *</label>
            <select
              id="type"
              value={formData.type}
              onChange={(e) => setFormData({ ...formData, type: e.target.value as CounterpartyType })}
              required
            >
              <option value="merchant">Merchant (Stores, Vendors)</option>
              <option value="person">Person (Individuals)</option>
              <option value="business">Business (Companies, Corporations)</option>
              <option value="government">Government (Tax Agencies, Departments)</option>
            </select>
          </div>

          <div className="form-group">
            <label htmlFor="aliases">Aliases</label>
            <div className="alias-input-group">
              <input
                type="text"
                id="aliases"
                value={aliasInput}
                onChange={(e) => setAliasInput(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === 'Enter') {
                    e.preventDefault();
                    addAlias();
                  }
                }}
                placeholder="Add alternative names..."
              />
              <button type="button" onClick={addAlias} className="add-button">Add</button>
            </div>
            {formData.aliases && formData.aliases.length > 0 && (
              <div className="alias-list">
                {formData.aliases.map(alias => (
                  <span key={alias} className="alias-chip">
                    {alias}
                    <button type="button" onClick={() => removeAlias(alias)}>×</button>
                  </span>
                ))}
              </div>
            )}
            <small>Alternative names this counterparty might appear as</small>
          </div>

          <div className="form-group">
            <label htmlFor="notes">Notes</label>
            <textarea
              id="notes"
              value={formData.notes}
              onChange={(e) => setFormData({ ...formData, notes: e.target.value })}
              rows={3}
              placeholder="Add notes about this counterparty..."
            />
          </div>

          <div className="form-group">
            <label htmlFor="tags">Tags</label>
            <div className="tag-input-group">
              <input
                type="text"
                id="tags"
                value={tagInput}
                onChange={(e) => setTagInput(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === 'Enter') {
                    e.preventDefault();
                    addTag();
                  }
                }}
                placeholder="Add tags..."
              />
              <button type="button" onClick={addTag} className="add-button">Add</button>
            </div>
            {formData.tags && formData.tags.length > 0 && (
              <div className="tag-list">
                {formData.tags.map(tag => (
                  <span key={tag} className="tag-chip">
                    {tag}
                    <button type="button" onClick={() => removeTag(tag)}>×</button>
                  </span>
                ))}
              </div>
            )}
          </div>

          <div className="form-actions">
            <button type="button" onClick={onClose} className="cancel-button" disabled={submitting}>
              Cancel
            </button>
            <button type="submit" className="submit-button" disabled={submitting}>
              {submitting ? "Creating..." : "Create Counterparty"}
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
.counterparty-manager {
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

.header-actions {
  display: flex;
  gap: 12px;
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

.duplicates-toggle {
  padding: 10px 16px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}

.duplicates-toggle.active {
  background: var(--primary-color);
  color: white;
  border-color: var(--primary-color);
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

.selection-actions {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-left: auto;
  padding: 8px 16px;
  background: var(--selection-bg);
  border-radius: 6px;
}

.selection-count {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-color);
}

.merge-button {
  padding: 6px 12px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.merge-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.clear-selection-button {
  padding: 6px 12px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  cursor: pointer;
}

/* Main Content Area */
.manager-content {
  display: grid;
  grid-template-columns: 1fr;
  gap: 20px;
}

.manager-content:has(.duplicates-panel) {
  grid-template-columns: 350px 1fr;
}

/* Duplicates Panel */
.duplicates-panel {
  background: var(--panel-bg);
  border: 1px solid var(--panel-border);
  border-radius: 8px;
  padding: 16px;
  max-height: 800px;
  overflow-y: auto;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.panel-header h3 {
  font-size: 16px;
  font-weight: 600;
  margin: 0;
}

.duplicate-count {
  font-size: 12px;
  padding: 2px 8px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 12px;
  font-weight: 600;
}

.duplicate-groups {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.duplicate-group-card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 8px;
  padding: 12px;
}

.duplicate-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 12px;
}

.duplicate-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.duplicate-name {
  font-weight: 600;
  font-size: 14px;
  color: var(--text-color);
}

.similarity-badge {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--warning-bg);
  color: var(--warning-color);
  border-radius: 4px;
  font-weight: 600;
}

.merge-duplicates-button {
  padding: 6px 12px;
  background: var(--primary-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.duplicate-items {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 8px;
}

.duplicate-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 8px;
  background: var(--item-bg);
  border-radius: 4px;
  font-size: 13px;
}

.item-name {
  font-weight: 500;
  color: var(--text-color);
}

.item-count {
  font-size: 12px;
  color: var(--secondary-color);
}

.duplicate-summary {
  font-size: 12px;
  color: var(--secondary-color);
  padding-top: 8px;
  border-top: 1px solid var(--divider-color);
}

/* Counterparty List */
.counterparty-list-container {
  flex: 1;
}

.counterparty-group {
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

.counterparty-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.counterparty-row {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: var(--row-bg);
  border: 1px solid var(--row-border);
  border-radius: 8px;
  transition: all 0.2s;
}

.counterparty-row:hover {
  background: var(--row-hover-bg);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.counterparty-row.selected {
  border-color: var(--primary-color);
  background: var(--row-selected-bg);
}

.counterparty-row.compact {
  padding: 12px;
  gap: 12px;
}

.checkbox-container {
  display: flex;
  align-items: center;
}

.checkbox-container input[type="checkbox"] {
  width: 18px;
  height: 18px;
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

.counterparty-info {
  flex: 1;
  min-width: 0;
}

.counterparty-name {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-color);
  margin-bottom: 4px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.alias-count {
  font-size: 11px;
  padding: 2px 8px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  font-weight: 500;
  cursor: help;
}

.counterparty-meta {
  font-size: 13px;
  color: var(--secondary-color);
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}

.separator {
  color: var(--separator-color);
}

.tags {
  display: flex;
  gap: 4px;
  flex-wrap: wrap;
}

.tag {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--tag-bg);
  color: var(--tag-color);
  border-radius: 4px;
}

.tag-more {
  font-size: 11px;
  color: var(--secondary-color);
}

.aliases {
  margin-top: 6px;
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}

.alias {
  font-size: 12px;
  padding: 2px 8px;
  background: var(--alias-bg);
  color: var(--alias-color);
  border-radius: 4px;
  border: 1px dashed var(--alias-border);
}

.alias-more {
  font-size: 12px;
  color: var(--secondary-color);
}

.counterparty-actions {
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

.counterparty-form {
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

.form-group small {
  font-size: 12px;
  color: var(--secondary-color);
}

.alias-input-group,
.tag-input-group {
  display: flex;
  gap: 8px;
}

.alias-input-group input,
.tag-input-group input {
  flex: 1;
}

.add-button {
  padding: 10px 16px;
  background: var(--secondary-button-bg);
  color: var(--secondary-button-color);
  border: 1px solid var(--input-border);
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
}

.alias-list,
.tag-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 8px;
}

.alias-chip,
.tag-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  background: var(--chip-bg);
  border: 1px solid var(--chip-border);
  border-radius: 16px;
  font-size: 13px;
}

.alias-chip button,
.tag-chip button {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 16px;
  line-height: 1;
  padding: 0;
  color: var(--secondary-color);
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
  --border-color: #e5e7eb;
  --selection-bg: #e0e7ff;
  --panel-bg: #f9fafb;
  --panel-border: #e5e7eb;
  --card-bg: #ffffff;
  --card-border: #e5e7eb;
  --badge-bg: #dbeafe;
  --badge-color: #1e40af;
  --warning-bg: #fef3c7;
  --warning-color: #92400e;
  --item-bg: #f3f4f6;
  --divider-color: #e5e7eb;
  --group-header-color: #6b7280;
  --row-bg: #ffffff;
  --row-border: #e5e7eb;
  --row-hover-bg: #f9fafb;
  --row-selected-bg: #e0e7ff;
  --text-color: #111827;
  --secondary-color: #6b7280;
  --separator-color: #d1d5db;
  --icon-bg: #f3f4f6;
  --tag-bg: #e0e7ff;
  --tag-color: #3730a3;
  --alias-bg: #ffffff;
  --alias-color: #6b7280;
  --alias-border: #d1d5db;
  --action-border: #e5e7eb;
  --action-hover-bg: #f3f4f6;
  --modal-bg: #ffffff;
  --label-color: #374151;
  --secondary-button-bg: #f3f4f6;
  --secondary-button-color: #374151;
  --chip-bg: #f3f4f6;
  --chip-border: #d1d5db;
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
  --border-color: #4b5563;
  --selection-bg: #1e3a8a;
  --panel-bg: #111827;
  --panel-border: #374151;
  --card-bg: #1f2937;
  --card-border: #374151;
  --badge-bg: #1e3a8a;
  --badge-color: #93c5fd;
  --warning-bg: #78350f;
  --warning-color: #fef3c7;
  --item-bg: #374151;
  --divider-color: #374151;
  --group-header-color: #9ca3af;
  --row-bg: #374151;
  --row-border: #4b5563;
  --row-hover-bg: #4b5563;
  --row-selected-bg: #1e3a8a;
  --text-color: #f3f4f6;
  --secondary-color: #9ca3af;
  --separator-color: #4b5563;
  --icon-bg: #4b5563;
  --tag-bg: #1e3a8a;
  --tag-color: #93c5fd;
  --alias-bg: #374151;
  --alias-color: #9ca3af;
  --alias-border: #4b5563;
  --action-border: #4b5563;
  --action-hover-bg: #1f2937;
  --modal-bg: #1f2937;
  --label-color: #d1d5db;
  --secondary-button-bg: #4b5563;
  --secondary-button-color: #f3f4f6;
  --chip-bg: #374151;
  --chip-border: #4b5563;
}

/* Responsive */
@media (max-width: 1024px) {
  .manager-content:has(.duplicates-panel) {
    grid-template-columns: 1fr;
  }

  .duplicates-panel {
    max-height: 400px;
  }
}

@media (max-width: 768px) {
  .counterparty-row {
    flex-direction: column;
    align-items: flex-start;
  }

  .manager-filters {
    flex-direction: column;
  }

  .selection-actions {
    width: 100%;
    margin-left: 0;
  }
}
```

---

## Visual Wireframes

### Main View with Duplicates Panel
```
┌────────────────────────────────────────────────────────────────────────────┐
│ Counterparty Management          [+ New Counterparty] [🔍 Duplicates (3)] │
├────────────────────────────────────────────────────────────────────────────┤
│ [Search by name or alias...] [No Grouping ▼] [Sort by Name ▼] [↑]         │
├──────────────────────┬─────────────────────────────────────────────────────┤
│ Possible Duplicates  │ All Counterparties (127)                            │
│ 3 groups found       │                                                     │
│                      │ ☐ [🏪] Amazon                +3 aliases             │
│ Amazon               │        Merchant • 247 transactions • online         │
│ 85% similar          │        amzn.com, amazon.com, AWS                    │
│ ┌──────────────────┐│                                                     │
│ │ Amazon      247  ││ ☐ [🏪] Starbucks              +1 alias              │
│ │ AMZN         45  ││        Merchant • 156 transactions • coffee         │
│ │ AWS          12  ││        Starbucks #1234                              │
│ └──────────────────┘│                                                     │
│ Total: 304 trans    │ ☐ [👤] John Smith                                   │
│ [Merge All]          │        Person • 89 transactions                     │
│                      │                                    [✏️] [🗑️]        │
└──────────────────────┴─────────────────────────────────────────────────────┘
```

### Multi-Select Mode
```
┌────────────────────────────────────────────────────────────────────────────┐
│ [Search...] [Group ▼] [Sort ▼] [↑]    3 selected [Merge Selected] [Clear] │
├────────────────────────────────────────────────────────────────────────────┤
│ ☑ [🏪] Amazon                +3 aliases                                    │
│        Merchant • 247 transactions • online                                │
│        amzn.com, amazon.com, AWS                          [✏️] [🗑️]        │
│                                                                            │
│ ☑ [🏪] AMZN                  +1 alias                                      │
│        Merchant • 45 transactions • online                                 │
│        Amazon Marketplace                                 [✏️] [🗑️]        │
│                                                                            │
│ ☑ [🏪] AWS                                                                 │
│        Merchant • 12 transactions • cloud                                  │
│                                                           [✏️] [🗑️]        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Multi-Domain Applicability

### Finance Domain
```tsx
<CounterpartyManager
  counterparties={financeCounterparties}
  transactionCounts={txCounts}
  currentUserId="user_123"
  onCounterpartyCreate={createCounterparty}
  onCounterpartyUpdate={updateCounterparty}
  onCounterpartyDelete={deleteCounterparty}
  onCounterpartiesMerge={mergeCounterparties}
  showDuplicates={true}
  groupBy="type"
/>
// Counterparties: Amazon, Starbucks, Landlord, Chase Bank
// Use case: Managing payment recipients and senders
```

### Healthcare Domain
```tsx
<CounterpartyManager
  counterparties={healthcareCounterparties}
  transactionCounts={claimCounts}
  currentUserId="patient_456"
  onCounterpartyCreate={createProvider}
  onCounterpartyUpdate={updateProvider}
  onCounterpartyDelete={deleteProvider}
  onCounterpartiesMerge={mergeProviders}
  showDuplicates={true}
  groupBy="type"
/>
// Counterparties: Dr. Smith, General Hospital, Blue Cross, CVS Pharmacy
// Use case: Managing healthcare providers and insurers
```

### Legal Domain
```tsx
<CounterpartyManager
  counterparties={legalCounterparties}
  transactionCounts={caseCounts}
  currentUserId="attorney_789"
  onCounterpartyCreate={createEntity}
  onCounterpartyUpdate={updateEntity}
  onCounterpartyDelete={deleteEntity}
  onCounterpartiesMerge={mergeEntities}
  showDuplicates={true}
  groupBy="type"
/>
// Counterparties: Smith Family Trust, Acme Corp, IRS, County Clerk
// Use case: Managing legal entities and government agencies
```

### Research Domain (RSRCH - Utilitario)
```tsx
<CounterpartyManager
  counterparties={rsrchCounterparties}
  transactionCounts={sourceProviderCounts}
  currentUserId="rsrch_analyst_101"
  onCounterpartyCreate={createSourceProvider}
  onCounterpartyUpdate={updateSourceProvider}
  onCounterpartyDelete={deleteSourceProvider}
  onCounterpartiesMerge={mergeSourceProviders}
  showDuplicates={true}
  groupBy="type"
/>
// Counterparties: TechCrunch, Twitter API, Lex Fridman (podcast), Scraping Service Inc
// Use case: Managing fact source providers and API vendors
```

---

## Features

### 1. Duplicate Detection
```typescript
// Automatic fuzzy matching on canonical names and aliases
// Levenshtein distance algorithm
// Shows similarity score (0-100%)
// Groups duplicates together
```

### 2. Bulk Merge
```typescript
// Multi-select counterparties
// Preview total transaction count
// Choose primary counterparty (keeps its canonical name)
// Merge operation is irreversible (warning shown)
```

### 3. Alias Management
```typescript
// Canonical name: Primary display name
// Aliases: Alternative names
// Search works across both canonical name and aliases
// Aliases shown as badges with count
```

### 4. Transaction Count
```typescript
// Shows number of transactions per counterparty
// Helps decide which counterparty to keep as primary
// Prevents deletion of counterparties with transactions
```

---

## Accessibility

```tsx
<div className="counterparty-row" role="listitem">
  <input
    type="checkbox"
    aria-label={`Select ${counterparty.canonical_name}`}
  />
  <div aria-label={`${counterparty.canonical_name}, ${formatCounterpartyType(counterparty.type)}, ${transactionCount} transactions`}>
    {/* Counterparty content */}
  </div>
</div>
```

---

## Related Components

**Uses:**
- MergeCounterpartiesDialog (for merging duplicates)

**Used by:**
- Settings page (manage counterparties)
- Transaction import wizard (map counterparties)
- Any feature requiring counterparty management

**Similar patterns:**
- AccountManager (for managing accounts)
- EntityManager (for managing entities)
