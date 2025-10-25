# ReconciliationStore (OL Primitive)

**Vertical:** 3.9 Reconciliation Strategies (Multi-Source Matching)
**Layer:** Objective Layer (OL)
**Type:** Persistent Storage with Match Audit Trail
**Status:** ✅ Specified

---

## Purpose

CRUD operations for reconciliation results (matches between items from different sources). Stores match metadata, confidence scores, method (auto/manual), cardinality (one-to-one, one-to-many, many-to-one), and provides audit trail for data lineage.

**Core Responsibility:** Persist match results with full provenance tracking (who matched, when, why, how confident). Support one-to-many and many-to-one matches (split payments, consolidated records). Enable match rollback and re-reconciliation after threshold adjustments. Provide query patterns for unmatched items, low-confidence suggestions, and match history.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY system requiring multi-source data matching with confidence scoring:

| Domain | Match Scenario | Source 1 | Source 2 | Cardinality Examples |
|--------|----------------|----------|----------|----------------------|
| **Finance** | Bank reconciliation | Bank statements | Accounting ledger | 1:1 (single payment), 1:N (split deposit), N:1 (batch payment) |
| **Healthcare** | Claim matching | Provider claims | Insurance payments | 1:1 (single claim), 1:N (partial payments), N:1 (bundled claim) |
| **Legal** | Document linkage | Court filings | Case management | 1:1 (single filing), 1:N (multi-document case), N:1 (consolidated filing) |
| **Research** | Citation reconciliation | Paper citations | Publication database | 1:1 (exact match), 1:N (citing multiple works), N:1 (duplicate citations) |
| **E-commerce** | Order reconciliation | Purchase orders | Shipment tracking | 1:1 (single item), 1:N (split shipment), N:1 (combined order) |

**Pattern:** Store match results with confidence scoring, support multiple cardinalities, enable audit trail for data quality, allow unmatch/rematch operations.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Dict, Literal
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal

@dataclass
class ReconciliationMatch:
    match_id: str
    user_id: str

    # Source items
    source_1_items: List[str]  # List of item IDs from source 1
    source_2_items: List[str]  # List of item IDs from source 2

    # Match metadata
    cardinality: Literal["one_to_one", "one_to_many", "many_to_one", "many_to_many"]
    confidence: Decimal  # 0.0 to 1.0
    match_method: Literal["auto", "manual", "assisted"]  # auto=system, manual=user, assisted=user+system

    # Match criteria
    feature_scores: Dict[str, Decimal]  # Individual feature scores
    # Example: {"amount_score": 1.0, "date_score": 0.95, "counterparty_score": 0.85}

    # Audit trail
    matched_at: datetime
    matched_by: Optional[str]  # user_id for manual matches, None for auto
    match_details: Optional[Dict]  # Additional match context (thresholds used, blocking key, etc.)

    # Status
    is_active: bool  # False = unmatched/deleted
    unmatched_at: Optional[datetime]
    unmatched_by: Optional[str]  # user_id who unmatched
    unmatch_reason: Optional[str]  # Reason for unmatch

@dataclass
class MatchCandidate:
    """Suggested match not yet confirmed."""
    candidate_id: str
    source_1_item: str
    source_2_item: str
    similarity_score: Decimal
    feature_scores: Dict[str, Decimal]
    decision: Literal["auto_link", "suggest", "manual", "no_match"]
    # auto_link >= 0.95, suggest 0.70-0.94, manual 0.50-0.69, no_match < 0.50
    match_details: Dict

class ReconciliationStore:
    """
    Manages CRUD operations for reconciliation matches.

    Enforces:
    - No item can be in multiple active matches (unless many-to-many)
    - Confidence must be in range [0.0, 1.0]
    - Manual matches must have matched_by user_id
    - Auto matches must have confidence >= threshold
    - Feature scores must be present and valid
    """

    def __init__(self, db_connection):
        self.db = db_connection

    def create_match(
        self,
        user_id: str,
        source_1_items: List[str],
        source_2_items: List[str],
        cardinality: str,
        confidence: Decimal,
        match_method: str,
        feature_scores: Dict[str, Decimal],
        matched_by: Optional[str] = None,
        match_details: Optional[Dict] = None
    ) -> ReconciliationMatch:
        """
        Create new reconciliation match.

        Validates:
        - Items exist and belong to user
        - Items not already matched (unless many-to-many)
        - Confidence in valid range
        - Feature scores present and valid
        - Manual matches have matched_by user

        Args:
            user_id: Owner of both sources
            source_1_items: List of item IDs from source 1
            source_2_items: List of item IDs from source 2
            cardinality: Match type (one_to_one, one_to_many, many_to_one, many_to_many)
            confidence: Overall match confidence (0.0 to 1.0)
            match_method: How match was created (auto, manual, assisted)
            feature_scores: Individual feature confidence scores
            matched_by: User ID for manual matches
            match_details: Optional context (thresholds, blocking key, etc.)

        Returns:
            Created ReconciliationMatch

        Raises:
            ValidationError: If items invalid or already matched
            InvalidConfidenceError: If confidence out of range
            InvalidMethodError: If manual match missing matched_by

        Example:
            # One-to-one bank transaction match
            match = store.create_match(
                user_id="user_darwin",
                source_1_items=["bank_txn_001"],
                source_2_items=["ledger_txn_001"],
                cardinality="one_to_one",
                confidence=Decimal("0.95"),
                match_method="auto",
                feature_scores={
                    "amount_score": Decimal("1.0"),
                    "date_score": Decimal("0.95"),
                    "description_score": Decimal("0.90")
                },
                match_details={
                    "amount_diff": Decimal("0.0"),
                    "date_diff_days": 0,
                    "thresholds": {"auto_link": 0.95, "suggest": 0.70}
                }
            )
        """

    def get(self, match_id: str, user_id: str) -> Optional[ReconciliationMatch]:
        """
        Get match by ID.

        Args:
            match_id: Match ID to fetch
            user_id: Owner (for authorization)

        Returns:
            ReconciliationMatch if found, None otherwise

        Raises:
            UnauthorizedError: If match exists but user doesn't own it
        """

    def find_matches_for_item(
        self,
        item_id: str,
        source: Literal["source_1", "source_2"],
        user_id: str,
        is_active: bool = True
    ) -> List[ReconciliationMatch]:
        """
        Find all matches containing specific item.

        Args:
            item_id: Item ID to search for
            source: Which source the item belongs to
            user_id: Owner
            is_active: Filter by active status (default: True)

        Returns:
            List of matches containing this item

        Example:
            # Find all matches for bank transaction
            matches = store.find_matches_for_item(
                item_id="bank_txn_001",
                source="source_1",
                user_id="user_darwin"
            )
        """

    def get_unmatched_items(
        self,
        user_id: str,
        source: Literal["source_1", "source_2"],
        limit: Optional[int] = None
    ) -> List[str]:
        """
        Get items not in any active match.

        This is the set of items that need reconciliation attention.

        Args:
            user_id: Owner
            source: Which source to check
            limit: Optional limit on results

        Returns:
            List of unmatched item IDs

        Example:
            # Get unmatched bank transactions
            unmatched = store.get_unmatched_items(
                user_id="user_darwin",
                source="source_1",
                limit=100
            )
        """

    def get_low_confidence_matches(
        self,
        user_id: str,
        threshold: Decimal,
        limit: Optional[int] = None
    ) -> List[ReconciliationMatch]:
        """
        Get matches below confidence threshold.

        Useful for identifying matches that may need manual review.

        Args:
            user_id: Owner
            threshold: Confidence threshold (e.g., 0.80)
            limit: Optional limit on results

        Returns:
            List of matches with confidence < threshold

        Example:
            # Get matches with confidence < 80%
            low_conf = store.get_low_confidence_matches(
                user_id="user_darwin",
                threshold=Decimal("0.80")
            )
        """

    def update_match(
        self,
        match_id: str,
        user_id: str,
        updates: dict
    ) -> ReconciliationMatch:
        """
        Update match details.

        Allowed updates:
        - confidence (if manually adjusting)
        - match_details (adding notes, context)

        Cannot update:
        - source_1_items, source_2_items (immutable after creation)
        - cardinality (would break match semantics)
        - match_method (historical record)

        Args:
            match_id: Match to update
            user_id: Owner (for authorization)
            updates: Dictionary of fields to update

        Returns:
            Updated ReconciliationMatch

        Raises:
            MatchNotFoundError: If match doesn't exist
            UnauthorizedError: If user doesn't own match
            ImmutableFieldError: If trying to update immutable field

        Example:
            # Update match confidence after manual review
            match = store.update_match(
                match_id="match_001",
                user_id="user_darwin",
                updates={
                    "confidence": Decimal("0.85"),
                    "match_details": {
                        "manual_review": True,
                        "reviewer_note": "Confirmed via email receipt"
                    }
                }
            )
        """

    def unmatch(
        self,
        match_id: str,
        user_id: str,
        unmatched_by: str,
        reason: Optional[str] = None
    ) -> ReconciliationMatch:
        """
        Unmatch items (soft delete).

        Sets is_active = False, preserves match record for audit trail.
        Items become available for new matches.

        Args:
            match_id: Match to unmatch
            user_id: Owner (for authorization)
            unmatched_by: User ID performing unmatch
            reason: Optional reason for unmatch

        Returns:
            Unmatched ReconciliationMatch (is_active = False)

        Raises:
            MatchNotFoundError: If match doesn't exist
            UnauthorizedError: If user doesn't own match
            AlreadyUnmatchedError: If match already unmatched

        Example:
            # Unmatch incorrect match
            match = store.unmatch(
                match_id="match_001",
                user_id="user_darwin",
                unmatched_by="user_darwin",
                reason="Incorrect match - different transaction"
            )
        """

    def get_audit_trail(
        self,
        item_id: str,
        source: Literal["source_1", "source_2"],
        user_id: str
    ) -> List[ReconciliationMatch]:
        """
        Get complete match history for item (active + unmatched).

        Returns all matches (is_active true and false) to show full history.

        Args:
            item_id: Item to get history for
            source: Which source the item belongs to
            user_id: Owner

        Returns:
            List of all matches (active and inactive) sorted by matched_at DESC

        Example:
            # Get full match history for bank transaction
            history = store.get_audit_trail(
                item_id="bank_txn_001",
                source="source_1",
                user_id="user_darwin"
            )
            # Returns: [current_match, previous_unmatch_1, previous_unmatch_2, ...]
        """

    def get_matches_by_date_range(
        self,
        user_id: str,
        start_date: datetime,
        end_date: datetime,
        is_active: bool = True
    ) -> List[ReconciliationMatch]:
        """
        Get matches created within date range.

        Args:
            user_id: Owner
            start_date: Start of range (inclusive)
            end_date: End of range (inclusive)
            is_active: Filter by active status

        Returns:
            List of matches created in date range
        """

    def get_matches_by_method(
        self,
        user_id: str,
        match_method: str,
        is_active: bool = True
    ) -> List[ReconciliationMatch]:
        """
        Get matches by creation method.

        Useful for analyzing auto vs manual match rates.

        Args:
            user_id: Owner
            match_method: Method filter (auto, manual, assisted)
            is_active: Filter by active status

        Returns:
            List of matches with specified method
        """

    def _generate_match_id(self) -> str:
        """Generate unique match ID (format: match_<uuid>)."""

    def _validate_items_exist(self, item_ids: List[str], user_id: str) -> None:
        """Validate all items exist and belong to user."""

    def _check_items_already_matched(
        self,
        item_ids: List[str],
        source: str,
        user_id: str
    ) -> None:
        """Check if any items already in active match (raises error if yes)."""

    def _validate_confidence(self, confidence: Decimal) -> None:
        """Validate confidence in range [0.0, 1.0]."""

    def _validate_feature_scores(self, feature_scores: Dict[str, Decimal]) -> None:
        """Validate all feature scores present and in valid range."""
```

---

## Implementation Details

### Database Schema

```sql
CREATE TABLE reconciliation_matches (
    match_id VARCHAR(100) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,

    -- Source items (stored as JSON arrays)
    source_1_items JSON NOT NULL,  -- ["item_id_1", "item_id_2", ...]
    source_2_items JSON NOT NULL,  -- ["item_id_1", "item_id_2", ...]

    -- Match metadata
    cardinality VARCHAR(20) NOT NULL CHECK (cardinality IN ('one_to_one', 'one_to_many', 'many_to_one', 'many_to_many')),
    confidence DECIMAL(3,2) NOT NULL CHECK (confidence >= 0 AND confidence <= 1),
    match_method VARCHAR(20) NOT NULL CHECK (match_method IN ('auto', 'manual', 'assisted')),

    -- Match criteria
    feature_scores JSON NOT NULL,  -- {"amount_score": 1.0, "date_score": 0.95, ...}
    match_details JSON,  -- Optional context

    -- Audit trail
    matched_at TIMESTAMP NOT NULL,
    matched_by VARCHAR(50),  -- NULL for auto matches, user_id for manual

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    unmatched_at TIMESTAMP,
    unmatched_by VARCHAR(50),
    unmatch_reason TEXT,

    -- Indexes
    INDEX idx_user (user_id, is_active),
    INDEX idx_matched_at (matched_at),
    INDEX idx_confidence (confidence),
    INDEX idx_method (match_method, is_active),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Many-to-many junction table for fast item lookups
CREATE TABLE reconciliation_match_items (
    match_id VARCHAR(100) NOT NULL,
    item_id VARCHAR(100) NOT NULL,
    source VARCHAR(10) NOT NULL CHECK (source IN ('source_1', 'source_2')),

    PRIMARY KEY (match_id, item_id, source),
    INDEX idx_item (item_id, source),
    FOREIGN KEY (match_id) REFERENCES reconciliation_matches(match_id) ON DELETE CASCADE
);
```

### Match Creation with Item Junction

```python
def create_match(
    self,
    user_id: str,
    source_1_items: List[str],
    source_2_items: List[str],
    cardinality: str,
    confidence: Decimal,
    match_method: str,
    feature_scores: Dict[str, Decimal],
    matched_by: Optional[str] = None,
    match_details: Optional[Dict] = None
) -> ReconciliationMatch:
    # Validations
    self._validate_items_exist(source_1_items + source_2_items, user_id)
    self._check_items_already_matched(source_1_items, "source_1", user_id)
    self._check_items_already_matched(source_2_items, "source_2", user_id)
    self._validate_confidence(confidence)
    self._validate_feature_scores(feature_scores)

    if match_method == "manual" and not matched_by:
        raise InvalidMethodError("Manual matches require matched_by user_id")

    # Generate ID
    match_id = self._generate_match_id()
    matched_at = datetime.utcnow()

    # Insert match
    self.db.execute(
        """
        INSERT INTO reconciliation_matches (
            match_id, user_id, source_1_items, source_2_items,
            cardinality, confidence, match_method, feature_scores,
            match_details, matched_at, matched_by, is_active
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """,
        (
            match_id, user_id,
            json.dumps(source_1_items), json.dumps(source_2_items),
            cardinality, confidence, match_method,
            json.dumps(feature_scores), json.dumps(match_details) if match_details else None,
            matched_at, matched_by, True
        )
    )

    # Insert junction table entries
    for item_id in source_1_items:
        self.db.execute(
            "INSERT INTO reconciliation_match_items (match_id, item_id, source) VALUES (?, ?, ?)",
            (match_id, item_id, "source_1")
        )

    for item_id in source_2_items:
        self.db.execute(
            "INSERT INTO reconciliation_match_items (match_id, item_id, source) VALUES (?, ?, ?)",
            (match_id, item_id, "source_2")
        )

    return ReconciliationMatch(
        match_id=match_id,
        user_id=user_id,
        source_1_items=source_1_items,
        source_2_items=source_2_items,
        cardinality=cardinality,
        confidence=confidence,
        match_method=match_method,
        feature_scores=feature_scores,
        matched_at=matched_at,
        matched_by=matched_by,
        match_details=match_details,
        is_active=True,
        unmatched_at=None,
        unmatched_by=None,
        unmatch_reason=None
    )
```

### Find Matches for Item (Fast Lookup via Junction)

```python
def find_matches_for_item(
    self,
    item_id: str,
    source: Literal["source_1", "source_2"],
    user_id: str,
    is_active: bool = True
) -> List[ReconciliationMatch]:
    """Use junction table for fast lookup."""
    query = """
        SELECT m.* FROM reconciliation_matches m
        INNER JOIN reconciliation_match_items i ON m.match_id = i.match_id
        WHERE i.item_id = ? AND i.source = ? AND m.user_id = ? AND m.is_active = ?
        ORDER BY m.matched_at DESC
    """

    rows = self.db.execute(query, (item_id, source, user_id, is_active)).fetchall()
    return [self._row_to_match(row) for row in rows]
```

### Get Unmatched Items

```python
def get_unmatched_items(
    self,
    user_id: str,
    source: Literal["source_1", "source_2"],
    limit: Optional[int] = None
) -> List[str]:
    """
    Get items not in any active match.

    Strategy: Get all items from source, exclude those in reconciliation_match_items
    where match is active.
    """
    # This assumes items are stored in a source-specific table (e.g., bank_transactions, ledger_entries)
    # The exact table depends on domain

    query = """
        SELECT item_id FROM source_items
        WHERE user_id = ? AND source = ?
        AND item_id NOT IN (
            SELECT DISTINCT i.item_id
            FROM reconciliation_match_items i
            INNER JOIN reconciliation_matches m ON i.match_id = m.match_id
            WHERE m.user_id = ? AND m.is_active = true AND i.source = ?
        )
    """

    params = [user_id, source, user_id, source]

    if limit:
        query += " LIMIT ?"
        params.append(limit)

    rows = self.db.execute(query, params).fetchall()
    return [row['item_id'] for row in rows]
```

---

## Usage Examples

### Example 1: Create One-to-One Match (Bank Reconciliation)

```python
store = ReconciliationStore(db_connection)

# Bank transaction: -$1,234.56 on 2024-10-15
# Ledger entry: -$1,234.56 on 2024-10-15
# Perfect match detected by ReconciliationEngine

match = store.create_match(
    user_id="user_darwin",
    source_1_items=["bank_txn_001"],
    source_2_items=["ledger_entry_001"],
    cardinality="one_to_one",
    confidence=Decimal("1.0"),
    match_method="auto",
    feature_scores={
        "amount_score": Decimal("1.0"),  # Exact match
        "date_score": Decimal("1.0"),  # Same date
        "description_score": Decimal("0.95")  # Very similar
    },
    match_details={
        "amount_diff": Decimal("0.0"),
        "date_diff_days": 0,
        "blocking_key": "2024-10-15_1234.56"
    }
)

print(match.match_id)  # "match_abc123"
print(match.confidence)  # 1.0
print(match.is_active)  # True
```

### Example 2: Create One-to-Many Match (Split Payment)

```python
# Single bank deposit: +$5,000 on 2024-10-20
# Two ledger entries: +$3,000 and +$2,000 on 2024-10-20
# Split deposit matched by ReconciliationEngine

match = store.create_match(
    user_id="user_darwin",
    source_1_items=["bank_txn_002"],  # Single deposit
    source_2_items=["ledger_entry_002", "ledger_entry_003"],  # Two entries
    cardinality="one_to_many",
    confidence=Decimal("0.92"),
    match_method="auto",
    feature_scores={
        "amount_score": Decimal("1.0"),  # $5,000 = $3,000 + $2,000
        "date_score": Decimal("1.0"),
        "description_score": Decimal("0.75")  # Partial match
    },
    match_details={
        "amount_diff": Decimal("0.0"),
        "split_type": "one_to_many",
        "split_amounts": [3000.0, 2000.0]
    }
)

print(match.cardinality)  # "one_to_many"
print(len(match.source_2_items))  # 2
```

### Example 3: Create Manual Match (Different Amounts)

```python
# Bank: -$47.32 (expense)
# Ledger: -$50.00 (rounded reimbursement)
# User manually links despite amount difference

match = store.create_match(
    user_id="user_darwin",
    source_1_items=["bank_txn_003"],
    source_2_items=["ledger_entry_004"],
    cardinality="one_to_one",
    confidence=Decimal("0.75"),  # Lower confidence due to amount diff
    match_method="manual",
    feature_scores={
        "amount_score": Decimal("0.85"),  # Close but not exact
        "date_score": Decimal("0.90"),  # 1 day difference
        "description_score": Decimal("0.50")  # Different descriptions
    },
    matched_by="user_darwin",
    match_details={
        "amount_diff": Decimal("2.68"),
        "manual_reason": "Employer rounds reimbursements to nearest $5",
        "override": True
    }
)

print(match.match_method)  # "manual"
print(match.matched_by)  # "user_darwin"
```

### Example 4: Find Matches for Item

```python
# Get all matches for bank transaction
matches = store.find_matches_for_item(
    item_id="bank_txn_001",
    source="source_1",
    user_id="user_darwin"
)

for match in matches:
    print(f"{match.match_id}: {match.cardinality} - confidence {match.confidence}")

# Output:
# match_abc123: one_to_one - confidence 1.0
```

### Example 5: Get Unmatched Items

```python
# Get bank transactions not matched to ledger
unmatched = store.get_unmatched_items(
    user_id="user_darwin",
    source="source_1",
    limit=50
)

print(f"Found {len(unmatched)} unmatched bank transactions")
# Output: Found 12 unmatched bank transactions

for item_id in unmatched:
    print(f"  - {item_id}")
```

### Example 6: Get Low Confidence Matches

```python
# Get matches with confidence < 80% for manual review
low_conf = store.get_low_confidence_matches(
    user_id="user_darwin",
    threshold=Decimal("0.80")
)

print(f"Found {len(low_conf)} low-confidence matches needing review")

for match in low_conf:
    print(f"{match.match_id}: {match.confidence} - {match.match_method}")
    print(f"  S1: {match.source_1_items}")
    print(f"  S2: {match.source_2_items}")
```

### Example 7: Unmatch Incorrect Match

```python
# User realizes match was incorrect
match = store.unmatch(
    match_id="match_abc123",
    user_id="user_darwin",
    unmatched_by="user_darwin",
    reason="Incorrect match - different transaction"
)

print(match.is_active)  # False
print(match.unmatched_at)  # 2024-10-25T16:30:00Z
print(match.unmatch_reason)  # "Incorrect match - different transaction"
```

### Example 8: Get Audit Trail

```python
# Get complete match history for item
history = store.get_audit_trail(
    item_id="bank_txn_001",
    source="source_1",
    user_id="user_darwin"
)

print(f"Match history for bank_txn_001:")
for match in history:
    status = "ACTIVE" if match.is_active else "UNMATCHED"
    print(f"  [{status}] {match.match_id} at {match.matched_at}")
    if not match.is_active:
        print(f"    Unmatched: {match.unmatch_reason}")

# Output:
# Match history for bank_txn_001:
#   [ACTIVE] match_def456 at 2024-10-25T16:45:00Z
#   [UNMATCHED] match_abc123 at 2024-10-15T14:15:00Z
#     Unmatched: Incorrect match - different transaction
```

---

## Multi-Domain Examples

### Healthcare: Claim-Payment Matching

```python
# Match insurance claim to payment
match = store.create_match(
    user_id="provider_123",
    source_1_items=["claim_001"],  # Provider claim: $350 for checkup
    source_2_items=["payment_001"],  # Insurance payment: $280 (80% coverage)
    cardinality="one_to_one",
    confidence=Decimal("0.95"),
    match_method="auto",
    feature_scores={
        "amount_score": Decimal("0.80"),  # $280 vs $350 (80% match)
        "date_score": Decimal("0.90"),  # 5 days difference
        "patient_score": Decimal("1.0"),  # Same patient
        "procedure_score": Decimal("1.0")  # Same CPT code
    },
    match_details={
        "claim_amount": 350.0,
        "payment_amount": 280.0,
        "coverage_rate": 0.80,
        "claim_id": "CLM12345",
        "patient_responsibility": 70.0
    }
)
```

### Legal: Document Version Linking

```python
# Match original filing to amended version
match = store.create_match(
    user_id="attorney_456",
    source_1_items=["filing_001"],  # Original complaint
    source_2_items=["filing_002"],  # Amended complaint
    cardinality="one_to_one",
    confidence=Decimal("1.0"),
    match_method="manual",
    feature_scores={
        "case_number_score": Decimal("1.0"),
        "filing_type_score": Decimal("1.0"),
        "party_score": Decimal("1.0")
    },
    matched_by="attorney_456",
    match_details={
        "case_number": "2024-CV-001",
        "relationship": "amendment",
        "amendment_date": "2024-10-20"
    }
)
```

### Research: Citation Deduplication

```python
# Match duplicate citations to same paper
match = store.create_match(
    user_id="researcher_789",
    source_1_items=["citation_001", "citation_002"],  # Two citations
    source_2_items=["paper_001"],  # Single paper in database
    cardinality="many_to_one",
    confidence=Decimal("0.98"),
    match_method="auto",
    feature_scores={
        "doi_score": Decimal("1.0"),  # Same DOI
        "title_score": Decimal("0.95"),  # Very similar titles
        "author_score": Decimal("1.0")  # Same first author
    },
    match_details={
        "doi": "10.1234/journal.2024.001",
        "deduplication": True
    }
)
```

### E-commerce: Order-Shipment Reconciliation

```python
# Match purchase order to split shipments
match = store.create_match(
    user_id="merchant_012",
    source_1_items=["order_001"],  # Single order: 10 items
    source_2_items=["shipment_001", "shipment_002"],  # Two shipments: 6 + 4 items
    cardinality="one_to_many",
    confidence=Decimal("0.95"),
    match_method="auto",
    feature_scores={
        "item_count_score": Decimal("1.0"),  # 10 = 6 + 4
        "order_number_score": Decimal("1.0"),
        "date_score": Decimal("0.90")
    },
    match_details={
        "order_number": "ORD-2024-001",
        "split_shipment": True,
        "shipment_ids": ["SHIP-001", "SHIP-002"]
    }
)
```

---

## Error Handling

### Custom Exceptions

```python
class ReconciliationStoreError(Exception):
    """Base exception for ReconciliationStore errors."""
    pass

class ValidationError(ReconciliationStoreError):
    pass

class ItemNotFoundError(ReconciliationStoreError):
    def __init__(self, message: str, item_id: str):
        self.message = message
        self.item_id = item_id
        super().__init__(message)

class ItemAlreadyMatchedError(ReconciliationStoreError):
    def __init__(self, message: str, item_id: str, existing_match_id: str):
        self.message = message
        self.item_id = item_id
        self.existing_match_id = existing_match_id
        super().__init__(message)

class InvalidConfidenceError(ReconciliationStoreError):
    def __init__(self, message: str, confidence: Decimal):
        self.message = message
        self.confidence = confidence
        super().__init__(message)

class InvalidMethodError(ReconciliationStoreError):
    pass

class MatchNotFoundError(ReconciliationStoreError):
    pass

class UnauthorizedError(ReconciliationStoreError):
    pass

class AlreadyUnmatchedError(ReconciliationStoreError):
    pass

class ImmutableFieldError(ReconciliationStoreError):
    def __init__(self, message: str, fields: List[str]):
        self.message = message
        self.fields = fields
        super().__init__(message)
```

---

## Performance Characteristics

### Query Performance

| Operation | Query Type | Expected Latency (p95) | Index Used |
|-----------|------------|------------------------|------------|
| `create_match()` | INSERT (2 tables) | <50ms | PRIMARY KEY + junction |
| `get()` | SELECT by PK | <10ms | PRIMARY KEY |
| `find_matches_for_item()` | JOIN | <20ms | idx_item (junction table) |
| `get_unmatched_items()` | NOT IN subquery | <100ms | idx_user, idx_item |
| `get_low_confidence_matches()` | SELECT with filter | <30ms | idx_confidence |
| `unmatch()` | UPDATE | <20ms | PRIMARY KEY |
| `get_audit_trail()` | JOIN + ORDER BY | <30ms | idx_item, idx_matched_at |

### Caching Strategy

```python
# Cache unmatched items count per user (changes infrequently)
cache_key = f"reconciliation:unmatched_count:{user_id}:{source}"
ttl = 300  # 5 minutes

def get_unmatched_count(user_id: str, source: str) -> int:
    cached = redis.get(cache_key)
    if cached:
        return int(cached)

    count = len(self.get_unmatched_items(user_id, source))
    redis.setex(cache_key, ttl, count)
    return count

# Invalidate on create/unmatch
def _invalidate_unmatched_cache(user_id: str):
    redis.delete(f"reconciliation:unmatched_count:{user_id}:source_1")
    redis.delete(f"reconciliation:unmatched_count:{user_id}:source_2")
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Create match
def test_create_match_one_to_one_success()
def test_create_match_one_to_many_success()
def test_create_match_many_to_one_success()
def test_create_match_item_already_matched_error()
def test_create_match_invalid_confidence_error()
def test_create_match_manual_missing_matched_by_error()

# Test: Get match
def test_get_match_success()
def test_get_match_not_found()
def test_get_match_unauthorized()

# Test: Find matches for item
def test_find_matches_for_item_single()
def test_find_matches_for_item_multiple()
def test_find_matches_for_item_none()

# Test: Get unmatched items
def test_get_unmatched_items_empty()
def test_get_unmatched_items_multiple()
def test_get_unmatched_items_with_limit()

# Test: Get low confidence matches
def test_get_low_confidence_matches()
def test_get_low_confidence_matches_none()

# Test: Update match
def test_update_match_confidence()
def test_update_match_immutable_field_error()

# Test: Unmatch
def test_unmatch_success()
def test_unmatch_already_unmatched_error()

# Test: Audit trail
def test_get_audit_trail_single_match()
def test_get_audit_trail_with_unmatches()
```

---

## Related Primitives

- **ReconciliationEngine** (OL) - Generates match candidates using this store
- **MatchScorer** (OL) - Calculates confidence scores for matches
- **ThresholdManager** (OL) - Manages confidence thresholds for auto-linking
- **ReconciliationDashboard** (IL) - UI for reviewing matches from this store
- **MatchReviewDialog** (IL) - UI for accepting/rejecting match candidates

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 4-5 days
**Dependencies:** Database (PostgreSQL/MySQL with JSON support), Validation library
