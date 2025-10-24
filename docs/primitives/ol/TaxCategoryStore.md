# TaxCategoryStore (OL Primitive)

**Vertical:** 3.4 Tax Categorization (Regulatory Classification)
**Layer:** Objective Layer (OL)
**Type:** Hybrid Registry Store (System + User Categories)
**Status:** ✅ Specified

---

## Purpose

CRUD operations for tax categories with support for both system-defined categories (e.g., Schedule C categories) and user-created custom categories. Provides hierarchical category management, uniqueness enforcement per jurisdiction, and parent-child relationship tracking.

**Core Responsibility:** Persist and manage tax categories as a hybrid registry: fixed system categories (read-only, shipped with application) + user-defined custom categories (fully editable). Enforce unique category names per jurisdiction (case-insensitive), support hierarchical parent-child relationships, and enable soft delete pattern.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY regulatory classification system with hierarchical taxonomies:

| Domain | Entity Name | Examples | Hierarchical Structure |
|--------|-------------|----------|------------------------|
| **Finance** | Tax Category | USA Schedule C (Advertising, Office Expenses), Mexico SAT (Gastos de Gestión) | Parent: "Expenses" → Child: "Office Expenses" → Grandchild: "Software Subscriptions" |
| **Healthcare** | Procedure Code | CPT codes (99213 Office Visit), ICD-10 diagnosis codes (E11 Type 2 Diabetes) | Parent: "Outpatient Services" → Child: "Office Visits" → Grandchild: "Established Patient" |
| **Legal** | Case Category | Civil litigation categories, jurisdiction-specific filing types | Parent: "Civil" → Child: "Contract Disputes" → Grandchild: "Breach of Contract" |
| **Research** | Grant Category | NSF expense categories, NIH allowable costs | Parent: "Direct Costs" → Child: "Personnel" → Grandchild: "Graduate Students" |
| **Manufacturing** | Safety Category | OSHA safety categories, EPA environmental compliance | Parent: "Workplace Safety" → Child: "Personal Protective Equipment" → Grandchild: "Respiratory Protection" |

**Pattern:** Hybrid Registry = system provides base taxonomy (read-only) + users extend with custom categories (within same hierarchy), enforce unique names per jurisdiction, support parent-child trees, allow archive (soft delete for custom only).

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Set
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal

@dataclass
class TaxCategory:
    category_id: str
    user_id: Optional[str]  # None for system categories, user_id for custom
    name: str
    jurisdiction: str  # usa_federal, mexico_federal, etc.
    taxonomy: str  # schedule_c, sat, cpt, icd10, etc.
    code: Optional[str]  # Official code (e.g., "line_8" for Schedule C, "84111506" for SAT)
    parent_id: Optional[str]  # For hierarchical categories
    deduction_rate: Decimal  # Finance: 1.0 (100%), 0.5 (50%), 0.0 (0%)
                            # Healthcare: reimbursement_rate
                            # Research: cost_share_rate
    is_system: bool  # True = read-only system category, False = user custom
    description: Optional[str]
    is_active: bool  # For custom categories only (system categories always active)
    created_at: datetime
    updated_at: datetime

@dataclass
class CategoryHierarchy:
    """Represents category tree structure."""
    category: TaxCategory
    children: List['CategoryHierarchy']  # Recursive structure
    depth: int  # 0 = root, 1 = child, 2 = grandchild, etc.
    path: List[str]  # Full path from root: ["Expenses", "Office", "Software"]

class TaxCategoryStore:
    """
    Manages CRUD operations for tax categories (system + custom).

    Enforces:
    - Unique names per (user_id, jurisdiction) tuple (case-insensitive)
    - System categories are read-only (cannot edit/delete)
    - Custom categories can have same name as system category in different jurisdiction
    - Parent must exist and be in same jurisdiction
    - Deduction rate must be in range [0.0, 1.0]
    - Soft delete for custom categories only
    """

    def __init__(self, db_connection):
        self.db = db_connection

    def create_custom(
        self,
        user_id: str,
        jurisdiction: str,
        name: str,
        parent_id: Optional[str],
        deduction_rate: Decimal,
        code: Optional[str] = None,
        description: Optional[str] = None
    ) -> TaxCategory:
        """
        Create custom tax category for user.

        Custom categories allow users to extend system taxonomy with domain-specific
        subcategories (e.g., "Social Media Advertising" under "Advertising").

        Args:
            user_id: Owner of the custom category
            jurisdiction: Regulatory framework (usa_federal, mexico_federal, etc.)
            name: Display name (e.g., "Social Media Advertising")
            parent_id: Optional parent category (must be in same jurisdiction)
            deduction_rate: Eligibility percentage [0.0, 1.0]
                           Finance: 1.0 = 100% deductible, 0.5 = 50%, 0.0 = non-deductible
                           Healthcare: 1.0 = 100% reimbursable, 0.8 = 80%, etc.
                           Research: 1.0 = 100% grant-funded, 0.5 = 50% cost-share, etc.
            code: Optional official code (e.g., SAT product/service code)
            description: Optional explanation (max 500 chars)

        Returns:
            Created TaxCategory object

        Raises:
            DuplicateCategoryNameError: If name exists for this user in this jurisdiction (case-insensitive)
            InvalidJurisdictionError: If jurisdiction not supported
            InvalidParentError: If parent_id doesn't exist or is in different jurisdiction
            InvalidDeductionRateError: If rate not in [0.0, 1.0]
            ValidationError: If name too long, empty, or contains invalid characters

        Example:
            # User creates custom category for social media ads under Advertising
            category = store.create_custom(
                user_id="user_darwin",
                jurisdiction="usa_federal",
                name="Social Media Advertising",
                parent_id="tax_cat_usa_schedule_c_advertising",  # System category
                deduction_rate=Decimal("1.0"),
                description="Facebook, Instagram, Twitter ads for business"
            )
        """

    def get(self, category_id: str, user_id: Optional[str] = None) -> Optional[TaxCategory]:
        """
        Get tax category by ID.

        Works for both system categories (user_id=None) and custom categories.

        Args:
            category_id: Category ID to fetch
            user_id: Owner (required only for custom categories)

        Returns:
            TaxCategory if found, None otherwise

        Raises:
            UnauthorizedError: If custom category exists but user doesn't own it
        """

    def get_hierarchy(
        self,
        jurisdiction: str,
        user_id: Optional[str] = None,
        parent_id: Optional[str] = None,
        include_system: bool = True,
        include_custom: bool = True
    ) -> List[CategoryHierarchy]:
        """
        Get category tree for jurisdiction (depth-first traversal).

        Returns full hierarchical structure with parent-child relationships.
        Useful for rendering tree views or category selectors.

        Args:
            jurisdiction: Regulatory framework (usa_federal, mexico_federal, etc.)
            user_id: Owner (required if include_custom=True)
            parent_id: Optional root parent (None = all root categories)
            include_system: Include system categories (default: True)
            include_custom: Include user's custom categories (default: True)

        Returns:
            List of CategoryHierarchy objects representing tree structure

        Example:
            # Get full Schedule C hierarchy for user
            hierarchy = store.get_hierarchy(
                jurisdiction="usa_federal",
                user_id="user_darwin"
            )
            # Returns:
            # - Advertising [system]
            #   - Social Media Advertising [custom]
            #   - Print Advertising [custom]
            # - Office Expenses [system]
            #   - Software Subscriptions [custom]
            # - Travel [system]
        """

    def search(
        self,
        jurisdiction: str,
        query: str,
        user_id: Optional[str] = None,
        include_system: bool = True,
        include_custom: bool = True
    ) -> List[TaxCategory]:
        """
        Full-text search across category names and descriptions.

        Args:
            jurisdiction: Regulatory framework to search within
            query: Search query (matches name and description)
            user_id: Owner (required if include_custom=True)
            include_system: Search system categories (default: True)
            include_custom: Search user's custom categories (default: True)

        Returns:
            List of matching TaxCategory objects (sorted by relevance, then name)

        Example:
            # User searches "advertising" in USA categories
            results = store.search(
                jurisdiction="usa_federal",
                query="advertising",
                user_id="user_darwin"
            )
            # Returns:
            # - Advertising [system]
            # - Social Media Advertising [custom]
            # - Print Advertising [custom]
        """

    def list_by_jurisdiction(
        self,
        jurisdiction: str,
        user_id: Optional[str] = None,
        is_active: Optional[bool] = True,
        include_system: bool = True,
        include_custom: bool = True
    ) -> List[TaxCategory]:
        """
        List all categories for jurisdiction (flat list, not hierarchical).

        Args:
            jurisdiction: Regulatory framework
            user_id: Owner (required if include_custom=True)
            is_active: Filter by active status (applies to custom categories only)
            include_system: Include system categories (default: True)
            include_custom: Include user's custom categories (default: True)

        Returns:
            List of TaxCategory objects (sorted by name ASC)
        """

    def update_custom(
        self,
        category_id: str,
        user_id: str,
        updates: dict
    ) -> TaxCategory:
        """
        Update custom category details.

        SYSTEM CATEGORIES (is_system=True):
        - Cannot be updated by users (read-only)
        - Only admins can update via system migration

        CUSTOM CATEGORIES (is_system=False):
        - Can update: name, deduction_rate, code, description
        - Cannot update: jurisdiction, parent_id (would break hierarchy)

        Args:
            category_id: Category to update
            user_id: Owner (for authorization)
            updates: Dictionary of fields to update

        Returns:
            Updated TaxCategory object

        Raises:
            CategoryNotFoundError: If category doesn't exist
            UnauthorizedError: If user doesn't own category
            SystemCategoryError: If trying to update system category
            DuplicateCategoryNameError: If new name conflicts with existing
            ImmutableFieldError: If trying to update jurisdiction or parent_id
            ValidationError: If updated values invalid

        Example:
            # User updates deduction rate after IRS rule change
            category = store.update_custom(
                category_id="tax_cat_custom_social_ads_1",
                user_id="user_darwin",
                updates={
                    "deduction_rate": Decimal("0.5"),  # Changed to 50%
                    "description": "Now only 50% deductible per new IRS rules"
                }
            )
        """

    def archive_custom(
        self,
        category_id: str,
        user_id: str
    ) -> TaxCategory:
        """
        Archive custom category (soft delete - set is_active = false).

        System categories cannot be archived (use system migration instead).
        Custom category remains in database to preserve historical classifications.

        Before archiving, checks if any transactions reference this category:
        - If yes: Warns user, archives anyway (historical classifications remain valid)
        - If no: Archives immediately

        Args:
            category_id: Category to archive
            user_id: Owner (for authorization)

        Returns:
            Archived TaxCategory object

        Raises:
            CategoryNotFoundError: If category doesn't exist
            UnauthorizedError: If user doesn't own category
            SystemCategoryError: If trying to archive system category

        Example:
            # User no longer needs custom category
            category = store.archive_custom(
                category_id="tax_cat_custom_old_category_1",
                user_id="user_darwin"
            )
        """

    def unarchive_custom(
        self,
        category_id: str,
        user_id: str
    ) -> TaxCategory:
        """
        Unarchive custom category (set is_active = true).

        Args:
            category_id: Category to unarchive
            user_id: Owner (for authorization)

        Returns:
            Unarchived TaxCategory object

        Raises:
            CategoryNotFoundError: If category doesn't exist
            UnauthorizedError: If user doesn't own category
        """

    def _generate_category_id(self, name: str, is_system: bool, user_id: Optional[str] = None) -> str:
        """
        Generate deterministic category_id.

        Format:
        - System: tax_cat_{jurisdiction}_{taxonomy}_{slug}
        - Custom: tax_cat_custom_{slug}_{sequence}

        Examples:
        - System: "tax_cat_usa_schedule_c_advertising"
        - Custom: "tax_cat_custom_social_ads_1"

        Args:
            name: Category name
            is_system: True for system category, False for custom
            user_id: Required for custom categories

        Returns:
            category_id string
        """

    def _validate_jurisdiction(self, jurisdiction: str) -> None:
        """
        Validate jurisdiction is supported.

        Supported jurisdictions:
        - Finance: usa_federal, usa_state_{code}, mexico_federal
        - Healthcare: hipaa, cms, icd10, cpt
        - Legal: jurisdiction_{state_code}, federal
        - Research: nsf, nih, doe

        Raises:
            InvalidJurisdictionError
        """

    def _validate_deduction_rate(self, rate: Decimal) -> None:
        """
        Validate deduction rate is in valid range [0.0, 1.0].

        Raises:
            InvalidDeductionRateError
        """
```

---

## Implementation Details

### Database Schema

```sql
CREATE TABLE tax_categories (
    category_id VARCHAR(100) PRIMARY KEY,
    user_id VARCHAR(50),                      -- NULL for system categories, user_id for custom

    -- Core fields
    name VARCHAR(200) NOT NULL,
    jurisdiction VARCHAR(50) NOT NULL,        -- usa_federal, mexico_federal, etc.
    taxonomy VARCHAR(50) NOT NULL,            -- schedule_c, sat, cpt, icd10, etc.
    code VARCHAR(50),                         -- Official code (e.g., "line_8", "84111506")
    parent_id VARCHAR(100),                   -- For hierarchical categories
    deduction_rate DECIMAL(3,2) NOT NULL CHECK (deduction_rate >= 0 AND deduction_rate <= 1),
    is_system BOOLEAN DEFAULT FALSE,
    description TEXT,

    -- Metadata
    is_active BOOLEAN DEFAULT TRUE,           -- For custom categories only
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    -- Constraints
    UNIQUE(user_id, jurisdiction, LOWER(name)),  -- Unique name per user per jurisdiction
    INDEX idx_jurisdiction (jurisdiction, is_system),
    INDEX idx_user_jurisdiction (user_id, jurisdiction, is_active),
    INDEX idx_parent (parent_id),
    INDEX idx_taxonomy (taxonomy),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_id) REFERENCES tax_categories(category_id)
);

-- Change log for audit trail (custom categories only)
CREATE TABLE tax_category_change_log (
    log_id VARCHAR(50) PRIMARY KEY,
    category_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(50) NOT NULL,

    operation VARCHAR(20) NOT NULL,           -- CREATE, UPDATE, ARCHIVE, UNARCHIVE
    changes JSONB,                            -- {field: {old: X, new: Y}}
    timestamp TIMESTAMP NOT NULL,

    FOREIGN KEY (category_id) REFERENCES tax_categories(category_id),
    INDEX idx_category (category_id),
    INDEX idx_user (user_id)
);
```

### Category ID Generation

```python
def _generate_category_id(
    self,
    name: str,
    is_system: bool,
    user_id: Optional[str] = None,
    jurisdiction: Optional[str] = None,
    taxonomy: Optional[str] = None
) -> str:
    """
    Generate category_id based on type.

    System categories (is_system=True):
      Format: tax_cat_{jurisdiction}_{taxonomy}_{slug}
      Examples:
        "Advertising" → "tax_cat_usa_schedule_c_advertising"
        "Gastos de Gestión" → "tax_cat_mx_sat_gastos_de_gestion"
        "Office Visit" → "tax_cat_cpt_office_visit"

    Custom categories (is_system=False):
      Format: tax_cat_custom_{slug}_{sequence}
      Examples:
        "Social Media Advertising" → "tax_cat_custom_social_media_advertising_1"
        "Software Subscriptions" → "tax_cat_custom_software_subscriptions_1"
    """
    # Create slug
    slug = name.lower()
    slug = slug.replace(' ', '_')
    slug = re.sub(r'[^a-z0-9_]', '', slug)  # Remove special chars
    slug = slug[:50]  # Limit slug length

    if is_system:
        # System category: include jurisdiction and taxonomy
        if not jurisdiction or not taxonomy:
            raise ValueError("jurisdiction and taxonomy required for system categories")

        category_id = f"tax_cat_{jurisdiction}_{taxonomy}_{slug}"
        return category_id
    else:
        # Custom category: use sequence
        existing = self.db.execute(
            "SELECT category_id FROM tax_categories WHERE category_id LIKE ? ORDER BY category_id DESC LIMIT 1",
            (f"tax_cat_custom_{slug}_%",)
        ).fetchone()

        if existing:
            # Extract sequence from "tax_cat_custom_social_ads_3" → 3
            last_seq = int(existing['category_id'].split('_')[-1])
            seq = last_seq + 1
        else:
            seq = 1

        return f"tax_cat_custom_{slug}_{seq}"
```

### Uniqueness Validation (Case-Insensitive)

```python
def _check_duplicate_name(
    self,
    user_id: Optional[str],
    jurisdiction: str,
    name: str,
    exclude_category_id: Optional[str] = None
) -> None:
    """
    Raise DuplicateCategoryNameError if name already exists (case-insensitive).

    Rules:
    - System categories (user_id=None): Name unique per jurisdiction
    - Custom categories (user_id=X): Name unique per (user_id, jurisdiction) tuple

    Example:
      System "Advertising" exists in usa_federal
      User can create custom "advertising" in usa_federal (different user_id)
      But user cannot create two custom "advertising" in same jurisdiction

    Args:
        user_id: Owner (None for system categories)
        jurisdiction: Regulatory framework
        name: Name to check
        exclude_category_id: Exclude this category from check (for updates)

    Raises:
        DuplicateCategoryNameError
    """
    query = """
        SELECT category_id FROM tax_categories
        WHERE user_id <=> ? AND jurisdiction = ? AND LOWER(name) = LOWER(?) AND is_active = true
    """
    params = [user_id, jurisdiction, name]

    if exclude_category_id:
        query += " AND category_id != ?"
        params.append(exclude_category_id)

    existing = self.db.execute(query, params).fetchone()

    if existing:
        raise DuplicateCategoryNameError(
            f"Category with name '{name}' already exists in jurisdiction '{jurisdiction}'",
            field="name",
            existing_category_id=existing['category_id']
        )
```

### Hierarchy Traversal (Depth-First)

```python
def get_hierarchy(
    self,
    jurisdiction: str,
    user_id: Optional[str] = None,
    parent_id: Optional[str] = None,
    include_system: bool = True,
    include_custom: bool = True
) -> List[CategoryHierarchy]:
    """
    Build category tree using depth-first traversal.

    Algorithm:
    1. Fetch root categories (parent_id = provided parent or NULL)
    2. For each root, recursively fetch children
    3. Build CategoryHierarchy objects with depth and path

    Returns tree structure:
    [
      CategoryHierarchy(
        category=Advertising,
        depth=0,
        path=["Advertising"],
        children=[
          CategoryHierarchy(
            category=Social Media Advertising,
            depth=1,
            path=["Advertising", "Social Media Advertising"],
            children=[]
          )
        ]
      )
    ]
    """
    def build_tree(parent_id: Optional[str], depth: int, path: List[str]) -> List[CategoryHierarchy]:
        # Fetch categories at this level
        query = """
            SELECT * FROM tax_categories
            WHERE jurisdiction = ? AND parent_id <=> ? AND is_active = true
        """
        params = [jurisdiction, parent_id]

        # Filter by system/custom
        if include_system and not include_custom:
            query += " AND is_system = true"
        elif include_custom and not include_system:
            query += " AND is_system = false"
            if user_id:
                query += " AND user_id = ?"
                params.append(user_id)
        elif include_custom and user_id:
            # Include both, but filter custom by user
            query += " AND (is_system = true OR user_id = ?)"
            params.append(user_id)

        query += " ORDER BY name ASC"

        categories = self.db.execute(query, params).fetchall()

        result = []
        for cat_row in categories:
            category = self._row_to_category(cat_row)
            current_path = path + [category.name]

            # Recursively fetch children
            children = build_tree(category.category_id, depth + 1, current_path)

            hierarchy = CategoryHierarchy(
                category=category,
                children=children,
                depth=depth,
                path=current_path
            )
            result.append(hierarchy)

        return result

    return build_tree(parent_id, 0, [])
```

### Parent Validation

```python
def _validate_parent(
    self,
    parent_id: Optional[str],
    jurisdiction: str
) -> None:
    """
    Validate parent category exists and is in same jurisdiction.

    Raises:
        InvalidParentError: If parent doesn't exist or jurisdiction mismatch
    """
    if parent_id is None:
        return  # Root category, no parent required

    parent = self.db.execute(
        "SELECT * FROM tax_categories WHERE category_id = ?",
        (parent_id,)
    ).fetchone()

    if not parent:
        raise InvalidParentError(
            f"Parent category '{parent_id}' not found",
            parent_id=parent_id
        )

    if parent['jurisdiction'] != jurisdiction:
        raise InvalidParentError(
            f"Parent category is in jurisdiction '{parent['jurisdiction']}', expected '{jurisdiction}'",
            parent_id=parent_id
        )
```

---

## Usage Examples

### Example 1: Create System Category (Admin Only)

```python
store = TaxCategoryStore(db_connection)

# System admin seeds Schedule C categories
advertising = store.create_custom(  # Internal method for system
    user_id=None,  # System category
    jurisdiction="usa_federal",
    name="Advertising",
    parent_id=None,  # Root category
    deduction_rate=Decimal("1.0"),
    code="line_8",
    description="Advertising and marketing expenses (Schedule C Line 8)"
)
# Override is_system flag (internal only)
advertising.is_system = True

print(advertising.category_id)  # "tax_cat_usa_schedule_c_advertising"
print(advertising.is_system)  # True
```

### Example 2: Create Custom Category (User)

```python
# User creates custom category under system "Advertising"
social_ads = store.create_custom(
    user_id="user_darwin",
    jurisdiction="usa_federal",
    name="Social Media Advertising",
    parent_id="tax_cat_usa_schedule_c_advertising",  # System category
    deduction_rate=Decimal("1.0"),
    description="Facebook, Instagram, Twitter ads for business promotion"
)

print(social_ads.category_id)  # "tax_cat_custom_social_media_advertising_1"
print(social_ads.is_system)  # False
print(social_ads.parent_id)  # "tax_cat_usa_schedule_c_advertising"
```

### Example 3: Get Category Hierarchy

```python
# Get full Schedule C hierarchy for user
hierarchy = store.get_hierarchy(
    jurisdiction="usa_federal",
    user_id="user_darwin",
    include_system=True,
    include_custom=True
)

for h in hierarchy:
    print(f"{'  ' * h.depth}{h.category.name} ({'system' if h.category.is_system else 'custom'})")
    for child in h.children:
        print(f"{'  ' * child.depth}{child.category.name} ({'system' if child.category.is_system else 'custom'})")

# Output:
# Advertising (system)
#   Social Media Advertising (custom)
#   Print Advertising (custom)
# Office Expenses (system)
#   Software Subscriptions (custom)
# Travel (system)
```

### Example 4: Search Categories

```python
# User searches for "advertising" categories
results = store.search(
    jurisdiction="usa_federal",
    query="advertising",
    user_id="user_darwin"
)

for cat in results:
    print(f"{cat.name} - {cat.description}")

# Output:
# Advertising - Advertising and marketing expenses (Schedule C Line 8)
# Social Media Advertising - Facebook, Instagram, Twitter ads
# Print Advertising - Newspaper, magazine, print media ads
```

### Example 5: Update Custom Category

```python
# User updates deduction rate after IRS rule change
updated = store.update_custom(
    category_id="tax_cat_custom_social_media_advertising_1",
    user_id="user_darwin",
    updates={
        "deduction_rate": Decimal("0.5"),  # Now only 50% deductible
        "description": "Social media ads - now 50% deductible per new IRS rules"
    }
)

print(updated.deduction_rate)  # Decimal("0.5")
```

### Example 6: Archive Custom Category

```python
# User archives old custom category
archived = store.archive_custom(
    category_id="tax_cat_custom_old_category_1",
    user_id="user_darwin"
)

print(archived.is_active)  # False
```

### Example 7: Handle Duplicate Name Error

```python
try:
    store.create_custom(
        user_id="user_darwin",
        jurisdiction="usa_federal",
        name="Social Media Advertising",  # Already exists
        parent_id="tax_cat_usa_schedule_c_advertising",
        deduction_rate=Decimal("1.0")
    )
except DuplicateCategoryNameError as e:
    print(e.message)  # "Category with name 'Social Media Advertising' already exists..."
    print(e.existing_category_id)  # "tax_cat_custom_social_media_advertising_1"
```

---

## Multi-Domain Examples

### Healthcare: CPT Code Categories

```python
# System CPT code category
office_visit = store.create_custom(
    user_id=None,  # System
    jurisdiction="cpt",
    name="Office Visits - Established Patient",
    parent_id="tax_cat_cpt_outpatient_services",
    deduction_rate=Decimal("1.0"),  # 100% reimbursable
    code="99213",
    description="Office visit, established patient, moderate complexity"
)

# User creates custom subcategory
annual_checkup = store.create_custom(
    user_id="doctor_123",
    jurisdiction="cpt",
    name="Annual Checkup - Routine",
    parent_id="tax_cat_cpt_office_visits_established",
    deduction_rate=Decimal("0.8"),  # 80% reimbursable
    description="Routine annual physical exam for established patients"
)
```

### Legal: Case Categories

```python
# System legal category
civil_litigation = store.create_custom(
    user_id=None,  # System
    jurisdiction="federal",
    name="Civil Litigation",
    parent_id=None,
    deduction_rate=Decimal("1.0"),  # 100% billable
    description="Civil litigation cases in federal court"
)

# Law firm creates custom subcategory
contract_disputes = store.create_custom(
    user_id="lawyer_456",
    jurisdiction="federal",
    name="Contract Disputes - Technology",
    parent_id="tax_cat_federal_civil_litigation",
    deduction_rate=Decimal("1.0"),
    description="Contract disputes involving software and tech licensing"
)
```

### Research: Grant Categories

```python
# System NSF category
personnel = store.create_custom(
    user_id=None,  # System
    jurisdiction="nsf",
    name="Personnel Costs",
    parent_id="tax_cat_nsf_direct_costs",
    deduction_rate=Decimal("1.0"),  # 100% grant-funded
    description="Salaries and wages for project personnel"
)

# Researcher creates custom subcategory
grad_students = store.create_custom(
    user_id="researcher_789",
    jurisdiction="nsf",
    name="Graduate Student Researchers",
    parent_id="tax_cat_nsf_personnel_costs",
    deduction_rate=Decimal("0.5"),  # 50% grant-funded, 50% institution cost-share
    description="Ph.D. students working on NSF-funded research projects"
)
```

---

## Error Handling

### Custom Exceptions

```python
class TaxCategoryStoreError(Exception):
    """Base exception for TaxCategoryStore errors."""
    pass

class DuplicateCategoryNameError(TaxCategoryStoreError):
    def __init__(self, message: str, field: str = "name", existing_category_id: str = None):
        self.message = message
        self.field = field
        self.existing_category_id = existing_category_id
        super().__init__(message)

class InvalidJurisdictionError(TaxCategoryStoreError):
    def __init__(self, message: str, jurisdiction: str):
        self.message = message
        self.jurisdiction = jurisdiction
        super().__init__(message)

class InvalidParentError(TaxCategoryStoreError):
    def __init__(self, message: str, parent_id: str):
        self.message = message
        self.parent_id = parent_id
        super().__init__(message)

class InvalidDeductionRateError(TaxCategoryStoreError):
    def __init__(self, message: str, rate: Decimal):
        self.message = message
        self.rate = rate
        super().__init__(message)

class CategoryNotFoundError(TaxCategoryStoreError):
    pass

class UnauthorizedError(TaxCategoryStoreError):
    pass

class SystemCategoryError(TaxCategoryStoreError):
    """Raised when attempting to modify system categories."""
    def __init__(self, message: str = "Cannot modify system categories"):
        self.message = message
        super().__init__(message)

class ImmutableFieldError(TaxCategoryStoreError):
    def __init__(self, message: str, fields: List[str]):
        self.message = message
        self.fields = fields
        super().__init__(message)

class ValidationError(TaxCategoryStoreError):
    pass
```

### Error Response Mapping (HTTP)

```python
ERROR_STATUS_CODES = {
    DuplicateCategoryNameError: 409,  # Conflict
    InvalidJurisdictionError: 400,    # Bad Request
    InvalidParentError: 400,          # Bad Request
    InvalidDeductionRateError: 400,   # Bad Request
    ValidationError: 400,             # Bad Request
    CategoryNotFoundError: 404,       # Not Found
    UnauthorizedError: 403,           # Forbidden
    SystemCategoryError: 403,         # Forbidden
    ImmutableFieldError: 400          # Bad Request
}
```

---

## Performance Characteristics

### Query Performance

| Operation | Query Type | Expected Latency (p95) | Index Used |
|-----------|------------|------------------------|------------|
| `create_custom()` | INSERT | <30ms | N/A |
| `get()` | SELECT by PK | <10ms | PRIMARY KEY |
| `get_hierarchy()` | Recursive SELECT | <100ms | idx_parent, idx_jurisdiction |
| `search()` | Full-text search | <50ms | Full-text index on name, description |
| `update_custom()` | UPDATE by PK | <20ms | PRIMARY KEY |
| `archive_custom()` | UPDATE | <20ms | PRIMARY KEY |

### Caching Strategy

```python
# Cache system categories per jurisdiction (rarely change)
cache_key = f"tax_categories:system:{jurisdiction}"
ttl = 86400  # 24 hours

def list_system_cached(jurisdiction: str) -> List[TaxCategory]:
    """Get system categories from cache or database."""
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    categories = self.list_by_jurisdiction(
        jurisdiction=jurisdiction,
        include_system=True,
        include_custom=False
    )
    redis.setex(cache_key, ttl, json.dumps([c.__dict__ for c in categories]))
    return categories

# Cache user's custom categories (change more frequently)
user_cache_key = f"tax_categories:custom:{user_id}:{jurisdiction}"
user_ttl = 300  # 5 minutes

def _invalidate_user_cache(user_id: str, jurisdiction: str):
    """Invalidate cache on create/update/archive."""
    redis.delete(f"tax_categories:custom:{user_id}:{jurisdiction}")
```

---

## Audit Trail

Every custom category operation is logged to `tax_category_change_log` table:

```python
def _log_change(
    self,
    category_id: str,
    user_id: str,
    operation: str,  # CREATE, UPDATE, ARCHIVE, UNARCHIVE
    changes: dict    # {field: {old: X, new: Y}}
):
    """Log category change for audit trail (custom categories only)."""
    log_id = f"tccl_{int(time.time() * 1000)}"

    self.db.execute(
        """
        INSERT INTO tax_category_change_log (log_id, category_id, user_id, operation, changes, timestamp)
        VALUES (?, ?, ?, ?, ?, ?)
        """,
        (log_id, category_id, user_id, operation, json.dumps(changes), datetime.utcnow())
    )
```

**Example Log Entries:**

```json
// CREATE operation
{
  "log_id": "tccl_1234567890",
  "category_id": "tax_cat_custom_social_media_advertising_1",
  "user_id": "user_darwin",
  "operation": "CREATE",
  "changes": {
    "name": {"old": null, "new": "Social Media Advertising"},
    "parent_id": {"old": null, "new": "tax_cat_usa_schedule_c_advertising"},
    "deduction_rate": {"old": null, "new": 1.0}
  },
  "timestamp": "2025-05-15T10:00:00Z"
}

// UPDATE operation
{
  "log_id": "tccl_1234567891",
  "category_id": "tax_cat_custom_social_media_advertising_1",
  "user_id": "user_darwin",
  "operation": "UPDATE",
  "changes": {
    "deduction_rate": {"old": 1.0, "new": 0.5},
    "description": {"old": "Facebook, Instagram ads", "new": "Social media ads - now 50% deductible"}
  },
  "timestamp": "2025-06-01T14:30:00Z"
}

// ARCHIVE operation
{
  "log_id": "tccl_1234567892",
  "category_id": "tax_cat_custom_old_category_1",
  "user_id": "user_darwin",
  "operation": "ARCHIVE",
  "changes": {
    "is_active": {"old": true, "new": false}
  },
  "timestamp": "2025-12-31T09:00:00Z"
}
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Create custom category
def test_create_custom_category_success()
def test_create_custom_category_duplicate_name()
def test_create_custom_category_invalid_jurisdiction()
def test_create_custom_category_invalid_parent()
def test_create_custom_category_invalid_deduction_rate()
def test_create_custom_category_with_parent()

# Test: Get category
def test_get_system_category()
def test_get_custom_category()
def test_get_category_not_found()
def test_get_custom_category_unauthorized()

# Test: Hierarchy
def test_get_hierarchy_root()
def test_get_hierarchy_with_parent()
def test_get_hierarchy_system_only()
def test_get_hierarchy_custom_only()
def test_get_hierarchy_depth_calculation()

# Test: Search
def test_search_by_name()
def test_search_by_description()
def test_search_case_insensitive()
def test_search_system_only()
def test_search_custom_only()

# Test: Update custom category
def test_update_custom_category_name()
def test_update_custom_category_deduction_rate()
def test_update_system_category_error()
def test_update_immutable_jurisdiction_error()
def test_update_immutable_parent_error()
def test_update_duplicate_name_error()

# Test: Archive custom category
def test_archive_custom_category()
def test_archive_system_category_error()
def test_unarchive_custom_category()

# Test: Validation
def test_validate_jurisdiction_valid()
def test_validate_jurisdiction_invalid()
def test_validate_deduction_rate_valid()
def test_validate_deduction_rate_negative()
def test_validate_deduction_rate_over_one()
def test_validate_parent_exists()
def test_validate_parent_same_jurisdiction()
```

---

## Related Primitives

- **TaxonomyEngine** (OL) - Navigates hierarchical category trees, calculates paths
- **TaxCategoryClassifier** (OL) - Auto-suggests categories based on transaction patterns
- **TaxCategorySelector** (IL) - UI component for selecting tax category
- **TaxCategoryManager** (IL) - UI component for managing custom categories
- **TransactionStore** (from Vertical 1.3) - Transactions reference tax_classification

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 3-4 days
**Dependencies:** Database (PostgreSQL/MySQL), Validation library
