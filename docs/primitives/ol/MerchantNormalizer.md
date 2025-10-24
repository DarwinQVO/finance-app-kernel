# Primitive: MerchantNormalizer (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Text Normalization Engine
> **Vertical:** 3.8 Cluster Rules
> **Last Updated:** 2025-10-24

---

## Overview

**MerchantNormalizer** is an Objective Layer primitive that normalizes raw merchant names using user-defined rules. It applies rules in priority order (exact match > regex > fuzzy > default) and logs all executions for audit trail.

**Core Responsibility:**
Apply text normalization rules to standardize inconsistent entity names (merchants, providers, institutions, etc.) while preserving original text for provenance.

**Domain Instantiation:**
- Finance: Normalize merchant names ("UBER EATS PENDING" → "Uber Eats")
- Healthcare: Normalize provider names ("ST MARY'S HOSP" → "St. Mary's Hospital")
- Legal: Normalize case/party names ("DOE V. SMITH" → "Doe v. Smith")
- Research: Normalize institution names ("MIT CSAIL" → "MIT Computer Science and AI Lab")

---

## Interface

### Methods

```python
class MerchantNormalizer:
    """
    Normalizes raw text using user-defined rules with priority ordering.

    Attributes:
        rule_store: NormalizationRuleStore instance
        fuzzy_matcher: FuzzyMatcher instance
        execution_logger: RuleExecutionLogger instance
    """

    def normalize(self, raw_name: str, user_id: str) -> NormalizedResult:
        """
        Normalizes raw merchant name using user's rules.

        Process:
        1. Load all enabled rules for user (ordered by priority desc)
        2. Try exact match first (highest priority)
        3. Try regex match (second priority)
        4. Try fuzzy match if threshold met (third priority)
        5. Return original if no match (default)

        Args:
            raw_name: Raw merchant name from transaction (e.g., "UBER EATS PENDING")
            user_id: User ID for rule lookup

        Returns:
            NormalizedResult with:
                - normalized_name: Standardized name (e.g., "Uber")
                - rule_id: ID of rule that matched (or None if no match)
                - match_type: "exact" | "regex" | "fuzzy" | "none"
                - confidence: 1.0 for exact/regex, [0.0-1.0] for fuzzy, None for none

        Raises:
            ValidationError: If raw_name empty or > 500 chars
            UserNotFoundError: If user_id invalid

        Example:
            >>> result = normalizer.normalize("UBER EATS PENDING", "user_darwin")
            >>> result.normalized_name
            "Uber"
            >>> result.match_type
            "regex"
            >>> result.confidence
            1.0
        """

    def batch_normalize(self, raw_names: List[str], user_id: str) -> Dict[str, NormalizedResult]:
        """
        Batch normalization for performance.

        Args:
            raw_names: List of raw merchant names
            user_id: User ID for rule lookup

        Returns:
            Dict mapping raw_name → NormalizedResult

        Performance:
            - Loads rules once (instead of per-item)
            - Caches compiled regex patterns
            - Parallelizes fuzzy matching across candidates

        Example:
            >>> results = normalizer.batch_normalize(
            ...     ["UBER EATS", "UBER *RIDE", "AMAZON.COM"],
            ...     "user_darwin"
            ... )
            >>> results["UBER EATS"].normalized_name
            "Uber"
            >>> results["AMAZON.COM"].normalized_name
            "Amazon"
        """

    def add_rule(self, user_id: str, pattern: str, replacement: str,
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
            NormalizationRule object

        Raises:
            DuplicatePatternError: If pattern exists for user
            InvalidPriorityError: If priority not in [0, 100]
            InvalidRegexError: If pattern invalid for regex type
            InvalidThresholdError: If fuzzy without threshold or threshold not in [0.0, 1.0]

        Example:
            >>> rule = normalizer.add_rule(
            ...     user_id="user_darwin",
            ...     pattern="UBER.*",
            ...     replacement="Uber",
            ...     rule_type="regex",
            ...     priority=90
            ... )
            >>> rule.rule_id
            "rule_uber_all_1"
        """

    def test_rule(self, pattern: str, rule_type: RuleType, user_id: str,
                  similarity_threshold: Optional[float] = None) -> RuleTestResult:
        """
        Tests rule without saving (preview matches).

        Args:
            pattern: Pattern to test
            rule_type: "exact" | "regex" | "fuzzy" | "soundex"
            user_id: User ID for transaction lookup
            similarity_threshold: [0.0, 1.0], required if rule_type=fuzzy

        Returns:
            RuleTestResult with:
                - matches: List of (raw_name, normalized_name, similarity)
                - total_transactions: Count of transactions that would be affected

        Example:
            >>> result = normalizer.test_rule(
            ...     pattern="McDonald's",
            ...     rule_type="fuzzy",
            ...     user_id="user_darwin",
            ...     similarity_threshold=0.80
            ... )
            >>> result.matches
            [
                ("MCDONALDS #1234", "McDonald's", 0.87),
                ("MCDONALD'S", "McDonald's", 0.95)
            ]
            >>> result.total_transactions
            23
        """

    def disable_rule(self, rule_id: str, user_id: str) -> None:
        """
        Disables rule (soft delete).

        Args:
            rule_id: Rule ID to disable
            user_id: User ID for ownership verification

        Raises:
            RuleNotFoundError: If rule_id not found
            UnauthorizedError: If user_id doesn't match rule owner
        """

    def enable_rule(self, rule_id: str, user_id: str) -> None:
        """
        Enables previously disabled rule.

        Args:
            rule_id: Rule ID to enable
            user_id: User ID for ownership verification

        Raises:
            RuleNotFoundError: If rule_id not found
            UnauthorizedError: If user_id doesn't match rule owner
        """

    def get_execution_log(self, rule_id: str, user_id: str,
                          limit: int = 100) -> List[RuleExecutionLog]:
        """
        Retrieves execution logs for rule (audit trail).

        Args:
            rule_id: Rule ID to query
            user_id: User ID for ownership verification
            limit: Max logs to return (default 100)

        Returns:
            List of RuleExecutionLog sorted by executed_at desc

        Example:
            >>> logs = normalizer.get_execution_log("rule_uber_all_1", "user_darwin")
            >>> logs[0].input_text
            "UBER EATS PENDING"
            >>> logs[0].output_text
            "Uber"
        """
```

---

## Data Model

### NormalizedResult

```python
@dataclass
class NormalizedResult:
    """
    Result of normalization operation.

    Attributes:
        normalized_name: Standardized name (e.g., "Uber Eats")
        rule_id: ID of rule that matched (None if no match)
        match_type: "exact" | "regex" | "fuzzy" | "none"
        confidence: 1.0 for exact/regex, [0.0-1.0] for fuzzy, None for none
        original_name: Preserved raw name for provenance
    """
    normalized_name: str
    rule_id: Optional[str]
    match_type: str
    confidence: Optional[float]
    original_name: str
```

### NormalizationRule

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

### RuleTestResult

```python
@dataclass
class RuleTestResult:
    """
    Result of test_rule() operation.

    Attributes:
        matches: List of (raw_name, normalized_name, similarity) tuples
        total_transactions: Count of transactions that would be affected
        sample_matches: First 10 matches for preview
    """
    matches: List[Tuple[str, str, float]]
    total_transactions: int
    sample_matches: List[Tuple[str, str, float]]
```

---

## Algorithm

### Normalization Process

```
1. INPUT: raw_name, user_id
2. Load all enabled rules for user (ORDER BY priority DESC, created_at ASC)
3. For each rule (highest priority first):
   a. If rule_type == "exact":
      - Compare raw_name == pattern (case-insensitive)
      - If match, return (replacement, rule_id, "exact", 1.0)
   b. If rule_type == "regex":
      - Try regex.match(pattern, raw_name)
      - If match, return (replacement, rule_id, "regex", 1.0)
   c. If rule_type == "fuzzy":
      - Calculate similarity = fuzzy_matcher.calculate_similarity(raw_name, pattern)
      - If similarity >= threshold, return (replacement, rule_id, "fuzzy", similarity)
   d. If rule_type == "soundex":
      - Calculate soundex_match = soundex(raw_name) == soundex(pattern)
      - If match, return (replacement, rule_id, "soundex", 0.9)
4. If no match, return (raw_name, None, "none", None)
5. Log execution to RuleExecutionLog
6. OUTPUT: NormalizedResult
```

### Priority Ordering

Rules are applied in this order:
1. **Priority** (descending): Higher priority rules execute first
2. **Rule Type** (implicit): Exact > Regex > Fuzzy > Soundex
3. **Creation Time** (ascending): Older rules win ties

Example:
```
Rule A: priority=95, type=exact, created=2025-01-01  ← Executes 1st
Rule B: priority=90, type=regex, created=2025-01-02  ← Executes 2nd
Rule C: priority=90, type=regex, created=2025-01-01  ← Executes 3rd (older)
Rule D: priority=85, type=fuzzy, created=2025-01-03  ← Executes 4th
```

### Regex Timeout

To prevent catastrophic backtracking (ReDoS):
```python
def safe_regex_match(pattern: str, text: str, timeout_ms: int = 100) -> Optional[re.Match]:
    """
    Executes regex with timeout to prevent ReDoS attacks.

    Raises:
        RegexTimeoutError: If execution exceeds timeout_ms
    """
    # Use regex library with timeout support
    # Block common backtracking patterns: (a+)+, (a*)*
```

---

## Multi-Domain Examples

### Finance: Merchant Name Normalization

```python
# Create rules for common merchants
normalizer.add_rule(
    user_id="user_darwin",
    pattern="UBER.*",
    replacement="Uber",
    rule_type="regex",
    priority=90
)

normalizer.add_rule(
    user_id="user_darwin",
    pattern="AMZN MKTP US",
    replacement="Amazon",
    rule_type="exact",
    priority=95
)

# Normalize transaction
result = normalizer.normalize("UBER EATS PENDING", "user_darwin")
# → NormalizedResult(normalized_name="Uber", match_type="regex", confidence=1.0)

# Batch normalize
results = normalizer.batch_normalize(
    ["UBER EATS", "AMZN MKTP US", "STARBUCKS"],
    "user_darwin"
)
# → {"UBER EATS": "Uber", "AMZN MKTP US": "Amazon", "STARBUCKS": "STARBUCKS"}
```

### Healthcare: Provider Name Normalization

```python
# Normalize hospital names
normalizer.add_rule(
    user_id="provider_system",
    pattern="ST MARY'S.*",
    replacement="St. Mary's Hospital",
    rule_type="regex",
    priority=90
)

normalizer.add_rule(
    user_id="provider_system",
    pattern="KAISER.*",
    replacement="Kaiser Permanente",
    rule_type="regex",
    priority=85
)

# Fuzzy match for misspellings
normalizer.add_rule(
    user_id="provider_system",
    pattern="Massachusetts General Hospital",
    replacement="Massachusetts General Hospital",
    rule_type="fuzzy",
    priority=80,
    similarity_threshold=0.85
)

result = normalizer.normalize("ST MARY'S HOSP", "provider_system")
# → "St. Mary's Hospital"

result = normalizer.normalize("Mass General Hosp", "provider_system")
# → "Massachusetts General Hospital" (fuzzy match, confidence=0.87)
```

### Legal: Case Name Normalization

```python
# Normalize case name format
normalizer.add_rule(
    user_id="law_firm",
    pattern=r"(\w+) V\. (\w+)",  # "DOE V. SMITH"
    replacement=r"\1 v. \2",      # "Doe v. Smith"
    rule_type="regex",
    priority=95
)

normalizer.add_rule(
    user_id="law_firm",
    pattern=r"(\w+) VS (\w+)",   # "DOE VS SMITH"
    replacement=r"\1 v. \2",      # "Doe v. Smith"
    rule_type="regex",
    priority=90
)

result = normalizer.normalize("DOE V. SMITH", "law_firm")
# → "Doe v. Smith"

result = normalizer.normalize("JOHNSON VS ACME CORP", "law_firm")
# → "Johnson v. ACME CORP"
```

### Research: Institution Name Normalization

```python
# Normalize university names
normalizer.add_rule(
    user_id="researcher",
    pattern="MIT CSAIL",
    replacement="MIT Computer Science and AI Lab",
    rule_type="exact",
    priority=95
)

normalizer.add_rule(
    user_id="researcher",
    pattern="Stanford Univ",
    replacement="Stanford University",
    rule_type="fuzzy",
    priority=85,
    similarity_threshold=0.80
)

result = normalizer.normalize("MIT CSAIL", "researcher")
# → "MIT Computer Science and AI Lab"

result = normalizer.normalize("Stanford U", "researcher")
# → "Stanford University" (fuzzy match, confidence=0.82)
```

### E-commerce: Vendor Name Normalization

```python
# Normalize vendor names for deduplication
normalizer.add_rule(
    user_id="ecommerce_platform",
    pattern="AMZN.*",
    replacement="Amazon",
    rule_type="regex",
    priority=90
)

normalizer.add_rule(
    user_id="ecommerce_platform",
    pattern="eBay Inc",
    replacement="eBay",
    rule_type="exact",
    priority=95
)

result = normalizer.normalize("AMZN MARKETPLACE", "ecommerce_platform")
# → "Amazon"
```

---

## Performance Characteristics

### Latency Targets

- `normalize()` single item: <10ms p95
- `batch_normalize()` 1000 items: <500ms p95
- `test_rule()` preview: <200ms p95

### Optimization Strategies

1. **Rule Caching:**
   - Cache user's rules in memory (TTL: 5 minutes)
   - Invalidate on create/update/delete

2. **Regex Compilation:**
   - Compile regex patterns once per rule
   - Cache compiled patterns for reuse

3. **Batch Processing:**
   - Load rules once for all items
   - Parallelize fuzzy matching across candidates

4. **Index Optimization:**
   - Index rules by (user_id, enabled, priority DESC)
   - Index execution logs by (rule_id, executed_at DESC)

### Scalability

- Rules per user: Unlimited (typical: 10-100)
- Normalization throughput: >1000 items/sec
- Batch size: Up to 10,000 items

---

## Error Handling

### Validation Errors

```python
class ValidationError(Exception):
    """Raised for invalid input."""
    pass

# Example
if len(raw_name) > 500:
    raise ValidationError("raw_name exceeds 500 chars")
```

### Duplicate Pattern

```python
class DuplicatePatternError(Exception):
    """Raised when pattern already exists for user."""
    pass

# Example
existing = rule_store.find_by_pattern(user_id, pattern)
if existing:
    raise DuplicatePatternError(f"Pattern '{pattern}' already exists")
```

### Regex Timeout

```python
class RegexTimeoutError(Exception):
    """Raised when regex execution exceeds timeout."""
    pass

# Example
try:
    match = safe_regex_match(pattern, text, timeout_ms=100)
except RegexTimeoutError:
    log.warning(f"Regex timeout: {pattern}")
    return None
```

---

## Testing

### Unit Tests

```python
def test_normalize_exact_match():
    normalizer = MerchantNormalizer()
    normalizer.add_rule("user_1", "UBER EATS", "Uber Eats", "exact", 90)
    result = normalizer.normalize("UBER EATS", "user_1")
    assert result.normalized_name == "Uber Eats"
    assert result.match_type == "exact"
    assert result.confidence == 1.0

def test_normalize_regex_match():
    normalizer.add_rule("user_1", "UBER.*", "Uber", "regex", 85)
    result = normalizer.normalize("UBER EATS PENDING", "user_1")
    assert result.normalized_name == "Uber"
    assert result.match_type == "regex"

def test_normalize_fuzzy_match():
    normalizer.add_rule("user_1", "McDonald's", "McDonald's", "fuzzy", 80, 0.80)
    result = normalizer.normalize("MCDONALDS #1234", "user_1")
    assert result.normalized_name == "McDonald's"
    assert result.match_type == "fuzzy"
    assert result.confidence >= 0.80

def test_normalize_priority_order():
    # Exact match (95) should win over regex (85)
    normalizer.add_rule("user_1", "UBER EATS", "Uber Eats", "exact", 95)
    normalizer.add_rule("user_1", "UBER.*", "Uber", "regex", 85)
    result = normalizer.normalize("UBER EATS", "user_1")
    assert result.normalized_name == "Uber Eats"  # Exact wins

def test_normalize_no_match():
    normalizer.add_rule("user_1", "UBER.*", "Uber", "regex", 90)
    result = normalizer.normalize("STARBUCKS", "user_1")
    assert result.normalized_name == "STARBUCKS"  # Original preserved
    assert result.match_type == "none"
    assert result.rule_id is None

def test_batch_normalize_performance():
    raw_names = ["UBER EATS"] * 1000
    start = time.time()
    results = normalizer.batch_normalize(raw_names, "user_1")
    duration = time.time() - start
    assert duration < 0.5  # <500ms for 1000 items
```

---

## Security Considerations

### Regex Injection Prevention

- Timeout regex execution at 100ms
- Validate common catastrophic backtracking patterns
- Block patterns: `(a+)+`, `(a*)*`, `(a|a)*`

### SQL Injection Prevention

- Use parameterized queries for all database operations
- Example: `SELECT * FROM rules WHERE user_id = ? AND pattern = ?`

### Input Sanitization

- Validate pattern/replacement for XSS (escape HTML)
- Max length: pattern 500 chars, replacement 200 chars

---

## Observability

### Metrics

```
normalization.executions.count (counter, labels: rule_id, match_type)
normalization.latency (histogram, labels: batch_size)
normalization.regex_timeout.count (counter, labels: pattern)
normalization.no_match.count (counter, labels: user_id)
```

### Structured Logs

```json
{
  "event": "merchant_normalized",
  "user_id": "user_darwin",
  "rule_id": "rule_uber_all_1",
  "input_text": "UBER EATS PENDING",
  "output_text": "Uber",
  "match_type": "regex",
  "confidence": 1.0,
  "latency_ms": 5,
  "timestamp": "2025-10-24T10:00:00Z"
}
```

---

## Related Primitives

- **NormalizationRuleStore** - Persists and queries rules
- **FuzzyMatcher** - Calculates similarity scores
- **ClusteringEngine** - Groups similar entities
- **TransactionStore** - Stores normalized merchant names

---

## Summary

**MerchantNormalizer** provides rule-based text normalization with:
- ✅ Priority-ordered rule execution (exact > regex > fuzzy)
- ✅ Multiple matching algorithms (exact, regex, fuzzy, soundex)
- ✅ Batch processing for performance
- ✅ Audit trail for all executions
- ✅ Multi-domain applicability (finance, healthcare, legal, research)
- ✅ Non-destructive (preserves original text)

**Universal Pattern:**
Applicable to any domain requiring text standardization and entity deduplication.
