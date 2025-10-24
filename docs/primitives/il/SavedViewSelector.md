# SavedViewSelector (IL Component)

## Definition

**SavedViewSelector** is a reusable dropdown component for selecting and managing saved views (dashboards, filters, queries, searches). It provides consistent UX for switching between pre-configured views and creating new ones.

**Problem it solves:**
- Users need to quickly switch between different saved configurations
- No standard UI pattern for managing saved views across features
- View management (create, edit, delete, set default) scattered across UI
- Inconsistent dropdown styling and behavior

**Solution:**
- Single dropdown component for all saved view scenarios
- Built-in view management (create, edit, delete, set default)
- Keyboard navigation support (arrow keys, search)
- Visual distinction between system views and user views
- Recent views history (MRU - Most Recently Used)

---

## Interface Contract

```typescript
interface SavedViewSelectorProps {
  // Data
  views: SavedView[];  // All available views (system + user)
  currentViewId?: string;  // Currently selected view
  defaultViewId?: string;  // Default view (shown with star icon)

  // Callbacks
  onViewSelect: (viewId: string) => void;  // User selects a view
  onViewCreate?: () => void;  // User clicks "New View"
  onViewEdit?: (viewId: string) => void;  // User edits a view
  onViewDelete?: (viewId: string) => void;  // User deletes a view
  onViewSetDefault?: (viewId: string) => void;  // User sets as default

  // UI customization
  placeholder?: string;  // Placeholder text (default: "Select a view...")
  showCreateButton?: boolean;  // Show "New View" button (default: true)
  showManageButton?: boolean;  // Show "Manage Views" button (default: true)
  maxRecentViews?: number;  // How many recent views to show (default: 5)
  theme?: "light" | "dark";
  disabled?: boolean;
}

interface SavedView {
  view_id: string;
  name: string;
  description?: string;
  is_system: boolean;  // System views cannot be deleted
  is_default: boolean;
  created_at: string;
  updated_at: string;
}
```

---

## Component Structure

```tsx
import React, { useState } from 'react';

export const SavedViewSelector: React.FC<SavedViewSelectorProps> = ({
  views,
  currentViewId,
  defaultViewId,
  onViewSelect,
  onViewCreate,
  onViewEdit,
  onViewDelete,
  onViewSetDefault,
  placeholder = "Select a view...",
  showCreateButton = true,
  showManageButton = true,
  maxRecentViews = 5,
  theme = "light",
  disabled = false
}) => {
  const [isOpen, setIsOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState("");
  const [recentViews, setRecentViews] = useState<string[]>([]);

  // Find current view
  const currentView = views.find(v => v.view_id === currentViewId);

  // Filter views by search query
  const filteredViews = views.filter(v =>
    v.name.toLowerCase().includes(searchQuery.toLowerCase())
  );

  // Separate system and user views
  const systemViews = filteredViews.filter(v => v.is_system);
  const userViews = filteredViews.filter(v => !v.is_system);

  // Handle view selection
  const handleSelect = (viewId: string) => {
    onViewSelect(viewId);
    setIsOpen(false);
    setSearchQuery("");

    // Update recent views (MRU)
    setRecentViews(prev => {
      const updated = [viewId, ...prev.filter(id => id !== viewId)];
      return updated.slice(0, maxRecentViews);
    });
  };

  // Keyboard navigation
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Escape') {
      setIsOpen(false);
    }
  };

  return (
    <div
      className={`saved-view-selector theme-${theme} ${disabled ? 'disabled' : ''}`}
      onKeyDown={handleKeyDown}
    >
      {/* Dropdown trigger */}
      <button
        className="selector-trigger"
        onClick={() => setIsOpen(!isOpen)}
        disabled={disabled}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
      >
        <span className="selected-view-name">
          {currentView ? currentView.name : placeholder}
        </span>
        <span className="dropdown-icon">▼</span>
      </button>

      {/* Dropdown menu */}
      {isOpen && (
        <div className="selector-dropdown" role="listbox">
          {/* Search box */}
          <div className="search-box">
            <input
              type="text"
              placeholder="Search views..."
              value={searchQuery}
              onChange={(e) => setSearchQuery(e.target.value)}
              autoFocus
            />
          </div>

          {/* Create new view button */}
          {showCreateButton && onViewCreate && (
            <button
              className="create-view-button"
              onClick={() => {
                onViewCreate();
                setIsOpen(false);
              }}
            >
              + New View
            </button>
          )}

          {/* Recent views (if any) */}
          {recentViews.length > 0 && (
            <>
              <div className="section-header">Recent</div>
              {recentViews.map(viewId => {
                const view = views.find(v => v.view_id === viewId);
                return view ? (
                  <ViewItem
                    key={viewId}
                    view={view}
                    isSelected={viewId === currentViewId}
                    isDefault={viewId === defaultViewId}
                    onSelect={handleSelect}
                    onEdit={onViewEdit}
                    onDelete={onViewDelete}
                    onSetDefault={onViewSetDefault}
                  />
                ) : null;
              })}
              <div className="section-divider" />
            </>
          )}

          {/* System views */}
          {systemViews.length > 0 && (
            <>
              <div className="section-header">System Views</div>
              {systemViews.map(view => (
                <ViewItem
                  key={view.view_id}
                  view={view}
                  isSelected={view.view_id === currentViewId}
                  isDefault={view.view_id === defaultViewId}
                  onSelect={handleSelect}
                  onEdit={onViewEdit}
                  onDelete={onViewDelete}
                  onSetDefault={onViewSetDefault}
                />
              ))}
              <div className="section-divider" />
            </>
          )}

          {/* User views */}
          {userViews.length > 0 && (
            <>
              <div className="section-header">My Views</div>
              {userViews.map(view => (
                <ViewItem
                  key={view.view_id}
                  view={view}
                  isSelected={view.view_id === currentViewId}
                  isDefault={view.view_id === defaultViewId}
                  onSelect={handleSelect}
                  onEdit={onViewEdit}
                  onDelete={onViewDelete}
                  onSetDefault={onViewSetDefault}
                />
              ))}
            </>
          )}

          {/* Empty state */}
          {filteredViews.length === 0 && (
            <div className="empty-state">
              No views found matching "{searchQuery}"
            </div>
          )}

          {/* Manage views button */}
          {showManageButton && (
            <>
              <div className="section-divider" />
              <button className="manage-views-button">
                Manage Views
              </button>
            </>
          )}
        </div>
      )}
    </div>
  );
};

// Individual view item component
const ViewItem: React.FC<{
  view: SavedView;
  isSelected: boolean;
  isDefault: boolean;
  onSelect: (viewId: string) => void;
  onEdit?: (viewId: string) => void;
  onDelete?: (viewId: string) => void;
  onSetDefault?: (viewId: string) => void;
}> = ({ view, isSelected, isDefault, onSelect, onEdit, onDelete, onSetDefault }) => {
  const [showActions, setShowActions] = useState(false);

  return (
    <div
      className={`view-item ${isSelected ? 'selected' : ''}`}
      onMouseEnter={() => setShowActions(true)}
      onMouseLeave={() => setShowActions(false)}
    >
      <button
        className="view-name"
        onClick={() => onSelect(view.view_id)}
        role="option"
        aria-selected={isSelected}
      >
        {view.name}
        {isDefault && <span className="default-badge">★</span>}
        {view.is_system && <span className="system-badge">System</span>}
      </button>

      {/* Action buttons (show on hover) */}
      {showActions && !view.is_system && (
        <div className="view-actions">
          {onEdit && (
            <button
              className="action-button edit"
              onClick={(e) => {
                e.stopPropagation();
                onEdit(view.view_id);
              }}
              title="Edit view"
            >
              ✏️
            </button>
          )}
          {onSetDefault && !isDefault && (
            <button
              className="action-button set-default"
              onClick={(e) => {
                e.stopPropagation();
                onSetDefault(view.view_id);
              }}
              title="Set as default"
            >
              ⭐
            </button>
          )}
          {onDelete && (
            <button
              className="action-button delete"
              onClick={(e) => {
                e.stopPropagation();
                if (confirm(`Delete view "${view.name}"?`)) {
                  onDelete(view.view_id);
                }
              }}
              title="Delete view"
            >
              🗑️
            </button>
          )}
        </div>
      )}
    </div>
  );
};
```

---

## Styling

```css
.saved-view-selector {
  position: relative;
  width: 300px;
}

.selector-trigger {
  width: 100%;
  padding: 8px 12px;
  background: var(--trigger-bg);
  border: 1px solid var(--trigger-border);
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 14px;
  transition: all 0.2s;
}

.selector-trigger:hover {
  border-color: var(--trigger-border-hover);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.selector-trigger.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.dropdown-icon {
  color: var(--icon-color);
  transition: transform 0.2s;
}

.selector-trigger[aria-expanded="true"] .dropdown-icon {
  transform: rotate(180deg);
}

.selector-dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  margin-top: 4px;
  background: var(--dropdown-bg);
  border: 1px solid var(--dropdown-border);
  border-radius: 6px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  max-height: 400px;
  overflow-y: auto;
  z-index: 1000;
}

.search-box {
  padding: 8px;
  border-bottom: 1px solid var(--divider-color);
}

.search-box input {
  width: 100%;
  padding: 6px 8px;
  border: 1px solid var(--input-border);
  border-radius: 4px;
  font-size: 13px;
}

.create-view-button {
  width: 100%;
  padding: 10px;
  background: var(--create-button-bg);
  color: var(--create-button-color);
  border: none;
  border-bottom: 1px solid var(--divider-color);
  cursor: pointer;
  font-weight: 600;
  text-align: left;
  transition: background 0.2s;
}

.create-view-button:hover {
  background: var(--create-button-hover-bg);
}

.section-header {
  padding: 8px 12px;
  font-size: 11px;
  font-weight: 600;
  color: var(--section-header-color);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.section-divider {
  height: 1px;
  background: var(--divider-color);
  margin: 4px 0;
}

.view-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 8px;
  transition: background 0.2s;
}

.view-item:hover {
  background: var(--item-hover-bg);
}

.view-item.selected {
  background: var(--item-selected-bg);
}

.view-name {
  flex: 1;
  padding: 10px 8px;
  background: none;
  border: none;
  text-align: left;
  cursor: pointer;
  font-size: 14px;
  color: var(--text-color);
  display: flex;
  align-items: center;
  gap: 8px;
}

.default-badge {
  color: #fbbf24;
  font-size: 12px;
}

.system-badge {
  font-size: 10px;
  padding: 2px 6px;
  background: var(--badge-bg);
  color: var(--badge-color);
  border-radius: 4px;
  text-transform: uppercase;
}

.view-actions {
  display: flex;
  gap: 4px;
}

.action-button {
  padding: 4px 8px;
  background: none;
  border: none;
  cursor: pointer;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.action-button:hover {
  opacity: 1;
}

.empty-state {
  padding: 20px;
  text-align: center;
  color: var(--secondary-color);
  font-size: 13px;
}

/* Theme: Light */
.theme-light {
  --trigger-bg: #ffffff;
  --trigger-border: #d1d5db;
  --trigger-border-hover: #9ca3af;
  --dropdown-bg: #ffffff;
  --dropdown-border: #d1d5db;
  --item-hover-bg: #f3f4f6;
  --item-selected-bg: #e0e7ff;
  --create-button-bg: #3b82f6;
  --create-button-color: #ffffff;
  --create-button-hover-bg: #2563eb;
  --section-header-color: #6b7280;
  --divider-color: #e5e7eb;
  --text-color: #111827;
  --icon-color: #6b7280;
  --badge-bg: #e5e7eb;
  --badge-color: #6b7280;
  --secondary-color: #9ca3af;
  --input-border: #d1d5db;
}

/* Theme: Dark */
.theme-dark {
  --trigger-bg: #1f2937;
  --trigger-border: #374151;
  --trigger-border-hover: #4b5563;
  --dropdown-bg: #1f2937;
  --dropdown-border: #374151;
  --item-hover-bg: #374151;
  --item-selected-bg: #1e3a8a;
  --create-button-bg: #3b82f6;
  --create-button-color: #ffffff;
  --create-button-hover-bg: #2563eb;
  --section-header-color: #9ca3af;
  --divider-color: #374151;
  --text-color: #f3f4f6;
  --icon-color: #9ca3af;
  --badge-bg: #374151;
  --badge-color: #9ca3af;
  --secondary-color: #6b7280;
  --input-border: #4b5563;
}
```

---

## Multi-Domain Applicability

### Finance Domain
```tsx
<SavedViewSelector
  views={dashboardViews}
  currentViewId="view_monthly_overview"
  defaultViewId="view_monthly_overview"
  onViewSelect={(id) => loadDashboardView(id)}
  onViewCreate={() => openCreateDashboardModal()}
  onViewDelete={(id) => deleteDashboardView(id)}
/>
// Views: "Monthly Overview", "Q1 Deductibles", "Annual Review"
```

### Healthcare Domain
```tsx
<SavedViewSelector
  views={patientQueryViews}
  currentViewId="view_diabetes_patients"
  onViewSelect={(id) => loadPatientQuery(id)}
  onViewCreate={() => openSaveQueryModal()}
/>
// Views: "Diabetes Patients", "High Blood Pressure", "Overdue Checkups"
```

### Legal Domain
```tsx
<SavedViewSelector
  views={caseSearchViews}
  currentViewId="view_active_cases"
  onViewSelect={(id) => loadCaseSearch(id)}
  onViewCreate={() => openSaveCaseSearchModal()}
/>
// Views: "Active Cases", "Won Cases 2025", "Pending Appeals"
```

### Research Domain
```tsx
<SavedViewSelector
  views={paperLibraryViews}
  currentViewId="view_my_papers"
  onViewSelect={(id) => loadPaperLibrary(id)}
  onViewCreate={() => openSaveLibraryViewModal()}
/>
// Views: "My Papers", "Peer-Reviewed Only", "AI & ML Topics"
```

### Manufacturing Domain
```tsx
<SavedViewSelector
  views={qcDashboardViews}
  currentViewId="view_daily_qc"
  onViewSelect={(id) => loadQCDashboard(id)}
  onViewCreate={() => openCreateQCViewModal()}
/>
// Views: "Daily QC", "Defect Trends", "Shift Comparison"
```

### Media Domain
```tsx
<SavedViewSelector
  views={analyticsViews}
  currentViewId="view_top_content"
  onViewSelect={(id) => loadAnalyticsView(id)}
  onViewCreate={() => openCreateAnalyticsViewModal()}
/>
// Views: "Top Content", "Engagement Trends", "Audience Demographics"
```

---

## Features

### 1. Recent Views (MRU - Most Recently Used)
```typescript
// Automatically tracks last 5 views selected by user
// Shown at top of dropdown for quick access
setRecentViews([
  "view_q1_deductibles",
  "view_monthly_overview",
  "view_annual_review"
]);
```

### 2. System vs User Views
```typescript
// System views: cannot be deleted, shown first
systemViews.filter(v => v.is_system)

// User views: can be edited/deleted, shown after system views
userViews.filter(v => !v.is_system)
```

### 3. Default View Indicator
```typescript
// View with star (★) icon is the default
// Loaded automatically on page load
view.is_default = true  // → Shows ★ badge
```

### 4. Search/Filter
```typescript
// Filter views by name as user types
filteredViews = views.filter(v =>
  v.name.toLowerCase().includes(searchQuery.toLowerCase())
)
```

### 5. Keyboard Navigation
```typescript
// ESC: Close dropdown
// Arrow keys: Navigate views (future enhancement)
// Enter: Select highlighted view (future enhancement)
```

---

## Accessibility

```tsx
<button
  role="combobox"
  aria-haspopup="listbox"
  aria-expanded={isOpen}
  aria-label="Select saved view"
  aria-activedescendant={currentViewId}
>
  {currentView?.name}
</button>

<div role="listbox">
  <div role="option" aria-selected={isSelected}>
    {view.name}
  </div>
</div>
```

---

## Related Components

**Used by:**
- Dashboard page (select dashboard view)
- Transaction list page (select saved filter)
- Any feature with saved configurations

**Uses:**
- SavedViewStore (loads available views)

**Similar patterns in other domains:**
- Filter selector (saved search queries)
- Report selector (saved reports)
- Template selector (saved templates)
