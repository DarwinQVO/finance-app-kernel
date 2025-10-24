# CounterpartyMatcher (OL Primitive)

**Vertical:** 3.2 Counterparty Registry
**Layer:** Objective Layer (OL)
**Type:** Fuzzy Matching Engine
**Status:** ✅ Specified

---

## Purpose

Fuzzy matching engine to find existing counterparties from raw merchant names using exact and similarity-based algorithms.

**Core Responsibility:** Match raw transaction merchant names to existing counterparties when exact alias match fails, reducing duplicate creation.

---

## Multi-Domain Applicability

Fuzzy matching applies to ANY domain with entity name variations:

| Domain | Entity | Fuzzy Matching Examples |
|--------|--------|------------------------|
| **Finance** | Merchant | "STARBUCKS #1234" → "Starbucks", "Amazon.com Inc" → "Amazon" |
| **Healthcare** | Provider | "Dr Smith MD" → "Smith Family Practice", "Memorial Hosp" → "Memorial Hospital" |
| **Legal** | Law Firm | "Jones & Assoc" → "Jones & Associates", "Jones Law LLP" → "Jones Legal Group" |
| **Research** | Institution | "Stanford Univ" → "Stanford University", "MIT-Cambridge" → "MIT" |
| **Manufacturing** | Supplier | "ACME STEEL CO" → "Acme Steel Corp", "Acme Inc." → "Acme Steel" |
| **Media** | Creator | "TechReviews YT" → "TechReviews Channel", "@techreviews" → "Tech Reviews" |

**Pattern:** Fuzzy Matcher = normalize inputs, calculate similarity scores, return best match above threshold.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Tuple
from dataclasses import dataclass
from enum import Enum

class MatchMethod(Enum):
    EXACT = "exact"              # Exact alias match (handled by CounterpartyStore)
    TOKEN = "token"              # Token-based matching (common words)
    LEVENSHTEIN = "levenshtein"  # Edit distance
    JARO_WINKLER = "jaro_winkler"  # Jaro-Winkler distance
    HYBRID = "hybrid"            # Combination of multiple methods

@dataclass
class MatchResult:
    counterparty_id: str
    canonical_name: str
    matched_alias: str           # Which alias matched
    similarity_score: float      # 0.0 to 1.0 (1.0 = perfect match)
    match_method: MatchMethod
    confidence: str              # "high", "medium", "low"

@dataclass
class MatchCandidates:
    raw_name: str
    matches: List[MatchResult]   # Sorted by similarity_score DESC
    best_match: Optional[MatchResult]  # Top match above threshold, or None

class CounterpartyMatcher:
    """
    Fuzzy matching engine for counterparty names.

    Matching Strategy:
    1. Exact match (case-insensitive) - handled by CounterpartyStore.find_by_alias()
    2. Token match - extract common tokens ("starbucks", "amazon") and find overlap
    3. Levenshtein distance - character-level edit distance
    4. Jaro-Winkler distance - string similarity optimized for typos
    5. Hybrid - combine multiple methods with weighted scoring

    Confidence Thresholds:
    - High: similarity >= 0.85 (auto-match recommended)
    - Medium: 0.70 <= similarity < 0.85 (suggest to user)
    - Low: 0.50 <= similarity < 0.70 (show as option, don't auto-match)
    - Below 0.50: ignore (too dissimilar)
    """

    DEFAULT_THRESHOLD = 0.70  # Minimum similarity for match
    HIGH_CONFIDENCE_THRESHOLD = 0.85
    LOW_CONFIDENCE_THRESHOLD = 0.50

    def __init__(self, counterparty_store):
        self.store = counterparty_store

    def find_matches(
        self,
        user_id: str,
        raw_name: str,
        threshold: float = DEFAULT_THRESHOLD,
        method: MatchMethod = MatchMethod.HYBRID,
        max_results: int = 5
    ) -> MatchCandidates:
        """
        Find matching counterparties for raw name.

        Args:
            user_id: Owner of counterparties
            raw_name: Raw merchant name from transaction
            threshold: Minimum similarity score (0.0 to 1.0)
            method: Matching method to use
            max_results: Maximum number of results to return

        Returns:
            MatchCandidates with all matches above threshold

        Example:
            candidates = matcher.find_matches(
                user_id="user_123",
                raw_name="STARBUCKS #1234",
                threshold=0.70
            )
            if candidates.best_match:
                print(f"Match: {candidates.best_match.canonical_name}")
                print(f"Score: {candidates.best_match.similarity_score}")
                print(f"Confidence: {candidates.best_match.confidence}")
        """

    def exact_match(
        self,
        user_id: str,
        alias: str
    ) -> Optional[MatchResult]:
        """
        Find exact alias match (case-insensitive).

        This is a wrapper around CounterpartyStore.find_by_alias().

        Args:
            user_id: Owner of counterparties
            alias: Exact alias to match

        Returns:
            MatchResult with similarity_score=1.0 if found, None otherwise

        Example:
            match = matcher.exact_match("user_123", "amazon.com")
            # match.similarity_score = 1.0
            # match.match_method = MatchMethod.EXACT
        """

    def fuzzy_match(
        self,
        user_id: str,
        raw_name: str,
        threshold: float = DEFAULT_THRESHOLD
    ) -> Optional[MatchResult]:
        """
        Find best fuzzy match using hybrid algorithm.

        Args:
            user_id: Owner of counterparties
            raw_name: Raw merchant name from transaction
            threshold: Minimum similarity score

        Returns:
            Best MatchResult above threshold, or None

        Example:
            match = matcher.fuzzy_match("user_123", "STARBUCKS #1234")
            # match.canonical_name = "Starbucks"
            # match.similarity_score = 0.89
            # match.match_method = MatchMethod.HYBRID
        """

    def calculate_similarity(
        self,
        str_a: str,
        str_b: str,
        method: MatchMethod = MatchMethod.HYBRID
    ) -> float:
        """
        Calculate similarity score between two strings.

        Args:
            str_a: First string
            str_b: Second string
            method: Matching method to use

        Returns:
            Similarity score (0.0 to 1.0)

        Example:
            score = matcher.calculate_similarity("Amazon", "AMZN Mktp")
            # score = 0.72 (token match: "amazon" vs "amzn")
        """

    def _normalize_name(self, name: str) -> str:
        """
        Normalize name for matching.

        Rules:
        1. Lowercase
        2. Remove special characters (keep only alphanumeric and spaces)
        3. Remove common suffixes (inc, llc, corp, ltd)
        4. Strip whitespace
        5. Remove transaction identifiers (#1234, *123)

        Args:
            name: Raw name

        Returns:
            Normalized name

        Example:
            _normalize_name("STARBUCKS #1234") → "starbucks"
            _normalize_name("Amazon.com Inc") → "amazon com"
            _normalize_name("Dr. Smith MD") → "dr smith"
        """

    def _token_similarity(self, str_a: str, str_b: str) -> float:
        """
        Calculate token-based similarity (Jaccard index).

        Steps:
        1. Normalize both strings
        2. Split into tokens (words)
        3. Calculate Jaccard similarity: |A ∩ B| / |A ∪ B|

        Args:
            str_a: First string
            str_b: Second string

        Returns:
            Token similarity score (0.0 to 1.0)

        Example:
            _token_similarity("Starbucks Coffee", "Starbucks")
            # tokens_a = {"starbucks", "coffee"}
            # tokens_b = {"starbucks"}
            # intersection = {"starbucks"} (1 token)
            # union = {"starbucks", "coffee"} (2 tokens)
            # similarity = 1/2 = 0.50
        """

    def _levenshtein_similarity(self, str_a: str, str_b: str) -> float:
        """
        Calculate Levenshtein distance similarity.

        Levenshtein distance = minimum number of single-character edits (insertions, deletions, substitutions).
        Similarity = 1 - (distance / max_length)

        Args:
            str_a: First string
            str_b: Second string

        Returns:
            Levenshtein similarity score (0.0 to 1.0)

        Example:
            _levenshtein_similarity("amazon", "amzn")
            # distance = 2 (delete "a", delete "o")
            # max_length = 6
            # similarity = 1 - (2/6) = 0.67
        """

    def _jaro_winkler_similarity(self, str_a: str, str_b: str) -> float:
        """
        Calculate Jaro-Winkler distance similarity.

        Jaro-Winkler is optimized for short strings and typos.
        Gives higher score to strings that match from the beginning.

        Args:
            str_a: First string
            str_b: Second string

        Returns:
            Jaro-Winkler similarity score (0.0 to 1.0)

        Example:
            _jaro_winkler_similarity("DWAYNE", "DUANE")
            # Jaro-Winkler = 0.84 (high similarity despite typo)
        """

    def _hybrid_similarity(self, str_a: str, str_b: str) -> float:
        """
        Calculate hybrid similarity (weighted combination).

        Weights:
        - Token similarity: 50% (most important for merchant matching)
        - Levenshtein similarity: 30% (handle typos)
        - Jaro-Winkler similarity: 20% (handle short strings)

        Args:
            str_a: First string
            str_b: Second string

        Returns:
            Hybrid similarity score (0.0 to 1.0)

        Example:
            _hybrid_similarity("STARBUCKS #1234", "Starbucks")
            # token = 1.0 (perfect token match)
            # levenshtein = 0.65
            # jaro_winkler = 0.72
            # hybrid = 0.5*1.0 + 0.3*0.65 + 0.2*0.72 = 0.84
        """

    def _get_confidence_level(self, similarity_score: float) -> str:
        """
        Convert similarity score to confidence level.

        Args:
            similarity_score: Score (0.0 to 1.0)

        Returns:
            Confidence level ("high", "medium", "low")
        """
```

---

## Implementation Details

### Normalization Logic

```python
import re

def _normalize_name(self, name: str) -> str:
    """
    Normalize merchant name for matching.

    Examples:
      "STARBUCKS #1234" → "starbucks"
      "Amazon.com Inc" → "amazon com"
      "Dr. Smith, MD" → "dr smith"
      "Shell*Gas 123" → "shell gas"
    """
    # 1. Lowercase
    normalized = name.lower()

    # 2. Remove transaction identifiers (#1234, *123, etc.)
    normalized = re.sub(r'[#*]\d+', '', normalized)

    # 3. Remove special characters (keep only alphanumeric and spaces)
    normalized = re.sub(r'[^a-z0-9\s]', ' ', normalized)

    # 4. Remove common business suffixes
    suffixes = ['inc', 'llc', 'corp', 'ltd', 'co', 'company', 'corporation']
    for suffix in suffixes:
        # Remove suffix only if it's a separate word (not part of another word)
        normalized = re.sub(rf'\b{suffix}\b', '', normalized)

    # 5. Collapse multiple spaces to single space
    normalized = re.sub(r'\s+', ' ', normalized)

    # 6. Strip whitespace
    normalized = normalized.strip()

    return normalized
```

### Token Similarity (Jaccard Index)

```python
def _token_similarity(self, str_a: str, str_b: str) -> float:
    """
    Calculate Jaccard similarity: |A ∩ B| / |A ∪ B|

    Examples:
      "Starbucks Coffee" vs "Starbucks"
      tokens_a = {"starbucks", "coffee"}
      tokens_b = {"starbucks"}
      similarity = 1/2 = 0.50

      "Amazon Prime" vs "Amazon"
      tokens_a = {"amazon", "prime"}
      tokens_b = {"amazon"}
      similarity = 1/2 = 0.50

      "Shell Gas" vs "Shell Gas Station"
      tokens_a = {"shell", "gas"}
      tokens_b = {"shell", "gas", "station"}
      similarity = 2/3 = 0.67
    """
    # Normalize and tokenize
    tokens_a = set(self._normalize_name(str_a).split())
    tokens_b = set(self._normalize_name(str_b).split())

    # Handle empty sets
    if not tokens_a or not tokens_b:
        return 0.0

    # Calculate Jaccard index
    intersection = len(tokens_a & tokens_b)
    union = len(tokens_a | tokens_b)

    if union == 0:
        return 0.0

    return intersection / union
```

### Levenshtein Distance

```python
def _levenshtein_distance(self, str_a: str, str_b: str) -> int:
    """
    Calculate minimum edit distance between two strings.

    Dynamic programming approach.

    Examples:
      "amazon" vs "amzn" → distance = 2
      "starbucks" vs "strbucks" → distance = 1
    """
    m, n = len(str_a), len(str_b)

    # Create DP table
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Initialize base cases
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    # Fill DP table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if str_a[i - 1] == str_b[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]  # No edit needed
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # Delete
                    dp[i][j - 1],      # Insert
                    dp[i - 1][j - 1]   # Substitute
                )

    return dp[m][n]

def _levenshtein_similarity(self, str_a: str, str_b: str) -> float:
    """
    Convert Levenshtein distance to similarity score.

    similarity = 1 - (distance / max_length)
    """
    # Normalize strings
    norm_a = self._normalize_name(str_a)
    norm_b = self._normalize_name(str_b)

    # Calculate distance
    distance = self._levenshtein_distance(norm_a, norm_b)

    # Convert to similarity
    max_length = max(len(norm_a), len(norm_b))
    if max_length == 0:
        return 0.0

    return 1.0 - (distance / max_length)
```

### Jaro-Winkler Similarity

```python
def _jaro_winkler_similarity(self, str_a: str, str_b: str) -> float:
    """
    Calculate Jaro-Winkler similarity.

    Optimized for typos and short strings.
    Gives bonus for common prefix.

    Uses third-party library: jellyfish or python-Levenshtein
    """
    import jellyfish

    # Normalize strings
    norm_a = self._normalize_name(str_a)
    norm_b = self._normalize_name(str_b)

    if not norm_a or not norm_b:
        return 0.0

    # Calculate Jaro-Winkler distance
    return jellyfish.jaro_winkler_similarity(norm_a, norm_b)
```

### Hybrid Matching Algorithm

```python
def _hybrid_similarity(self, str_a: str, str_b: str) -> float:
    """
    Weighted combination of multiple similarity methods.

    Weights tuned for merchant name matching:
    - Token: 50% (most important - captures core business name)
    - Levenshtein: 30% (handles typos and abbreviations)
    - Jaro-Winkler: 20% (handles short strings and prefix matching)
    """
    token_sim = self._token_similarity(str_a, str_b)
    levenshtein_sim = self._levenshtein_similarity(str_a, str_b)
    jaro_sim = self._jaro_winkler_similarity(str_a, str_b)

    # Weighted average
    hybrid_sim = (
        0.50 * token_sim +
        0.30 * levenshtein_sim +
        0.20 * jaro_sim
    )

    return hybrid_sim
```

### Find Matches Implementation

```python
def find_matches(
    self,
    user_id: str,
    raw_name: str,
    threshold: float = 0.70,
    method: MatchMethod = MatchMethod.HYBRID,
    max_results: int = 5
) -> MatchCandidates:
    """
    Find all matching counterparties above threshold.

    Algorithm:
    1. Get all active counterparties for user
    2. For each counterparty, calculate similarity against all aliases
    3. Keep best similarity score per counterparty
    4. Filter by threshold
    5. Sort by similarity score DESC
    6. Return top N results
    """
    # 1. Get all counterparties
    counterparties = self.store.list(user_id=user_id, is_active=True)

    # 2. Calculate similarity for each counterparty
    match_results = []

    for cp in counterparties:
        best_score = 0.0
        best_alias = None

        # Check similarity against all aliases
        for alias in cp.aliases:
            if method == MatchMethod.TOKEN:
                score = self._token_similarity(raw_name, alias)
            elif method == MatchMethod.LEVENSHTEIN:
                score = self._levenshtein_similarity(raw_name, alias)
            elif method == MatchMethod.JARO_WINKLER:
                score = self._jaro_winkler_similarity(raw_name, alias)
            else:  # HYBRID
                score = self._hybrid_similarity(raw_name, alias)

            if score > best_score:
                best_score = score
                best_alias = alias

        # 3. Keep if above threshold
        if best_score >= threshold:
            confidence = self._get_confidence_level(best_score)
            match_results.append(MatchResult(
                counterparty_id=cp.counterparty_id,
                canonical_name=cp.canonical_name,
                matched_alias=best_alias,
                similarity_score=best_score,
                match_method=method,
                confidence=confidence
            ))

    # 4. Sort by similarity score DESC
    match_results.sort(key=lambda m: m.similarity_score, reverse=True)

    # 5. Limit results
    match_results = match_results[:max_results]

    # 6. Return
    best_match = match_results[0] if match_results else None

    return MatchCandidates(
        raw_name=raw_name,
        matches=match_results,
        best_match=best_match
    )
```

---

## Usage Examples

### Example 1: Exact Match (No Fuzzy Needed)

```python
matcher = CounterpartyMatcher(counterparty_store)

# Exact alias match
match = matcher.exact_match("user_123", "Amazon.com")

print(match.similarity_score)  # 1.0 (perfect match)
print(match.match_method)  # MatchMethod.EXACT
print(match.confidence)  # "high"
```

### Example 2: Fuzzy Match (Transaction Identifier)

```python
# Transaction arrives: "STARBUCKS #1234"
# User has counterparty "Starbucks" with alias "Starbucks"

match = matcher.fuzzy_match("user_123", "STARBUCKS #1234", threshold=0.70)

print(match.canonical_name)  # "Starbucks"
print(match.similarity_score)  # 0.95 (high - normalized to "starbucks")
print(match.confidence)  # "high"
print(match.match_method)  # MatchMethod.HYBRID
```

### Example 3: Multiple Candidates

```python
# Transaction: "Shell Gas Station"
# User has:
# - "Shell Gas" (alias: "Shell Gas")
# - "Shell" (alias: "Shell")
# - "Gas Station" (alias: "Gas Station")

candidates = matcher.find_matches("user_123", "Shell Gas Station", threshold=0.60)

for match in candidates.matches:
    print(f"{match.canonical_name}: {match.similarity_score:.2f} ({match.confidence})")

# Output:
# Shell Gas: 0.85 (high)
# Shell: 0.67 (medium)
# Gas Station: 0.67 (medium)

print(candidates.best_match.canonical_name)  # "Shell Gas"
```

### Example 4: Abbreviation Matching

```python
# Transaction: "AMZN Mktp"
# User has counterparty "Amazon" with aliases: {"Amazon", "Amazon.com"}

match = matcher.fuzzy_match("user_123", "AMZN Mktp", threshold=0.70)

print(match.canonical_name)  # "Amazon"
print(match.similarity_score)  # 0.72 (medium - token match on "amzn" vs "amazon")
print(match.confidence)  # "medium"
```

### Example 5: No Match (Below Threshold)

```python
# Transaction: "Random Store XYZ"
# User has no similar counterparties

match = matcher.fuzzy_match("user_123", "Random Store XYZ", threshold=0.70)

print(match)  # None (no match above threshold)

# With lower threshold
candidates = matcher.find_matches("user_123", "Random Store XYZ", threshold=0.40)
print(len(candidates.matches))  # 0 or very few low-confidence matches
```

---

## Multi-Domain Examples

### Healthcare: Provider Matching

```python
# Medical claim: "Dr Smith MD"
# User has provider: "Smith Family Practice"

match = matcher.fuzzy_match("patient_123", "Dr Smith MD", threshold=0.70)
# match.canonical_name = "Smith Family Practice"
# match.similarity_score = 0.78 (token match on "smith")
```

### Legal: Law Firm Matching

```python
# Billing record: "Jones & Assoc"
# User has firm: "Jones & Associates"

match = matcher.fuzzy_match("lawyer_456", "Jones & Assoc", threshold=0.70)
# match.canonical_name = "Jones & Associates"
# match.similarity_score = 0.92 (high similarity)
```

### Research: Institution Matching

```python
# Publication: "Stanford Univ"
# User has institution: "Stanford University"

match = matcher.fuzzy_match("researcher_789", "Stanford Univ", threshold=0.70)
# match.canonical_name = "Stanford University"
# match.similarity_score = 0.88 (high similarity)
```

---

## Performance Characteristics

### Complexity Analysis

| Operation | Time Complexity | Space Complexity |
|-----------|----------------|------------------|
| `_normalize_name()` | O(n) | O(n) |
| `_token_similarity()` | O(n + m) | O(n + m) |
| `_levenshtein_distance()` | O(n × m) | O(n × m) |
| `_jaro_winkler_similarity()` | O(n) | O(1) |
| `find_matches()` | O(C × A × M) | O(C × A) |

Where:
- n, m = string lengths
- C = number of counterparties
- A = average aliases per counterparty
- M = matching algorithm complexity

### Performance Optimization

```python
# Cache normalized names and alias maps
class CounterpartyMatcher:
    def __init__(self, counterparty_store):
        self.store = counterparty_store
        self._normalized_cache = {}  # Cache normalized strings
        self._alias_map = None       # Cache user's alias→counterparty mapping

    def _get_normalized(self, name: str) -> str:
        """Get normalized name from cache."""
        if name not in self._normalized_cache:
            self._normalized_cache[name] = self._normalize_name(name)
        return self._normalized_cache[name]

    def _build_alias_map(self, user_id: str):
        """Build alias→counterparty_id map for fast lookup."""
        if self._alias_map is None:
            self._alias_map = {}
            counterparties = self.store.list(user_id=user_id, is_active=True)
            for cp in counterparties:
                for alias in cp.aliases:
                    self._alias_map[alias.lower()] = cp.counterparty_id
        return self._alias_map
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Normalization
def test_normalize_removes_transaction_id()
def test_normalize_removes_special_chars()
def test_normalize_removes_business_suffixes()
def test_normalize_case_insensitive()

# Test: Token similarity
def test_token_similarity_perfect_match()
def test_token_similarity_partial_match()
def test_token_similarity_no_match()

# Test: Levenshtein similarity
def test_levenshtein_exact_match()
def test_levenshtein_one_char_diff()
def test_levenshtein_abbreviation()

# Test: Jaro-Winkler similarity
def test_jaro_winkler_typo()
def test_jaro_winkler_prefix_match()

# Test: Hybrid similarity
def test_hybrid_combines_methods()
def test_hybrid_weights_token_highest()

# Test: Find matches
def test_find_matches_exact()
def test_find_matches_fuzzy()
def test_find_matches_multiple_candidates()
def test_find_matches_below_threshold()
def test_find_matches_max_results()

# Test: Confidence levels
def test_confidence_high()
def test_confidence_medium()
def test_confidence_low()
```

---

## Related Primitives

- **CounterpartyStore** (OL) - Uses CounterpartyMatcher in find_or_create() workflow
- **AliasMerger** (OL) - Uses CounterpartyMatcher to identify merge candidates
- **CounterpartySelector** (IL) - UI component that shows fuzzy match suggestions to user

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 2-3 days
**Dependencies:** jellyfish or python-Levenshtein library
