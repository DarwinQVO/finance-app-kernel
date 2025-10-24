# Primitive: FuzzyMatcher (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Similarity Calculator
> **Vertical:** 3.8 Cluster Rules
> **Last Updated:** 2025-10-24

---

## Overview

**FuzzyMatcher** is an Objective Layer primitive that calculates text similarity scores using multiple algorithms (Levenshtein, Jaro-Winkler, Soundex, Metaphone). It enables fuzzy string matching for entity deduplication and normalization.

**Core Responsibility:**
Calculate similarity scores between text strings to support fuzzy matching, clustering, and entity resolution across any domain requiring approximate string matching.

**Domain Instantiation:**
- Finance: Match merchant name variants ("MCDONALDS #1234" ≈ "McDonald's")
- Healthcare: Match provider names with misspellings ("KAISER PERM" ≈ "Kaiser Permanente")
- Legal: Match case names with different formats ("DOE V. SMITH" ≈ "Doe v. Smith")
- Research: Match author/institution names with variations ("MIT CSAIL" ≈ "MIT")

---

## Interface

### Methods

```python
class FuzzyMatcher:
    """
    Calculates text similarity using multiple algorithms.

    Attributes:
        default_algorithm: Default algorithm to use ("jaro_winkler")
        min_threshold: Minimum similarity threshold (0.70)
    """

    def calculate_similarity(self, text1: str, text2: str,
                            algorithm: str = "jaro_winkler",
                            case_sensitive: bool = False) -> float:
        """
        Calculates similarity score between two strings.

        Algorithms:
        - levenshtein: Edit distance (character insertions/deletions/substitutions)
          Range: [0.0, 1.0], 1.0 = identical
          Good for: Typos, character-level differences
          Example: "kitten" vs "sitting" → 0.57

        - jaro_winkler: String similarity with prefix weighting
          Range: [0.0, 1.0], 1.0 = identical
          Good for: General text similarity, person names
          Example: "MARTHA" vs "MARHTA" → 0.96 (transposition)

        - soundex: Phonetic matching (English pronunciation)
          Range: [0.0, 1.0], 1.0 = same pronunciation
          Good for: Name matching with misspellings
          Example: "Smith" vs "Smyth" → 1.0 (both S530)

        - metaphone: Improved phonetic matching
          Range: [0.0, 1.0], 1.0 = same pronunciation
          Good for: Name matching, better than Soundex
          Example: "knight" vs "night" → 1.0 (both NT)

        Args:
            text1: First string
            text2: Second string
            algorithm: "levenshtein" | "jaro_winkler" | "soundex" | "metaphone"
            case_sensitive: If True, preserve case (default False)

        Returns:
            Similarity score [0.0, 1.0]

        Example:
            >>> matcher = FuzzyMatcher()
            >>> matcher.calculate_similarity("MCDONALDS", "McDonald's", "jaro_winkler")
            0.92
            >>> matcher.calculate_similarity("UBER EATS", "UBER", "levenshtein")
            0.67
        """

    def find_best_matches(self, text: str, candidates: List[str],
                          threshold: float = 0.70,
                          algorithm: str = "jaro_winkler",
                          max_results: int = 10) -> List[Match]:
        """
        Finds candidates above similarity threshold, sorted by score.

        Args:
            text: Text to match
            candidates: List of candidate strings
            threshold: Minimum similarity [0.0, 1.0] (default 0.70)
            algorithm: "levenshtein" | "jaro_winkler" | "soundex" | "metaphone"
            max_results: Maximum matches to return (default 10)

        Returns:
            List of Match objects sorted by similarity desc

        Example:
            >>> candidates = ["MCDONALDS #1234", "MCDONALD'S", "BURGER KING"]
            >>> matches = matcher.find_best_matches("McDonald's", candidates, threshold=0.80)
            >>> len(matches)
            2  # MCDONALDS #1234 (0.87), MCDONALD'S (0.95)
            >>> matches[0].candidate
            "MCDONALD'S"
            >>> matches[0].similarity
            0.95
        """

    def get_algorithm(self, name: str) -> SimilarityAlgorithm:
        """
        Returns algorithm implementation by name.

        Args:
            name: "levenshtein" | "jaro_winkler" | "soundex" | "metaphone"

        Returns:
            SimilarityAlgorithm instance

        Raises:
            UnknownAlgorithmError: If name invalid
        """

    def batch_calculate(self, pairs: List[Tuple[str, str]],
                        algorithm: str = "jaro_winkler") -> List[float]:
        """
        Batch similarity calculation for performance.

        Args:
            pairs: List of (text1, text2) tuples
            algorithm: Algorithm to use

        Returns:
            List of similarity scores in same order as pairs

        Example:
            >>> pairs = [("UBER EATS", "Uber"), ("AMAZON", "AMZN")]
            >>> scores = matcher.batch_calculate(pairs)
            >>> scores
            [0.67, 0.78]
        """

    def calculate_with_confidence(self, text1: str, text2: str,
                                   algorithm: str = "jaro_winkler") -> ConfidenceScore:
        """
        Calculates similarity with confidence interval.

        Returns:
            ConfidenceScore with:
                - similarity: [0.0, 1.0]
                - confidence_lower: Lower bound (95% CI)
                - confidence_upper: Upper bound (95% CI)
                - is_ambiguous: True if multiple close matches

        Example:
            >>> score = matcher.calculate_with_confidence("MCDON", "McDonald's")
            >>> score.similarity
            0.82
            >>> score.is_ambiguous
            False
        """
```

---

## Data Model

### Match

```python
@dataclass
class Match:
    """
    Result of fuzzy match.

    Attributes:
        candidate: Matched candidate string
        similarity: Similarity score [0.0, 1.0]
        algorithm: Algorithm used
        query: Original query text
    """
    candidate: str
    similarity: float
    algorithm: str
    query: str
```

### ConfidenceScore

```python
@dataclass
class ConfidenceScore:
    """
    Similarity with confidence interval.

    Attributes:
        similarity: Similarity score [0.0, 1.0]
        confidence_lower: Lower bound (95% CI)
        confidence_upper: Upper bound (95% CI)
        is_ambiguous: True if multiple close matches exist
    """
    similarity: float
    confidence_lower: float
    confidence_upper: float
    is_ambiguous: bool
```

---

## Algorithms

### Levenshtein Distance

**Description:**
Minimum number of single-character edits (insertions, deletions, substitutions) to transform text1 into text2.

**Normalized Formula:**
```
similarity = 1 - (levenshtein_distance / max(len(text1), len(text2)))
```

**Example:**
```python
text1 = "kitten"  # 6 chars
text2 = "sitting" # 7 chars

# Edit path:
# kitten → sitten (substitute k→s)
# sitten → sittin (substitute e→i)
# sittin → sitting (insert g)

distance = 3
similarity = 1 - (3 / 7) = 0.57
```

**Use Cases:**
- Typo detection
- Spelling correction
- Character-level differences

### Jaro-Winkler Distance

**Description:**
String similarity with prefix weighting (common prefixes increase score).

**Formula:**
```
jaro = (1/3) * (m/|text1| + m/|text2| + (m-t)/m)
where:
  m = matching characters
  t = transpositions / 2

jaro_winkler = jaro + (L * P * (1 - jaro))
where:
  L = common prefix length (max 4)
  P = prefix scaling factor (0.1)
```

**Example:**
```python
text1 = "MARTHA"
text2 = "MARHTA"

# Matching chars: M, A, R, H, T, A (6 out of 6)
# Transpositions: T-H swapped (1 transposition)

jaro = (1/3) * (6/6 + 6/6 + (6-1)/6) = 0.94
prefix = "MAR" (3 chars)
jaro_winkler = 0.94 + (3 * 0.1 * (1 - 0.94)) = 0.96
```

**Use Cases:**
- Person names
- General text similarity
- Transposition tolerance

### Soundex (Phonetic Matching)

**Description:**
Encodes strings by pronunciation (English), assigns same code to similar-sounding words.

**Algorithm:**
```
1. Keep first letter
2. Remove vowels (A, E, I, O, U, H, W, Y)
3. Replace consonants with digits:
   - B, F, P, V → 1
   - C, G, J, K, Q, S, X, Z → 2
   - D, T → 3
   - L → 4
   - M, N → 5
   - R → 6
4. Pad with zeros to 4 chars
```

**Example:**
```python
text1 = "Smith"
# S → S (keep first)
# m → 5
# i → remove (vowel)
# t → 3
# h → remove
# Result: S530

text2 = "Smyth"
# S → S
# m → 5
# y → remove
# t → 3
# h → remove
# Result: S530

similarity = 1.0 (same code)
```

**Use Cases:**
- Name matching with misspellings
- Database search (fuzzy name lookup)

### Metaphone (Improved Phonetic)

**Description:**
Improved Soundex, better handles English pronunciation rules.

**Improvements:**
- Handles consonant clusters (TH, CH, SH)
- Better vowel handling
- Longer codes (more precision)

**Example:**
```python
text1 = "knight"
# Metaphone: NT

text2 = "night"
# Metaphone: NT

similarity = 1.0 (same code)
```

**Use Cases:**
- Name matching
- Better accuracy than Soundex

---

## Multi-Domain Examples

### Finance: Merchant Name Matching

```python
matcher = FuzzyMatcher()

# Exact match
matcher.calculate_similarity("UBER EATS", "UBER EATS", "jaro_winkler")
# → 1.0

# High similarity (partial match)
matcher.calculate_similarity("UBER EATS PENDING", "Uber Eats", "jaro_winkler")
# → 0.85

# Medium similarity (abbreviation)
matcher.calculate_similarity("AMZN MKTP US", "Amazon", "jaro_winkler")
# → 0.65

# Low similarity (different)
matcher.calculate_similarity("UBER", "WALMART", "jaro_winkler")
# → 0.15

# Find best matches
candidates = [
    "MCDONALDS #1234",
    "MCDONALD'S",
    "MC DONALDS",
    "BURGER KING"
]
matches = matcher.find_best_matches("McDonald's", candidates, threshold=0.80)
# → [
#     Match("MCDONALD'S", 0.95),
#     Match("MCDONALDS #1234", 0.87),
#     Match("MC DONALDS", 0.82)
# ]
```

### Healthcare: Provider Name Matching

```python
# Hospital name variations
matcher.calculate_similarity(
    "ST MARY'S HOSP",
    "St. Mary's Hospital",
    "jaro_winkler"
)
# → 0.90

matcher.calculate_similarity(
    "KAISER PERM",
    "Kaiser Permanente",
    "levenshtein"
)
# → 0.75

# Phonetic matching for misspellings
matcher.calculate_similarity(
    "MASSACHUSETTS GEN",
    "Mass General Hosp",
    "metaphone"
)
# → 0.85 (similar pronunciation)
```

### Legal: Case Name Matching

```python
# Case name format variations
matcher.calculate_similarity(
    "DOE V. SMITH",
    "Doe v. Smith",
    "jaro_winkler"
)
# → 0.98 (case difference only)

matcher.calculate_similarity(
    "JOHNSON VS ACME CORP",
    "Johnson v. ACME Corporation",
    "levenshtein"
)
# → 0.82
```

### Research: Author Name Matching

```python
# Author name variations
candidates = [
    "J. Smith",
    "John Smith",
    "Smith, J.",
    "Smith, John",
    "J. R. Smith"
]

matches = matcher.find_best_matches(
    "John Smith",
    candidates,
    threshold=0.70,
    algorithm="jaro_winkler"
)
# → All variants matched (similarity > 0.70)

# Institution abbreviations
matcher.calculate_similarity(
    "MIT CSAIL",
    "MIT Computer Science and AI Lab",
    "jaro_winkler"
)
# → 0.75
```

---

## Performance Characteristics

### Latency Targets

- `calculate_similarity()`: <1ms per pair
- `find_best_matches()` 100 candidates: <50ms p95
- `batch_calculate()` 1000 pairs: <100ms p95

### Algorithm Complexity

| Algorithm    | Time Complexity | Space Complexity |
|-------------|-----------------|------------------|
| Levenshtein | O(m * n)       | O(m * n)        |
| Jaro-Winkler| O(m + n)       | O(1)            |
| Soundex     | O(n)           | O(1)            |
| Metaphone   | O(n)           | O(1)            |

where m, n = lengths of text1, text2

### Optimization Strategies

1. **Pre-computation:**
   - Cache Soundex/Metaphone codes
   - Compute once, reuse for matching

2. **Early Termination:**
   - If len(text1) - len(text2) > threshold * max_len, skip calculation

3. **Parallel Processing:**
   - Use multiprocessing for large candidate sets
   - Split candidates across CPU cores

---

## Error Handling

```python
class UnknownAlgorithmError(Exception):
    """Raised when algorithm name invalid."""
    pass

# Example
if algorithm not in ["levenshtein", "jaro_winkler", "soundex", "metaphone"]:
    raise UnknownAlgorithmError(f"Unknown algorithm: {algorithm}")
```

---

## Testing

### Unit Tests

```python
def test_levenshtein_identical():
    score = matcher.calculate_similarity("hello", "hello", "levenshtein")
    assert score == 1.0

def test_levenshtein_one_edit():
    score = matcher.calculate_similarity("kitten", "sitten", "levenshtein")
    expected = 1 - (1/6)  # 1 edit, 6 chars
    assert abs(score - expected) < 0.01

def test_jaro_winkler_transposition():
    score = matcher.calculate_similarity("MARTHA", "MARHTA", "jaro_winkler")
    assert score > 0.95  # High score despite transposition

def test_soundex_phonetic_match():
    score = matcher.calculate_similarity("Smith", "Smyth", "soundex")
    assert score == 1.0  # Same Soundex code

def test_find_best_matches_threshold():
    candidates = ["MCDONALDS #1234", "MCDONALD'S", "BURGER KING"]
    matches = matcher.find_best_matches("McDonald's", candidates, threshold=0.80)
    assert len(matches) == 2  # Only 2 above 0.80
    assert matches[0].similarity > matches[1].similarity  # Sorted desc

def test_batch_calculate():
    pairs = [("UBER", "UBER EATS"), ("AMAZON", "AMZN")]
    scores = matcher.batch_calculate(pairs, "jaro_winkler")
    assert len(scores) == 2
    assert all(0.0 <= s <= 1.0 for s in scores)
```

---

## Security Considerations

### Resource Limits

- Max string length: 500 chars (prevent DoS)
- Timeout calculation at 100ms (prevent hang)
- Max candidates for `find_best_matches()`: 10,000

### Input Validation

```python
def validate_input(text: str) -> None:
    """
    Validates input text.

    Raises:
        ValidationError: If text invalid
    """
    if len(text) > 500:
        raise ValidationError("Text exceeds 500 chars")
    if not text:
        raise ValidationError("Text cannot be empty")
```

---

## Observability

### Metrics

```
fuzzy_match.calculations.count (counter, labels: algorithm)
fuzzy_match.latency (histogram, labels: algorithm, candidate_count)
fuzzy_match.threshold_misses.count (counter, labels: algorithm)
```

### Structured Logs

```json
{
  "event": "fuzzy_match_calculated",
  "text1": "MCDONALDS",
  "text2": "McDonald's",
  "algorithm": "jaro_winkler",
  "similarity": 0.92,
  "latency_ms": 0.5,
  "timestamp": "2025-10-24T10:00:00Z"
}
```

---

## Related Primitives

- **MerchantNormalizer** - Uses fuzzy matching for normalization
- **ClusteringEngine** - Uses similarity scores for clustering
- **NormalizationRuleStore** - Stores fuzzy matching rules

---

## Summary

**FuzzyMatcher** provides text similarity calculation with:
- ✅ 4 algorithms (Levenshtein, Jaro-Winkler, Soundex, Metaphone)
- ✅ Threshold-based filtering
- ✅ Batch processing for performance
- ✅ Confidence scoring
- ✅ Multi-domain applicability (finance, healthcare, legal, research)
- ✅ <1ms latency per calculation

**Universal Pattern:**
Applicable to any domain requiring approximate string matching and entity deduplication.
