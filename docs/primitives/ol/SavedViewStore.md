# SavedViewStore (OL Primitive)

## Definition

**SavedViewStore** persists saved dashboard view configurations (filters, widgets, layouts). It enables users to save custom dashboard configurations as named presets and load them later.

**Problem it solves:**
- Users need to recreate complex filter combinations manually (tedious)
- No way to save frequently used dashboard configurations
- Cannot share reproducible views with team members (future feature)
- Dashboard state is lost on page refresh

**Solution:**
- Persist view configurations to database
- CRUD operations for saved views (create, read, update, delete)
- System-provided default views (cannot be deleted)
- User-created custom views (can be deleted)

---

## Interface Contract

```python
from typing import Optional, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class SavedView:
    """Saved dashboard view configuration"""
    view_id: str  # Format: "view_{slug}"
    name: str  # User-facing name
    description: Optional[str]
    is_default: bool  # Whether this is default view on page load
    is_system: bool  # Whether this is system view (cannot be deleted)
    user_id: Optional[str]  # Owner (null for system views)
    config: dict  # DashboardConfig (date_range, filters, widgets)
    created_at: datetime
    updated_at: datetime

class SavedViewStore:
    """
    Persist and retrieve saved dashboard views.

    Domain-agnostic - works for ANY dashboard configuration.
    """

    def create_view(
        self,
        user_id: str,
        name: str,
        config: dict,
        description: Optional[str] = None
    ) -> SavedView:
        """
        Create new saved view.

        Args:
            user_id: Owner of this view
            name: User-facing view name (max 100 chars)
            config: Dashboard configuration (filters, widgets)
            description: Optional description (max 500 chars)

        Returns:
            Created SavedView

        Raises:
            MaxViewsExceededError: If user has 50+ views
            DuplicateViewNameError: If view name already exists for user
            InvalidConfigError: If config validation fails
        """

    def get_view(self, view_id: str, user_id: str) -> SavedView:
        """
        Get single view by ID.

        Args:
            view_id: View identifier
            user_id: Current user (for access control)

        Returns:
            SavedView if found and user has access

        Raises:
            ViewNotFoundError: If view doesn't exist
            UnauthorizedError: If user doesn't own this view
        """

    def get_user_views(self, user_id: str) -> List[SavedView]:
        """
        Get all views for user (system views + user's custom views).

        Returns:
            List of SavedView (system views first, then user views sorted by name)
        """

    def update_view(
        self,
        view_id: str,
        user_id: str,
        name: Optional[str] = None,
        description: Optional[str] = None,
        config: Optional[dict] = None
    ) -> SavedView:
        """
        Update existing view.

        Args:
            view_id: View to update
            user_id: Current user (for access control)
            name: New name (if provided)
            description: New description (if provided)
            config: New config (if provided)

        Returns:
            Updated SavedView

        Raises:
            ViewNotFoundError: If view doesn't exist
            UnauthorizedError: If user doesn't own this view
            CannotModifySystemViewError: If trying to update system view
        """

    def delete_view(self, view_id: str, user_id: str) -> None:
        """
        Delete user's custom view.

        Args:
            view_id: View to delete
            user_id: Current user (for access control)

        Raises:
            ViewNotFoundError: If view doesn't exist
            UnauthorizedError: If user doesn't own this view
            CannotDeleteSystemViewError: If trying to delete system view
        """

    def set_default_view(self, view_id: str, user_id: str) -> None:
        """
        Set view as default (loaded on dashboard page load).

        Only one view can be default per user.
        Setting new default unsets previous default.
        """
```

---

## Multi-Domain Applicability

### Finance Domain
```python
view = SavedView(
    view_id="view_q1_deductibles",
    name="Q1 2025 Deductibles",
    config={
        "date_range": {"from": "2025-01-01", "to": "2025-03-31"},
        "filters": {"deductible_usa": True},
        "widgets": [
            {"type": "deductible_summary", "position": {"row": 0, "col": 0, "width": 8}},
            {"type": "category_breakdown", "position": {"row": 1, "col": 0, "width": 8}}
        ]
    }
)
```

### Healthcare Domain
```python
view = SavedView(
    view_id="view_diabetes_patients",
    name="Diabetes Patients - Monthly Vitals",
    config={
        "date_range": "current_month",
        "filters": {"diagnosis": "diabetes"},
        "widgets": [
            {"type": "avg_glucose", "position": {"row": 0, "col": 0, "width": 4}},
            {"type": "hba1c_trend", "position": {"row": 0, "col": 4, "width": 4}}
        ]
    }
)
```

### Legal Domain
```python
view = SavedView(
    view_id="view_won_cases_2025",
    name="Won Cases 2025",
    config={
        "date_range": "current_year",
        "filters": {"outcome": "won"},
        "widgets": [
            {"type": "case_count", "position": {"row": 0, "col": 0, "width": 4}},
            {"type": "hours_billed", "position": {"row": 0, "col": 4, "width": 4}}
        ]
    }
)
```

### Research Domain
```python
view = SavedView(
    view_id="view_h_index_dashboard",
    name="Publication Metrics 2025",
    config={
        "date_range": "current_year",
        "filters": {"type": "peer_reviewed"},
        "widgets": [
            {"type": "papers_published", "position": {"row": 0, "col": 0, "width": 4}},
            {"type": "citation_count", "position": {"row": 0, "col": 4, "width": 4}},
            {"type": "h_index", "position": {"row": 1, "col": 0, "width": 4}}
        ]
    }
)
```

### Manufacturing Domain
```python
view = SavedView(
    view_id="view_qc_monthly",
    name="QC Dashboard - Monthly",
    config={
        "date_range": "current_month",
        "filters": {"product_line": "widgets"},
        "widgets": [
            {"type": "units_produced", "position": {"row": 0, "col": 0, "width": 4}},
            {"type": "defect_rate", "position": {"row": 0, "col": 4, "width": 4}},
            {"type": "pass_fail_ratio", "position": {"row": 1, "col": 0, "width": 8}}
        ]
    }
)
```

### Media Domain
```python
view = SavedView(
    view_id="view_top_content",
    name="Top Performing Content - Q1",
    config={
        "date_range": {"from": "2025-01-01", "to": "2025-03-31"},
        "filters": {"content_type": "video"},
        "widgets": [
            {"type": "total_views", "position": {"row": 0, "col": 0, "width": 4}},
            {"type": "avg_watch_time", "position": {"row": 0, "col": 4, "width": 4}},
            {"type": "top_clips", "position": {"row": 1, "col": 0, "width": 8}}
        ]
    }
)
```

---

## Responsibilities

**SavedViewStore is responsible for:**
- ✅ Persisting view configurations to database
- ✅ Validating view configurations (max widgets, no overlaps)
- ✅ Enforcing max views limit (50 per user)
- ✅ Access control (user can only modify own views)
- ✅ Preventing deletion of system views
- ✅ Managing default view per user

---

## NOT Responsibilities

**SavedViewStore is NOT responsible for:**
- ❌ Calculating dashboard metrics (DashboardEngine's job)
- ❌ Rendering dashboard UI (DashboardGrid's job)
- ❌ Querying transactions (TransactionQuery's job)
- ❌ Exporting dashboards (SnapshotExporter's job)
- ❌ Validating widget-specific config (widget's job)

---

## Implementation Notes

**Storage:**
```sql
CREATE TABLE saved_views (
    view_id VARCHAR(100) PRIMARY KEY,
    user_id VARCHAR(100),  -- NULL for system views
    name VARCHAR(100) NOT NULL,
    description TEXT,
    is_default BOOLEAN DEFAULT FALSE,
    is_system BOOLEAN DEFAULT FALSE,
    config JSONB NOT NULL,  -- DashboardConfig
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),

    CONSTRAINT unique_user_view_name UNIQUE (user_id, name),
    CONSTRAINT max_user_views CHECK (
        -- Enforced via trigger: user cannot have >50 views
    )
);

CREATE INDEX idx_saved_views_user ON saved_views(user_id);
CREATE INDEX idx_saved_views_is_system ON saved_views(is_system);
```

**View ID generation:**
```python
def generate_view_id(name: str) -> str:
    """
    Generate view_id from name.

    Example: "Q1 2025 Deductibles" → "view_q1_2025_deductibles"
    """
    slug = name.lower().replace(" ", "_").replace("-", "_")
    slug = re.sub(r'[^a-z0-9_]', '', slug)  # Remove special chars
    return f"view_{slug}"
```

**System views (seeded on deployment):**
```python
SYSTEM_VIEWS = [
    {
        "view_id": "view_monthly_overview",
        "name": "Monthly Overview",
        "description": "Current month income, expenses, net",
        "is_default": True,
        "is_system": True,
        "config": {
            "date_range": "current_month",
            "widgets": [...]
        }
    },
    {
        "view_id": "view_quarterly_summary",
        "name": "Quarterly Summary",
        "description": "Current quarter financial overview",
        "is_default": False,
        "is_system": True,
        "config": {
            "date_range": "current_quarter",
            "widgets": [...]
        }
    }
]
```

---

## Related Primitives

**Dependencies:**
- None (standalone primitive)

**Used by:**
- DashboardEngine (loads view config to determine which metrics to calculate)
- SnapshotExporter (includes view config in PDF export)

**Reuses:**
- None (primitive-level storage)
