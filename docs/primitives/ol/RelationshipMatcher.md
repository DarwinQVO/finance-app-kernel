# RelationshipMatcher (OL Primitive)

**Vertical:** 3.5 Relationships
**Layer:** Objective Layer (OL)
**Type:** Fuzzy Matching Engine
**Status:** ✅ Specified

---

## Purpose

Fuzzy matching engine for paired transactions with configurable thresholds, enabling automatic detection of transfers, FX conversions, and other transaction relationships.

**Core Responsibility:** Match transactions based on configurable criteria (amount tolerance, date window, account differences) to identify potential relationships with confidence scoring, reducing manual linking effort.

---

## Multi-Domain Applicability

Fuzzy record matching with thresholds applies to ANY domain with paired records:

| Domain | Entity | Fuzzy Matching Examples |
|--------|--------|------------------------|
| **Finance** | Transaction Pairs | Transfer (BofA -$1,000 ↔ Wise +$998), FX conversion (USD -$1,000 ↔ MXN +$18,500) |
| **Healthcare** | Claim-Payment | Insurance claim -$500 ↔ Reimbursement +$450 (60-day delay, 90% amount) |
| **Legal** | Document Versions | Original brief v1 ↔ Amended brief v2 (same case, similar content, different dates) |
| **Research** | Dataset Lineage | Source dataset ↔ Derived dataset (processing relationship, version tracking) |
| **E-commerce** | Order-Return | Original order $100 ↔ Refund $100 (same product, opposite amounts) |
| **Manufacturing** | Invoice-Payment | Purchase order $10,000 ↔ Payment $9,950 (2% discount, 30-day terms) |

**Pattern:** Fuzzy Matcher = define matching criteria, calculate similarity score, return candidates above threshold, support batch processing.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Tuple
from dataclasses import dataclass
from enum import Enum
from datetime import date, timedelta
from decimal import Decimal

class RelationshipType(Enum):
    TRANSFER = "transfer"
    FX_CONVERSION = "fx_conversion"
    REIMBURSEMENT = "reimbursement"
    SPLIT = "split"
    CORRECTION = "correction"
    OTHER = "other"

@dataclass
class MatchConfig:
    """Configuration for relationship matching thresholds."""
    amount_tolerance_pct: float = 0.05  # 5% tolerance (0.00 = exact match, 1.00 = 100% variance allowed)
    date_window_days: int = 7           # Maximum days between transactions
    min_confidence: float = 0.50        # Minimum confidence score to consider match
    require_opposite_signs: bool = True # Must have opposite signs (- vs +)
    require_different_accounts: bool = True  # Must be from different accounts
    same_currency_only: bool = False    # Only match same currency pairs

    # FX conversion specific
    fx_rate_tolerance_pct: float = 0.10  # 10% tolerance for exchange rates

    # User-customizable per relationship type
    type_specific: dict = None  # {"transfer": {...}, "fx_conversion": {...}}

@dataclass
class MatchResult:
    candidate_txn_id: str               # Transaction ID of match candidate
    confidence: float                   # 0.0 to 1.0 (1.0 = perfect match)
    match_type: RelationshipType        # Suggested relationship type
    match_details: dict                 # Details: amount_diff, date_diff, etc.

    # Breakdown scores
    amount_score: float                 # 0.0 to 1.0
    date_score: float                   # 0.0 to 1.0
    account_score: float                # 0.0 to 1.0
    currency_score: float               # 0.0 to 1.0

    # FX conversion specific
    exchange_rate: Optional[Decimal] = None
    rate_source: Optional[str] = None   # "calculated", "market"

@dataclass
class MatchCandidates:
    """Container for all match candidates for a transaction."""
    source_txn_id: str
    matches: List[MatchResult]          # Sorted by confidence DESC
    best_match: Optional[MatchResult]   # Top match above threshold, or None

class RelationshipMatcher:
    """
    Fuzzy matching engine for transaction pairs with configurable thresholds.

    Matching Strategy:
    1. Amount similarity - Calculate % difference vs tolerance
    2. Date proximity - Days apart vs window
    3. Account difference - Same vs different accounts
    4. Currency match - Same vs different currencies
    5. Sign verification - Opposite vs same signs
    6. Combine scores → final confidence

    Confidence Calculation:
    - Amount score: 40% weight (most important)
    - Date score: 30% weight
    - Account score: 15% weight
    - Currency score: 15% weight

    Confidence Thresholds:
    - High: ≥0.90 (auto-suggest with high confidence)
    - Medium: 0.70-0.89 (auto-suggest with caution)
    - Low: 0.50-0.69 (show in possible matches list)
    - Below 0.50: ignore (too dissimilar)
    """

    DEFAULT_CONFIG = MatchConfig()
    HIGH_CONFIDENCE = 0.90
    MEDIUM_CONFIDENCE = 0.70
    LOW_CONFIDENCE = 0.50

    def __init__(self, transaction_store, fx_converter=None):
        """
        Initialize matcher.

        Args:
            transaction_store: CanonicalStore instance for querying transactions
            fx_converter: Optional FXConverter for exchange rate lookups
        """
        self.txn_store = transaction_store
        self.fx_converter = fx_converter
        self._cache = {}  # Cache for recent match queries

    def match(
        self,
        txn_id: str,
        user_id: str,
        config: Optional[MatchConfig] = None,
        max_results: int = 5
    ) -> MatchCandidates:
        """
        Find matching transactions for a single transaction.

        Args:
            txn_id: Transaction ID to find matches for
            user_id: Owner of transactions
            config: Match configuration (uses default if None)
            max_results: Maximum number of results to return

        Returns:
            MatchCandidates with all matches above threshold

        Raises:
            TransactionNotFoundError: If txn_id doesn't exist
            UnauthorizedError: If user doesn't own transaction

        Example:
            matcher = RelationshipMatcher(txn_store)

            # Find transfer candidates for $1,000 withdrawal
            candidates = matcher.match(
                txn_id="txn_bofa_001",
                user_id="user_darwin",
                config=MatchConfig(
                    amount_tolerance_pct=0.02,  # 2% tolerance
                    date_window_days=3          # Within 3 days
                )
            )

            if candidates.best_match:
                print(f"Best match: {candidates.best_match.candidate_txn_id}")
                print(f"Confidence: {candidates.best_match.confidence:.2f}")
                print(f"Type: {candidates.best_match.match_type.value}")
        """

    def bulk_match(
        self,
        user_id: str,
        date_from: date,
        date_to: date,
        config: Optional[MatchConfig] = None,
        batch_size: int = 100
    ) -> List[Tuple[str, MatchCandidates]]:
        """
        Batch match all unlinked transactions in date range.

        Used for:
        - Initial historical data processing
        - Periodic re-matching to find missed links
        - Bulk operations after config changes

        Args:
            user_id: Owner of transactions
            date_from: Start date (inclusive)
            date_to: End date (inclusive)
            config: Match configuration (uses default if None)
            batch_size: Number of transactions to process per batch

        Returns:
            List of (txn_id, MatchCandidates) tuples
            Only returns transactions with matches (filters out no-match cases)

        Performance:
        - Processes in batches to avoid memory issues
        - Uses optimized bulk queries
        - Caches intermediate results

        Example:
            # Re-match all transactions from last year
            results = matcher.bulk_match(
                user_id="user_darwin",
                date_from=date(2024, 1, 1),
                date_to=date(2024, 12, 31),
                config=MatchConfig(amount_tolerance_pct=0.05)
            )

            print(f"Found {len(results)} transactions with matches")

            for txn_id, candidates in results:
                if candidates.best_match:
                    print(f"{txn_id} → {candidates.best_match.candidate_txn_id} "
                          f"({candidates.best_match.confidence:.2f})")
        """

    def validate_match(
        self,
        txn_1_id: str,
        txn_2_id: str,
        user_id: str,
        relationship_type: RelationshipType
    ) -> MatchResult:
        """
        Validate a proposed match before creating relationship.

        Used when:
        - User manually links two transactions
        - System validates auto-detected matches
        - Pre-creation sanity checks

        Args:
            txn_1_id: First transaction ID
            txn_2_id: Second transaction ID
            user_id: Owner of transactions
            relationship_type: Intended relationship type

        Returns:
            MatchResult with confidence and validation details

        Raises:
            TransactionNotFoundError: If either transaction doesn't exist
            UnauthorizedError: If user doesn't own both transactions
            SameTransactionError: If txn_1_id == txn_2_id
            AlreadyLinkedError: If either transaction already in relationship

        Example:
            # User wants to manually link reimbursement
            result = matcher.validate_match(
                txn_1_id="txn_personal_expense",
                txn_2_id="txn_employer_reimbursement",
                user_id="user_darwin",
                relationship_type=RelationshipType.REIMBURSEMENT
            )

            if result.confidence < 0.50:
                print(f"Warning: Low confidence match ({result.confidence:.2f})")
                print(f"Amount diff: {result.match_details['amount_diff']}")
                print(f"Date diff: {result.match_details['date_diff']} days")
        """

    def calculate_confidence(
        self,
        txn_1: dict,
        txn_2: dict,
        config: Optional[MatchConfig] = None
    ) -> Tuple[float, dict]:
        """
        Calculate confidence score for a transaction pair.

        Args:
            txn_1: First transaction (dict with amount, date, account_id, currency)
            txn_2: Second transaction (dict with amount, date, account_id, currency)
            config: Match configuration (uses default if None)

        Returns:
            Tuple of (confidence_score, score_breakdown)
            - confidence_score: float (0.0 to 1.0)
            - score_breakdown: dict with individual component scores

        Confidence Formula:
            confidence = (
                0.40 * amount_score +
                0.30 * date_score +
                0.15 * account_score +
                0.15 * currency_score
            )

        Amount Score:
            Perfect match (0% diff): 1.0
            Within tolerance: 1.0 - (diff_pct / tolerance_pct)
            Beyond tolerance: 0.0

        Date Score:
            Same day: 1.0
            Next day: 0.90
            2 days: 0.80
            ...linear decay to window edge
            Beyond window: 0.0

        Account Score:
            Different accounts: 1.0 (expected for transfers)
            Same account: 0.5 (unusual but allowed)

        Currency Score:
            Different currencies: 1.0 (FX conversion)
            Same currency: 1.0 (transfer/reimbursement)

        Example:
            # Calculate confidence for potential transfer
            txn_1 = {
                "amount": Decimal("-1000.00"),
                "date": date(2025, 10, 15),
                "account_id": "acc_bofa",
                "currency": "USD"
            }
            txn_2 = {
                "amount": Decimal("998.00"),
                "date": date(2025, 10, 15),
                "account_id": "acc_wise",
                "currency": "USD"
            }

            confidence, breakdown = matcher.calculate_confidence(txn_1, txn_2)

            print(f"Confidence: {confidence:.2f}")
            print(f"Amount score: {breakdown['amount_score']:.2f}")
            print(f"Date score: {breakdown['date_score']:.2f}")
            print(f"Account score: {breakdown['account_score']:.2f}")
        """

    def _calculate_amount_score(
        self,
        amount_1: Decimal,
        amount_2: Decimal,
        tolerance_pct: float
    ) -> float:
        """
        Calculate amount similarity score.

        Args:
            amount_1: First transaction amount
            amount_2: Second transaction amount
            tolerance_pct: Tolerance percentage (0.05 = 5%)

        Returns:
            Score from 0.0 to 1.0

        Example:
            # Perfect match
            score = _calculate_amount_score(
                Decimal("-1000.00"),
                Decimal("1000.00"),
                0.05
            )
            # score = 1.0 (0% difference)

            # Within tolerance
            score = _calculate_amount_score(
                Decimal("-1000.00"),
                Decimal("998.00"),
                0.05
            )
            # diff = 2.00, diff_pct = 0.002 (0.2%)
            # score = 1.0 - (0.002 / 0.05) = 0.96

            # Beyond tolerance
            score = _calculate_amount_score(
                Decimal("-1000.00"),
                Decimal("900.00"),
                0.05
            )
            # diff = 100.00, diff_pct = 0.10 (10% > 5% tolerance)
            # score = 0.0
        """

    def _calculate_date_score(
        self,
        date_1: date,
        date_2: date,
        window_days: int
    ) -> float:
        """
        Calculate date proximity score.

        Args:
            date_1: First transaction date
            date_2: Second transaction date
            window_days: Maximum days window

        Returns:
            Score from 0.0 to 1.0 (linear decay)

        Example:
            # Same day
            score = _calculate_date_score(
                date(2025, 10, 15),
                date(2025, 10, 15),
                7
            )
            # score = 1.0

            # Next day
            score = _calculate_date_score(
                date(2025, 10, 15),
                date(2025, 10, 16),
                7
            )
            # diff = 1 day
            # score = 1.0 - (1 / 7) = 0.86

            # Beyond window
            score = _calculate_date_score(
                date(2025, 10, 15),
                date(2025, 10, 25),
                7
            )
            # diff = 10 days > 7 day window
            # score = 0.0
        """

    def _calculate_account_score(
        self,
        account_1: str,
        account_2: str,
        require_different: bool
    ) -> float:
        """
        Calculate account difference score.

        Args:
            account_1: First transaction account ID
            account_2: Second transaction account ID
            require_different: True if different accounts expected

        Returns:
            Score from 0.0 to 1.0

        Example:
            # Different accounts (expected for transfer)
            score = _calculate_account_score("acc_bofa", "acc_wise", True)
            # score = 1.0

            # Same account (expected for correction)
            score = _calculate_account_score("acc_bofa", "acc_bofa", False)
            # score = 1.0

            # Same account when different expected (unusual)
            score = _calculate_account_score("acc_bofa", "acc_bofa", True)
            # score = 0.5
        """

    def _classify_relationship_type(
        self,
        txn_1: dict,
        txn_2: dict
    ) -> RelationshipType:
        """
        Classify suggested relationship type based on transaction characteristics.

        Decision Logic:
            - Different currencies → FX_CONVERSION
            - Same amount, opposite signs, different accounts → TRANSFER
            - Different amounts, opposite signs, same account → CORRECTION
            - Otherwise → REIMBURSEMENT (catch-all)

        Args:
            txn_1: First transaction
            txn_2: Second transaction

        Returns:
            RelationshipType enum

        Example:
            # FX conversion
            txn_1 = {"amount": -1000, "currency": "USD"}
            txn_2 = {"amount": 18500, "currency": "MXN"}
            type = _classify_relationship_type(txn_1, txn_2)
            # type = RelationshipType.FX_CONVERSION

            # Transfer
            txn_1 = {"amount": -1000, "currency": "USD", "account_id": "acc_bofa"}
            txn_2 = {"amount": 1000, "currency": "USD", "account_id": "acc_wise"}
            type = _classify_relationship_type(txn_1, txn_2)
            # type = RelationshipType.TRANSFER
        """

    def _find_candidates_sql(
        self,
        txn: dict,
        user_id: str,
        config: MatchConfig
    ) -> str:
        """
        Build optimized SQL query to find match candidates.

        Query optimizations:
        - Filter by user_id (partition key)
        - Filter by date window (indexed)
        - Filter by amount range (calculated tolerance)
        - Exclude already linked transactions
        - Limit results for performance

        Args:
            txn: Source transaction
            user_id: User ID
            config: Match configuration

        Returns:
            SQL query string
        """

    def clear_cache(self) -> None:
        """Clear internal match result cache."""
        self._cache.clear()
```

---

## Implementation Details

### Confidence Calculation Algorithm

```python
def calculate_confidence(
    self,
    txn_1: dict,
    txn_2: dict,
    config: Optional[MatchConfig] = None
) -> Tuple[float, dict]:
    """
    Calculate weighted confidence score.

    Weights:
    - Amount: 40% (most important - amounts must be close)
    - Date: 30% (important - timing matters)
    - Account: 15% (moderate - same/different matters for type)
    - Currency: 15% (moderate - same/different determines FX vs transfer)
    """
    if config is None:
        config = self.DEFAULT_CONFIG

    # Extract transaction attributes
    amount_1 = abs(txn_1["amount"])
    amount_2 = abs(txn_2["amount"])
    date_1 = txn_1["date"]
    date_2 = txn_2["date"]
    account_1 = txn_1["account_id"]
    account_2 = txn_2["account_id"]
    currency_1 = txn_1.get("currency", "USD")
    currency_2 = txn_2.get("currency", "USD")

    # Calculate individual scores
    amount_score = self._calculate_amount_score(
        amount_1,
        amount_2,
        config.amount_tolerance_pct
    )

    date_score = self._calculate_date_score(
        date_1,
        date_2,
        config.date_window_days
    )

    account_score = self._calculate_account_score(
        account_1,
        account_2,
        config.require_different_accounts
    )

    # Currency score
    if currency_1 == currency_2:
        currency_score = 1.0
    else:
        # Different currencies OK for FX conversion
        currency_score = 1.0 if not config.same_currency_only else 0.0

    # Sign check (must have opposite signs for transfer/FX)
    if config.require_opposite_signs:
        has_opposite_signs = (
            (txn_1["amount"] > 0 and txn_2["amount"] < 0) or
            (txn_1["amount"] < 0 and txn_2["amount"] > 0)
        )
        if not has_opposite_signs:
            return 0.0, {
                "amount_score": 0.0,
                "date_score": 0.0,
                "account_score": 0.0,
                "currency_score": 0.0,
                "reason": "same_sign"
            }

    # Weighted sum
    confidence = (
        0.40 * amount_score +
        0.30 * date_score +
        0.15 * account_score +
        0.15 * currency_score
    )

    # Return confidence and breakdown
    breakdown = {
        "amount_score": amount_score,
        "date_score": date_score,
        "account_score": account_score,
        "currency_score": currency_score,
        "amount_diff": float(abs(amount_1 - amount_2)),
        "amount_diff_pct": float(abs(amount_1 - amount_2) / amount_1) if amount_1 > 0 else 0.0,
        "date_diff_days": abs((date_1 - date_2).days),
        "same_account": account_1 == account_2,
        "same_currency": currency_1 == currency_2
    }

    return confidence, breakdown
```

### Amount Score Calculation

```python
def _calculate_amount_score(
    self,
    amount_1: Decimal,
    amount_2: Decimal,
    tolerance_pct: float
) -> float:
    """
    Calculate amount similarity score with tolerance.

    Examples:
      amount_1 = 1000.00, amount_2 = 1000.00, tolerance = 0.05
      diff = 0, diff_pct = 0.00, score = 1.0 (perfect)

      amount_1 = 1000.00, amount_2 = 998.00, tolerance = 0.05
      diff = 2.00, diff_pct = 0.002, score = 1.0 - (0.002/0.05) = 0.96

      amount_1 = 1000.00, amount_2 = 950.00, tolerance = 0.05
      diff = 50.00, diff_pct = 0.05, score = 1.0 - (0.05/0.05) = 0.0

      amount_1 = 1000.00, amount_2 = 900.00, tolerance = 0.05
      diff = 100.00, diff_pct = 0.10 > 0.05, score = 0.0
    """
    # Calculate absolute difference
    diff = abs(amount_1 - amount_2)

    # Handle zero amounts
    if amount_1 == 0 or amount_2 == 0:
        return 0.0

    # Calculate percentage difference (relative to larger amount)
    base_amount = max(amount_1, amount_2)
    diff_pct = float(diff / base_amount)

    # Perfect match
    if diff_pct == 0.0:
        return 1.0

    # Beyond tolerance
    if diff_pct > tolerance_pct:
        return 0.0

    # Within tolerance - linear decay
    # score = 1.0 at 0% diff, 0.0 at tolerance_pct
    score = 1.0 - (diff_pct / tolerance_pct)

    return max(0.0, min(1.0, score))  # Clamp to [0, 1]
```

### Date Score Calculation

```python
def _calculate_date_score(
    self,
    date_1: date,
    date_2: date,
    window_days: int
) -> float:
    """
    Calculate date proximity score with linear decay.

    Examples:
      date_1 = 2025-10-15, date_2 = 2025-10-15, window = 7
      diff = 0, score = 1.0 (same day)

      date_1 = 2025-10-15, date_2 = 2025-10-16, window = 7
      diff = 1, score = 1.0 - (1/7) = 0.857

      date_1 = 2025-10-15, date_2 = 2025-10-19, window = 7
      diff = 4, score = 1.0 - (4/7) = 0.429

      date_1 = 2025-10-15, date_2 = 2025-10-22, window = 7
      diff = 7, score = 1.0 - (7/7) = 0.0 (at edge)

      date_1 = 2025-10-15, date_2 = 2025-10-25, window = 7
      diff = 10 > 7, score = 0.0 (beyond window)
    """
    # Calculate absolute day difference
    diff_days = abs((date_1 - date_2).days)

    # Same day
    if diff_days == 0:
        return 1.0

    # Beyond window
    if diff_days > window_days:
        return 0.0

    # Linear decay within window
    # score = 1.0 at 0 days, 0.0 at window_days
    score = 1.0 - (diff_days / window_days)

    return max(0.0, min(1.0, score))  # Clamp to [0, 1]
```

### Match Candidates Query

```python
def match(
    self,
    txn_id: str,
    user_id: str,
    config: Optional[MatchConfig] = None,
    max_results: int = 5
) -> MatchCandidates:
    """Find matching transactions."""
    if config is None:
        config = self.DEFAULT_CONFIG

    # 1. Get source transaction
    source_txn = self.txn_store.get(txn_id, user_id)
    if not source_txn:
        raise TransactionNotFoundError(f"Transaction {txn_id} not found")

    # 2. Check cache
    cache_key = f"{txn_id}:{config.__hash__()}"
    if cache_key in self._cache:
        return self._cache[cache_key]

    # 3. Calculate search boundaries
    amount_min = abs(source_txn.amount) * (1 - config.amount_tolerance_pct)
    amount_max = abs(source_txn.amount) * (1 + config.amount_tolerance_pct)
    date_min = source_txn.date - timedelta(days=config.date_window_days)
    date_max = source_txn.date + timedelta(days=config.date_window_days)

    # 4. Find candidates with optimized query
    candidates_query = """
        SELECT t.*
        FROM canonical_transactions t
        LEFT JOIN relationships r ON (
            t.txn_id = r.txn_1_id OR t.txn_id = r.txn_2_id
        )
        WHERE t.user_id = :user_id
          AND t.txn_id != :source_txn_id
          AND ABS(t.amount) BETWEEN :amount_min AND :amount_max
          AND t.date BETWEEN :date_min AND :date_max
          AND r.relationship_id IS NULL  -- Not already linked
    """

    # Additional filters
    if config.require_opposite_signs:
        candidates_query += " AND SIGN(t.amount) != SIGN(:source_amount)"

    if config.require_different_accounts:
        candidates_query += " AND t.account_id != :source_account_id"

    if config.same_currency_only:
        candidates_query += " AND t.currency = :source_currency"

    candidates_query += " ORDER BY ABS(t.date - :source_date), ABS(ABS(t.amount) - ABS(:source_amount)) LIMIT :limit"

    # Execute query
    raw_candidates = self.txn_store.query(
        candidates_query,
        user_id=user_id,
        source_txn_id=source_txn.txn_id,
        source_amount=source_txn.amount,
        source_account_id=source_txn.account_id,
        source_currency=source_txn.currency,
        source_date=source_txn.date,
        amount_min=amount_min,
        amount_max=amount_max,
        date_min=date_min,
        date_max=date_max,
        limit=max_results * 2  # Get extra for filtering
    )

    # 5. Calculate confidence for each candidate
    match_results = []
    for candidate in raw_candidates:
        confidence, breakdown = self.calculate_confidence(
            source_txn.__dict__,
            candidate.__dict__,
            config
        )

        # Filter by minimum confidence
        if confidence < config.min_confidence:
            continue

        # Classify relationship type
        rel_type = self._classify_relationship_type(
            source_txn.__dict__,
            candidate.__dict__
        )

        # Calculate FX rate if applicable
        exchange_rate = None
        rate_source = None
        if rel_type == RelationshipType.FX_CONVERSION:
            exchange_rate = abs(candidate.amount) / abs(source_txn.amount)
            rate_source = "calculated"

        match_results.append(MatchResult(
            candidate_txn_id=candidate.txn_id,
            confidence=confidence,
            match_type=rel_type,
            match_details=breakdown,
            amount_score=breakdown["amount_score"],
            date_score=breakdown["date_score"],
            account_score=breakdown["account_score"],
            currency_score=breakdown["currency_score"],
            exchange_rate=exchange_rate,
            rate_source=rate_source
        ))

    # 6. Sort by confidence DESC
    match_results.sort(key=lambda m: m.confidence, reverse=True)

    # 7. Limit results
    match_results = match_results[:max_results]

    # 8. Get best match
    best_match = match_results[0] if match_results else None

    # 9. Create result
    result = MatchCandidates(
        source_txn_id=txn_id,
        matches=match_results,
        best_match=best_match
    )

    # 10. Cache result
    self._cache[cache_key] = result

    return result
```

### Bulk Matching for Historical Data

```python
def bulk_match(
    self,
    user_id: str,
    date_from: date,
    date_to: date,
    config: Optional[MatchConfig] = None,
    batch_size: int = 100
) -> List[Tuple[str, MatchCandidates]]:
    """
    Batch process all unlinked transactions in date range.

    Algorithm:
    1. Get all unlinked transactions in date range
    2. Process in batches (for memory efficiency)
    3. For each transaction, find matches
    4. Collect results
    5. Return only transactions with matches
    """
    if config is None:
        config = self.DEFAULT_CONFIG

    # Get all unlinked transactions
    unlinked_txns = self.txn_store.list_unlinked(
        user_id=user_id,
        date_from=date_from,
        date_to=date_to
    )

    results = []
    total_txns = len(unlinked_txns)

    # Process in batches
    for i in range(0, total_txns, batch_size):
        batch = unlinked_txns[i:i+batch_size]

        for txn in batch:
            # Find matches for this transaction
            candidates = self.match(
                txn_id=txn.txn_id,
                user_id=user_id,
                config=config
            )

            # Only include if matches found
            if candidates.matches:
                results.append((txn.txn_id, candidates))

        # Progress logging (optional)
        processed = min(i + batch_size, total_txns)
        print(f"Processed {processed}/{total_txns} transactions...")

    return results
```

### Relationship Type Classification

```python
def _classify_relationship_type(
    self,
    txn_1: dict,
    txn_2: dict
) -> RelationshipType:
    """
    Classify relationship type based on transaction characteristics.

    Decision tree:
    1. Different currencies? → FX_CONVERSION
    2. Same amount, opposite signs, different accounts? → TRANSFER
    3. Same account, opposite signs? → CORRECTION
    4. Different amounts, opposite signs? → REIMBURSEMENT
    5. Default → OTHER
    """
    currency_1 = txn_1.get("currency", "USD")
    currency_2 = txn_2.get("currency", "USD")
    account_1 = txn_1["account_id"]
    account_2 = txn_2["account_id"]
    amount_1 = abs(txn_1["amount"])
    amount_2 = abs(txn_2["amount"])

    # Different currencies → FX conversion
    if currency_1 != currency_2:
        return RelationshipType.FX_CONVERSION

    # Calculate amount similarity
    amount_diff_pct = abs(amount_1 - amount_2) / max(amount_1, amount_2) if max(amount_1, amount_2) > 0 else 0

    # Same amount (within 2%), different accounts → Transfer
    if amount_diff_pct <= 0.02 and account_1 != account_2:
        return RelationshipType.TRANSFER

    # Same account, opposite signs → Correction
    if account_1 == account_2:
        return RelationshipType.CORRECTION

    # Different amounts → Reimbursement
    if amount_diff_pct > 0.02:
        return RelationshipType.REIMBURSEMENT

    # Default
    return RelationshipType.OTHER
```

---

## Usage Examples

### Example 1: Simple Transfer Detection

```python
matcher = RelationshipMatcher(txn_store)

# Find matches for BofA withdrawal
candidates = matcher.match(
    txn_id="txn_bofa_001",  # BofA -$1,000 on 2025-10-15
    user_id="user_darwin"
)

if candidates.best_match:
    print(f"Best match: {candidates.best_match.candidate_txn_id}")
    print(f"Confidence: {candidates.best_match.confidence:.2%}")
    print(f"Type: {candidates.best_match.match_type.value}")

# Output:
# Best match: txn_wise_002
# Confidence: 95%
# Type: transfer
```

### Example 2: Custom Match Configuration

```python
# Strict matching for exact transfers
strict_config = MatchConfig(
    amount_tolerance_pct=0.01,  # 1% tolerance
    date_window_days=3,         # Within 3 days
    min_confidence=0.85         # High confidence only
)

candidates = matcher.match(
    txn_id="txn_bofa_001",
    user_id="user_darwin",
    config=strict_config
)
```

### Example 3: FX Conversion with Tolerance

```python
# Looser matching for FX conversions (rates vary)
fx_config = MatchConfig(
    amount_tolerance_pct=0.10,  # 10% tolerance (FX spreads)
    date_window_days=1,         # Same day
    require_opposite_signs=True,
    same_currency_only=False    # Allow different currencies
)

candidates = matcher.match(
    txn_id="txn_usd_conversion",  # USD -$1,000
    user_id="user_darwin",
    config=fx_config
)

if candidates.best_match:
    if candidates.best_match.exchange_rate:
        print(f"Exchange rate: {candidates.best_match.exchange_rate:.4f}")
```

### Example 4: Validate Manual Link

```python
# User wants to manually link two transactions
try:
    result = matcher.validate_match(
        txn_1_id="txn_personal_expense",
        txn_2_id="txn_employer_reimbursement",
        user_id="user_darwin",
        relationship_type=RelationshipType.REIMBURSEMENT
    )

    if result.confidence < 0.50:
        print("⚠️ Warning: Low confidence match")
        print(f"Amount difference: ${result.match_details['amount_diff']:.2f}")
        print(f"Date difference: {result.match_details['date_diff_days']} days")

        # Still allow user to proceed
        confirm = input("Continue with link? (y/n): ")

except AlreadyLinkedError:
    print("❌ One or both transactions already linked")
```

### Example 5: Bulk Historical Matching

```python
# Re-match all transactions from 2024
results = matcher.bulk_match(
    user_id="user_darwin",
    date_from=date(2024, 1, 1),
    date_to=date(2024, 12, 31),
    config=MatchConfig(
        amount_tolerance_pct=0.05,
        date_window_days=7,
        min_confidence=0.70
    )
)

print(f"Found {len(results)} transactions with potential matches")

# Create auto-suggested relationships
for txn_id, candidates in results:
    if candidates.best_match and candidates.best_match.confidence >= 0.85:
        print(f"Auto-suggest: {txn_id} → {candidates.best_match.candidate_txn_id} "
              f"({candidates.best_match.confidence:.2%})")
```

### Example 6: Handle Multiple Candidates

```python
# Transaction with multiple potential matches
candidates = matcher.match(
    txn_id="txn_100_withdrawal",  # -$100
    user_id="user_darwin",
    max_results=10
)

print(f"Found {len(candidates.matches)} potential matches:")

for i, match in enumerate(candidates.matches, 1):
    print(f"{i}. {match.candidate_txn_id} - {match.confidence:.2%} confidence")
    print(f"   Amount: ${abs(match.match_details['amount_diff']):.2f} diff")
    print(f"   Date: {match.match_details['date_diff_days']} days apart")
    print()

# User selects correct match from list
```

### Example 7: Relationship Type-Specific Config

```python
# Custom config per relationship type
type_configs = {
    "transfer": MatchConfig(
        amount_tolerance_pct=0.02,
        date_window_days=3,
        min_confidence=0.85,
        require_opposite_signs=True,
        require_different_accounts=True
    ),
    "reimbursement": MatchConfig(
        amount_tolerance_pct=0.20,  # Allow larger variance
        date_window_days=60,        # Up to 2 months delay
        min_confidence=0.50,
        require_opposite_signs=False,
        require_different_accounts=False
    )
}

# Use appropriate config
candidates = matcher.match(
    txn_id="txn_expense",
    user_id="user_darwin",
    config=type_configs["reimbursement"]
)
```

---

## Multi-Domain Examples

### Healthcare: Claim-Payment Matching

```python
# Insurance claim submitted: -$500
# Reimbursement received 60 days later: +$450 (90% coverage)

healthcare_config = MatchConfig(
    amount_tolerance_pct=0.20,  # 20% tolerance (copay, deductible)
    date_window_days=90,        # Up to 3 months
    min_confidence=0.50,
    require_opposite_signs=True
)

candidates = matcher.match(
    txn_id="claim_doctor_visit",
    user_id="patient_123",
    config=healthcare_config
)

# Match found: payment_insurance_reimbursement
# Confidence: 72% (amount differs by 10%, 60-day delay)
```

### Legal: Document Version Linking

```python
# Original brief filed: doc_v1 (2025-10-15)
# Amended brief filed: doc_v2 (2025-10-20)

legal_config = MatchConfig(
    amount_tolerance_pct=0.50,  # Amounts may vary significantly
    date_window_days=30,        # Within 30 days
    min_confidence=0.60,
    require_opposite_signs=False,  # Both are expenses
    require_different_accounts=False
)

candidates = matcher.match(
    txn_id="doc_v1",
    user_id="lawyer_456",
    config=legal_config
)
```

### Research: Dataset Lineage

```python
# Source dataset: raw_data.csv (collected 2025-01-15)
# Derived dataset: processed_data.csv (created 2025-01-20)

research_config = MatchConfig(
    amount_tolerance_pct=1.00,  # Size may differ significantly after processing
    date_window_days=30,        # Within 30 days
    min_confidence=0.50,
    require_opposite_signs=False,
    require_different_accounts=False
)

candidates = matcher.match(
    txn_id="dataset_raw_001",
    user_id="researcher_789",
    config=research_config
)
```

### E-commerce: Order-Return Matching

```python
# Original order: +$100 (customer purchase)
# Return processed: -$100 (refund)

ecommerce_config = MatchConfig(
    amount_tolerance_pct=0.05,  # Amounts should match closely
    date_window_days=30,        # Return window
    min_confidence=0.80,
    require_opposite_signs=True
)

candidates = matcher.match(
    txn_id="order_12345",
    user_id="customer_101",
    config=ecommerce_config
)
```

---

## Performance Characteristics

### Complexity Analysis

| Operation | Time Complexity | Space Complexity | Notes |
|-----------|----------------|------------------|-------|
| `match()` | O(C) | O(C) | C = candidate count (filtered by SQL) |
| `bulk_match()` | O(T × C) | O(T × C) | T = transactions, C = avg candidates per txn |
| `calculate_confidence()` | O(1) | O(1) | Simple arithmetic |
| `validate_match()` | O(1) | O(1) | Single pair validation |

### Performance Optimization

```python
# 1. Database indexes
CREATE INDEX idx_transfer_lookup ON canonical_transactions(
    user_id, date, amount
) WHERE deleted_at IS NULL;

CREATE INDEX idx_relationship_check ON relationships(
    txn_1_id, txn_2_id
) WHERE deleted_at IS NULL;

# 2. Query optimization
# - Filter by date range first (most selective)
# - Then amount range
# - Then currency/account
# - Exclude already linked (JOIN optimization)

# 3. Caching strategy
# - Cache match results for frequently queried transactions
# - TTL: 5 minutes
# - Invalidate on relationship creation

# 4. Batch processing
# - Process bulk_match in batches of 100 transactions
# - Use connection pooling
# - Parallel processing for independent batches
```

### Performance Targets

| Metric | Target (p95) | Notes |
|--------|--------------|-------|
| Single match query | <200ms | Including DB query + scoring |
| Bulk match (100 txns) | <10s | With batch processing |
| Validate match | <50ms | Simple pair validation |
| Cache hit rate | >80% | For frequently matched txns |

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Amount score calculation
def test_amount_score_perfect_match()
def test_amount_score_within_tolerance()
def test_amount_score_beyond_tolerance()
def test_amount_score_zero_amount()

# Test: Date score calculation
def test_date_score_same_day()
def test_date_score_next_day()
def test_date_score_within_window()
def test_date_score_beyond_window()

# Test: Confidence calculation
def test_confidence_perfect_transfer()
def test_confidence_with_fee()
def test_confidence_delayed_transfer()
def test_confidence_fx_conversion()

# Test: Match finding
def test_match_single_candidate()
def test_match_multiple_candidates()
def test_match_no_candidates()
def test_match_already_linked()

# Test: Bulk matching
def test_bulk_match_empty_range()
def test_bulk_match_with_results()
def test_bulk_match_batch_processing()

# Test: Validation
def test_validate_match_success()
def test_validate_match_already_linked()
def test_validate_match_same_transaction()
def test_validate_match_unauthorized()

# Test: Relationship type classification
def test_classify_transfer()
def test_classify_fx_conversion()
def test_classify_reimbursement()
def test_classify_correction()
```

### Integration Tests

```python
def test_end_to_end_transfer_matching():
    """Test complete transfer detection flow."""
    # Setup
    txn_1 = create_transaction(
        user_id="user_test",
        account_id="acc_bofa",
        amount=Decimal("-1000.00"),
        date=date(2025, 10, 15)
    )

    txn_2 = create_transaction(
        user_id="user_test",
        account_id="acc_wise",
        amount=Decimal("1000.00"),
        date=date(2025, 10, 15)
    )

    # Execute
    matcher = RelationshipMatcher(txn_store)
    candidates = matcher.match(txn_1.txn_id, "user_test")

    # Verify
    assert len(candidates.matches) == 1
    assert candidates.best_match.candidate_txn_id == txn_2.txn_id
    assert candidates.best_match.confidence >= 0.90
    assert candidates.best_match.match_type == RelationshipType.TRANSFER

def test_fx_conversion_with_rate():
    """Test FX conversion detection and rate calculation."""
    txn_usd = create_transaction(
        user_id="user_test",
        amount=Decimal("-1000.00"),
        currency="USD",
        date=date(2025, 10, 15)
    )

    txn_mxn = create_transaction(
        user_id="user_test",
        amount=Decimal("18500.00"),
        currency="MXN",
        date=date(2025, 10, 15)
    )

    matcher = RelationshipMatcher(txn_store)
    candidates = matcher.match(txn_usd.txn_id, "user_test")

    assert candidates.best_match.match_type == RelationshipType.FX_CONVERSION
    assert candidates.best_match.exchange_rate == Decimal("18.5")
```

### Edge Case Tests

```python
def test_match_with_multiple_same_amount():
    """Three $100 transactions on same day."""
    txn_1 = create_transaction(amount=Decimal("-100.00"), account="acc_1")
    txn_2 = create_transaction(amount=Decimal("100.00"), account="acc_2")
    txn_3 = create_transaction(amount=Decimal("100.00"), account="acc_3")
    txn_4 = create_transaction(amount=Decimal("100.00"), account="acc_4")

    candidates = matcher.match(txn_1.txn_id, "user_test")

    assert len(candidates.matches) == 3
    assert {m.candidate_txn_id for m in candidates.matches} == {
        txn_2.txn_id, txn_3.txn_id, txn_4.txn_id
    }

def test_match_with_tolerance_boundary():
    """Test exact tolerance boundary (5% = $50 on $1,000)."""
    txn_1 = create_transaction(amount=Decimal("-1000.00"))
    txn_2 = create_transaction(amount=Decimal("950.00"))  # Exactly 5% diff

    candidates = matcher.match(txn_1.txn_id, "user_test")

    assert len(candidates.matches) == 1
    assert candidates.best_match.confidence > 0.0  # Should match at boundary
```

---

## Related Primitives

- **RelationshipStore** (OL) - Persists matched relationships
- **TransferDetector** (OL) - Uses RelationshipMatcher to auto-detect transfers
- **FXConverter** (OL) - Calculates exchange rates for FX conversions
- **CanonicalStore** (OL) - Provides transaction data for matching
- **TransferLinkDialog** (IL) - UI for manual linking with match preview
- **RelationshipPanel** (IL) - Displays matched relationships

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 3-4 days
**Dependencies:** CanonicalStore (1.3), FXConverter (3.5), Database with optimized indexes
