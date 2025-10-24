# Primitive: NormalizationRuleStore (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Data Store (CRUD)
> **Vertical:** 3.8 Cluster Rules
> **Last Updated:** 2025-10-24

---

## Overview

**NormalizationRuleStore** is an Objective Layer primitive that persists and manages normalization rules. It provides CRUD operations, pattern matching, priority ordering, and rule versioning.

**Core Responsibility:**
Store, retrieve, and execute user-defined text normalization rules with proper ownership, priority ordering, and audit trail.

**Domain Instantiation:**
- Finance: Store merchant normalization rules
- Healthcare: Store provider/medication normalization rules
- Legal: Store case/party name normalization rules
- Research: Store institution/author normalization rules

---

## Interface

### Methods

```python
class NormalizationRuleStore:
    """
    Persists and manages normalization rules.

    Attributes:
        db: Database connection
        cache: LRU cache for rule lookups
    """

    def create_rule(self, user_id: str, pattern: str, replacement: str,
                    rule_type: RuleType, priority: int,
                    similarity_threshold: Optional[float] = None) -> NormalizationRule:
        """
        Creates normalization rule.

        Args:
            user_id: User ID for rule ownership
            pattern: Pattern to match (exact string, regex, or fuzzy template)
            replacement: Normalized output (e.g., "Uber Eats")
            rule_type: "exact" | "regex" | "fuzzy" | "soundex"
            priority: [0, 100], higher priority executes first
            similarity_threshold: [0.0, 1.0], required if rule_type=fuzzy

        Returns:
            NormalizationRule object with generated rule_id

        Raises:
            DuplicatePatternError: If pattern exists for user
            InvalidPriorityError: If priority not in [0, 100]
            InvalidRegexError: If pattern invalid for regex type
            InvalidThresholdError: If fuzzy without threshold or threshold not in [0.0, 1.0]

        Example:
            >>> rule = store.create_rule(
            ...     user_id="user_darwin",
            ...     pattern="UBER.*",
            ...     replacement="Uber",
            ...     rule_type="regex",
            ...     priority=90
            ... )
            >>> rule.rule_id
            "rule_uber_all_1"
        """

    def get_rule(self, rule_id: str, user_id: str) -> Optional[NormalizationRule]:
        """
        Retrieves rule by ID.

        Args:
            rule_id: Rule ID to fetch
            user_id: User ID for ownership verification

        Returns:
            NormalizationRule object or None if not found

        Raises:
            UnauthorizedError: If user_id doesn't match rule owner
        """

    def list_rules(self, user_id: str, enabled_only: bool = True,
                   order_by: str = "priority") -> List[NormalizationRule]:
        """
        Lists all rules for user.

        Args:
            user_id: User ID to query
            enabled_only: If True, only return enabled rules (default True)
            order_by: "priority" | "created_at" | "match_count"

        Returns:
            List of NormalizationRule objects sorted by order_by

        Example:
            >>> rules = store.list_rules("user_darwin", enabled_only=True)
            >>> rules[0].priority
            95  # Highest priority first
        """

    def find_matching_rules(self, text: str, user_id: str) -> List[NormalizationRule]:
        """
        Finds all rules that match text.

        Args:
            text: Text to match against rules
            user_id: User ID for rule lookup

        Returns:
            List of matching rules, ordered by priority desc

        Process:
        1. Load all enabled rules for user
        2. Filter rules that match text (exact, regex, fuzzy)
        3. Sort by priority desc, then created_at asc
        4. Return matches

        Example:
            >>> rules = store.find_matching_rules("UBER EATS PENDING", "user_darwin")
            >>> rules[0].pattern
            "UBER.*"
        """

    def execute_rules(self, text: str, rules: List[NormalizationRule]) -> Optional[str]:
        """
        Executes rules in priority order, returns first match.

        Args:
            text: Text to normalize
            rules: List of rules to try (should be pre-sorted by priority)

        Returns:
            Normalized text from first matching rule, or None if no match

        Example:
            >>> result = store.execute_rules("UBER EATS", [rule1, rule2])
            >>> result
            "Uber"
        """

    def update_rule(self, rule_id: str, user_id: str, **kwargs) -> NormalizationRule:
        """
        Updates rule fields.

        Args:
            rule_id: Rule ID to update
            user_id: User ID for ownership verification
            **kwargs: Fields to update (pattern, replacement, priority, enabled, etc.)

        Returns:
            Updated NormalizationRule object

        Raises:
            RuleNotFoundError: If rule_id not found
            UnauthorizedError: If user_id doesn't match rule owner

        Example:
            >>> rule = store.update_rule(
            ...     "rule_uber_1",
            ...     "user_darwin",
            ...     priority=95,
            ...     enabled=True
            ... )
        """

    def soft_delete(self, rule_id: str, user_id: str) -> None:
        """
        Soft deletes rule (sets enabled=False).

        Args:
            rule_id: Rule ID to delete
            user_id: User ID for ownership verification

        Raises:
            RuleNotFoundError: If rule_id not found
            UnauthorizedError: If user_id doesn't match rule owner

        Note:
            Rule is preserved for audit trail and historical normalization provenance.
        """

    def hard_delete(self, rule_id: str, user_id: str) -> None:
        """
        Permanently deletes rule.

        Args:
            rule_id: Rule ID to delete
            user_id: User ID for ownership verification

        Raises:
            RuleNotFoundError: If rule_id not found
            UnauthorizedError: If user_id doesn't match rule owner
            RuleInUseError: If rule has been applied to transactions

        Warning:
            Only allowed if rule has never been used (match_count=0).
        """

    def increment_match_count(self, rule_id: str) -> None:
        """
        Increments match_count for rule (called when rule matches).

        Args:
            rule_id: Rule ID to increment
        """

    def find_by_pattern(self, user_id: str, pattern: str) -> Optional[NormalizationRule]:
        """
        Finds rule by exact pattern match (case-insensitive).

        Args:
            user_id: User ID to query
            pattern: Pattern to search

        Returns:
            NormalizationRule object or None if not found

        Example:
            >>> rule = store.find_by_pattern("user_darwin", "UBER.*")
            >>> rule.rule_id if rule else None
            "rule_uber_all_1"
        """
```

---

## Data Model

### NormalizationRule (Database Schema)

```sql
CREATE TABLE normalization_rules (
    rule_id VARCHAR(100) PRIMARY KEY,
    user_id UUID NOT NULL,
    pattern VARCHAR(500) NOT NULL,
    replacement VARCHAR(200) NOT NULL,
    rule_type VARCHAR(20) NOT NULL CHECK (rule_type IN ('exact', 'regex', 'fuzzy', 'soundex')),
    priority INTEGER NOT NULL CHECK (priority >= 0 AND priority <= 100),
    similarity_threshold DECIMAL(3,2) CHECK (similarity_threshold >= 0 AND similarity_threshold <= 1),
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    match_count INTEGER DEFAULT 0,
    UNIQUE(user_id, pattern),
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

CREATE INDEX idx_rules_user_priority ON normalization_rules(user_id, enabled, priority DESC);
CREATE INDEX idx_rules_pattern ON normalization_rules(user_id, pattern);
CREATE INDEX idx_rules_match_count ON normalization_rules(match_count DESC);
```

### Python Dataclass

```python
@dataclass
class NormalizationRule:
    """
    User-defined normalization rule.

    Attributes:
        rule_id: Unique identifier (e.g., "rule_uber_all_1")
        user_id: User who created rule
        pattern: Pattern to match (exact string, regex, or fuzzy template)
        replacement: Normalized output (e.g., "Uber Eats")
        rule_type: "exact" | "regex" | "fuzzy" | "soundex"
        priority: [0, 100], higher priority executes first
        similarity_threshold: [0.0, 1.0], required if rule_type=fuzzy
        enabled: True if active, False if disabled
        created_at: Creation timestamp
        updated_at: Last update timestamp
        match_count: Count of times rule has matched
    """
    rule_id: str
    user_id: str
    pattern: str
    replacement: str
    rule_type: str
    priority: int
    similarity_threshold: Optional[float]
    enabled: bool
    created_at: datetime
    updated_at: datetime
    match_count: int
```

---

## Validation Logic

### Pattern Validation

```python
def validate_pattern(pattern: str, rule_type: str) -> None:
    """
    Validates pattern based on rule type.

    Raises:
        ValidationError: If pattern invalid
    """
    # Length check
    if len(pattern) == 0 or len(pattern) > 500:
        raise ValidationError("Pattern must be 1-500 chars")

    # Regex validation
    if rule_type == "regex":
        try:
            re.compile(pattern)
        except re.error as e:
            raise InvalidRegexError(f"Invalid regex pattern: {e}")

        # Block catastrophic backtracking patterns
        if any(p in pattern for p in ["(a+)+", "(a*)*", "(a|a)*"]):
            raise InvalidRegexError("Pattern may cause catastrophic backtracking")

    # Fuzzy validation
    if rule_type == "fuzzy":
        if len(pattern) < 3:
            raise ValidationError("Fuzzy pattern must be at least 3 chars")
```

### Priority Validation

```python
def validate_priority(priority: int) -> None:
    """
    Validates priority is in range [0, 100].

    Raises:
        InvalidPriorityError: If priority out of range
    """
    if priority < 0 or priority > 100:
        raise InvalidPriorityError("Priority must be in [0, 100]")
```

### Threshold Validation

```python
def validate_threshold(rule_type: str, threshold: Optional[float]) -> None:
    """
    Validates fuzzy threshold.

    Raises:
        InvalidThresholdError: If threshold invalid
    """
    if rule_type == "fuzzy":
        if threshold is None:
            raise InvalidThresholdError("Fuzzy rule requires similarity_threshold")
        if threshold < 0.0 or threshold > 1.0:
            raise InvalidThresholdError("Threshold must be in [0.0, 1.0]")
```

---

## Multi-Domain Examples

### Finance: Merchant Normalization Rules

```python
# Create exact match rule
rule1 = store.create_rule(
    user_id="user_darwin",
    pattern="AMZN MKTP US",
    replacement="Amazon",
    rule_type="exact",
    priority=95
)

# Create regex rule
rule2 = store.create_rule(
    user_id="user_darwin",
    pattern="UBER.*",
    replacement="Uber",
    rule_type="regex",
    priority=90
)

# Create fuzzy rule
rule3 = store.create_rule(
    user_id="user_darwin",
    pattern="McDonald's",
    replacement="McDonald's",
    rule_type="fuzzy",
    priority=85,
    similarity_threshold=0.80
)

# List all rules (ordered by priority)
rules = store.list_rules("user_darwin", enabled_only=True)
# → [rule1 (95), rule2 (90), rule3 (85)]

# Find matching rules
matches = store.find_matching_rules("UBER EATS PENDING", "user_darwin")
# → [rule2] (pattern "UBER.*" matches)

# Execute rules
normalized = store.execute_rules("UBER EATS PENDING", matches)
# → "Uber"
```

### Healthcare: Provider Normalization Rules

```python
# Normalize hospital names
store.create_rule(
    user_id="hospital_system",
    pattern="ST MARY'S.*",
    replacement="St. Mary's Hospital",
    rule_type="regex",
    priority=90
)

store.create_rule(
    user_id="hospital_system",
    pattern="KAISER.*",
    replacement="Kaiser Permanente",
    rule_type="regex",
    priority=85
)

# Fuzzy match for misspellings
store.create_rule(
    user_id="hospital_system",
    pattern="Massachusetts General Hospital",
    replacement="Massachusetts General Hospital",
    rule_type="fuzzy",
    priority=80,
    similarity_threshold=0.85
)
```

### Legal: Case Name Normalization Rules

```python
# Normalize case name format
store.create_rule(
    user_id="law_firm",
    pattern=r"(\w+) V\. (\w+)",
    replacement=r"\1 v. \2",
    rule_type="regex",
    priority=95
)

store.create_rule(
    user_id="law_firm",
    pattern=r"(\w+) VS (\w+)",
    replacement=r"\1 v. \2",
    rule_type="regex",
    priority=90
)
```

### Research: Institution Normalization Rules

```python
# Normalize university abbreviations
store.create_rule(
    user_id="researcher",
    pattern="MIT CSAIL",
    replacement="MIT Computer Science and AI Lab",
    rule_type="exact",
    priority=95
)

store.create_rule(
    user_id="researcher",
    pattern="Stanford Univ",
    replacement="Stanford University",
    rule_type="fuzzy",
    priority=85,
    similarity_threshold=0.80
)
```

---

## Caching Strategy

### LRU Cache for Rule Lookups

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_cached_rules(user_id: str) -> List[NormalizationRule]:
    """
    Caches user's rules for 5 minutes.

    Invalidate on:
    - create_rule()
    - update_rule()
    - soft_delete()
    - hard_delete()
    """
    return db.query(
        "SELECT * FROM normalization_rules WHERE user_id = ? AND enabled = TRUE ORDER BY priority DESC",
        user_id
    )

def invalidate_cache(user_id: str) -> None:
    """Invalidates cache for user."""
    get_cached_rules.cache_clear()
```

---

## Performance Characteristics

### Latency Targets

- `create_rule()`: <50ms p95
- `list_rules()`: <20ms p95 (cached)
- `find_matching_rules()`: <30ms p95
- `execute_rules()`: <10ms p95 (per rule)

### Database Indexes

```sql
-- Fast lookup by user + priority
CREATE INDEX idx_rules_user_priority ON normalization_rules(user_id, enabled, priority DESC);

-- Fast pattern search
CREATE INDEX idx_rules_pattern ON normalization_rules(user_id, pattern);

-- Analytics query (top rules by usage)
CREATE INDEX idx_rules_match_count ON normalization_rules(match_count DESC);
```

### Scalability

- Rules per user: Unlimited (typical: 10-100)
- Query performance: O(log n) with indexes
- Cache hit rate target: >95%

---

## Error Handling

```python
class DuplicatePatternError(Exception):
    """Raised when pattern already exists for user."""
    pass

class InvalidPriorityError(Exception):
    """Raised when priority not in [0, 100]."""
    pass

class InvalidRegexError(Exception):
    """Raised when regex pattern invalid."""
    pass

class InvalidThresholdError(Exception):
    """Raised when fuzzy threshold invalid."""
    pass

class RuleNotFoundError(Exception):
    """Raised when rule_id not found."""
    pass

class UnauthorizedError(Exception):
    """Raised when user_id doesn't match rule owner."""
    pass

class RuleInUseError(Exception):
    """Raised when trying to hard delete rule with match_count > 0."""
    pass
```

---

## Testing

### Unit Tests

```python
def test_create_rule_exact():
    rule = store.create_rule(
        user_id="user_1",
        pattern="UBER EATS",
        replacement="Uber Eats",
        rule_type="exact",
        priority=90
    )
    assert rule.rule_id.startswith("rule_")
    assert rule.pattern == "UBER EATS"
    assert rule.enabled == True

def test_create_rule_duplicate_pattern():
    store.create_rule("user_1", "UBER.*", "Uber", "regex", 90)
    with pytest.raises(DuplicatePatternError):
        store.create_rule("user_1", "UBER.*", "Uber Eats", "regex", 85)

def test_create_rule_invalid_priority():
    with pytest.raises(InvalidPriorityError):
        store.create_rule("user_1", "UBER.*", "Uber", "regex", 150)

def test_create_rule_invalid_regex():
    with pytest.raises(InvalidRegexError):
        store.create_rule("user_1", "(a+)+", "Bad Pattern", "regex", 90)

def test_list_rules_ordered_by_priority():
    store.create_rule("user_1", "Pattern A", "A", "exact", 95)
    store.create_rule("user_1", "Pattern B", "B", "exact", 90)
    store.create_rule("user_1", "Pattern C", "C", "exact", 85)
    rules = store.list_rules("user_1")
    assert rules[0].priority == 95
    assert rules[1].priority == 90
    assert rules[2].priority == 85

def test_find_matching_rules():
    store.create_rule("user_1", "UBER.*", "Uber", "regex", 90)
    store.create_rule("user_1", "UBER EATS", "Uber Eats", "exact", 95)
    matches = store.find_matching_rules("UBER EATS PENDING", "user_1")
    assert len(matches) == 1  # Only regex matches (exact doesn't match "PENDING")
    assert matches[0].pattern == "UBER.*"

def test_soft_delete():
    rule = store.create_rule("user_1", "TEST", "Test", "exact", 90)
    store.soft_delete(rule.rule_id, "user_1")
    deleted = store.get_rule(rule.rule_id, "user_1")
    assert deleted.enabled == False

def test_hard_delete_with_usage():
    rule = store.create_rule("user_1", "TEST", "Test", "exact", 90)
    store.increment_match_count(rule.rule_id)
    with pytest.raises(RuleInUseError):
        store.hard_delete(rule.rule_id, "user_1")
```

---

## Security Considerations

### Ownership Verification

```python
def verify_ownership(rule_id: str, user_id: str) -> None:
    """
    Verifies user owns rule.

    Raises:
        UnauthorizedError: If user_id doesn't match rule owner
    """
    rule = db.query("SELECT user_id FROM normalization_rules WHERE rule_id = ?", rule_id)
    if not rule or rule.user_id != user_id:
        raise UnauthorizedError(f"User {user_id} doesn't own rule {rule_id}")
```

### SQL Injection Prevention

```python
# Use parameterized queries
db.query(
    "SELECT * FROM normalization_rules WHERE user_id = ? AND pattern = ?",
    user_id, pattern
)

# NEVER use string formatting
# BAD: f"SELECT * FROM rules WHERE user_id = '{user_id}'"
```

### Input Sanitization

```python
def sanitize_replacement(replacement: str) -> str:
    """
    Sanitizes replacement text to prevent XSS.

    Returns:
        HTML-escaped replacement
    """
    return html.escape(replacement)
```

---

## Observability

### Metrics

```
normalization_rules.created.count (counter, labels: user_id, rule_type)
normalization_rules.deleted.count (counter, labels: user_id, soft_delete)
normalization_rules.executed.count (counter, labels: rule_id)
normalization_rules.cache_hit_rate (gauge)
```

### Structured Logs

```json
{
  "event": "rule_created",
  "rule_id": "rule_uber_all_1",
  "user_id": "user_darwin",
  "pattern": "UBER.*",
  "rule_type": "regex",
  "priority": 90,
  "timestamp": "2025-10-24T10:00:00Z"
}
```

---

## Related Primitives

- **MerchantNormalizer** - Uses rules to normalize text
- **ClusteringEngine** - Suggests rules from clusters
- **FuzzyMatcher** - Calculates similarity for fuzzy rules

---

## Summary

**NormalizationRuleStore** provides rule persistence with:
- ✅ CRUD operations with ownership verification
- ✅ Priority-based rule ordering
- ✅ Pattern validation (exact, regex, fuzzy)
- ✅ Soft/hard delete with usage protection
- ✅ LRU caching for performance
- ✅ Audit trail (match_count tracking)
- ✅ Multi-domain applicability

**Universal Pattern:**
Applicable to any domain requiring user-defined text transformation rules.
