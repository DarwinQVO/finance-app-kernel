# MatchScorer (OL Primitive)

**Vertical:** 3.9 Reconciliation Strategies (Multi-Source Transaction Matching)
**Layer:** Objective Layer (OL)
**Type:** Similarity Calculation Engine
**Status:** ✅ Specified

---

## Purpose

Calculate weighted similarity scores for match candidates by comparing features (amount, date, counterparty name, description) with configurable tolerances and penalties. Produces granular confidence scores that enable threshold-based automated decisions.

**Core Responsibility:** Compute feature-level similarity scores (0.0-1.0) for each comparison dimension, apply tolerance-based penalties, combine scores using weighted aggregation, and return overall confidence with detailed breakdowns for explainability.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY entity similarity calculation:

| Domain | Amount Feature | Date Feature | Counterparty Feature | Description Feature | Custom Features |
|--------|----------------|--------------|----------------------|---------------------|-----------------|
| **Finance** | Transaction amount (±5%) | Transaction date (±7 days) | Merchant/Vendor name | Description text | Currency, account type |
| **Healthcare** | Claim amount (±10%) | Service date (±60 days) | Patient name | Procedure codes (CPT) | Provider ID, diagnosis |
| **Legal** | Filing fee (exact) | Filing date (±30 days) | Party names | Document type | Case number, jurisdiction |
| **Research (RSRCH - Utilitario)** | Investment amount (±10%) | Discovery date (±30 days) | Entity names (founders/companies) | Fact claim text | Source type, source credibility |
| **E-commerce** | Order amount (±5%) | Order date (±14 days) | Customer name | Product SKUs | Shipping address |
| **Logistics** | Shipment value (±10%) | Ship date (±7 days) | Consignee name | Tracking number | Origin/destination |

**Pattern:** For each feature, calculate raw similarity [0.0, 1.0], apply tolerance penalty if variance outside bounds, multiply by feature weight, sum weighted scores to get overall confidence.

---

## Interface Contract

### Python Interface

```python
from typing import Dict, Optional, Tuple
from dataclasses import dataclass
from datetime import date
from decimal import Decimal
from enum import Enum

@dataclass
class MatchScore:
    """Complete scoring breakdown for match candidate."""
    confidence: float  # Overall weighted confidence [0.0, 1.0]
    decision: 'MatchDecision'  # Based on confidence thresholds

    # Feature-level scores [0.0, 1.0]
    scores: Dict[str, float]  # {amount: 1.0, date: 0.95, counterparty: 0.95, description: 0.0}

    # Feature weights (sum to 1.0)
    weights: Dict[str, float]  # {amount: 0.4, date: 0.3, counterparty: 0.2, description: 0.1}

    # Feature details for explainability
    features: Dict[str, any]  # {amount_diff: 0.00, date_diff_days: 1, counterparty_similarity: 0.95}

    # Penalties applied
    penalties: Dict[str, str]  # {amount: "none", date: "1_day_difference", counterparty: "minor_variation"}

class MatchScorer:
    """
    Calculate weighted similarity scores for match candidates.

    Supports configurable features, weights, and tolerances.
    Default features: amount, date, counterparty, description
    Custom features: domain-specific (e.g., currency, patient_id, tracking_number)

    Scoring algorithm:
    1. For each feature, calculate raw similarity [0.0, 1.0]
    2. Apply tolerance penalty if variance outside bounds
    3. Multiply by feature weight
    4. Sum weighted scores = overall confidence
    """

    def __init__(self, fuzzy_matcher: 'FuzzyMatcher'):
        self.fuzzy_matcher = fuzzy_matcher  # For string similarity (Jaro-Winkler, Levenshtein)

    def calculate_confidence(
        self,
        item_a: 'ReconciliationItem',
        item_b: 'ReconciliationItem',
        config: 'ReconciliationConfig'
    ) -> MatchScore:
        """
        Calculate weighted confidence score for match candidate.

        Algorithm:
        1. Score amount: 1.0 if exact, penalty for % difference (within tolerance)
        2. Score date: 1.0 if same day, decay based on days difference (within tolerance)
        3. Score counterparty: String similarity (Jaro-Winkler) on names
        4. Score description: String similarity (optional, if both non-null)
        5. Compute weighted confidence: Σ(score_i * weight_i)

        Args:
            item_a: First item (e.g., bank transaction)
            item_b: Second item (e.g., invoice)
            config: ReconciliationConfig with weights and tolerances

        Returns:
            MatchScore with confidence, feature scores, weights, details, penalties

        Example (Finance - High Confidence Match):
            item_a = BankTransaction(amount=-5000.00, date="2024-10-15", counterparty="Acme Corp")
            item_b = Invoice(amount=5000.00, date="2024-10-14", vendor="Acme Corporation")

            score = scorer.calculate_confidence(item_a, item_b, config)

            # Returns:
            MatchScore(
                confidence=0.98,
                decision=MatchDecision.AUTO_LINK,
                scores={amount: 1.0, date: 0.95, counterparty: 0.95, description: 0.0},
                weights={amount: 0.4, date: 0.3, counterparty: 0.2, description: 0.1},
                features={amount_diff: 0.00, date_diff_days: 1, counterparty_similarity: 0.95},
                penalties={amount: "none", date: "1_day", counterparty: "minor_variation"}
            )

        Example (Finance - Medium Confidence Match):
            item_a = BankTransaction(amount=-142.50, date="2024-10-20", counterparty="AMZN MKTP US")
            item_b = Invoice(amount=142.50, date="2024-10-22", vendor="Amazon.com")

            score = scorer.calculate_confidence(item_a, item_b, config)

            # Returns:
            MatchScore(
                confidence=0.82,
                decision=MatchDecision.AUTO_SUGGEST,
                scores={amount: 1.0, date: 0.90, counterparty: 0.75, description: 0.0},
                features={amount_diff: 0.00, date_diff_days: 2, counterparty_similarity: 0.75}
            )

        Example (Healthcare - Amount Variance):
            claim = Claim(amount=1250.00, date="2024-09-15", patient="John Doe")
            payment = Payment(amount=1125.00, date="2024-10-30", patient="John Doe")  # 10% co-pay

            score = scorer.calculate_confidence(claim, payment, healthcare_config)

            # Returns:
            MatchScore(
                confidence=0.85,
                scores={amount: 0.90, date: 0.70, counterparty: 1.0, description: 0.0},  # Amount penalty for 10% diff
                features={amount_diff: 125.00, amount_diff_pct: 0.10, date_diff_days: 45},
                penalties={amount: "10_percent_variance", date: "45_days"}
            )
        """
        pass

    def score_amount(
        self,
        amount_a: Decimal,
        amount_b: Decimal,
        tolerance_pct: Decimal
    ) -> Tuple[float, Dict]:
        """
        Calculate amount similarity score with tolerance.

        Algorithm:
        1. Compute absolute difference: abs(amount_a - amount_b)
        2. Compute percentage difference: diff / min(amount_a, amount_b)
        3. If diff_pct == 0.0: score = 1.0 (exact match)
        4. If diff_pct <= tolerance_pct: score = 1.0 - (diff_pct / tolerance_pct) * 0.05  # Small penalty
        5. If diff_pct > tolerance_pct: score = 0.0 (outside tolerance)

        Args:
            amount_a: Amount from item A (can be negative for transactions)
            amount_b: Amount from item B
            tolerance_pct: Acceptable variance as decimal (0.05 = 5%)

        Returns:
            (score [0.0, 1.0], details {amount_diff, amount_diff_pct, penalty})

        Examples (Finance - 5% tolerance):
            score_amount(100.00, 100.00, 0.05) → (1.0, {diff: 0.00, diff_pct: 0.00, penalty: "none"})
            score_amount(100.00, 102.00, 0.05) → (0.96, {diff: 2.00, diff_pct: 0.02, penalty: "2_percent"})
            score_amount(100.00, 110.00, 0.05) → (0.0, {diff: 10.00, diff_pct: 0.10, penalty: "outside_tolerance"})

        Examples (Healthcare - 10% tolerance for adjustments):
            score_amount(1250.00, 1125.00, 0.10) → (0.90, {diff: 125.00, diff_pct: 0.10, penalty: "at_tolerance_limit"})
        """
        pass

    def score_date(
        self,
        date_a: date,
        date_b: date,
        tolerance_days: int
    ) -> Tuple[float, Dict]:
        """
        Calculate date similarity score with decay function.

        Algorithm:
        1. Compute days difference: abs((date_a - date_b).days)
        2. If days_diff == 0: score = 1.0 (same day)
        3. If days_diff <= tolerance_days: score = 1.0 - (days_diff / tolerance_days) * 0.20  # Linear decay
        4. If days_diff > tolerance_days: score = 0.0 (outside tolerance)

        Args:
            date_a: Date from item A
            date_b: Date from item B
            tolerance_days: Acceptable variance in days (7 = ±7 days)

        Returns:
            (score [0.0, 1.0], details {date_diff_days, penalty})

        Examples (Finance - 7 days tolerance):
            score_date("2024-10-15", "2024-10-15", 7) → (1.0, {diff_days: 0, penalty: "none"})
            score_date("2024-10-15", "2024-10-17", 7) → (0.94, {diff_days: 2, penalty: "2_days"})
            score_date("2024-10-15", "2024-10-22", 7) → (0.80, {diff_days: 7, penalty: "at_tolerance_limit"})
            score_date("2024-10-15", "2024-10-30", 7) → (0.0, {diff_days: 15, penalty: "outside_tolerance"})

        Examples (Healthcare - 60 days tolerance):
            score_date("2024-09-15", "2024-10-30", 60) → (0.85, {diff_days: 45, penalty: "45_days"})
        """
        pass

    def score_counterparty(
        self,
        counterparty_a: str,
        counterparty_b: str
    ) -> Tuple[float, Dict]:
        """
        Calculate counterparty name similarity using fuzzy string matching.

        Algorithm:
        1. Normalize names: lowercase, remove punctuation, remove common suffixes (Inc, LLC, Corp)
        2. Calculate Jaro-Winkler similarity [0.0, 1.0]
        3. Apply bonus for exact match after normalization
        4. Apply penalty for very short strings (<3 chars)

        Args:
            counterparty_a: Name from item A (e.g., "Acme Corp")
            counterparty_b: Name from item B (e.g., "Acme Corporation")

        Returns:
            (score [0.0, 1.0], details {normalized_a, normalized_b, algorithm, raw_similarity})

        Examples (Finance):
            score_counterparty("Acme Corp", "Acme Corporation") → (0.95, {algorithm: "jaro_winkler", raw: 0.95})
            score_counterparty("AMZN MKTP US", "Amazon.com") → (0.75, {algorithm: "jaro_winkler", raw: 0.75})
            score_counterparty("Apple Inc", "APPLE COM BILL") → (0.85, {algorithm: "jaro_winkler", raw: 0.85})

        Examples (Healthcare - Patient names):
            score_counterparty("John Doe", "John Doe") → (1.0, {exact_match: True})
            score_counterparty("John A. Doe", "John Doe") → (0.92, {normalized: "john doe", raw: 0.92})

        Examples (Research - Author names):
            score_counterparty("Smith, J.", "J. Smith") → (0.88, {normalized: "j smith", raw: 0.88})
        """
        pass

    def score_description(
        self,
        description_a: Optional[str],
        description_b: Optional[str]
    ) -> Tuple[float, Dict]:
        """
        Calculate description similarity using fuzzy string matching.

        Algorithm:
        1. If either description is null/empty: score = 0.0 (no data to compare)
        2. Normalize: lowercase, remove punctuation
        3. Calculate Jaro-Winkler similarity [0.0, 1.0]
        4. Optional: Use TF-IDF for longer descriptions (>50 chars)

        Args:
            description_a: Description from item A (optional)
            description_b: Description from item B (optional)

        Returns:
            (score [0.0, 1.0], details {has_data, algorithm, raw_similarity})

        Examples (Finance):
            score_description("ACH Payment", "ACH Transfer") → (0.85, {algorithm: "jaro_winkler", raw: 0.85})
            score_description("Consulting Services - Q3 2024", "Q3 2024 Consulting") → (0.80, {algorithm: "jaro_winkler"})
            score_description(None, "Some description") → (0.0, {has_data: False})

        Note: Description is typically weighted low (10%) since it's optional and highly variable.
        """
        pass

    def score_custom_feature(
        self,
        feature_name: str,
        value_a: any,
        value_b: any,
        config: Dict
    ) -> Tuple[float, Dict]:
        """
        Calculate similarity for custom domain-specific features.

        Examples:
        - Finance: currency (exact match required)
        - Healthcare: provider_id (exact match), diagnosis_code (prefix match)
        - Legal: case_number (prefix match), jurisdiction (exact match)
        - Research: journal (fuzzy match), doi_prefix (exact match)
        - E-commerce: shipping_address (fuzzy match), sku (exact match)

        Args:
            feature_name: Name of custom feature
            value_a: Value from item A
            value_b: Value from item B
            config: Feature-specific configuration (e.g., exact_match, fuzzy_threshold)

        Returns:
            (score [0.0, 1.0], details)

        Example (Finance - Currency):
            score_custom_feature("currency", "USD", "USD", {match_type: "exact"}) → (1.0, {exact: True})
            score_custom_feature("currency", "USD", "EUR", {match_type: "exact"}) → (0.0, {exact: False})

        Example (Healthcare - Procedure Code):
            score_custom_feature("cpt_code", "99213", "99213", {match_type: "exact"}) → (1.0, {exact: True})
            score_custom_feature("cpt_code", "99213", "99214", {match_type: "exact"}) → (0.0, {exact: False})

        Example (Research - Journal):
            score_custom_feature("journal", "Nature", "Nat Rev", {match_type: "fuzzy"}) → (0.75, {fuzzy: 0.75})
        """
        pass

    def _normalize_name(self, name: str) -> str:
        """
        Normalize counterparty/vendor/patient name for comparison.

        Steps:
        1. Lowercase
        2. Remove punctuation (.,-')
        3. Remove common suffixes (Inc, LLC, Corp, Ltd, GmbH)
        4. Remove extra whitespace
        5. Remove common prefixes (The, A, An)

        Examples:
            "Acme Corp." → "acme"
            "The Apple Inc" → "apple"
            "AMZN MKTP US" → "amzn mktp us"
            "John A. Doe" → "john a doe"

        Args:
            name: Raw name string

        Returns:
            Normalized name for fuzzy matching
        """
        import re

        if not name:
            return ""

        # Lowercase
        normalized = name.lower()

        # Remove punctuation
        normalized = re.sub(r'[.,\-\'"]', ' ', normalized)

        # Remove common suffixes
        suffixes = [' inc', ' llc', ' corp', ' corporation', ' ltd', ' limited', ' gmbh', ' co', ' company']
        for suffix in suffixes:
            if normalized.endswith(suffix):
                normalized = normalized[:-len(suffix)]

        # Remove common prefixes
        prefixes = ['the ', 'a ', 'an ']
        for prefix in prefixes:
            if normalized.startswith(prefix):
                normalized = normalized[len(prefix):]

        # Remove extra whitespace
        normalized = ' '.join(normalized.split())

        return normalized

    def _aggregate_scores(
        self,
        scores: Dict[str, float],
        weights: Dict[str, float]
    ) -> float:
        """
        Aggregate feature scores using weighted sum.

        Formula: confidence = Σ(score_i * weight_i)

        Validation:
        - Weights must sum to 1.0 (within 0.01 tolerance for floating point)
        - All scores must be [0.0, 1.0]

        Args:
            scores: Feature scores {amount: 1.0, date: 0.95, counterparty: 0.95, description: 0.0}
            weights: Feature weights {amount: 0.4, date: 0.3, counterparty: 0.2, description: 0.1}

        Returns:
            Overall confidence [0.0, 1.0]

        Example:
            scores = {amount: 1.0, date: 0.95, counterparty: 0.95, description: 0.0}
            weights = {amount: 0.4, date: 0.3, counterparty: 0.2, description: 0.1}

            confidence = (1.0 * 0.4) + (0.95 * 0.3) + (0.95 * 0.2) + (0.0 * 0.1)
                       = 0.4 + 0.285 + 0.19 + 0.0
                       = 0.875
                       ≈ 0.88
        """
        # Validate weights sum to 1.0
        weight_sum = sum(weights.values())
        if not (0.99 <= weight_sum <= 1.01):  # Allow floating point variance
            raise ValueError(f"Weights must sum to 1.0, got {weight_sum}")

        # Validate all scores in [0.0, 1.0]
        for feature, score in scores.items():
            if not (0.0 <= score <= 1.0):
                raise ValueError(f"Score for {feature} must be [0.0, 1.0], got {score}")

        # Compute weighted sum
        confidence = sum(scores[feature] * weights[feature] for feature in scores.keys())

        # Clamp to [0.0, 1.0] (safety)
        return max(0.0, min(1.0, confidence))
```

---

## Multi-Domain Examples

### Finance: Bank-to-Invoice Scoring

```python
# Setup
item_a = BankTransaction(
    amount=-5000.00,
    date=date(2024, 10, 15),
    counterparty="Acme Corp",
    description="ACH Payment"
)

item_b = Invoice(
    amount=5000.00,
    date=date(2024, 10, 14),
    counterparty="Acme Corporation",
    description="Consulting Services - Q3 2024"
)

config = ReconciliationConfig(
    amount_tolerance_pct=Decimal("0.05"),
    date_tolerance_days=7,
    weights={"amount": 0.4, "date": 0.3, "counterparty": 0.2, "description": 0.1}
)

# Calculate confidence
score = scorer.calculate_confidence(item_a, item_b, config)

# Result:
# confidence = 0.98 (AUTO_LINK)
# scores = {amount: 1.0, date: 0.95, counterparty: 0.95, description: 0.80}
# features = {amount_diff: 0.00, date_diff_days: 1, counterparty_similarity: 0.95}
```

### Healthcare: Claim-to-Payment Scoring

```python
# Setup
claim = InsuranceClaim(
    amount=Decimal("1250.00"),
    date=date(2024, 9, 15),
    counterparty="John Doe",  # Patient name
    description="CPT 99213 - Office Visit"
)

payment = Payment(
    amount=Decimal("1125.00"),  # 10% co-pay deducted
    date=date(2024, 10, 30),   # 45 days later
    counterparty="John Doe",
    description="EOB - Claim #67890"
)

healthcare_config = ReconciliationConfig(
    amount_tolerance_pct=Decimal("0.10"),  # 10% tolerance for adjustments
    date_tolerance_days=60,
    weights={"amount": 0.3, "date": 0.2, "counterparty": 0.3, "description": 0.2}
)

# Calculate confidence
score = scorer.calculate_confidence(claim, payment, healthcare_config)

# Result:
# confidence = 0.85 (AUTO_SUGGEST)
# scores = {amount: 0.90, date: 0.70, counterparty: 1.0, description: 0.60}
# features = {amount_diff: 125.00, amount_diff_pct: 0.10, date_diff_days: 45}
# penalties = {amount: "10_percent_variance", date: "45_days"}
```

### Legal: Court Filing Scoring

```python
# Setup
pacer_filing = CourtFiling(
    amount=Decimal("402.00"),  # Filing fee
    date=date(2024, 10, 1),
    counterparty="Smith v. Jones",  # Party names
    description="Civil Complaint"
)

state_filing = CourtFiling(
    amount=Decimal("402.00"),
    date=date(2024, 10, 3),   # Filed in state court 2 days later
    counterparty="Smith vs Jones",
    description="Civil Action - Complaint"
)

legal_config = ReconciliationConfig(
    amount_tolerance_pct=Decimal("0.0"),  # Filing fees exact
    date_tolerance_days=30,
    weights={"amount": 0.2, "date": 0.2, "counterparty": 0.4, "description": 0.2}
)

# Calculate confidence
score = scorer.calculate_confidence(pacer_filing, state_filing, legal_config)

# Result:
# confidence = 0.88 (AUTO_SUGGEST)
# scores = {amount: 1.0, date: 0.94, counterparty: 0.90, description: 0.75}
# features = {amount_diff: 0.00, date_diff_days: 2, counterparty_similarity: 0.90}
```

### Research (RSRCH - Utilitario): Fact Deduplication Scoring

```python
# Setup
techcrunch_fact = Fact(
    amount=375000000,  # $375M investment
    date=date(2025, 1, 15),  # TechCrunch article date
    counterparty="Sam Altman",  # Subject entity
    description="Sam Altman invested $375M in Helion Energy"
)

podcast_fact = Fact(
    amount=375000000,
    date=date(2025, 1, 20),  # Lex Fridman podcast date (5 days later)
    counterparty="sama",  # Twitter handle variant
    description="sama invested 375 million in Helion"
)

fact_config = ReconciliationConfig(
    amount_tolerance_pct=Decimal("0.10"),  # ±10% for investment amounts
    date_tolerance_days=30,  # ±30 days
    weights={"amount": 0.3, "date": 0.2, "counterparty": 0.3, "description": 0.2}
)

# Calculate confidence
score = scorer.calculate_confidence(techcrunch_fact, podcast_fact, fact_config)

# Result:
# confidence = 0.94 (AUTO_LINK - same investment fact from 2 sources)
# scores = {amount: 1.0, date: 0.83, counterparty: 0.95, description: 0.98}
# features = {date_diff_days: 0, counterparty_similarity: 0.95, description_similarity: 0.90}
```

---

## Performance Characteristics

**Target Latencies (p95):**
- `calculate_confidence()`: <5ms per comparison
- `score_amount()`: <0.1ms
- `score_date()`: <0.1ms
- `score_counterparty()`: <2ms (fuzzy string matching)
- `score_description()`: <2ms (fuzzy string matching)

**Scalability:**
- Can score 1000s of comparisons per second
- Fuzzy matching is the bottleneck (Jaro-Winkler ~2ms per pair)
- Caching normalized names improves performance 10x

**Optimization:**
- Pre-normalize all counterparty names (one-time cost)
- Use fast string similarity libraries (python-Levenshtein, jellyfish)
- Cache similarity scores for common name pairs (e.g., "AMZN" → "Amazon")

---

## Testing Strategy

**Unit Tests:**
```python
def test_score_amount_exact_match():
    score, details = scorer.score_amount(Decimal("100.00"), Decimal("100.00"), Decimal("0.05"))
    assert score == 1.0
    assert details["amount_diff"] == Decimal("0.00")
    assert details["amount_diff_pct"] == Decimal("0.00")

def test_score_amount_within_tolerance():
    score, details = scorer.score_amount(Decimal("100.00"), Decimal("102.00"), Decimal("0.05"))
    assert 0.95 <= score <= 0.97  # Small penalty for 2% difference
    assert details["amount_diff_pct"] == Decimal("0.02")

def test_score_amount_outside_tolerance():
    score, details = scorer.score_amount(Decimal("100.00"), Decimal("110.00"), Decimal("0.05"))
    assert score == 0.0
    assert details["penalty"] == "outside_tolerance"

def test_score_date_same_day():
    score, details = scorer.score_date(date(2024, 10, 15), date(2024, 10, 15), 7)
    assert score == 1.0

def test_score_counterparty_fuzzy_match():
    score, details = scorer.score_counterparty("Acme Corp", "Acme Corporation")
    assert score >= 0.90  # High similarity
```

---

## Summary

MatchScorer provides **granular similarity calculation** for reconciliation:

✅ **Feature-level scoring** (amount, date, counterparty, description)
✅ **Tolerance-based penalties** (configurable per feature)
✅ **Weighted aggregation** (customize importance of each feature)
✅ **Explainable results** (detailed breakdowns for user review)
✅ **Domain-agnostic** (finance, healthcare, legal, research, e-commerce)

**Key Design Decisions:**
- Amount: Exact match = 1.0, penalty for % difference, 0.0 if outside tolerance
- Date: Same day = 1.0, linear decay, 0.0 if outside tolerance
- Counterparty: Fuzzy string matching (Jaro-Winkler), normalization critical
- Description: Optional feature (low weight 10%), fuzzy matching
- Custom features: Extensible for domain-specific requirements
