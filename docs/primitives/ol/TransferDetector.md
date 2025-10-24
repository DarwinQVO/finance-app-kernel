# TransferDetector (OL Primitive)

**Vertical:** 3.5 Relationships (Transaction-to-Transaction Linking)
**Layer:** Objective Layer (OL)
**Type:** Paired Record Matching + Confidence Scoring Engine
**Status:** ✅ Specified

---

## Purpose

Auto-detect transfer pairs between accounts based on matching criteria (amount, date, account, opposite signs) and calculate confidence scores for pairing suggestions.

**Core Responsibility:** Analyze newly uploaded or existing transactions to identify potential transfer relationships. Match on amount (±5% tolerance), date proximity (±7 days), opposite signs, different accounts, and same user. Return ranked candidates with confidence scores to enable auto-suggestion and reduce manual linking effort.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY paired record detection task:

| Domain | Paired Record Type | Matching Criteria | Edge Cases |
|--------|-------------------|-------------------|------------|
| **Finance** | Account transfers, FX conversions | Amount, date, opposite signs, different accounts | Fees (amount mismatch), delays, multiple candidates |
| **Healthcare** | Claim-Payment pairs | Claim ID, patient, amount, date | Partial reimbursements, processing delays |
| **Legal** | Billing-Payment pairs | Invoice number, client, amount | Payment plans, partial payments |
| **Research** | Grant-Expense pairs | Grant ID, amount, date | Multi-grant expenses, cost sharing |
| **E-commerce** | Order-Return pairs | Order ID, SKU, amount | Partial returns, restocking fees |
| **Logistics** | Shipment-Delivery pairs | Tracking number, destination, date | Split shipments, delivery delays |

**Pattern:** Paired Record Matching = find candidates based on fuzzy criteria → calculate confidence score from weighted features → return ranked suggestions → support manual override.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Dict
from dataclasses import dataclass
from datetime import date, timedelta
from decimal import Decimal

@dataclass
class TransferCandidate:
    """Represents a potential transfer match."""
    txn_id: str
    confidence: float  # [0.0, 1.0]
    match_type: str  # "transfer", "fx_conversion"
    amount: Decimal
    date: date
    account_id: str
    currency: str
    match_details: Dict  # Feature breakdown (amount_score, date_score, etc.)

@dataclass
class MatchConfig:
    """Configuration for transfer detection."""
    amount_tolerance_pct: float = 0.05  # 5% tolerance
    date_window_days: int = 7  # ±7 days
    search_window_days: int = 30  # Search last 30 days for performance
    min_confidence: float = 0.50  # Minimum confidence to suggest
    require_opposite_signs: bool = True
    require_different_accounts: bool = True
    allow_fx_conversion: bool = True  # Detect different currencies

class TransferDetector:
    """
    Auto-detect transfer pairs based on matching criteria.

    Detection algorithm:
    1. Find candidates: opposite amount, same user, recent dates
    2. Calculate confidence: weighted score from features
    3. Rank by confidence
    4. Filter by minimum threshold
    5. Exclude already-linked transactions

    Confidence scoring:
    - Amount match: 40% weight (perfect = 0.40, ±2% = 0.35, ±5% = 0.25)
    - Date proximity: 30% weight (same day = 0.30, +1 day = 0.25, +3 days = 0.20)
    - Opposite signs: 20% weight (required)
    - Different accounts: 10% weight (required)

    Confidence thresholds:
    - ≥0.90: High confidence (auto-suggest with default accept)
    - 0.70-0.89: Medium confidence (auto-suggest)
    - 0.50-0.69: Low confidence (show in "Possible Matches")
    - <0.50: No suggestion (manual link only)
    """

    def __init__(
        self,
        canonical_store,
        relationship_store,
        account_store,
        config: Optional[MatchConfig] = None
    ):
        self.canonical_store = canonical_store
        self.relationship_store = relationship_store
        self.account_store = account_store
        self.config = config or MatchConfig()

    def detect_on_upload(
        self,
        txn_id: str
    ) -> List[TransferCandidate]:
        """
        Detect transfer candidates for newly uploaded transaction.

        This is the primary entry point, called automatically after
        transaction normalization completes. Runs as background job
        to avoid blocking user.

        Algorithm:
        1. Get transaction details
        2. Query candidates: opposite amount, same user, recent dates
        3. Calculate confidence for each candidate
        4. Filter by minimum confidence threshold
        5. Sort by confidence (descending)
        6. Return top 10 candidates

        Args:
            txn_id: Transaction to find matches for

        Returns:
            List of TransferCandidate objects, sorted by confidence DESC
            Empty list if no matches above threshold

        Example:
            # After user uploads BofA statement showing $1,000 withdrawal
            txn_bofa = canonical_store.get("txn_001")
            # amount: -1000.00, date: 2025-10-15, account: BofA Checking

            # Later uploads Wise statement showing $1,000 deposit
            txn_wise = canonical_store.get("txn_002")
            # amount: +1000.00, date: 2025-10-15, account: Wise USD

            # Auto-detection runs
            candidates = detector.detect_on_upload("txn_002")

            # Returns:
            # [TransferCandidate(
            #   txn_id="txn_001",
            #   confidence=1.0,
            #   match_type="transfer",
            #   amount=-1000.00,
            #   date=2025-10-15,
            #   account_id="acc_bofa_checking",
            #   currency="USD",
            #   match_details={
            #     "amount_score": 0.40,  # Perfect match
            #     "date_score": 0.30,    # Same day
            #     "sign_score": 0.20,    # Opposite signs
            #     "account_score": 0.10  # Different accounts
            #   }
            # )]
        """

    def detect_batch(
        self,
        user_id: str,
        date_from: date,
        date_to: date,
        limit: int = 100
    ) -> Dict[str, List[TransferCandidate]]:
        """
        Detect transfers for multiple transactions in batch (optimized).

        Useful for:
        - Bulk analysis of existing transactions
        - Re-running detection after config changes
        - Background job to process unlinked transactions

        Optimization: Single query to load all transactions in range,
        then in-memory matching to avoid N+1 queries.

        Args:
            user_id: User to analyze
            date_from: Start date for analysis
            date_to: End date for analysis
            limit: Max transactions to analyze (default 100)

        Returns:
            Dictionary: txn_id → List[TransferCandidate]
            Only includes transactions with matches

        Example:
            # Analyze October transactions
            results = detector.detect_batch(
                user_id="user_darwin",
                date_from=date(2025, 10, 1),
                date_to=date(2025, 10, 31)
            )

            # Results:
            # {
            #   "txn_001": [TransferCandidate(...), TransferCandidate(...)],
            #   "txn_005": [TransferCandidate(...)],
            #   ...
            # }

            # Show summary
            for txn_id, candidates in results.items():
                txn = canonical_store.get(txn_id)
                print(f"{txn.amount:>10} | {candidates[0].confidence:.0%} match")
        """

    def calculate_confidence(
        self,
        txn_1: Transaction,
        txn_2: Transaction
    ) -> float:
        """
        Calculate confidence score for transfer pair.

        Confidence formula (weighted sum):

        1. Amount Match (40% weight):
           - amount_diff = |abs(txn_1.amount) - abs(txn_2.amount)|
           - amount_ratio = amount_diff / abs(txn_1.amount)
           - If ratio == 0: score += 0.40 (perfect)
           - If ratio <= 0.02: score += 0.35 (±2%)
           - If ratio <= 0.05: score += 0.25 (±5%)
           - Else: score += 0.10 (weak)

        2. Date Proximity (30% weight):
           - date_diff = |txn_1.date - txn_2.date| (days)
           - If diff == 0: score += 0.30 (same day)
           - If diff <= 1: score += 0.25 (next day)
           - If diff <= 3: score += 0.20 (within 3 days)
           - If diff <= 7: score += 0.10 (within week)
           - Else: score += 0.0 (too far)

        3. Opposite Signs (20% weight):
           - If (txn_1.amount > 0 AND txn_2.amount < 0) OR
                (txn_1.amount < 0 AND txn_2.amount > 0):
             score += 0.20
           - Else: score += 0.0

        4. Different Accounts, Same User (10% weight):
           - If txn_1.account_id != txn_2.account_id AND
                txn_1.user_id == txn_2.user_id:
             score += 0.10
           - Else: score += 0.0

        Total: min(score, 1.0)  # Cap at 1.0

        Args:
            txn_1: First transaction
            txn_2: Second transaction

        Returns:
            Confidence score [0.0, 1.0]

        Example:
            # Perfect match
            txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15), account_id="acc_bofa")
            txn_2 = Transaction(amount=+1000.00, date=date(2025,10,15), account_id="acc_wise")
            confidence = detector.calculate_confidence(txn_1, txn_2)
            # Returns: 1.0 (0.40 + 0.30 + 0.20 + 0.10)

            # With $2 fee
            txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15), account_id="acc_bofa")
            txn_2 = Transaction(amount=+998.00, date=date(2025,10,15), account_id="acc_wise")
            confidence = detector.calculate_confidence(txn_1, txn_2)
            # Returns: 0.95 (0.35 + 0.30 + 0.20 + 0.10)
            # Amount ratio: 2/1000 = 0.002 = 0.2% (within 2%)

            # With 4-day delay
            txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15), account_id="acc_bofa")
            txn_2 = Transaction(amount=+1000.00, date=date(2025,10,19), account_id="acc_wise")
            confidence = detector.calculate_confidence(txn_1, txn_2)
            # Returns: 0.80 (0.40 + 0.10 + 0.20 + 0.10)
            # Date diff: 4 days (within 7 but >3)
        """

    def find_fx_conversion(
        self,
        txn_id: str
    ) -> Optional[TransferCandidate]:
        """
        Specialized detection for FX conversions (different currencies).

        FX conversions have unique characteristics:
        - Different currencies (USD → MXN)
        - Same institution (Wise, Revolut, etc.)
        - Usually same day or next day
        - Different amounts (exchange rate applied)

        Confidence scoring adjustments for FX:
        - Amount match NOT required (different currencies)
        - Date proximity MORE important (30% → 40%)
        - Same institution bonus (+10%)
        - Exchange rate reasonableness check

        Args:
            txn_id: Transaction to find FX pair for

        Returns:
            TransferCandidate with match_type="fx_conversion"
            None if no FX conversion found

        Example:
            # Wise USD → MXN conversion
            txn_usd = Transaction(
                amount=-1000.00, currency="USD",
                date=date(2025,10,16), account_id="acc_wise_usd"
            )
            txn_mxn = Transaction(
                amount=+18500.00, currency="MXN",
                date=date(2025,10,16), account_id="acc_wise_mxn"
            )

            candidate = detector.find_fx_conversion("txn_usd")

            # Returns:
            # TransferCandidate(
            #   txn_id="txn_mxn",
            #   confidence=0.98,
            #   match_type="fx_conversion",
            #   match_details={
            #     "exchange_rate": 18.5,
            #     "same_institution": True,
            #     "date_score": 0.40,
            #     ...
            #   }
            # )
        """

    def is_already_linked(
        self,
        txn_id: str
    ) -> bool:
        """
        Check if transaction is already in a relationship.

        Used to prevent:
        - Duplicate suggestions
        - Linking already-linked transactions
        - Re-suggesting dismissed links

        Args:
            txn_id: Transaction to check

        Returns:
            True if transaction has active relationship
            False if transaction is unlinked

        Example:
            if not detector.is_already_linked("txn_001"):
                candidates = detector.detect_on_upload("txn_001")
                # Show suggestions...
            else:
                # Skip detection, already linked
                pass
        """

    def get_match_explanation(
        self,
        txn_1: Transaction,
        txn_2: Transaction
    ) -> Dict:
        """
        Generate human-readable explanation for match.

        Useful for:
        - UI tooltips
        - User feedback
        - Debugging detection logic

        Args:
            txn_1: First transaction
            txn_2: Second transaction

        Returns:
            Explanation dictionary with feature breakdown

        Example:
            explanation = detector.get_match_explanation(txn_bofa, txn_wise)

            # Returns:
            # {
            #   "confidence": 1.0,
            #   "features": {
            #     "amount": {
            #       "match": True,
            #       "txn_1": -1000.00,
            #       "txn_2": +1000.00,
            #       "diff": 0.0,
            #       "diff_pct": 0.0,
            #       "score": 0.40,
            #       "explanation": "Amounts match exactly"
            #     },
            #     "date": {
            #       "match": True,
            #       "txn_1": "2025-10-15",
            #       "txn_2": "2025-10-15",
            #       "diff_days": 0,
            #       "score": 0.30,
            #       "explanation": "Same day"
            #     },
            #     "signs": {
            #       "match": True,
            #       "txn_1": "negative",
            #       "txn_2": "positive",
            #       "score": 0.20,
            #       "explanation": "Opposite signs (withdrawal → deposit)"
            #     },
            #     "accounts": {
            #       "match": True,
            #       "txn_1": "BofA Checking",
            #       "txn_2": "Wise USD",
            #       "same_user": True,
            #       "score": 0.10,
            #       "explanation": "Different accounts, same user"
            #     }
            #   },
            #   "total_score": 1.0,
            #   "recommendation": "Very high confidence - suggest auto-link"
            # }
        """

    def _query_candidates(
        self,
        txn: Transaction,
        config: MatchConfig
    ) -> List[Transaction]:
        """
        Query database for potential transfer candidates.

        SQL query optimized with indexes:
        - user_id, date, amount index
        - Exclude already-linked transactions
        - Limit search window for performance

        Internal method.
        """

    def _calculate_amount_score(
        self,
        amount_1: Decimal,
        amount_2: Decimal
    ) -> float:
        """
        Calculate amount match score (0.0 to 0.40).

        Internal method.
        """

    def _calculate_date_score(
        self,
        date_1: date,
        date_2: date
    ) -> float:
        """
        Calculate date proximity score (0.0 to 0.30).

        Internal method.
        """

    def _calculate_sign_score(
        self,
        amount_1: Decimal,
        amount_2: Decimal
    ) -> float:
        """
        Calculate opposite signs score (0.0 or 0.20).

        Internal method.
        """

    def _calculate_account_score(
        self,
        txn_1: Transaction,
        txn_2: Transaction
    ) -> float:
        """
        Calculate account difference score (0.0 or 0.10).

        Internal method.
        """
```

---

## Implementation Details

### Core Detection Algorithm

```python
def detect_on_upload(self, txn_id: str) -> List[TransferCandidate]:
    """
    Detect transfer candidates for newly uploaded transaction.

    Steps:
    1. Validate transaction exists and is not already linked
    2. Query candidates from database
    3. Calculate confidence for each candidate
    4. Filter by minimum threshold
    5. Sort by confidence
    6. Return top candidates
    """
    # Step 1: Get transaction
    txn = self.canonical_store.get(txn_id)
    if not txn:
        raise NotFoundError(f"Transaction {txn_id} not found")

    # Check if already linked
    if self.is_already_linked(txn_id):
        return []  # Don't suggest for already-linked transactions

    # Step 2: Query candidates
    candidates = self._query_candidates(txn, self.config)

    # Step 3: Calculate confidence for each
    results = []
    for candidate in candidates:
        # Skip if candidate is already linked
        if self.is_already_linked(candidate.txn_id):
            continue

        confidence = self.calculate_confidence(txn, candidate)

        # Filter by threshold
        if confidence < self.config.min_confidence:
            continue

        # Determine match type
        match_type = "fx_conversion" if txn.currency != candidate.currency else "transfer"

        # Build match details
        match_details = {
            "amount_score": self._calculate_amount_score(txn.amount, candidate.amount),
            "date_score": self._calculate_date_score(txn.date, candidate.date),
            "sign_score": self._calculate_sign_score(txn.amount, candidate.amount),
            "account_score": self._calculate_account_score(txn, candidate)
        }

        results.append(TransferCandidate(
            txn_id=candidate.txn_id,
            confidence=confidence,
            match_type=match_type,
            amount=candidate.amount,
            date=candidate.date,
            account_id=candidate.account_id,
            currency=candidate.currency,
            match_details=match_details
        ))

    # Step 4: Sort by confidence (descending)
    results.sort(key=lambda x: x.confidence, reverse=True)

    # Step 5: Return top 10
    return results[:10]
```

### Candidate Query (Optimized)

```python
def _query_candidates(
    self,
    txn: Transaction,
    config: MatchConfig
) -> List[Transaction]:
    """
    Query database for potential transfer candidates.

    Optimization strategies:
    1. Use indexed columns (user_id, date, amount)
    2. Limit search window (last 30 days) for performance
    3. Pre-filter by amount tolerance (±5%) to reduce rows
    4. Exclude already-linked transactions
    5. Limit results to 50 for scoring
    """
    # Calculate amount range
    amount_min = abs(txn.amount) * (1 - config.amount_tolerance_pct)
    amount_max = abs(txn.amount) * (1 + config.amount_tolerance_pct)

    # Calculate date range
    date_min = txn.date - timedelta(days=config.search_window_days)
    date_max = txn.date + timedelta(days=config.search_window_days)

    # Query with optimizations
    query = """
        SELECT t.*
        FROM canonical_transactions t
        WHERE t.user_id = :user_id
          AND t.txn_id != :txn_id
          AND t.date >= :date_min
          AND t.date <= :date_max
          AND ABS(t.amount) >= :amount_min
          AND ABS(t.amount) <= :amount_max
          AND SIGN(t.amount) != SIGN(:amount)  -- Opposite signs
          AND t.account_id != :account_id  -- Different accounts
          AND t.deleted_at IS NULL
          AND t.txn_id NOT IN (
              -- Exclude already-linked transactions
              SELECT txn_1_id FROM relationships WHERE deleted_at IS NULL
              UNION
              SELECT txn_2_id FROM relationships WHERE deleted_at IS NULL
          )
        ORDER BY
          ABS(t.date - :date),  -- Closest date first
          ABS(ABS(t.amount) - ABS(:amount))  -- Closest amount second
        LIMIT 50
    """

    params = {
        "user_id": txn.user_id,
        "txn_id": txn.txn_id,
        "date_min": date_min,
        "date_max": date_max,
        "amount_min": amount_min,
        "amount_max": amount_max,
        "amount": txn.amount,
        "account_id": txn.account_id,
        "date": txn.date
    }

    return self.canonical_store.query(query, params)
```

### Confidence Calculation (Detailed)

```python
def calculate_confidence(
    self,
    txn_1: Transaction,
    txn_2: Transaction
) -> float:
    """
    Calculate confidence score using weighted feature sum.

    Formula:
      confidence = amount_score + date_score + sign_score + account_score

    Maximum possible: 1.0
    """
    score = 0.0

    # Feature 1: Amount Match (40% weight)
    score += self._calculate_amount_score(txn_1.amount, txn_2.amount)

    # Feature 2: Date Proximity (30% weight)
    score += self._calculate_date_score(txn_1.date, txn_2.date)

    # Feature 3: Opposite Signs (20% weight)
    score += self._calculate_sign_score(txn_1.amount, txn_2.amount)

    # Feature 4: Different Accounts (10% weight)
    score += self._calculate_account_score(txn_1, txn_2)

    return min(score, 1.0)  # Cap at 1.0

def _calculate_amount_score(
    self,
    amount_1: Decimal,
    amount_2: Decimal
) -> float:
    """
    Calculate amount match score (0.0 to 0.40).

    Scoring:
    - Perfect match (0% diff): 0.40
    - ≤2% difference: 0.35
    - ≤5% difference: 0.25
    - >5% difference: 0.10
    """
    amount_diff = abs(abs(amount_1) - abs(amount_2))
    amount_ratio = amount_diff / abs(amount_1)

    if amount_ratio == 0:
        return 0.40  # Perfect match
    elif amount_ratio <= 0.02:
        return 0.35  # Within 2%
    elif amount_ratio <= 0.05:
        return 0.25  # Within 5%
    else:
        return 0.10  # Weak match

def _calculate_date_score(
    self,
    date_1: date,
    date_2: date
) -> float:
    """
    Calculate date proximity score (0.0 to 0.30).

    Scoring:
    - Same day: 0.30
    - +1 day: 0.25
    - +2-3 days: 0.20
    - +4-7 days: 0.10
    - >7 days: 0.0
    """
    date_diff = abs((date_1 - date_2).days)

    if date_diff == 0:
        return 0.30  # Same day
    elif date_diff <= 1:
        return 0.25  # Next day
    elif date_diff <= 3:
        return 0.20  # Within 3 days
    elif date_diff <= 7:
        return 0.10  # Within week
    else:
        return 0.0  # Too far apart

def _calculate_sign_score(
    self,
    amount_1: Decimal,
    amount_2: Decimal
) -> float:
    """
    Calculate opposite signs score (0.0 or 0.20).

    Transfers must have opposite signs:
    - One withdrawal (negative)
    - One deposit (positive)
    """
    opposite_signs = (
        (amount_1 > 0 and amount_2 < 0) or
        (amount_1 < 0 and amount_2 > 0)
    )

    return 0.20 if opposite_signs else 0.0

def _calculate_account_score(
    self,
    txn_1: Transaction,
    txn_2: Transaction
) -> float:
    """
    Calculate account difference score (0.0 or 0.10).

    Transfers must be between different accounts
    but same user.
    """
    different_accounts = txn_1.account_id != txn_2.account_id
    same_user = txn_1.user_id == txn_2.user_id

    return 0.10 if (different_accounts and same_user) else 0.0
```

### Batch Detection (Optimized)

```python
def detect_batch(
    self,
    user_id: str,
    date_from: date,
    date_to: date,
    limit: int = 100
) -> Dict[str, List[TransferCandidate]]:
    """
    Detect transfers for multiple transactions in batch.

    Optimization: Load all transactions once, match in-memory
    to avoid N+1 queries.
    """
    # Step 1: Load all unlinked transactions in date range
    all_txns = self.canonical_store.list(
        user_id=user_id,
        date_from=date_from,
        date_to=date_to,
        deleted_at=None,
        limit=limit
    )

    # Step 2: Filter out already-linked transactions
    linked_txn_ids = set(self._get_linked_transaction_ids(user_id))
    unlinked_txns = [t for t in all_txns if t.txn_id not in linked_txn_ids]

    # Step 3: For each transaction, find matches from same list
    results = {}

    for txn in unlinked_txns:
        candidates = []

        # Compare against all other transactions
        for candidate_txn in unlinked_txns:
            # Skip self
            if candidate_txn.txn_id == txn.txn_id:
                continue

            # Skip if already processed (avoid duplicate pairs)
            if candidate_txn.txn_id in results:
                continue

            # Check basic criteria
            if not self._meets_basic_criteria(txn, candidate_txn):
                continue

            # Calculate confidence
            confidence = self.calculate_confidence(txn, candidate_txn)

            # Filter by threshold
            if confidence < self.config.min_confidence:
                continue

            # Build candidate
            match_type = "fx_conversion" if txn.currency != candidate_txn.currency else "transfer"

            candidates.append(TransferCandidate(
                txn_id=candidate_txn.txn_id,
                confidence=confidence,
                match_type=match_type,
                amount=candidate_txn.amount,
                date=candidate_txn.date,
                account_id=candidate_txn.account_id,
                currency=candidate_txn.currency,
                match_details=self._build_match_details(txn, candidate_txn)
            ))

        # Sort and store
        if candidates:
            candidates.sort(key=lambda x: x.confidence, reverse=True)
            results[txn.txn_id] = candidates[:10]  # Top 10 per transaction

    return results

def _meets_basic_criteria(
    self,
    txn_1: Transaction,
    txn_2: Transaction
) -> bool:
    """
    Fast pre-filter before calculating confidence.

    Checks:
    - Different accounts
    - Same user
    - Opposite signs
    - Amount within tolerance
    - Date within window
    """
    # Different accounts
    if txn_1.account_id == txn_2.account_id:
        return False

    # Same user
    if txn_1.user_id != txn_2.user_id:
        return False

    # Opposite signs
    if (txn_1.amount > 0 and txn_2.amount > 0) or \
       (txn_1.amount < 0 and txn_2.amount < 0):
        return False

    # Amount tolerance
    amount_diff = abs(abs(txn_1.amount) - abs(txn_2.amount))
    amount_ratio = amount_diff / abs(txn_1.amount)
    if amount_ratio > self.config.amount_tolerance_pct:
        return False

    # Date window
    date_diff = abs((txn_1.date - txn_2.date).days)
    if date_diff > self.config.date_window_days:
        return False

    return True
```

### FX Conversion Detection

```python
def find_fx_conversion(
    self,
    txn_id: str
) -> Optional[TransferCandidate]:
    """
    Specialized detection for FX conversions.

    Key differences from regular transfers:
    - Requires different currencies
    - Prefers same institution/account family
    - More lenient on amount matching
    - Stricter on date matching
    """
    txn = self.canonical_store.get(txn_id)
    if not txn:
        return None

    # Query candidates: different currency, same user, close date
    query = """
        SELECT t.*
        FROM canonical_transactions t
        WHERE t.user_id = :user_id
          AND t.txn_id != :txn_id
          AND t.currency != :currency  -- Different currency
          AND t.date >= :date - INTERVAL '3 days'
          AND t.date <= :date + INTERVAL '3 days'  -- Tighter date window
          AND SIGN(t.amount) != SIGN(:amount)  -- Opposite signs
          AND t.deleted_at IS NULL
          AND t.txn_id NOT IN (
              SELECT txn_1_id FROM relationships WHERE deleted_at IS NULL
              UNION
              SELECT txn_2_id FROM relationships WHERE deleted_at IS NULL
          )
        ORDER BY ABS(t.date - :date)
        LIMIT 10
    """

    candidates = self.canonical_store.query(query, {
        "user_id": txn.user_id,
        "txn_id": txn.txn_id,
        "currency": txn.currency,
        "date": txn.date,
        "amount": txn.amount
    })

    # Calculate FX-specific confidence
    best_candidate = None
    best_confidence = 0.0

    for candidate in candidates:
        # FX confidence scoring
        confidence = 0.0

        # Date proximity (40% weight - more important for FX)
        date_diff = abs((txn.date - candidate.date).days)
        if date_diff == 0:
            confidence += 0.40
        elif date_diff <= 1:
            confidence += 0.30
        else:
            confidence += 0.15

        # Same institution bonus (20% weight)
        same_institution = self._same_institution(txn.account_id, candidate.account_id)
        if same_institution:
            confidence += 0.20

        # Opposite signs (20% weight)
        opposite_signs = (txn.amount > 0 and candidate.amount < 0) or \
                        (txn.amount < 0 and candidate.amount > 0)
        if opposite_signs:
            confidence += 0.20

        # Exchange rate reasonableness (20% weight)
        exchange_rate = abs(candidate.amount) / abs(txn.amount)
        if self._is_reasonable_fx_rate(txn.currency, candidate.currency, exchange_rate):
            confidence += 0.20
        else:
            confidence += 0.10  # Suspicious rate, lower score

        if confidence > best_confidence:
            best_confidence = confidence
            best_candidate = candidate

    # Return best match if above threshold
    if best_candidate and best_confidence >= self.config.min_confidence:
        return TransferCandidate(
            txn_id=best_candidate.txn_id,
            confidence=best_confidence,
            match_type="fx_conversion",
            amount=best_candidate.amount,
            date=best_candidate.date,
            account_id=best_candidate.account_id,
            currency=best_candidate.currency,
            match_details={
                "exchange_rate": abs(best_candidate.amount) / abs(txn.amount),
                "same_institution": self._same_institution(txn.account_id, best_candidate.account_id),
                "date_diff_days": abs((txn.date - best_candidate.date).days)
            }
        )

    return None

def _same_institution(self, account_id_1: str, account_id_2: str) -> bool:
    """
    Check if two accounts belong to same institution.

    Example: "acc_wise_usd" and "acc_wise_mxn" → True
    """
    account_1 = self.account_store.get(account_id_1)
    account_2 = self.account_store.get(account_id_2)

    return account_1.institution_id == account_2.institution_id

def _is_reasonable_fx_rate(
    self,
    from_currency: str,
    to_currency: str,
    rate: Decimal
) -> bool:
    """
    Sanity check on exchange rate.

    Prevents matching unrelated transactions with
    wildly different amounts.

    Example checks:
    - USD/MXN rate should be ~15-25 (not 1000)
    - USD/EUR rate should be ~0.8-1.2 (not 0.01)
    """
    # Define expected ranges for common currency pairs
    expected_ranges = {
        ("USD", "MXN"): (15.0, 25.0),
        ("USD", "EUR"): (0.8, 1.2),
        ("USD", "GBP"): (0.7, 0.9),
        ("USD", "CAD"): (1.2, 1.4),
        ("USD", "JPY"): (100.0, 150.0),
        # ... more pairs
    }

    # Check both directions
    pair = (from_currency, to_currency)
    reverse_pair = (to_currency, from_currency)

    if pair in expected_ranges:
        min_rate, max_rate = expected_ranges[pair]
        return min_rate <= rate <= max_rate
    elif reverse_pair in expected_ranges:
        min_rate, max_rate = expected_ranges[reverse_pair]
        inverse_rate = 1 / rate
        return min_rate <= inverse_rate <= max_rate
    else:
        # Unknown pair, allow if rate is not extreme
        return 0.001 <= rate <= 1000
```

---

## Edge Cases

### EC1: Transfer with Fee (Amount Mismatch)

**Scenario:** BofA sends $1,000, Wise receives $998 (Wise charges $2 fee).

**Detection:**
```python
txn_bofa = Transaction(amount=-1000.00, date=date(2025,10,15))
txn_wise = Transaction(amount=+998.00, date=date(2025,10,15))

# Amount difference: $2
# Amount ratio: 2/1000 = 0.002 = 0.2%

confidence = detector.calculate_confidence(txn_bofa, txn_wise)
# Returns: 0.95
# Breakdown: 0.35 (amount, within 2%) + 0.30 (date) + 0.20 (signs) + 0.10 (accounts)
```

**Outcome:** Still high confidence (0.95), auto-suggest with note about fee.

**UI Message:**
```
⚠️ Amount Mismatch
From: BofA  -$1,000.00
To:   Wise  +$998.00

Difference: $2.00 (may be transfer fee)

[Link as Transfer]  [Dismiss]
```

### EC2: Delayed Transfer (Date Gap)

**Scenario:** BofA withdrawal on 10/15, Wise deposit on 10/19 (4-day delay).

**Detection:**
```python
txn_bofa = Transaction(amount=-1000.00, date=date(2025,10,15))
txn_wise = Transaction(amount=+1000.00, date=date(2025,10,19))

# Date difference: 4 days (within 7-day window)

confidence = detector.calculate_confidence(txn_bofa, txn_wise)
# Returns: 0.80
# Breakdown: 0.40 (amount) + 0.10 (date, 4-7 days) + 0.20 (signs) + 0.10 (accounts)
```

**Outcome:** Medium-high confidence (0.80), still auto-suggest.

**UI Message:**
```
🔗 Potential Transfer (4-day delay)
From: BofA Checking  -$1,000.00  (Oct 15)
To:   Wise USD       +$1,000.00  (Oct 19)

Confidence: 80%

[Link as Transfer]  [Dismiss]
```

### EC3: Multiple Candidates (Ambiguous)

**Scenario:** Three $100 transactions on same day.

**Detection:**
```python
txn_bofa = Transaction(amount=-100.00, date=date(2025,10,15), account_id="acc_bofa")
txn_wise = Transaction(amount=+100.00, date=date(2025,10,15), account_id="acc_wise")
txn_chase = Transaction(amount=+100.00, date=date(2025,10,15), account_id="acc_chase")
txn_venmo = Transaction(amount=+100.00, date=date(2025,10,15), account_id="acc_venmo")

candidates = detector.detect_on_upload(txn_bofa.txn_id)
# Returns 3 candidates, all with confidence = 1.0
```

**Outcome:** Show all candidates, let user choose.

**UI Message:**
```
🔗 Multiple Potential Transfers Found

BofA Checking  -$100.00  (Oct 15)

Could be linked to:
◯ Wise USD       +$100.00  (Oct 15)  [95% confidence]
◯ Chase Savings  +$100.00  (Oct 15)  [95% confidence]
◯ Venmo         +$100.00  (Oct 15)  [95% confidence]

Select one:
[Link Selected]  [Dismiss All]
```

### EC4: Already Linked Transaction

**Scenario:** User tries to link transaction that's already in a relationship.

**Detection:**
```python
# Transaction already linked
if detector.is_already_linked("txn_001"):
    # Don't show in suggestions
    return []

# Validation on manual link creation
def create_relationship(txn_1_id, txn_2_id):
    if detector.is_already_linked(txn_1_id):
        raise AlreadyLinkedError(f"Transaction {txn_1_id} already linked")
```

**UI Message:**
```
❌ Cannot Link Transaction

This transaction is already linked:
• Existing: Transfer to Wise USD (rel_001)

Unlink the existing relationship first.
```

### EC5: Chain Transfers (Multi-Hop)

**Scenario:** BofA → Wise USD → Wise MXN → Scotia.

**Detection:**
```python
# Each pair detected separately
pair_1 = detector.detect_on_upload("txn_bofa")
# Finds: txn_wise_usd (confidence: 0.95)

pair_2 = detector.detect_on_upload("txn_wise_usd_out")
# Finds: txn_wise_mxn (confidence: 0.98, type: fx_conversion)

pair_3 = detector.detect_on_upload("txn_wise_mxn_out")
# Finds: txn_scotia (confidence: 0.95)
```

**Outcome:** Three separate relationships created.

**UI Display (Transaction Detail):**
```
Transfer Chain:
BofA → Wise USD → Wise MXN → Scotia

Details:
1. BofA → Wise USD (transfer, $1,000)
2. Wise USD → Wise MXN (FX conversion, 18.5 rate)
3. Wise MXN → Scotia (transfer, $18,500)
```

### EC6: FX Conversion with Unreasonable Rate

**Scenario:** System finds candidate with suspicious exchange rate.

**Detection:**
```python
txn_usd = Transaction(amount=-100.00, currency="USD")
txn_mxn = Transaction(amount=+5000.00, currency="MXN")  # Rate: 50 (too high)

# Normal USD/MXN rate: 15-25
# This rate: 50 (suspicious)

confidence = detector.find_fx_conversion(txn_usd.txn_id).confidence
# Lower confidence due to unreasonable rate
# Uses 0.10 instead of 0.20 for rate score
```

**Outcome:** Lower confidence, still suggest but with warning.

**UI Message:**
```
💱 Potential FX Conversion (verify rate)

From: Wise USD  -$100.00
To:   Wise MXN  +$5,000.00

Exchange Rate: 50.0 MXN/USD
⚠️ This rate seems unusual (expected: 15-25)

[Link as FX Conversion]  [Dismiss]
```

---

## Multi-Domain Examples

### Healthcare: Claim-Payment Matching

```python
# Insurance claim submission
claim = Transaction(
    amount=-500.00,
    date=date(2025, 9, 1),
    category="healthcare",
    description="Doctor visit - insurance claim"
)

# Insurance reimbursement (60 days later, 90% coverage)
payment = Transaction(
    amount=+450.00,
    date=date(2025, 10, 30),
    description="Insurance reimbursement - claim #12345"
)

# Detection
candidates = detector.detect_on_upload(payment.txn_id)
# Low confidence due to:
# - Large date gap (60 days > 7-day window)
# - Amount mismatch ($50 difference = 10%)
# Returns: [] (below 0.50 threshold)

# User must manually link
relationship = RelationshipStore.create(
    txn_1_id=claim.txn_id,
    txn_2_id=payment.txn_id,
    type="reimbursement",
    confidence=None,
    detection_method="manual",
    notes="90% coverage, 10% copay"
)
```

### Legal: Billing-Payment Matching

```python
# Law firm invoice sent
invoice = Transaction(
    amount=-5000.00,
    date=date(2025, 10, 1),
    description="Legal services - October invoice"
)

# Client payment (partial, installment plan)
payment = Transaction(
    amount=+1000.00,
    date=date(2025, 10, 15),
    description="Payment - October installment"
)

# Detection
candidates = detector.detect_on_upload(payment.txn_id)
# Low confidence due to:
# - Amount mismatch (1000 vs 5000 = 80% difference)
# Returns: [] (below threshold)

# Manual link with notes
relationship = RelationshipStore.create(
    txn_1_id=invoice.txn_id,
    txn_2_id=payment.txn_id,
    type="partial_payment",
    confidence=None,
    detection_method="manual",
    notes="Installment 1 of 5 ($1,000/month)"
)
```

### E-commerce: Order-Return Matching

```python
# Original order
order = Transaction(
    amount=-250.00,
    date=date(2025, 10, 5),
    description="Amazon order #123-456"
)

# Partial return (2 of 3 items)
refund = Transaction(
    amount=+150.00,
    date=date(2025, 10, 20),
    description="Amazon refund - order #123-456"
)

# Detection
candidates = detector.detect_on_upload(refund.txn_id)
# Medium confidence:
# - Amount mismatch (150 vs 250 = 40% difference)
# - Date gap: 15 days (>7 days)
# Returns: [] (below threshold)

# Manual link
relationship = RelationshipStore.create(
    txn_1_id=order.txn_id,
    txn_2_id=refund.txn_id,
    type="partial_refund",
    confidence=None,
    detection_method="manual",
    notes="Returned 2 of 3 items"
)
```

---

## Performance Characteristics

### Query Performance

| Operation | Expected Latency (p95) | Optimization |
|-----------|------------------------|--------------|
| `detect_on_upload()` | <200ms | Indexed query, limit search window |
| `detect_batch()` (100 txns) | <2000ms | In-memory matching, avoid N+1 |
| `calculate_confidence()` | <1ms | Pure calculation, no I/O |
| `is_already_linked()` | <10ms | Cached lookup |
| `find_fx_conversion()` | <150ms | Specialized query, tighter date window |

### Database Indexes Required

```sql
-- Index for candidate lookup (most important)
CREATE INDEX idx_transfers_lookup ON canonical_transactions(
    user_id, date, amount
) WHERE deleted_at IS NULL;

-- Index for opposite sign filter
CREATE INDEX idx_transfers_amount_sign ON canonical_transactions(
    user_id, SIGN(amount), ABS(amount)
) WHERE deleted_at IS NULL;

-- Index for already-linked check
CREATE INDEX idx_relationships_txn_1 ON relationships(txn_1_id)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_relationships_txn_2 ON relationships(txn_2_id)
    WHERE deleted_at IS NULL;
```

### Search Window Optimization

**Problem:** Searching all user transactions is expensive (O(n²)).

**Solution:** Limit search window to ±30 days.

```python
# Instead of searching all transactions:
# SELECT * FROM canonical_transactions WHERE user_id = ?

# Search limited window:
SELECT * FROM canonical_transactions
WHERE user_id = ?
  AND date >= ? - INTERVAL '30 days'
  AND date <= ? + INTERVAL '30 days'
```

**Impact:**
- Old transactions excluded from search (reduces candidates)
- Misses delayed transfers >30 days (acceptable tradeoff)
- Performance: 100x faster for users with years of history

### Batch Processing Optimization

**Problem:** Calling `detect_on_upload()` for 100 transactions = 100 queries.

**Solution:** Load all transactions once, match in-memory.

```python
# BAD: N+1 queries
for txn_id in transaction_ids:
    candidates = detector.detect_on_upload(txn_id)  # Query per txn

# GOOD: Single query, in-memory matching
results = detector.detect_batch(user_id, date_from, date_to)
```

**Speedup:** 50-100x for batch operations.

---

## Confidence Scoring

### Confidence Ranges

| Confidence | Interpretation | User Action | Example |
|-----------|----------------|-------------|---------|
| **1.00** | Perfect match | Auto-suggest with high confidence | Same amount, same day, opposite signs, perfect alignment |
| **0.90-0.99** | Very high confidence | Auto-suggest | ±2% amount, same day or +1 day |
| **0.70-0.89** | High confidence | Auto-suggest | ±5% amount, +2-3 days |
| **0.50-0.69** | Medium confidence | Show in "Possible Matches" | ±5% amount, +4-7 days |
| **<0.50** | Low confidence | Manual link only | Large differences, not suggested |

### Threshold Recommendations

| Confidence Threshold | Use Case | Precision | Recall |
|---------------------|----------|-----------|--------|
| **≥0.90** | Conservative (high precision) | 98% | 60% |
| **≥0.70** | Balanced (recommended) | 90% | 80% |
| **≥0.50** | Aggressive (high recall) | 75% | 95% |

**Default:** 0.70 (balanced)

**Configurable per user:**
```python
config = MatchConfig(min_confidence=0.80)  # More conservative
detector = TransferDetector(..., config=config)
```

### Feature Weight Tuning

**Current weights:**
- Amount: 40%
- Date: 30%
- Signs: 20%
- Accounts: 10%

**Alternative for delayed transfers:**
- Amount: 50% (more important)
- Date: 20% (less important)
- Signs: 20%
- Accounts: 10%

**Alternative for FX conversions:**
- Same institution: 30%
- Date: 30%
- Signs: 20%
- Rate reasonableness: 20%
- Amount: N/A (different currencies)

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Perfect match
def test_calculate_confidence_perfect_match():
    txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15), account_id="acc_bofa")
    txn_2 = Transaction(amount=+1000.00, date=date(2025,10,15), account_id="acc_wise")

    confidence = detector.calculate_confidence(txn_1, txn_2)
    assert confidence == 1.0

# Test: With fee
def test_calculate_confidence_with_fee():
    txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15))
    txn_2 = Transaction(amount=+998.00, date=date(2025,10,15))

    confidence = detector.calculate_confidence(txn_1, txn_2)
    assert 0.90 <= confidence <= 0.95

# Test: Delayed transfer
def test_calculate_confidence_delayed():
    txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15))
    txn_2 = Transaction(amount=+1000.00, date=date(2025,10,19))

    confidence = detector.calculate_confidence(txn_1, txn_2)
    assert 0.75 <= confidence <= 0.85

# Test: Below threshold
def test_calculate_confidence_below_threshold():
    txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15))
    txn_2 = Transaction(amount=+800.00, date=date(2025,10,25))  # 20% diff, 10 days

    confidence = detector.calculate_confidence(txn_1, txn_2)
    assert confidence < 0.50

# Test: Same signs (should not match)
def test_calculate_confidence_same_signs():
    txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15))
    txn_2 = Transaction(amount=-1000.00, date=date(2025,10,15))

    confidence = detector.calculate_confidence(txn_1, txn_2)
    assert confidence < 0.50  # Missing 0.20 from sign score

# Test: FX conversion detection
def test_find_fx_conversion():
    txn_usd = Transaction(amount=-1000.00, currency="USD", date=date(2025,10,16))
    txn_mxn = Transaction(amount=+18500.00, currency="MXN", date=date(2025,10,16))

    # Setup database with both transactions
    canonical_store.create(txn_usd)
    canonical_store.create(txn_mxn)

    candidate = detector.find_fx_conversion(txn_usd.txn_id)

    assert candidate is not None
    assert candidate.match_type == "fx_conversion"
    assert candidate.confidence >= 0.90
    assert candidate.match_details["exchange_rate"] == Decimal("18.5")

# Test: Multiple candidates
def test_detect_multiple_candidates():
    txn_bofa = Transaction(amount=-100.00, date=date(2025,10,15))
    txn_wise = Transaction(amount=+100.00, date=date(2025,10,15))
    txn_chase = Transaction(amount=+100.00, date=date(2025,10,15))

    # Setup database
    canonical_store.create(txn_bofa)
    canonical_store.create(txn_wise)
    canonical_store.create(txn_chase)

    candidates = detector.detect_on_upload(txn_bofa.txn_id)

    assert len(candidates) == 2
    assert all(c.confidence == 1.0 for c in candidates)

# Test: Already linked exclusion
def test_detect_excludes_already_linked():
    txn_1 = Transaction(amount=-1000.00, date=date(2025,10,15))
    txn_2 = Transaction(amount=+1000.00, date=date(2025,10,15))
    txn_3 = Transaction(amount=-1000.00, date=date(2025,10,16))

    # Setup: txn_1 and txn_2 already linked
    canonical_store.create(txn_1)
    canonical_store.create(txn_2)
    canonical_store.create(txn_3)
    relationship_store.create(txn_1.txn_id, txn_2.txn_id, "transfer")

    # Detect for txn_3
    candidates = detector.detect_on_upload(txn_3.txn_id)

    # Should not include txn_1 or txn_2
    assert txn_1.txn_id not in [c.txn_id for c in candidates]
    assert txn_2.txn_id not in [c.txn_id for c in candidates]

# Test: Batch detection performance
def test_detect_batch_performance():
    # Create 100 transactions
    for i in range(100):
        canonical_store.create(Transaction(...))

    start = time.time()
    results = detector.detect_batch(user_id, date_from, date_to)
    duration = time.time() - start

    assert duration < 2.0  # Should complete in <2s

# Test: Config customization
def test_custom_config():
    config = MatchConfig(
        amount_tolerance_pct=0.10,  # 10% tolerance
        date_window_days=14,  # 2 weeks
        min_confidence=0.80  # Higher threshold
    )

    detector = TransferDetector(..., config=config)

    # Should find more candidates due to wider tolerance
    candidates = detector.detect_on_upload(txn_id)
    assert len(candidates) > 0
```

### Integration Test Examples

```python
# Test: End-to-end detection on upload
def test_e2e_detection_on_upload():
    # Step 1: Upload first transaction
    txn_1 = create_transaction(amount=-1000.00, date=date(2025,10,15))

    # No candidates yet
    candidates = detector.detect_on_upload(txn_1.txn_id)
    assert len(candidates) == 0

    # Step 2: Upload matching transaction
    txn_2 = create_transaction(amount=+1000.00, date=date(2025,10,15))

    # Should detect match
    candidates = detector.detect_on_upload(txn_2.txn_id)
    assert len(candidates) == 1
    assert candidates[0].txn_id == txn_1.txn_id
    assert candidates[0].confidence >= 0.90

# Test: Chain transfer detection
def test_chain_transfer_detection():
    # Create chain: BofA → Wise USD → Wise MXN → Scotia
    txn_bofa = create_transaction(amount=-1000.00, account_id="acc_bofa", date=date(2025,10,15))
    txn_wise_usd = create_transaction(amount=+1000.00, account_id="acc_wise_usd", date=date(2025,10,15))
    txn_wise_usd_out = create_transaction(amount=-1000.00, account_id="acc_wise_usd", date=date(2025,10,16))
    txn_wise_mxn = create_transaction(amount=+18500.00, currency="MXN", account_id="acc_wise_mxn", date=date(2025,10,16))
    txn_wise_mxn_out = create_transaction(amount=-18500.00, currency="MXN", account_id="acc_wise_mxn", date=date(2025,10,17))
    txn_scotia = create_transaction(amount=+18500.00, currency="MXN", account_id="acc_scotia", date=date(2025,10,17))

    # Detect each pair
    pair_1 = detector.detect_on_upload(txn_wise_usd.txn_id)
    assert len(pair_1) == 1
    assert pair_1[0].match_type == "transfer"

    pair_2 = detector.find_fx_conversion(txn_wise_usd_out.txn_id)
    assert pair_2.match_type == "fx_conversion"

    pair_3 = detector.detect_on_upload(txn_scotia.txn_id)
    assert len(pair_3) == 1
    assert pair_3[0].match_type == "transfer"
```

---

## Related Primitives

- **RelationshipStore** (OL) - Stores created relationships
- **FXConverter** (OL) - Calculates exchange rates for FX conversions
- **RelationshipMatcher** (OL) - Generic fuzzy matcher (shares logic)
- **CanonicalStore** (OL) - Source of transaction data
- **AccountStore** (OL) - Validates account ownership
- **TransferLinkDialog** (IL) - Manual link creation UI
- **RelationshipPanel** (IL) - Displays detected relationships

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 3-4 days
**Dependencies:** CanonicalStore, RelationshipStore, AccountStore, PostgreSQL with indexes

**Performance Targets:**
- detect_on_upload: <200ms p95
- detect_batch (100 txns): <2000ms p95
- calculate_confidence: <1ms
- Query optimization: Indexed lookups, ±30 day search window

**Next Steps:**
1. Implement core detection algorithm
2. Add database indexes
3. Build batch processing
4. Tune confidence thresholds based on real data
5. Add FX conversion specialization
6. Create UI integration hooks
