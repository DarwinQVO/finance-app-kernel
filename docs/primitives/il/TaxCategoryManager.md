# TaxCategoryManager (IL Component)

## Definition

**TaxCategoryManager** is a full CRUD UI component for managing custom tax categories with hierarchical tree view, bulk operations, usage statistics, and system/user category distinction. It provides a complete interface for regulatory classification lifecycle management across multiple jurisdictions.

**Problem it solves:**
- No standardized UI for managing custom regulatory categories across different domains
- Difficult to visualize hierarchical category relationships (parent-child trees)
- No clear distinction between system-provided and user-created categories
- Cannot track which categories are actively used (transaction counts)
- No way to merge or archive unused categories
- Scattered category management across multiple screens
- Hard to identify orphan categories or circular references

**Solution:**
- Single component for all category CRUD operations
- Tree view with expand/collapse for hierarchical visualization
- Visual distinction between system and custom categories
- Usage statistics showing transaction count per category
- Bulk operations (merge similar categories, archive unused)
- Search and filter across all categories
- Archive confirmation with transaction reassignment workflow
- Keyboard shortcuts for power users (N = new, Esc = close)
- Responsive design (tree on desktop, list on mobile)

---

## Interface Contract

```typescript
interface TaxCategoryManagerProps {
  // Data
  categories: TaxCategory[];  // All categories (system + custom)
  usageStats: Record<string, number>;  // Transaction count per category
  jurisdiction: string;  // Active jurisdiction
  availableJurisdictions: string[];  // All supported jurisdictions
  currentUserId: string;  // For ownership checks

  // Callbacks
  onCategoryCreate: (category: CreateCategoryInput) => Promise<TaxCategory>;
  onCategoryUpdate: (categoryId: string, updates: Partial<TaxCategory>) => Promise<TaxCategory>;
  onCategoryArchive: (categoryId: string) => Promise<void>;
  onCategoryRestore: (categoryId: string) => Promise<void>;
  onCategoriesMerge: (primaryId: string, mergeIds: string[]) => Promise<void>;

  // UI states
  loading?: boolean;
  error?: string | null;

  // Filters & Display
  initialFilter?: CategoryFilter;
  showArchived?: boolean;  // Show archived categories (default: false)
  showUsageStats?: boolean;  // Show transaction counts (default: true)
  allowCreate?: boolean;  // Show "New Category" button (default: true)
  allowEdit?: boolean;  // Allow editing custom categories (default: true)
  allowArchive?: boolean;  // Allow archiving categories (default: true)
  allowMerge?: boolean;  // Allow merging categories (default: true)

  // Customization
  theme?: "light" | "dark";
  compact?: boolean;  // Compact view for smaller spaces
}

interface TaxCategory {
  category_id: string;
  user_id: string | null;  // null for system categories
  name: string;
  jurisdiction: string;  // "usa_federal", "mexico_federal", etc.
  taxonomy: string;  // "schedule_c", "sat", "cpt", "icd10", etc.
  code: string;  // "line_8", "84111506", "99213", "E11.9", etc.
  parent_id: string | null;
  deduction_rate: number;  // 0.0 to 1.0
  is_system: boolean;
  is_archived: boolean;
  description?: string;
  created_at: string;
  updated_at: string;
}

interface CreateCategoryInput {
  name: string;
  jurisdiction: string;
  parent_id: string | null;
  deduction_rate: number;
  code?: string;  // Optional for custom categories
  description?: string;
}

interface CategoryFilter {
  searchQuery?: string;  // Search by name or code
  taxonomies?: string[];  // Filter by taxonomy
  showSystemOnly?: boolean;  // Show only system categories
  showCustomOnly?: boolean;  // Show only user categories
  showUnused?: boolean;  // Show categories with 0 transactions
  showArchived?: boolean;  // Include archived categories
}
```

---

## Component Structure

```tsx
import React, { useState, useMemo } from 'react';

export const TaxCategoryManager: React.FC<TaxCategoryManagerProps> = ({
  categories,
  usageStats,
  jurisdiction,
  availableJurisdictions,
  currentUserId,
  onCategoryCreate,
  onCategoryUpdate,
  onCategoryArchive,
  onCategoryRestore,
  onCategoriesMerge,
  loading = false,
  error = null,
  initialFilter = {},
  showArchived = false,
  showUsageStats = true,
  allowCreate = true,
  allowEdit = true,
  allowArchive = true,
  allowMerge = true,
  theme = "light",
  compact = false
}) => {
  const [selectedJurisdiction, setSelectedJurisdiction] = useState(jurisdiction);
  const [filter, setFilter] = useState<CategoryFilter>({
    showArchived,
    ...initialFilter
  });
  const [expandedNodes, setExpandedNodes] = useState<Set<string>>(new Set());
  const [showCreateModal, setShowCreateModal] = useState(false);
  const [editingCategory, setEditingCategory] = useState<TaxCategory | null>(null);
  const [archiveConfirm, setArchiveConfirm] = useState<TaxCategory | null>(null);
  const [selectedForMerge, setSelectedForMerge] = useState<Set<string>>(new Set());
  const [showMergeDialog, setShowMergeDialog] = useState(false);

  // Keyboard shortcuts
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'n' && (e.metaKey || e.ctrlKey)) {
        e.preventDefault();
        if (allowCreate) setShowCreateModal(true);
      }
      if (e.key === 'Escape') {
        setShowCreateModal(false);
        setEditingCategory(null);
        setArchiveConfirm(null);
        setShowMergeDialog(false);
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [allowCreate]);

  // Filter categories
  const filteredCategories = useMemo(() => {
    let result = categories.filter(c => c.jurisdiction === selectedJurisdiction);

    // Filter by archived status
    if (!filter.showArchived) {
      result = result.filter(c => !c.is_archived);
    }

    // Search by name or code
    if (filter.searchQuery) {
      const query = filter.searchQuery.toLowerCase();
      result = result.filter(c =>
        c.name.toLowerCase().includes(query) ||
        c.code.toLowerCase().includes(query) ||
        c.description?.toLowerCase().includes(query)
      );
    }

    // Filter by taxonomy
    if (filter.taxonomies && filter.taxonomies.length > 0) {
      result = result.filter(c => filter.taxonomies!.includes(c.taxonomy));
    }

    // Filter system vs custom
    if (filter.showSystemOnly) {
      result = result.filter(c => c.is_system);
    }
    if (filter.showCustomOnly) {
      result = result.filter(c => !c.is_system);
    }

    // Filter unused categories
    if (filter.showUnused) {
      result = result.filter(c => (usageStats[c.category_id] || 0) === 0);
    }

    return result;
  }, [categories, selectedJurisdiction, filter, usageStats]);

  // Build category tree
  const categoryTree = useMemo(() => {
    const buildTree = (parentId: string | null): TaxCategory[] => {
      const children = filteredCategories
        .filter(c => c.parent_id === parentId)
        .sort((a, b) => a.name.localeCompare(b.name));

      return children.flatMap(child => [
        child,
        ...buildTree(child.category_id)
      ]);
    };

    return buildTree(null);
  }, [filteredCategories]);

  // Group by taxonomy
  const groupedByTaxonomy = useMemo(() => {
    const groups: Record<string, TaxCategory[]> = {};

    categoryTree.forEach(category => {
      const taxonomyKey = category.taxonomy;
      if (!groups[taxonomyKey]) groups[taxonomyKey] = [];

      // Only add root-level categories to groups (children will be rendered recursively)
      if (category.parent_id === null) {
        groups[taxonomyKey].push(category);
      }
    });

    return groups;
  }, [categoryTree]);

  // Get children for a category
  const getChildren = (categoryId: string): TaxCategory[] => {
    return filteredCategories.filter(c => c.parent_id === categoryId);
  };

  // Toggle node expansion
  const toggleExpand = (categoryId: string) => {
    const newExpanded = new Set(expandedNodes);
    if (newExpanded.has(categoryId)) {
      newExpanded.delete(categoryId);
    } else {
      newExpanded.add(categoryId);
    }
    setExpandedNodes(newExpanded);
  };

  // Expand all nodes
  const expandAll = () => {
    const allParents = new Set(
      filteredCategories
        .filter(c => getChildren(c.category_id).length > 0)
        .map(c => c.category_id)
    );
    setExpandedNodes(allParents);
  };

  // Collapse all nodes
  const collapseAll = () => {
    setExpandedNodes(new Set());
  };

  // Handle multi-select for merge
  const toggleSelect = (categoryId: string) => {
    const category = categories.find(c => c.category_id === categoryId);

    // Can only merge custom categories
    if (category?.is_system) return;

    const newSelection = new Set(selectedForMerge);
    if (newSelection.has(categoryId)) {
      newSelection.delete(categoryId);
    } else {
      newSelection.add(categoryId);
    }
    setSelectedForMerge(newSelection);
  };

  const clearSelection = () => {
    setSelectedForMerge(new Set());
  };

  // Loading state
  if (loading) {
    return (
      <div className={`tax-category-manager theme-${theme} loading ${compact ? 'compact' : ''}`}>
        <div className="skeleton skeleton-header"></div>
        <div className="skeleton skeleton-tree"></div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`tax-category-manager theme-${theme} error ${compact ? 'compact' : ''}`}>
        <div className="error-icon">⚠️</div>
        <div className="error-message">{error}</div>
      </div>
    );
  }

  return (
    <div className={`tax-category-manager theme-${theme} ${compact ? 'compact' : ''}`}>
      {/* Header */}
      <div className="manager-header">
        <div className="header-title">
          <h2>Tax Category Management</h2>
          <span className="category-count">
            {filteredCategories.length} categories
            {filter.searchQuery && ` matching "${filter.searchQuery}"`}
          </span>
        </div>

        <div className="header-actions">
          {allowCreate && (
            <button
              className="create-button"
              onClick={() => setShowCreateModal(true)}
              title="Create custom category (⌘N)"
            >
              + New Category
            </button>
          )}
        </div>
      </div>

      {/* Jurisdiction Tabs */}
      {availableJurisdictions.length > 1 && (
        <div className="jurisdiction-tabs">
          {availableJurisdictions.map(j => (
            <button
              key={j}
              className={`jurisdiction-tab ${selectedJurisdiction === j ? 'active' : ''}`}
              onClick={() => setSelectedJurisdiction(j)}
            >
              {formatJurisdictionName(j)}
            </button>
          ))}
        </div>
      )}

      {/* Search & Filters */}
      <div className="manager-filters">
        <input
          type="text"
          className="search-input"
          placeholder="Search categories by name or code..."
          value={filter.searchQuery || ""}
          onChange={(e) => setFilter({ ...filter, searchQuery: e.target.value })}
        />

        <div className="filter-toggles">
          <label className="filter-toggle">
            <input
              type="checkbox"
              checked={filter.showSystemOnly || false}
              onChange={(e) => setFilter({
                ...filter,
                showSystemOnly: e.target.checked,
                showCustomOnly: e.target.checked ? false : filter.showCustomOnly
              })}
            />
            System Only
          </label>

          <label className="filter-toggle">
            <input
              type="checkbox"
              checked={filter.showCustomOnly || false}
              onChange={(e) => setFilter({
                ...filter,
                showCustomOnly: e.target.checked,
                showSystemOnly: e.target.checked ? false : filter.showSystemOnly
              })}
            />
            Custom Only
          </label>

          <label className="filter-toggle">
            <input
              type="checkbox"
              checked={filter.showUnused || false}
              onChange={(e) => setFilter({ ...filter, showUnused: e.target.checked })}
            />
            Unused Only
          </label>

          <label className="filter-toggle">
            <input
              type="checkbox"
              checked={filter.showArchived || false}
              onChange={(e) => setFilter({ ...filter, showArchived: e.target.checked })}
            />
            Show Archived
          </label>
        </div>

        <div className="tree-controls">
          <button className="tree-control-button" onClick={expandAll}>
            Expand All
          </button>
          <button className="tree-control-button" onClick={collapseAll}>
            Collapse All
          </button>
        </div>
      </div>

      {/* Merge Selection Bar */}
      {selectedForMerge.size > 0 && allowMerge && (
        <div className="selection-bar">
          <span className="selection-count">
            {selectedForMerge.size} categories selected
          </span>
          <button
            className="merge-button"
            onClick={() => setShowMergeDialog(true)}
            disabled={selectedForMerge.size < 2}
          >
            Merge Categories
          </button>
          <button className="clear-selection-button" onClick={clearSelection}>
            Clear Selection
          </button>
        </div>
      )}

      {/* Category Tree */}
      {Object.keys(groupedByTaxonomy).length === 0 ? (
        <div className="empty-state">
          <div className="empty-icon">📂</div>
          <div className="empty-message">
            {filter.searchQuery
              ? `No categories found matching "${filter.searchQuery}"`
              : "No custom categories yet. Create your first category to get started."}
          </div>
          {allowCreate && !filter.searchQuery && (
            <button
              className="create-button-secondary"
              onClick={() => setShowCreateModal(true)}
            >
              Create Category
            </button>
          )}
        </div>
      ) : (
        <div className="category-tree">
          {Object.entries(groupedByTaxonomy).map(([taxonomyKey, taxonomyCategories]) => (
            <div key={taxonomyKey} className="taxonomy-section">
              <div className="taxonomy-header">
                <span className="taxonomy-name">{formatTaxonomyName(taxonomyKey)}</span>
                <span className="taxonomy-count">({taxonomyCategories.length})</span>
              </div>

              <div className="taxonomy-categories">
                {taxonomyCategories.map(category => (
                  <CategoryTreeNode
                    key={category.category_id}
                    category={category}
                    children={getChildren(category.category_id)}
                    isExpanded={expandedNodes.has(category.category_id)}
                    isSelected={selectedForMerge.has(category.category_id)}
                    usageCount={usageStats[category.category_id] || 0}
                    showUsageStats={showUsageStats}
                    allowEdit={allowEdit}
                    allowArchive={allowArchive}
                    allowMerge={allowMerge}
                    onToggleExpand={toggleExpand}
                    onToggleSelect={toggleSelect}
                    onEdit={setEditingCategory}
                    onArchive={setArchiveConfirm}
                    onRestore={onCategoryRestore}
                    getChildren={getChildren}
                    expandedNodes={expandedNodes}
                    selectedForMerge={selectedForMerge}
                    usageStats={usageStats}
                  />
                ))}
              </div>
            </div>
          ))}
        </div>
      )}

      {/* Create Category Modal */}
      {showCreateModal && (
        <CategoryCreateModal
          jurisdiction={selectedJurisdiction}
          categories={categories.filter(c => c.jurisdiction === selectedJurisdiction)}
          onClose={() => setShowCreateModal(false)}
          onCreate={async (data) => {
            await onCategoryCreate(data);
            setShowCreateModal(false);
          }}
          theme={theme}
        />
      )}

      {/* Edit Category Modal */}
      {editingCategory && (
        <CategoryEditModal
          category={editingCategory}
          categories={categories.filter(c => c.jurisdiction === selectedJurisdiction)}
          onClose={() => setEditingCategory(null)}
          onUpdate={async (updates) => {
            await onCategoryUpdate(editingCategory.category_id, updates);
            setEditingCategory(null);
          }}
          theme={theme}
        />
      )}

      {/* Archive Confirmation Modal */}
      {archiveConfirm && (
        <CategoryArchiveModal
          category={archiveConfirm}
          usageCount={usageStats[archiveConfirm.category_id] || 0}
          onClose={() => setArchiveConfirm(null)}
          onConfirm={async () => {
            await onCategoryArchive(archiveConfirm.category_id);
            setArchiveConfirm(null);
          }}
          theme={theme}
        />
      )}

      {/* Merge Categories Dialog */}
      {showMergeDialog && (
        <CategoryMergeDialog
          categories={categories.filter(c => selectedForMerge.has(c.category_id))}
          usageStats={usageStats}
          onClose={() => setShowMergeDialog(false)}
          onConfirm={async (primaryId, mergeIds) => {
            await onCategoriesMerge(primaryId, mergeIds);
            setShowMergeDialog(false);
            clearSelection();
          }}
          theme={theme}
        />
      )}
    </div>
  );
};

// Category Tree Node Component (Recursive)
const CategoryTreeNode: React.FC<{
  category: TaxCategory;
  children: TaxCategory[];
  isExpanded: boolean;
  isSelected: boolean;
  usageCount: number;
  showUsageStats: boolean;
  allowEdit: boolean;
  allowArchive: boolean;
  allowMerge: boolean;
  onToggleExpand: (categoryId: string) => void;
  onToggleSelect: (categoryId: string) => void;
  onEdit: (category: TaxCategory) => void;
  onArchive: (category: TaxCategory) => void;
  onRestore: (categoryId: string) => void;
  getChildren: (categoryId: string) => TaxCategory[];
  expandedNodes: Set<string>;
  selectedForMerge: Set<string>;
  usageStats: Record<string, number>;
}> = ({
  category,
  children,
  isExpanded,
  isSelected,
  usageCount,
  showUsageStats,
  allowEdit,
  allowArchive,
  allowMerge,
  onToggleExpand,
  onToggleSelect,
  onEdit,
  onArchive,
  onRestore,
  getChildren,
  expandedNodes,
  selectedForMerge,
  usageStats
}) => {
  const hasChildren = children.length > 0;

  return (
    <div className="tree-node">
      <div
        className={`node-row ${category.is_archived ? 'archived' : ''} ${isSelected ? 'selected' : ''}`}
      >
        {/* Expand/Collapse Button */}
        {hasChildren ? (
          <button
            className="expand-button"
            onClick={() => onToggleExpand(category.category_id)}
            aria-label={isExpanded ? "Collapse" : "Expand"}
          >
            {isExpanded ? "▼" : "▶"}
          </button>
        ) : (
          <div className="expand-spacer"></div>
        )}

        {/* Select Checkbox (for merge) */}
        {allowMerge && !category.is_system && (
          <input
            type="checkbox"
            className="select-checkbox"
            checked={isSelected}
            onChange={() => onToggleSelect(category.category_id)}
            disabled={category.is_archived}
          />
        )}

        {/* Category Info */}
        <div className="node-info">
          <div className="node-main">
            <span className="node-name">{category.name}</span>
            {!category.is_system && <span className="custom-badge">Custom</span>}
            {category.is_archived && <span className="archived-badge">Archived</span>}
          </div>

          <div className="node-meta">
            <span className="node-code">{category.code}</span>
            <DeductionRateBadge rate={category.deduction_rate} />
            {showUsageStats && (
              <span className="usage-count" title={`Used in ${usageCount} transactions`}>
                {usageCount} txns
              </span>
            )}
          </div>
        </div>

        {/* Actions */}
        <div className="node-actions">
          {allowEdit && !category.is_system && !category.is_archived && (
            <button
              className="action-button edit"
              onClick={() => onEdit(category)}
              title="Edit category"
            >
              ✏️
            </button>
          )}
          {allowArchive && !category.is_system && !category.is_archived && (
            <button
              className="action-button archive"
              onClick={() => onArchive(category)}
              title="Archive category"
            >
              📦
            </button>
          )}
          {allowArchive && category.is_archived && (
            <button
              className="action-button restore"
              onClick={() => onRestore(category.category_id)}
              title="Restore category"
            >
              ↺
            </button>
          )}
        </div>
      </div>

      {/* Render Children Recursively */}
      {hasChildren && isExpanded && (
        <div className="node-children">
          {children.map(child => (
            <CategoryTreeNode
              key={child.category_id}
              category={child}
              children={getChildren(child.category_id)}
              isExpanded={expandedNodes.has(child.category_id)}
              isSelected={selectedForMerge.has(child.category_id)}
              usageCount={usageStats[child.category_id] || 0}
              showUsageStats={showUsageStats}
              allowEdit={allowEdit}
              allowArchive={allowArchive}
              allowMerge={allowMerge}
              onToggleExpand={onToggleExpand}
              onToggleSelect={onToggleSelect}
              onEdit={onEdit}
              onArchive={onArchive}
              onRestore={onRestore}
              getChildren={getChildren}
              expandedNodes={expandedNodes}
              selectedForMerge={selectedForMerge}
              usageStats={usageStats}
            />
          ))}
        </div>
      )}
    </div>
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
      {percentage}%
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

## Create Category Modal

```tsx
const CategoryCreateModal: React.FC<{
  jurisdiction: string;
  categories: TaxCategory[];
  onClose: () => void;
  onCreate: (data: CreateCategoryInput) => Promise<void>;
  theme: "light" | "dark";
}> = ({ jurisdiction, categories, onClose, onCreate, theme }) => {
  const [formData, setFormData] = useState<CreateCategoryInput>({
    name: "",
    jurisdiction,
    parent_id: null,
    deduction_rate: 1.0,
    code: "",
    description: ""
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
        throw new Error("Category name is required");
      }
      if (formData.deduction_rate < 0 || formData.deduction_rate > 1) {
        throw new Error("Deduction rate must be between 0% and 100%");
      }

      // Check for duplicate name
      const duplicate = categories.find(
        c => c.name.toLowerCase() === formData.name.trim().toLowerCase()
      );
      if (duplicate) {
        throw new Error(`Category "${formData.name}" already exists`);
      }

      await onCreate(formData);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to create category");
      setSubmitting(false);
    }
  };

  // Get parent categories (for hierarchical structure)
  const parentCategories = categories.filter(c => !c.is_archived);

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Create Custom Category</h3>
          <button className="close-button" onClick={onClose}>×</button>
        </div>

        <form onSubmit={handleSubmit} className="category-form">
          {error && <div className="form-error">{error}</div>}

          <div className="form-group">
            <label htmlFor="name">Category Name *</label>
            <input
              type="text"
              id="name"
              value={formData.name}
              onChange={(e) => setFormData({ ...formData, name: e.target.value })}
              placeholder="e.g., Software Subscriptions, Social Media Ads"
              required
              autoFocus
            />
            <small>Must be unique within jurisdiction</small>
          </div>

          <div className="form-group">
            <label htmlFor="parent">Parent Category (Optional)</label>
            <select
              id="parent"
              value={formData.parent_id || ""}
              onChange={(e) => setFormData({
                ...formData,
                parent_id: e.target.value || null
              })}
            >
              <option value="">None (Top-level category)</option>
              {parentCategories.map(c => (
                <option key={c.category_id} value={c.category_id}>
                  {c.name} ({c.code})
                </option>
              ))}
            </select>
            <small>Create a subcategory under an existing category</small>
          </div>

          <div className="form-group">
            <label htmlFor="deduction_rate">Deduction Rate *</label>
            <div className="deduction-rate-input">
              <input
                type="number"
                id="deduction_rate"
                value={formData.deduction_rate * 100}
                onChange={(e) => setFormData({
                  ...formData,
                  deduction_rate: parseFloat(e.target.value) / 100
                })}
                min="0"
                max="100"
                step="1"
                required
              />
              <span className="unit">%</span>
            </div>
            <div className="deduction-presets">
              <button
                type="button"
                onClick={() => setFormData({ ...formData, deduction_rate: 1.0 })}
                className={formData.deduction_rate === 1.0 ? 'active' : ''}
              >
                100% (Fully deductible)
              </button>
              <button
                type="button"
                onClick={() => setFormData({ ...formData, deduction_rate: 0.5 })}
                className={formData.deduction_rate === 0.5 ? 'active' : ''}
              >
                50% (Partially deductible)
              </button>
              <button
                type="button"
                onClick={() => setFormData({ ...formData, deduction_rate: 0.0 })}
                className={formData.deduction_rate === 0.0 ? 'active' : ''}
              >
                0% (Non-deductible)
              </button>
            </div>
          </div>

          <div className="form-group">
            <label htmlFor="code">Code (Optional)</label>
            <input
              type="text"
              id="code"
              value={formData.code}
              onChange={(e) => setFormData({ ...formData, code: e.target.value })}
              placeholder="e.g., line_8, 84111506"
            />
            <small>Reference code for reporting (e.g., Schedule C line, SAT code)</small>
          </div>

          <div className="form-group">
            <label htmlFor="description">Description (Optional)</label>
            <textarea
              id="description"
              value={formData.description}
              onChange={(e) => setFormData({ ...formData, description: e.target.value })}
              rows={3}
              placeholder="Add notes about this category..."
            />
          </div>

          <div className="form-actions">
            <button type="button" onClick={onClose} className="cancel-button" disabled={submitting}>
              Cancel
            </button>
            <button type="submit" className="submit-button" disabled={submitting}>
              {submitting ? "Creating..." : "Create Category"}
            </button>
          </div>
        </form>
      </div>
    </div>
  );
};
```

---

## Archive Confirmation Modal

```tsx
const CategoryArchiveModal: React.FC<{
  category: TaxCategory;
  usageCount: number;
  onClose: () => void;
  onConfirm: () => Promise<void>;
  theme: "light" | "dark";
}> = ({ category, usageCount, onClose, onConfirm, theme }) => {
  const [submitting, setSubmitting] = useState(false);

  const handleConfirm = async () => {
    setSubmitting(true);
    try {
      await onConfirm();
    } catch (err) {
      setSubmitting(false);
    }
  };

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content confirm-modal" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Archive Category?</h3>
          <button className="close-button" onClick={onClose}>×</button>
        </div>

        <div className="modal-body">
          <p>
            Are you sure you want to archive <strong>{category.name}</strong>?
          </p>

          {usageCount > 0 && (
            <div className="warning-box">
              <div className="warning-icon">⚠️</div>
              <div className="warning-text">
                <strong>This category is used in {usageCount} transactions.</strong>
                <br />
                Archiving will hide it from selection, but historical classifications will be preserved.
                You can restore it later if needed.
              </div>
            </div>
          )}

          {usageCount === 0 && (
            <p className="info-text">
              This category is not used in any transactions. It will be hidden from selection.
            </p>
          )}
        </div>

        <div className="modal-actions">
          <button className="cancel-button" onClick={onClose} disabled={submitting}>
            Cancel
          </button>
          <button className="danger-button" onClick={handleConfirm} disabled={submitting}>
            {submitting ? "Archiving..." : "Archive Category"}
          </button>
        </div>
      </div>
    </div>
  );
};
```

---

## Merge Dialog

```tsx
const CategoryMergeDialog: React.FC<{
  categories: TaxCategory[];
  usageStats: Record<string, number>;
  onClose: () => void;
  onConfirm: (primaryId: string, mergeIds: string[]) => Promise<void>;
  theme: "light" | "dark";
}> = ({ categories, usageStats, onClose, onConfirm, theme }) => {
  const [primaryId, setPrimaryId] = useState(categories[0]?.category_id || "");
  const [submitting, setSubmitting] = useState(false);

  const handleConfirm = async () => {
    setSubmitting(true);
    try {
      const mergeIds = categories
        .filter(c => c.category_id !== primaryId)
        .map(c => c.category_id);
      await onConfirm(primaryId, mergeIds);
    } catch (err) {
      setSubmitting(false);
    }
  };

  const totalTransactions = categories.reduce(
    (sum, c) => sum + (usageStats[c.category_id] || 0),
    0
  );

  return (
    <div className={`modal-overlay theme-${theme}`} onClick={onClose}>
      <div className="modal-content merge-modal" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Merge Categories</h3>
          <button className="close-button" onClick={onClose}>×</button>
        </div>

        <div className="modal-body">
          <p>Merge {categories.length} categories into one. Select the primary category to keep:</p>

          <div className="merge-options">
            {categories.map(category => (
              <label key={category.category_id} className="merge-option">
                <input
                  type="radio"
                  name="primary"
                  value={category.category_id}
                  checked={primaryId === category.category_id}
                  onChange={(e) => setPrimaryId(e.target.value)}
                />
                <div className="option-info">
                  <div className="option-name">{category.name}</div>
                  <div className="option-meta">
                    {category.code} • {usageStats[category.category_id] || 0} transactions
                  </div>
                </div>
              </label>
            ))}
          </div>

          <div className="info-box">
            <div className="info-icon">ℹ️</div>
            <div className="info-text">
              All {totalTransactions} transactions from merged categories will be reassigned to the primary category.
              This action cannot be undone.
            </div>
          </div>
        </div>

        <div className="modal-actions">
          <button className="cancel-button" onClick={onClose} disabled={submitting}>
            Cancel
          </button>
          <button className="submit-button" onClick={handleConfirm} disabled={submitting}>
            {submitting ? "Merging..." : "Merge Categories"}
          </button>
        </div>
      </div>
    </div>
  );
};
```

---

## Styling

```css
.tax-category-manager {
  background: var(--manager-bg);
  border: 1px solid var(--manager-border);
  border-radius: 8px;
  padding: 24px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.tax-category-manager.compact {
  padding: 16px;
  font-size: 14px;
}

/* Header */
.manager-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.header-title h2 {
  font-size: 24px;
  font-weight: 600;
  color: var(--heading-color);
  margin: 0 0 4px 0;
}

.category-count {
  font-size: 14px;
  color: var(--text-muted);
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

/* Jurisdiction Tabs */
.jurisdiction-tabs {
  display: flex;
  gap: 8px;
  margin-bottom: 20px;
  border-bottom: 2px solid var(--divider-color);
}

.jurisdiction-tab {
  padding: 10px 16px;
  background: transparent;
  border: none;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  cursor: pointer;
  font-weight: 600;
  color: var(--text-muted);
  transition: all 0.2s;
}

.jurisdiction-tab:hover {
  color: var(--text-color);
}

.jurisdiction-tab.active {
  color: var(--primary-color);
  border-bottom-color: var(--primary-color);
}

/* Filters */
.manager-filters {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 20px;
}

.search-input {
  width: 100%;
  padding: 10px 16px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
}

.filter-toggles {
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
}

.filter-toggle {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  color: var(--text-color);
  cursor: pointer;
}

.filter-toggle input[type="checkbox"] {
  width: 16px;
  height: 16px;
  cursor: pointer;
}

.tree-controls {
  display: flex;
  gap: 8px;
}

.tree-control-button {
  padding: 6px 12px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s;
}

.tree-control-button:hover {
  background: var(--hover-bg);
}

/* Selection Bar */
.selection-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: var(--selection-bg);
  border-radius: 6px;
  margin-bottom: 16px;
}

.selection-count {
  font-weight: 600;
  color: var(--text-color);
}

.merge-button {
  padding: 8px 16px;
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
  padding: 8px 16px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  cursor: pointer;
}

/* Category Tree */
.category-tree {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.taxonomy-section {
  background: var(--section-bg);
  border: 1px solid var(--section-border);
  border-radius: 8px;
  overflow: hidden;
}

.taxonomy-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  background: var(--taxonomy-header-bg);
  border-bottom: 1px solid var(--divider-color);
  font-weight: 600;
  color: var(--text-color);
}

.taxonomy-count {
  color: var(--text-muted);
  font-weight: 400;
}

.taxonomy-categories {
  padding: 8px;
}

/* Tree Node */
.tree-node {
  display: flex;
  flex-direction: column;
}

.node-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  background: var(--node-bg);
  border-radius: 6px;
  margin-bottom: 4px;
  transition: all 0.2s;
}

.node-row:hover {
  background: var(--node-hover-bg);
}

.node-row.selected {
  background: var(--node-selected-bg);
  border-left: 3px solid var(--primary-color);
}

.node-row.archived {
  opacity: 0.6;
}

.expand-button {
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  color: var(--text-muted);
  cursor: pointer;
  font-size: 12px;
}

.expand-spacer {
  width: 24px;
}

.select-checkbox {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.node-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.node-main {
  display: flex;
  align-items: center;
  gap: 8px;
}

.node-name {
  font-weight: 600;
  color: var(--text-color);
  font-size: 14px;
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

.archived-badge {
  font-size: 10px;
  padding: 2px 6px;
  background: var(--muted-bg);
  color: var(--muted-color);
  border-radius: 4px;
  font-weight: 600;
  text-transform: uppercase;
}

.node-meta {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 12px;
}

.node-code {
  color: var(--text-muted);
  font-family: monospace;
}

.deduction-badge {
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: 600;
  font-size: 11px;
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

.usage-count {
  color: var(--text-muted);
}

.node-actions {
  display: flex;
  gap: 8px;
}

.action-button {
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.2s;
}

.action-button:hover {
  background: var(--hover-bg);
}

/* Nested Children */
.node-children {
  margin-left: 32px;
  padding-left: 12px;
  border-left: 2px solid var(--tree-line-color);
}

/* Empty State */
.empty-state {
  padding: 60px 20px;
  text-align: center;
}

.empty-icon {
  font-size: 64px;
  margin-bottom: 16px;
  opacity: 0.5;
}

.empty-message {
  font-size: 16px;
  color: var(--text-muted);
  margin-bottom: 24px;
}

.create-button-secondary {
  padding: 10px 20px;
  background: var(--secondary-bg);
  color: var(--text-color);
  border: 2px solid var(--primary-color);
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

/* Modal Styles */
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
  animation: fadeIn 0.2s;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.modal-content {
  background: var(--modal-bg);
  border-radius: 12px;
  max-width: 600px;
  width: 90%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  animation: slideUp 0.3s;
}

@keyframes slideUp {
  from {
    transform: translateY(40px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 24px;
  border-bottom: 1px solid var(--divider-color);
}

.modal-header h3 {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
}

.close-button {
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: var(--text-muted);
}

.modal-body {
  padding: 24px;
}

.category-form {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 24px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.form-group label {
  font-weight: 600;
  font-size: 14px;
  color: var(--text-color);
}

.form-group input,
.form-group select,
.form-group textarea {
  padding: 10px 12px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  font-size: 14px;
  font-family: inherit;
}

.form-group small {
  font-size: 12px;
  color: var(--text-muted);
}

.form-error {
  padding: 12px;
  background: var(--error-bg);
  color: var(--error-color);
  border-radius: 6px;
  font-size: 14px;
}

.deduction-rate-input {
  display: flex;
  align-items: center;
  gap: 8px;
}

.deduction-rate-input input {
  flex: 1;
}

.deduction-rate-input .unit {
  font-weight: 600;
  color: var(--text-muted);
}

.deduction-presets {
  display: flex;
  gap: 8px;
  margin-top: 8px;
}

.deduction-presets button {
  flex: 1;
  padding: 8px 12px;
  background: var(--button-bg);
  border: 2px solid var(--border-color);
  border-radius: 6px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
}

.deduction-presets button.active {
  border-color: var(--primary-color);
  background: var(--primary-bg-light);
  font-weight: 600;
}

.form-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
  padding-top: 12px;
}

.cancel-button {
  padding: 10px 20px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
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

.danger-button {
  padding: 10px 20px;
  background: var(--error-color);
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.warning-box,
.info-box {
  display: flex;
  gap: 12px;
  padding: 16px;
  border-radius: 8px;
  margin: 16px 0;
}

.warning-box {
  background: var(--warning-bg);
  border: 1px solid var(--warning-border);
}

.info-box {
  background: var(--info-bg);
  border: 1px solid var(--info-border);
}

.warning-icon,
.info-icon {
  font-size: 20px;
}

.merge-options {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin: 16px 0;
}

.merge-option {
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

.merge-option:has(input:checked) {
  border-color: var(--primary-color);
  background: var(--option-selected-bg);
}

.merge-option input[type="radio"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.option-info {
  flex: 1;
}

.option-name {
  font-weight: 600;
  margin-bottom: 4px;
}

.option-meta {
  font-size: 12px;
  color: var(--text-muted);
}

/* Theme Variables */
.tax-category-manager.theme-light {
  --manager-bg: #ffffff;
  --manager-border: #e5e7eb;
  --heading-color: #111827;
  --text-color: #111827;
  --text-muted: #6b7280;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --section-bg: #f9fafb;
  --section-border: #e5e7eb;
  --taxonomy-header-bg: #f3f4f6;
  --node-bg: #ffffff;
  --node-hover-bg: #f9fafb;
  --node-selected-bg: #eff6ff;
  --divider-color: #e5e7eb;
  --hover-bg: #f3f4f6;
  --input-border: #d1d5db;
  --border-color: #d1d5db;
  --tree-line-color: #d1d5db;
  --success-bg: #d1fae5;
  --success-color: #065f46;
  --warning-bg: #fef3c7;
  --warning-color: #92400e;
  --error-bg: #fee2e2;
  --error-color: #991b1b;
  --info-bg: #dbeafe;
  --info-color: #1e40af;
  --muted-bg: #e5e7eb;
  --muted-color: #6b7280;
  --selection-bg: #eff6ff;
  --modal-bg: #ffffff;
}

.tax-category-manager.theme-dark {
  --manager-bg: #1f2937;
  --manager-border: #374151;
  --heading-color: #f9fafb;
  --text-color: #f9fafb;
  --text-muted: #9ca3af;
  --primary-color: #3b82f6;
  --primary-color-hover: #60a5fa;
  --section-bg: #111827;
  --section-border: #374151;
  --taxonomy-header-bg: #1f2937;
  --node-bg: #1f2937;
  --node-hover-bg: #374151;
  --node-selected-bg: #1e3a8a;
  --divider-color: #374151;
  --hover-bg: #374151;
  --input-border: #4b5563;
  --border-color: #4b5563;
  --tree-line-color: #4b5563;
  --success-bg: #064e3b;
  --success-color: #d1fae5;
  --warning-bg: #78350f;
  --warning-color: #fef3c7;
  --error-bg: #7f1d1d;
  --error-color: #fee2e2;
  --info-bg: #1e3a8a;
  --info-color: #dbeafe;
  --muted-bg: #374151;
  --muted-color: #9ca3af;
  --selection-bg: #1e3a8a;
  --modal-bg: #1f2937;
}

/* Responsive Design */
@media (max-width: 768px) {
  .manager-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 16px;
  }

  .manager-filters {
    flex-direction: column;
  }

  .filter-toggles {
    flex-direction: column;
  }

  .node-children {
    margin-left: 16px;
  }

  .modal-content {
    max-width: 100%;
    border-radius: 16px 16px 0 0;
    max-height: 80vh;
  }
}
```

---

## User Interactions

### 1. Create Custom Category
```
User: Clicks "New Category" button (or presses ⌘N)
System:
  1. Opens create modal
  2. User fills form:
     - Name: "Software Subscriptions"
     - Parent: "Office Expenses"
     - Deduction Rate: 100%
     - Code: "custom_software"
  3. Validates name uniqueness
  4. Creates category with user_id
  5. Adds to tree under parent
  6. Success notification
```

### 2. Edit Custom Category
```
User: Clicks ✏️ edit icon on custom category
System:
  1. Opens edit modal with pre-filled data
  2. User changes deduction rate from 100% to 50%
  3. Validates changes
  4. Updates category
  5. Refreshes tree view
```

### 3. Archive Category with Transactions
```
User: Clicks 📦 archive icon on category with 10 transactions
System:
  1. Shows warning: "This category is used in 10 transactions"
  2. User confirms
  3. Soft deletes category (is_archived=true)
  4. Hides from future selections
  5. Preserves historical transaction classifications
  6. User can restore later if needed
```

### 4. Merge Duplicate Categories
```
User: Selects 3 similar categories ("Uber", "Uber Rides", "Uber Eats")
System:
  1. Enables "Merge Categories" button
  2. User clicks merge
  3. Dialog shows total transaction count (45)
  4. User selects primary: "Uber"
  5. System reassigns all 45 transactions to "Uber"
  6. Archives "Uber Rides" and "Uber Eats"
  7. Success: "3 categories merged, 45 transactions updated"
```

### 5. Search Categories
```
User: Types "advertising" in search
System:
  1. Filters tree in real-time
  2. Shows matches:
     - "Advertising" (Schedule C, line_8)
     - "Publicidad" (SAT, 84111506)
     - "Social Media Ads" (Custom)
  3. Maintains hierarchy (shows parents of matches)
```

### 6. Expand/Collapse Tree
```
User: Clicks expand arrow on "Office Expenses"
System:
  1. Expands to show children:
     - Software (Custom)
     - Supplies
     - Postage
  2. Arrow changes: ▶ → ▼

User: Clicks "Collapse All"
System:
  1. Collapses entire tree
  2. Shows only root categories
```

---

## Wireframes (ASCII Art)

### Tree View (Expanded)
```
┌───────────────────────────────────────────────────┐
│ Tax Category Management          + New Category   │
│ 25 categories                                     │
├───────────────────────────────────────────────────┤
│ [USA Federal] [Mexico Federal]                    │
├───────────────────────────────────────────────────┤
│ 🔍 Search categories...                           │
│ ☐ System Only ☐ Custom Only ☐ Unused ☐ Archived │
│ [Expand All] [Collapse All]                       │
├───────────────────────────────────────────────────┤
│ Schedule C (USA)                              (8) │
│ ┌───────────────────────────────────────────────┐ │
│ │ ▼ ☐ Advertising    line_8  [100%] 5 txns ✏️📦│ │
│ │ ▼ ☐ Office Expenses line_18 [100%] 12 txns   │ │
│ │   ▶ ☐ Software (Custom)     [100%] 8 txns ✏️ │ │
│ │   ▶ ☐ Supplies              [100%] 4 txns    │ │
│ │ ▼ ☐ Meals (Business) line_24b [50%] 3 txns   │ │
│ │ ▼ ☐ Travel          line_24a [100%] 7 txns   │ │
│ └───────────────────────────────────────────────┘ │
│                                                   │
│ SAT (Mexico)                                  (4) │
│ ┌───────────────────────────────────────────────┐ │
│ │ ▼ ☐ Gastos de Gestión 84111500 [100%] 2 txns │ │
│ │   ▶ ☐ Publicidad     84111506 [100%] 2 txns  │ │
│ └───────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────┘
```

### Create Modal
```
┌─────────────────────────────────────────┐
│ Create Custom Category               × │
├─────────────────────────────────────────┤
│                                         │
│ Category Name *                         │
│ ┌─────────────────────────────────────┐ │
│ │ Software Subscriptions              │ │
│ └─────────────────────────────────────┘ │
│ Must be unique within jurisdiction      │
│                                         │
│ Parent Category (Optional)              │
│ ┌─────────────────────────────────────┐ │
│ │ Office Expenses (line_18)        ▼ │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Deduction Rate *                        │
│ ┌───────┐                               │
│ │ 100   │ %                             │
│ └───────┘                               │
│ [100% Fully] [50% Partial] [0% None]    │
│                                         │
│ Code (Optional)                         │
│ ┌─────────────────────────────────────┐ │
│ │ custom_software                     │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Description (Optional)                  │
│ ┌─────────────────────────────────────┐ │
│ │ Monthly SaaS subscriptions...       │ │
│ └─────────────────────────────────────┘ │
│                                         │
│              [Cancel] [Create Category] │
└─────────────────────────────────────────┘
```

### Archive Confirmation
```
┌─────────────────────────────────────────┐
│ Archive Category?                    × │
├─────────────────────────────────────────┤
│                                         │
│ Are you sure you want to archive        │
│ Software Subscriptions?                 │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ ⚠️  This category is used in 8 txns.│ │
│ │                                     │ │
│ │ Archiving will hide it from         │ │
│ │ selection, but historical           │ │
│ │ classifications will be preserved.  │ │
│ └─────────────────────────────────────┘ │
│                                         │
│              [Cancel] [Archive Category]│
└─────────────────────────────────────────┘
```

### Merge Dialog
```
┌─────────────────────────────────────────┐
│ Merge Categories                     × │
├─────────────────────────────────────────┤
│                                         │
│ Merge 3 categories into one. Select     │
│ the primary category to keep:           │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ ◉ Uber                              │ │
│ │   uber • 20 transactions            │ │
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │ ○ Uber Rides                        │ │
│ │   uber_rides • 15 transactions      │ │
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │ ○ Uber Eats                         │ │
│ │   uber_eats • 10 transactions       │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ ℹ️  All 45 transactions will be     │ │
│ │    reassigned to the primary        │ │
│ │    category. This cannot be undone. │ │
│ └─────────────────────────────────────┘ │
│                                         │
│             [Cancel] [Merge Categories] │
└─────────────────────────────────────────┘
```

---

## Accessibility

```tsx
// ARIA attributes
<button
  role="treeitem"
  aria-expanded={isExpanded}
  aria-level={depth}
  aria-label={`${category.name}, ${usageCount} transactions`}
>
  {/* Category content */}
</button>

// Keyboard navigation
// N or ⌘N: New category
// Esc: Close modals
// Space: Toggle checkbox (for merge)
// Enter: Edit category

// Screen reader announcements
aria-live="polite"
// "Custom category Software Subscriptions created"
// "Category Uber merged with 2 others, 45 transactions updated"
// "Category archived, will be hidden from selection"
```

---

## Multi-Domain Examples

### Finance Domain (Tax Categories)
```tsx
<TaxCategoryManager
  categories={taxCategories}
  usageStats={taxUsageStats}
  jurisdiction="usa_federal"
  availableJurisdictions={["usa_federal", "mexico_federal"]}
  currentUserId="user_123"
  onCategoryCreate={createTaxCategory}
  onCategoryUpdate={updateTaxCategory}
  onCategoryArchive={archiveTaxCategory}
  onCategoryRestore={restoreTaxCategory}
  onCategoriesMerge={mergeTaxCategories}
  showUsageStats={true}
  allowMerge={true}
/>
```

### Healthcare Domain (Procedure Codes)
```tsx
<TaxCategoryManager
  categories={cptCodes}
  usageStats={procedureUsageStats}
  jurisdiction="hipaa"
  availableJurisdictions={["hipaa", "medicare"]}
  currentUserId="provider_456"
  onCategoryCreate={createProcedureCode}
  onCategoryUpdate={updateProcedureCode}
  onCategoryArchive={archiveProcedureCode}
  onCategoryRestore={restoreProcedureCode}
  onCategoriesMerge={mergeProcedureCodes}
  showUsageStats={true}
/>
```

### Legal Domain (Case Categories)
```tsx
<TaxCategoryManager
  categories={caseLawCategories}
  usageStats={caseUsageStats}
  jurisdiction="ca_state"
  availableJurisdictions={["ca_state", "federal"]}
  currentUserId="attorney_789"
  onCategoryCreate={createCaseCategory}
  onCategoryUpdate={updateCaseCategory}
  onCategoryArchive={archiveCaseCategory}
  onCategoryRestore={restoreCaseCategory}
  onCategoriesMerge={mergeCaseCategories}
  showUsageStats={true}
/>
```

### Research Domain (Grant Categories)
```tsx
<TaxCategoryManager
  categories={grantCategories}
  usageStats={grantUsageStats}
  jurisdiction="nsf"
  availableJurisdictions={["nsf", "nih"]}
  currentUserId="researcher_101"
  onCategoryCreate={createGrantCategory}
  onCategoryUpdate={updateGrantCategory}
  onCategoryArchive={archiveGrantCategory}
  onCategoryRestore={restoreGrantCategory}
  onCategoriesMerge={mergeGrantCategories}
  showUsageStats={true}
/>
```

---

## Related Components

**Uses:**
- TaxCategorySelector (for selecting parent category)

**Used by:**
- Settings page (category management section)
- Transaction classification workflow (custom category creation)
- Admin dashboard (system category management)

**Similar patterns:**
- CounterpartyManager (for managing merchants/people)
- AccountManager (for managing accounts)
- SeriesManager (for managing recurring series)

**OL Dependencies:**
- TaxCategoryStore (CRUD operations)
- TaxonomyEngine (build hierarchy tree)
- TransactionStore (usage statistics)
