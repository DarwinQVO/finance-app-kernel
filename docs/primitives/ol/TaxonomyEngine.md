# TaxonomyEngine (OL Primitive)

**Vertical:** 3.4 Tax Categorization (Regulatory Classification)
**Layer:** Objective Layer (OL)
**Type:** Tree Navigation Engine
**Status:** ✅ Specified

---

## Purpose

Navigate and traverse hierarchical category trees (taxonomies) with support for multiple traversal algorithms, path calculation, and tree operations. Provides efficient tree navigation for regulatory classification systems.

**Core Responsibility:** Load taxonomy data structures from database, provide depth-first and breadth-first traversal, calculate category paths from root to leaf, and support multiple concurrent taxonomies (Schedule C, SAT, CPT, ICD-10, etc.).

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY hierarchical classification system:

| Domain | Taxonomy Name | Tree Structure | Use Case |
|--------|---------------|----------------|----------|
| **Finance** | USA Schedule C, Mexico SAT | Root: Expenses → Office → Software | Navigate tax categories for transaction classification |
| **Healthcare** | CPT codes, ICD-10 diagnoses | Root: Services → Outpatient → Office Visits | Navigate procedure codes for billing |
| **Legal** | Case law categories | Root: Civil → Contract → Breach | Navigate legal categories for case classification |
| **Research** | NSF grant categories | Root: Direct Costs → Personnel → Students | Navigate expense categories for grant compliance |
| **E-commerce** | Product categories | Root: Electronics → Computers → Laptops | Navigate product catalog |
| **Content Management** | Document taxonomy | Root: Articles → News → Technology | Navigate content hierarchies |

**Pattern:** Tree Navigation = load hierarchical data, traverse with DFS/BFS, calculate paths, support multiple trees simultaneously, cache for performance.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Dict, Callable
from dataclasses import dataclass
from enum import Enum

class TraversalOrder(Enum):
    """Tree traversal algorithms."""
    DEPTH_FIRST_PRE = "depth_first_pre"      # Parent before children
    DEPTH_FIRST_POST = "depth_first_post"    # Children before parent
    BREADTH_FIRST = "breadth_first"          # Level by level

@dataclass
class TaxonomyNode:
    """Represents a node in the taxonomy tree."""
    category_id: str
    name: str
    parent_id: Optional[str]
    depth: int  # 0 = root, 1 = child, 2 = grandchild, etc.
    path: List[str]  # Full path from root: ["Expenses", "Office", "Software"]
    path_ids: List[str]  # Category IDs in path
    children: List['TaxonomyNode']  # Child nodes
    metadata: Dict  # Additional data (deduction_rate, code, etc.)

@dataclass
class TaxonomyTree:
    """Represents a complete taxonomy tree."""
    jurisdiction: str
    taxonomy: str
    root_nodes: List[TaxonomyNode]  # Top-level categories
    node_map: Dict[str, TaxonomyNode]  # category_id → node lookup
    max_depth: int  # Maximum depth of tree

class TaxonomyEngine:
    """
    Navigate hierarchical category trees (taxonomies).

    Provides:
    - Multiple traversal algorithms (DFS, BFS)
    - Path calculation (root to leaf)
    - Tree search and filtering
    - Multi-taxonomy support
    - In-memory caching for performance
    """

    def __init__(self, category_store):
        self.category_store = category_store
        self._cache: Dict[str, TaxonomyTree] = {}

    def load_taxonomy(
        self,
        jurisdiction: str,
        user_id: Optional[str] = None,
        include_system: bool = True,
        include_custom: bool = True,
        force_reload: bool = False
    ) -> TaxonomyTree:
        """
        Load taxonomy tree into memory.

        Fetches all categories for jurisdiction, builds tree structure,
        and caches for subsequent operations.

        Args:
            jurisdiction: Regulatory framework (usa_federal, mexico_federal, etc.)
            user_id: Owner (required if include_custom=True)
            include_system: Include system categories (default: True)
            include_custom: Include user's custom categories (default: True)
            force_reload: Force reload from database (ignore cache)

        Returns:
            TaxonomyTree with root nodes and node lookup map

        Example:
            # Load Schedule C taxonomy
            tree = engine.load_taxonomy(
                jurisdiction="usa_federal",
                user_id="user_darwin"
            )
            print(f"Loaded {len(tree.node_map)} categories")
            print(f"Max depth: {tree.max_depth}")
        """

    def get_children(
        self,
        taxonomy_tree: TaxonomyTree,
        parent_id: Optional[str] = None
    ) -> List[TaxonomyNode]:
        """
        Get immediate children of parent node.

        Args:
            taxonomy_tree: Loaded taxonomy tree
            parent_id: Parent category ID (None = root categories)

        Returns:
            List of TaxonomyNode objects (direct children only, not recursive)

        Example:
            # Get top-level categories
            roots = engine.get_children(tree, parent_id=None)
            # Returns: [Advertising, Office Expenses, Travel, ...]

            # Get children of "Office Expenses"
            office_children = engine.get_children(tree, parent_id="tax_cat_usa_schedule_c_office")
            # Returns: [Software Subscriptions, Supplies, ...]
        """

    def get_path(
        self,
        taxonomy_tree: TaxonomyTree,
        category_id: str
    ) -> List[str]:
        """
        Get full path from root to category (category names).

        Args:
            taxonomy_tree: Loaded taxonomy tree
            category_id: Target category ID

        Returns:
            List of category names from root to target

        Raises:
            CategoryNotFoundError: If category_id not in tree

        Example:
            path = engine.get_path(tree, "tax_cat_custom_social_ads_1")
            # Returns: ["Expenses", "Advertising", "Social Media Advertising"]
        """

    def get_path_ids(
        self,
        taxonomy_tree: TaxonomyTree,
        category_id: str
    ) -> List[str]:
        """
        Get full path from root to category (category IDs).

        Args:
            taxonomy_tree: Loaded taxonomy tree
            category_id: Target category ID

        Returns:
            List of category IDs from root to target

        Raises:
            CategoryNotFoundError: If category_id not in tree

        Example:
            path_ids = engine.get_path_ids(tree, "tax_cat_custom_social_ads_1")
            # Returns: [
            #   "tax_cat_usa_schedule_c_expenses",
            #   "tax_cat_usa_schedule_c_advertising",
            #   "tax_cat_custom_social_ads_1"
            # ]
        """

    def traverse(
        self,
        taxonomy_tree: TaxonomyTree,
        order: TraversalOrder = TraversalOrder.DEPTH_FIRST_PRE,
        start_node: Optional[str] = None,
        visitor: Optional[Callable[[TaxonomyNode], None]] = None
    ) -> List[TaxonomyNode]:
        """
        Traverse taxonomy tree in specified order.

        Args:
            taxonomy_tree: Loaded taxonomy tree
            order: Traversal algorithm (DFS pre-order, DFS post-order, BFS)
            start_node: Optional starting node (None = start from all roots)
            visitor: Optional callback function called for each node

        Returns:
            List of TaxonomyNode objects in traversal order

        Example:
            # Depth-first traversal (pre-order)
            nodes = engine.traverse(tree, order=TraversalOrder.DEPTH_FIRST_PRE)
            # Order: Parent → Children → Grandchildren
            # Advertising → Social Media Advertising → Print Advertising

            # Breadth-first traversal
            nodes = engine.traverse(tree, order=TraversalOrder.BREADTH_FIRST)
            # Order: Level 0 → Level 1 → Level 2
            # Advertising, Office, Travel → Social Media, Software → ...

            # Custom visitor function
            def print_category(node):
                print(f"{'  ' * node.depth}{node.name}")

            engine.traverse(tree, visitor=print_category)
        """

    def search_tree(
        self,
        taxonomy_tree: TaxonomyTree,
        predicate: Callable[[TaxonomyNode], bool]
    ) -> List[TaxonomyNode]:
        """
        Search tree for nodes matching predicate.

        Args:
            taxonomy_tree: Loaded taxonomy tree
            predicate: Function that returns True for matching nodes

        Returns:
            List of matching TaxonomyNode objects

        Example:
            # Find all categories with deduction_rate = 1.0
            fully_deductible = engine.search_tree(
                tree,
                lambda node: node.metadata.get('deduction_rate') == 1.0
            )

            # Find all leaf categories (no children)
            leaves = engine.search_tree(
                tree,
                lambda node: len(node.children) == 0
            )

            # Find all categories at depth 2
            level_2 = engine.search_tree(
                tree,
                lambda node: node.depth == 2
            )
        """

    def get_ancestors(
        self,
        taxonomy_tree: TaxonomyTree,
        category_id: str
    ) -> List[TaxonomyNode]:
        """
        Get all ancestor nodes from root to parent.

        Args:
            taxonomy_tree: Loaded taxonomy tree
            category_id: Target category ID

        Returns:
            List of ancestor TaxonomyNode objects (root to parent, excluding target)

        Example:
            ancestors = engine.get_ancestors(tree, "tax_cat_custom_social_ads_1")
            # Returns: [
            #   TaxonomyNode(name="Expenses"),
            #   TaxonomyNode(name="Advertising")
            # ]
        """

    def get_descendants(
        self,
        taxonomy_tree: TaxonomyTree,
        category_id: str,
        max_depth: Optional[int] = None
    ) -> List[TaxonomyNode]:
        """
        Get all descendant nodes (children, grandchildren, etc.).

        Args:
            taxonomy_tree: Loaded taxonomy tree
            category_id: Target category ID
            max_depth: Optional maximum depth to traverse (None = unlimited)

        Returns:
            List of descendant TaxonomyNode objects (all levels below target)

        Example:
            # Get all descendants of "Advertising"
            descendants = engine.get_descendants(tree, "tax_cat_usa_schedule_c_advertising")
            # Returns: [
            #   TaxonomyNode(name="Social Media Advertising"),
            #   TaxonomyNode(name="Print Advertising"),
            #   ... all children and grandchildren ...
            # ]

            # Get only immediate children (max_depth=1)
            children = engine.get_descendants(tree, "tax_cat_usa_schedule_c_advertising", max_depth=1)
        """

    def get_siblings(
        self,
        taxonomy_tree: TaxonomyTree,
        category_id: str
    ) -> List[TaxonomyNode]:
        """
        Get sibling nodes (same parent).

        Args:
            taxonomy_tree: Loaded taxonomy tree
            category_id: Target category ID

        Returns:
            List of sibling TaxonomyNode objects (excluding target)

        Example:
            siblings = engine.get_siblings(tree, "tax_cat_usa_schedule_c_advertising")
            # Returns: [Office Expenses, Travel, Meals, ...] (other root categories)
        """

    def get_leaf_nodes(
        self,
        taxonomy_tree: TaxonomyTree
    ) -> List[TaxonomyNode]:
        """
        Get all leaf nodes (categories with no children).

        Args:
            taxonomy_tree: Loaded taxonomy tree

        Returns:
            List of leaf TaxonomyNode objects

        Example:
            leaves = engine.get_leaf_nodes(tree)
            # Returns all categories with no children
            # Useful for finding most specific categories
        """

    def calculate_depth(
        self,
        taxonomy_tree: TaxonomyTree,
        category_id: str
    ) -> int:
        """
        Calculate depth of category in tree.

        Args:
            taxonomy_tree: Loaded taxonomy tree
            category_id: Target category ID

        Returns:
            Depth (0 = root, 1 = child, 2 = grandchild, etc.)

        Example:
            depth = engine.calculate_depth(tree, "tax_cat_custom_social_ads_1")
            # Returns: 2 (Expenses → Advertising → Social Media Advertising)
        """

    def _build_tree(
        self,
        categories: List[TaxCategory]
    ) -> TaxonomyTree:
        """
        Build tree structure from flat category list.

        Internal method used by load_taxonomy.

        Algorithm:
        1. Create node_map: category_id → TaxonomyNode
        2. Build parent-child relationships
        3. Calculate depths and paths
        4. Identify root nodes (parent_id = None)
        5. Return TaxonomyTree

        Args:
            categories: Flat list of TaxCategory objects

        Returns:
            TaxonomyTree with hierarchical structure
        """
```

---

## Implementation Details

### Tree Building Algorithm

```python
def _build_tree(self, categories: List[TaxCategory]) -> TaxonomyTree:
    """
    Build tree structure from flat category list.

    Algorithm:
    1. Create nodes and map by ID
    2. Build parent-child relationships
    3. Calculate depths (BFS from roots)
    4. Calculate paths (DFS from roots)
    5. Return tree structure

    Time complexity: O(n) where n = number of categories
    Space complexity: O(n) for node_map + children lists
    """
    if not categories:
        return TaxonomyTree(
            jurisdiction="",
            taxonomy="",
            root_nodes=[],
            node_map={},
            max_depth=0
        )

    jurisdiction = categories[0].jurisdiction
    taxonomy = categories[0].taxonomy

    # Step 1: Create nodes and map
    node_map: Dict[str, TaxonomyNode] = {}

    for cat in categories:
        node = TaxonomyNode(
            category_id=cat.category_id,
            name=cat.name,
            parent_id=cat.parent_id,
            depth=0,  # Will calculate later
            path=[],  # Will calculate later
            path_ids=[],  # Will calculate later
            children=[],
            metadata={
                'deduction_rate': cat.deduction_rate,
                'code': cat.code,
                'is_system': cat.is_system,
                'description': cat.description
            }
        )
        node_map[cat.category_id] = node

    # Step 2: Build parent-child relationships
    root_nodes: List[TaxonomyNode] = []

    for node in node_map.values():
        if node.parent_id is None:
            root_nodes.append(node)
        else:
            parent = node_map.get(node.parent_id)
            if parent:
                parent.children.append(node)

    # Step 3: Calculate depths and paths (DFS from roots)
    max_depth = 0

    def calculate_depth_and_path(node: TaxonomyNode, depth: int, path: List[str], path_ids: List[str]):
        nonlocal max_depth

        node.depth = depth
        node.path = path + [node.name]
        node.path_ids = path_ids + [node.category_id]

        max_depth = max(max_depth, depth)

        # Recursively process children
        for child in node.children:
            calculate_depth_and_path(child, depth + 1, node.path, node.path_ids)

    for root in root_nodes:
        calculate_depth_and_path(root, 0, [], [])

    return TaxonomyTree(
        jurisdiction=jurisdiction,
        taxonomy=taxonomy,
        root_nodes=root_nodes,
        node_map=node_map,
        max_depth=max_depth
    )
```

### Depth-First Traversal (Pre-Order)

```python
def _traverse_dfs_pre(
    self,
    node: TaxonomyNode,
    visitor: Optional[Callable[[TaxonomyNode], None]] = None
) -> List[TaxonomyNode]:
    """
    Depth-first traversal (pre-order): Parent before children.

    Algorithm:
    1. Visit parent
    2. Recursively visit children

    Example tree:
        A
       / \
      B   C
     /
    D

    Order: A, B, D, C
    """
    result = []

    def dfs(n: TaxonomyNode):
        # Visit parent first
        if visitor:
            visitor(n)
        result.append(n)

        # Then visit children
        for child in n.children:
            dfs(child)

    dfs(node)
    return result
```

### Depth-First Traversal (Post-Order)

```python
def _traverse_dfs_post(
    self,
    node: TaxonomyNode,
    visitor: Optional[Callable[[TaxonomyNode], None]] = None
) -> List[TaxonomyNode]:
    """
    Depth-first traversal (post-order): Children before parent.

    Algorithm:
    1. Recursively visit children
    2. Visit parent

    Example tree:
        A
       / \
      B   C
     /
    D

    Order: D, B, C, A
    """
    result = []

    def dfs(n: TaxonomyNode):
        # Visit children first
        for child in n.children:
            dfs(child)

        # Then visit parent
        if visitor:
            visitor(n)
        result.append(n)

    dfs(node)
    return result
```

### Breadth-First Traversal

```python
def _traverse_bfs(
    self,
    node: TaxonomyNode,
    visitor: Optional[Callable[[TaxonomyNode], None]] = None
) -> List[TaxonomyNode]:
    """
    Breadth-first traversal: Level by level.

    Algorithm:
    1. Use queue (FIFO)
    2. Visit nodes level by level

    Example tree:
        A
       / \
      B   C
     /
    D

    Order: A, B, C, D
    """
    result = []
    queue = [node]

    while queue:
        current = queue.pop(0)  # Dequeue

        # Visit current node
        if visitor:
            visitor(current)
        result.append(current)

        # Enqueue children
        queue.extend(current.children)

    return result
```

### Path Calculation

```python
def get_path(self, taxonomy_tree: TaxonomyTree, category_id: str) -> List[str]:
    """
    Get full path from root to category (names).

    Uses pre-calculated path from tree building (O(1) lookup).
    """
    node = taxonomy_tree.node_map.get(category_id)

    if not node:
        raise CategoryNotFoundError(f"Category '{category_id}' not found in taxonomy")

    return node.path

def get_path_ids(self, taxonomy_tree: TaxonomyTree, category_id: str) -> List[str]:
    """
    Get full path from root to category (IDs).

    Uses pre-calculated path_ids from tree building (O(1) lookup).
    """
    node = taxonomy_tree.node_map.get(category_id)

    if not node:
        raise CategoryNotFoundError(f"Category '{category_id}' not found in taxonomy")

    return node.path_ids
```

---

## Usage Examples

### Example 1: Load Taxonomy

```python
engine = TaxonomyEngine(category_store)

# Load Schedule C taxonomy
tree = engine.load_taxonomy(
    jurisdiction="usa_federal",
    user_id="user_darwin"
)

print(f"Loaded {len(tree.node_map)} categories")
print(f"Root categories: {len(tree.root_nodes)}")
print(f"Max depth: {tree.max_depth}")

# Output:
# Loaded 42 categories
# Root categories: 8
# Max depth: 3
```

### Example 2: Get Children

```python
# Get top-level categories
roots = engine.get_children(tree, parent_id=None)
for node in roots:
    print(f"{node.name} ({len(node.children)} children)")

# Output:
# Advertising (3 children)
# Office Expenses (5 children)
# Travel (2 children)
# Meals (0 children)
```

### Example 3: Get Path

```python
# Get path to "Social Media Advertising"
path = engine.get_path(tree, "tax_cat_custom_social_ads_1")
print(" → ".join(path))

# Output:
# Expenses → Advertising → Social Media Advertising

# Get path IDs
path_ids = engine.get_path_ids(tree, "tax_cat_custom_social_ads_1")
print(path_ids)

# Output:
# ['tax_cat_usa_schedule_c_expenses', 'tax_cat_usa_schedule_c_advertising', 'tax_cat_custom_social_ads_1']
```

### Example 4: Depth-First Traversal

```python
# Traverse tree in pre-order (parent before children)
nodes = engine.traverse(tree, order=TraversalOrder.DEPTH_FIRST_PRE)

for node in nodes:
    print(f"{'  ' * node.depth}{node.name}")

# Output:
# Expenses
#   Advertising
#     Social Media Advertising
#     Print Advertising
#   Office Expenses
#     Software Subscriptions
#     Office Supplies
#   Travel
```

### Example 5: Breadth-First Traversal

```python
# Traverse tree level by level
nodes = engine.traverse(tree, order=TraversalOrder.BREADTH_FIRST)

for node in nodes:
    print(f"Depth {node.depth}: {node.name}")

# Output:
# Depth 0: Expenses
# Depth 0: Advertising
# Depth 0: Office Expenses
# Depth 1: Social Media Advertising
# Depth 1: Print Advertising
# Depth 1: Software Subscriptions
```

### Example 6: Search Tree

```python
# Find all fully deductible categories (rate = 1.0)
fully_deductible = engine.search_tree(
    tree,
    lambda node: node.metadata.get('deduction_rate') == 1.0
)

print(f"Found {len(fully_deductible)} fully deductible categories:")
for node in fully_deductible:
    print(f"  - {node.name}")

# Find all leaf categories
leaves = engine.search_tree(tree, lambda node: len(node.children) == 0)
print(f"Found {len(leaves)} leaf categories")
```

### Example 7: Get Ancestors and Descendants

```python
# Get ancestors of "Social Media Advertising"
ancestors = engine.get_ancestors(tree, "tax_cat_custom_social_ads_1")
print("Ancestors:", [node.name for node in ancestors])
# Output: ['Expenses', 'Advertising']

# Get descendants of "Advertising"
descendants = engine.get_descendants(tree, "tax_cat_usa_schedule_c_advertising")
print("Descendants:", [node.name for node in descendants])
# Output: ['Social Media Advertising', 'Print Advertising', 'TV Advertising', ...]
```

### Example 8: Custom Visitor Function

```python
# Print tree with indentation
def print_category(node):
    indent = "  " * node.depth
    rate = node.metadata.get('deduction_rate', 0)
    print(f"{indent}{node.name} ({int(rate * 100)}% deductible)")

engine.traverse(tree, visitor=print_category)

# Output:
# Expenses (100% deductible)
#   Advertising (100% deductible)
#     Social Media Advertising (100% deductible)
#   Meals (50% deductible)
```

---

## Multi-Domain Examples

### Healthcare: CPT Code Navigation

```python
# Load CPT code taxonomy
cpt_tree = engine.load_taxonomy(
    jurisdiction="cpt",
    user_id="doctor_123"
)

# Get path to specific procedure
path = engine.get_path(cpt_tree, "cpt_99213")
print(" → ".join(path))
# Output: Evaluation and Management → Office Visits → Established Patient → Moderate Complexity

# Find all office visit codes
office_visits = engine.search_tree(
    cpt_tree,
    lambda node: "Office Visit" in node.path
)
```

### Legal: Case Category Navigation

```python
# Load legal case taxonomy
legal_tree = engine.load_taxonomy(
    jurisdiction="federal",
    user_id="lawyer_456"
)

# Get all civil litigation subcategories
civil_descendants = engine.get_descendants(
    legal_tree,
    "cat_federal_civil_litigation"
)

for node in civil_descendants:
    print(f"{'  ' * node.depth}{node.name}")

# Output:
# Contract Disputes
#   Breach of Contract
#   Non-Performance
# Torts
#   Negligence
#   Product Liability
```

### Research: Grant Category Navigation

```python
# Load NSF grant taxonomy
nsf_tree = engine.load_taxonomy(
    jurisdiction="nsf",
    user_id="researcher_789"
)

# Get all direct cost categories
direct_costs = engine.get_descendants(
    nsf_tree,
    "cat_nsf_direct_costs"
)

# Find categories with cost-share requirement
cost_share_categories = engine.search_tree(
    nsf_tree,
    lambda node: node.metadata.get('deduction_rate', 1.0) < 1.0
)
```

---

## Performance Characteristics

### Time Complexity

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `load_taxonomy()` | O(n) | Build tree from n categories |
| `get_children()` | O(1) | Direct access via node.children |
| `get_path()` | O(1) | Pre-calculated during tree build |
| `traverse()` DFS | O(n) | Visit each node once |
| `traverse()` BFS | O(n) | Visit each node once |
| `search_tree()` | O(n) | Check each node |
| `get_ancestors()` | O(1) | Direct access via path_ids |
| `get_descendants()` | O(d) | Visit d descendants |

### Space Complexity

| Operation | Space Complexity | Notes |
|-----------|------------------|-------|
| `load_taxonomy()` | O(n) | Store n nodes in memory |
| Tree structure | O(n) | node_map + children lists |
| Traversal | O(h) | Recursion stack depth h (tree height) |

### Caching Strategy

```python
def load_taxonomy(
    self,
    jurisdiction: str,
    user_id: Optional[str] = None,
    include_system: bool = True,
    include_custom: bool = True,
    force_reload: bool = False
) -> TaxonomyTree:
    """
    Load taxonomy with in-memory caching.

    Cache key: {jurisdiction}:{user_id}:{include_system}:{include_custom}
    TTL: 5 minutes for custom, 24 hours for system-only
    """
    cache_key = f"{jurisdiction}:{user_id}:{include_system}:{include_custom}"

    # Check cache
    if not force_reload and cache_key in self._cache:
        return self._cache[cache_key]

    # Load from database
    categories = self.category_store.list_by_jurisdiction(
        jurisdiction=jurisdiction,
        user_id=user_id,
        include_system=include_system,
        include_custom=include_custom
    )

    # Build tree
    tree = self._build_tree(categories)

    # Cache result
    self._cache[cache_key] = tree

    return tree

def invalidate_cache(self, jurisdiction: str, user_id: str):
    """
    Invalidate cache for jurisdiction and user.

    Called when user creates/updates/deletes custom categories.
    """
    # Remove all cache entries for this user and jurisdiction
    keys_to_remove = [
        key for key in self._cache.keys()
        if key.startswith(f"{jurisdiction}:{user_id}:")
    ]

    for key in keys_to_remove:
        del self._cache[key]
```

---

## Algorithm Details

### Depth-First Search (Pre-Order)

**Use Case:** Process parent before children (e.g., permissions inheritance)

**Algorithm:**
```
DFS-Pre(node):
  1. Visit node
  2. For each child:
       DFS-Pre(child)
```

**Example:**
```
Tree:
    A
   / \
  B   C
 /
D

Order: A → B → D → C
```

**Implementation:**
```python
def _dfs_pre(node, result, visitor):
    # Visit parent first
    if visitor:
        visitor(node)
    result.append(node)

    # Then children
    for child in node.children:
        _dfs_pre(child, result, visitor)
```

---

### Depth-First Search (Post-Order)

**Use Case:** Process children before parent (e.g., delete operations, aggregations)

**Algorithm:**
```
DFS-Post(node):
  1. For each child:
       DFS-Post(child)
  2. Visit node
```

**Example:**
```
Tree:
    A
   / \
  B   C
 /
D

Order: D → B → C → A
```

**Implementation:**
```python
def _dfs_post(node, result, visitor):
    # Visit children first
    for child in node.children:
        _dfs_post(child, result, visitor)

    # Then parent
    if visitor:
        visitor(node)
    result.append(node)
```

---

### Breadth-First Search

**Use Case:** Process by levels (e.g., shortest path, level-order display)

**Algorithm:**
```
BFS(root):
  1. queue = [root]
  2. While queue not empty:
       node = dequeue()
       Visit node
       Enqueue all children
```

**Example:**
```
Tree:
    A
   / \
  B   C
 /
D

Order: A → B → C → D
```

**Implementation:**
```python
def _bfs(node, result, visitor):
    queue = [node]

    while queue:
        current = queue.pop(0)  # Dequeue

        # Visit
        if visitor:
            visitor(current)
        result.append(current)

        # Enqueue children
        queue.extend(current.children)
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Load taxonomy
def test_load_taxonomy_success()
def test_load_taxonomy_empty()
def test_load_taxonomy_cache()
def test_load_taxonomy_force_reload()

# Test: Tree building
def test_build_tree_single_root()
def test_build_tree_multiple_roots()
def test_build_tree_deep_hierarchy()
def test_build_tree_calculate_depths()
def test_build_tree_calculate_paths()

# Test: Get children
def test_get_children_root()
def test_get_children_parent()
def test_get_children_leaf()

# Test: Get path
def test_get_path_root()
def test_get_path_leaf()
def test_get_path_not_found()

# Test: Traversal
def test_traverse_dfs_pre()
def test_traverse_dfs_post()
def test_traverse_bfs()
def test_traverse_with_visitor()

# Test: Search
def test_search_tree_predicate()
def test_search_tree_no_matches()
def test_search_tree_all_match()

# Test: Ancestors/descendants
def test_get_ancestors()
def test_get_descendants()
def test_get_descendants_max_depth()
def test_get_siblings()

# Test: Edge cases
def test_circular_reference_detection()
def test_orphan_node_handling()
def test_empty_tree()
```

---

## Related Primitives

- **TaxCategoryStore** (OL) - Provides category data for taxonomy
- **TaxCategoryClassifier** (OL) - Uses taxonomy for pattern matching
- **TaxCategorySelector** (IL) - Renders taxonomy in UI
- **TaxCategoryManager** (IL) - Visualizes taxonomy tree

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 2-3 days
**Dependencies:** TaxCategoryStore, Tree algorithms (DFS/BFS)
