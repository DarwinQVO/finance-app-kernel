# InstanceTracker (OL Primitive)

**Vertical:** 3.3 Series Registry
**Layer:** Objective Layer (OL)
**Type:** Matching Engine + State Manager
**Status:** ✅ Specified

---

## Purpose

Auto-link transactions to series instances based on matching criteria, manage instance status lifecycle, detect variances, and provide missing payment alerts.

**Core Responsibility:** When a transaction arrives, determine if it matches an expected series instance. Track the relationship between expected occurrences and actual transactions, computing variance and managing status transitions (upcoming → matched/missing).

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY expected-vs-actual tracking:

| Domain | Expected Event | Actual Event | Matching Criteria | Status Tracking |
|--------|---------------|--------------|-------------------|-----------------|
| **Finance** | Recurring payment | Bank transaction | Account + counterparty + amount + date | matched, missing, variance |
| **Healthcare** | Scheduled appointment | Check-in record | Patient + provider + date | attended, no-show, rescheduled |
| **Legal** | Filing deadline | Court filing | Case + court + deadline | filed, late, missed |
| **Research** | Data collection | Lab result | Study + subject + date | collected, pending, missing |
| **Manufacturing** | Maintenance schedule | Service record | Equipment + vendor + date | completed, overdue, skipped |
| **Media** | Content schedule | Publication record | Channel + date + topic | published, delayed, canceled |

**Pattern:** Instance Tracking = match expected events to actual events based on multi-criteria matching, maintain status state machine, detect anomalies.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Dict
from dataclasses import dataclass
from datetime import date, timedelta
from decimal import Decimal

@dataclass
class SeriesInstance:
    """Represents a single occurrence of a series (expected + actual)."""
    instance_id: str
    series_id: str
    user_id: str
    transaction_id: Optional[str]  # None = no match yet
    expected_date: date
    actual_date: Optional[date]  # None = no match yet
    expected_amount: Decimal
    actual_amount: Optional[Decimal]  # None = no match yet
    status: str  # "matched", "matched_manual", "variance", "missing", "upcoming", "skipped"
    variance: Optional[Decimal]  # actual - expected
    link_type: Optional[str]  # "auto", "manual", "forced"
    created_at: datetime
    updated_at: datetime

@dataclass
class MatchCriteria:
    """Criteria used to match transaction to series instance."""
    account_match: bool
    counterparty_match: bool
    amount_within_tolerance: bool
    date_within_window: bool
    no_duplicate: bool

@dataclass
class MatchResult:
    """Result of auto-link attempt."""
    matched: bool
    instance: Optional[SeriesInstance]
    criteria: MatchCriteria
    reason: Optional[str]  # If not matched, why?

@dataclass
class MissingInstance:
    """Represents a missing payment/event."""
    series_id: str
    series_name: str
    expected_date: date
    expected_amount: Decimal
    days_overdue: int
    category: str

class InstanceTracker:
    """
    Tracks relationship between expected series instances and actual transactions.

    Responsibilities:
    - Auto-link transactions to series instances based on matching criteria
    - Manual linking with override support
    - Status management (upcoming → matched/missing)
    - Variance detection
    - Missing payment alerts
    - Duplicate prevention
    """

    def __init__(self, db_connection):
        self.db = db_connection
        self.date_window_days = 3  # ±3 days for matching

    def auto_link(
        self,
        transaction_id: str,
        user_id: str
    ) -> MatchResult:
        """
        Attempt to auto-link transaction to a series instance.

        Called after transaction normalization (from Vertical 1.3).

        Matching criteria (ALL must be true):
        1. Account match: transaction.account_id == series.account_id
        2. Counterparty match: transaction.counterparty_id == series.counterparty_id
        3. Amount tolerance: |transaction.amount - series.expected_amount| <= series.tolerance
        4. Date window: |transaction.date - expected_date| <= 3 days
        5. No duplicate: No existing instance with this transaction_id

        Args:
            transaction_id: Transaction to link
            user_id: Owner of transaction/series

        Returns:
            MatchResult with matched flag, instance (if matched), criteria, reason

        Example:
            # Transaction: Netflix charge $15.99 on Feb 15
            # Series: Netflix subscription $15.99 monthly on 15th
            result = tracker.auto_link("txn_12345", "user_darwin")
            if result.matched:
                print(f"Linked to {result.instance.series_id}")
            else:
                print(f"No match: {result.reason}")
        """

    def manual_link(
        self,
        user_id: str,
        series_id: str,
        transaction_id: str,
        force: bool = False
    ) -> SeriesInstance:
        """
        Manually link transaction to series.

        User explicitly specifies: "This transaction belongs to this series"

        Validations (unless force=True):
        - Amount within tolerance
        - Account matches
        - Transaction not already linked to another series

        Args:
            user_id: Owner (for authorization)
            series_id: Series to link to
            transaction_id: Transaction to link
            force: If True, bypass tolerance checks (still require account match)

        Returns:
            Created/updated SeriesInstance

        Raises:
            AmountOutOfToleranceError: If amount exceeds tolerance (unless force=True)
            AccountMismatchError: If transaction account != series account (even with force)
            TransactionAlreadyLinkedError: If transaction already linked to another series
            SeriesNotFoundError: If series doesn't exist
            TransactionNotFoundError: If transaction doesn't exist
            UnauthorizedError: If user doesn't own series or transaction

        Example:
            # User manually links rent payment that auto-link missed
            instance = tracker.manual_link(
                user_id="user_darwin",
                series_id="series_rent_1",
                transaction_id="txn_67890",
                force=False
            )
            print(f"Status: {instance.status}")  # "matched_manual"
        """

    def unlink(
        self,
        user_id: str,
        instance_id: str
    ) -> None:
        """
        Remove instance link (delete instance, keep transaction).

        Args:
            user_id: Owner (for authorization)
            instance_id: Instance to unlink

        Raises:
            InstanceNotFoundError: If instance doesn't exist
            UnauthorizedError: If user doesn't own instance

        Example:
            # User realizes this transaction wasn't actually Netflix
            tracker.unlink("user_darwin", "instance_series_netflix_1_202402")
        """

    def mark_skipped(
        self,
        user_id: str,
        series_id: str,
        expected_date: date,
        reason: Optional[str] = None
    ) -> SeriesInstance:
        """
        Mark expected instance as intentionally skipped.

        Use case: User knows payment didn't happen (canceled subscription, skipped month).

        Args:
            user_id: Owner (for authorization)
            series_id: Series
            expected_date: Date to mark skipped
            reason: Optional reason (e.g., "Canceled subscription")

        Returns:
            Updated SeriesInstance with status="skipped"

        Example:
            # User skipped gym payment this month (out of town)
            instance = tracker.mark_skipped(
                user_id="user_darwin",
                series_id="series_gym_1",
                expected_date=date(2024, 8, 1),
                reason="Vacation month"
            )
        """

    def get_missing_instances(
        self,
        user_id: str,
        as_of_date: Optional[date] = None,
        days_overdue_min: int = 0
    ) -> List[MissingInstance]:
        """
        Get all expected instances with no match (status="missing").

        Args:
            user_id: Owner
            as_of_date: Check as of this date (default: today)
            days_overdue_min: Only return instances overdue by at least N days

        Returns:
            List of MissingInstance objects

        Example:
            # Get all missing payments
            missing = tracker.get_missing_instances("user_darwin")
            for m in missing:
                print(f"{m.series_name}: expected {m.expected_date}, {m.days_overdue} days overdue")
        """

    def detect_missing(
        self,
        user_id: str,
        as_of_date: Optional[date] = None
    ) -> int:
        """
        Background job: Detect missing instances.

        For all instances with expected_date <= as_of_date and status="upcoming",
        update status to "missing".

        Args:
            user_id: Owner (optional - if None, process all users)
            as_of_date: Check as of this date (default: today)

        Returns:
            Count of instances marked missing

        Example:
            # Daily cron job
            count = tracker.detect_missing(user_id=None)  # All users
            logger.info(f"Marked {count} instances as missing")
        """

    def get_instance_for_date(
        self,
        series_id: str,
        user_id: str,
        expected_date: date
    ) -> Optional[SeriesInstance]:
        """
        Get instance for specific expected date.

        Args:
            series_id: Series
            user_id: Owner
            expected_date: Expected date

        Returns:
            SeriesInstance if exists, None otherwise
        """

    def list_instances(
        self,
        series_id: str,
        user_id: str,
        status: Optional[str] = None,
        limit: int = 100
    ) -> List[SeriesInstance]:
        """
        List instances for series.

        Args:
            series_id: Series
            user_id: Owner
            status: Filter by status (optional)
            limit: Max results

        Returns:
            List of SeriesInstance objects (sorted by expected_date DESC)
        """

    def _check_match_criteria(
        self,
        transaction: Transaction,
        series: Series,
        expected_date: date
    ) -> MatchCriteria:
        """
        Evaluate all matching criteria.

        Returns:
            MatchCriteria with boolean flags
        """

    def _calculate_variance(
        self,
        expected_amount: Decimal,
        actual_amount: Decimal
    ) -> Decimal:
        """
        Calculate variance: actual - expected.

        Examples:
          Expected: -$100, Actual: -$100 → variance = 0
          Expected: -$100, Actual: -$105 → variance = -5 (spent $5 more)
          Expected: -$100, Actual: -$95 → variance = 5 (spent $5 less)
          Expected: $1000, Actual: $1050 → variance = 50 (received $50 more)
        """

    def _determine_status(
        self,
        criteria: MatchCriteria,
        link_type: str,
        tolerance: Decimal,
        variance: Decimal
    ) -> str:
        """
        Determine instance status based on criteria and link type.

        Logic:
        - If link_type="auto" and all criteria met → "matched"
        - If link_type="manual" → "matched_manual"
        - If link_type="forced" → check variance
          - If |variance| <= tolerance → "matched_manual"
          - If |variance| > tolerance → "variance"
        - If expected_date passed and no link → "missing"
        - If expected_date future → "upcoming"

        Returns:
            Status string
        """

    def _generate_instance_id(self, series_id: str, expected_date: date) -> str:
        """
        Generate deterministic instance_id.

        Format: instance_{series_id}_{YYYYMM}_{seq}
        Example: "instance_series_netflix_1_202402_1"

        Args:
            series_id: Series ID
            expected_date: Expected date

        Returns:
            instance_id string
        """
```

---

## Matching Algorithm

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Transaction Arrives (from Normalization)                     │
│    - transaction_id: txn_12345                                  │
│    - date: 2024-02-15                                           │
│    - amount: -$15.99                                            │
│    - account_id: acc_chase_credit_1                             │
│    - counterparty_id: cpty_netflix_1                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. Query Active Series for User                                 │
│    SELECT * FROM series WHERE user_id = ? AND is_active = true  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Filter Series by Account Match                               │
│    series.account_id == transaction.account_id                  │
│    → Found: series_netflix_1                                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Filter Series by Counterparty Match                          │
│    series.counterparty_id == transaction.counterparty_id        │
│    → Found: series_netflix_1                                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. Calculate Expected Date Window                               │
│    transaction.date ± 3 days = [2024-02-12, 2024-02-18]        │
│    series next_expected_date: 2024-02-15                        │
│    → Match! (within window)                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. Check Amount Tolerance                                       │
│    |transaction.amount - series.expected_amount| <= tolerance   │
│    |-15.99 - (-15.99)| = 0 <= 2.00                             │
│    → Match! (within tolerance)                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. Check No Duplicate                                           │
│    SELECT * FROM series_instances WHERE transaction_id = ?      │
│    → None found                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 8. Create Instance Link                                         │
│    INSERT INTO series_instances (...)                           │
│    - status: "matched"                                          │
│    - link_type: "auto"                                          │
│    - variance: 0.00                                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 9. Update Series Next Expected Date                             │
│    UPDATE series SET next_expected_date = ?                     │
│    (calculated by RecurrenceEngine)                             │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```python
def auto_link(self, transaction_id: str, user_id: str) -> MatchResult:
    """
    Auto-link transaction to series instance.

    Algorithm:
    1. Load transaction
    2. Query active series for user
    3. Filter by account match
    4. Filter by counterparty match
    5. For each candidate series:
       a. Calculate expected date window
       b. Check if transaction date within window
       c. Check amount tolerance
       d. Check no duplicate
       e. If all pass, create instance link
    6. Return result
    """
    # 1. Load transaction
    transaction = self._get_transaction(transaction_id, user_id)
    if not transaction:
        return MatchResult(
            matched=False,
            instance=None,
            criteria=MatchCriteria(False, False, False, False, False),
            reason="Transaction not found"
        )

    # 2. Query active series for user with account + counterparty match
    candidate_series = self.db.execute(
        """
        SELECT * FROM series
        WHERE user_id = ?
          AND is_active = true
          AND account_id = ?
          AND counterparty_id = ?
        """,
        (user_id, transaction.account_id, transaction.counterparty_id)
    ).fetchall()

    if not candidate_series:
        return MatchResult(
            matched=False,
            instance=None,
            criteria=MatchCriteria(True, False, False, False, False),
            reason="No series found with matching account/counterparty"
        )

    # 3. Find best match
    for series in candidate_series:
        # Calculate expected date for this transaction
        expected_date = self._find_expected_date_near(
            series.series_id,
            transaction.date,
            window_days=self.date_window_days
        )

        if not expected_date:
            continue  # No expected date within window

        # Check all criteria
        criteria = self._check_match_criteria(
            transaction,
            series,
            expected_date
        )

        if all([
            criteria.account_match,
            criteria.counterparty_match,
            criteria.amount_within_tolerance,
            criteria.date_within_window,
            criteria.no_duplicate
        ]):
            # All criteria met - create link
            variance = self._calculate_variance(
                series.expected_amount,
                transaction.amount
            )

            instance = self._create_instance(
                series_id=series.series_id,
                user_id=user_id,
                transaction_id=transaction_id,
                expected_date=expected_date,
                actual_date=transaction.date,
                expected_amount=series.expected_amount,
                actual_amount=transaction.amount,
                status="matched",
                variance=variance,
                link_type="auto"
            )

            # Update series next expected date
            self._update_series_next_expected(series.series_id)

            return MatchResult(
                matched=True,
                instance=instance,
                criteria=criteria,
                reason=None
            )

    # No match found
    return MatchResult(
        matched=False,
        instance=None,
        criteria=criteria,  # Last checked criteria
        reason="No series matched all criteria"
    )
```

### Criteria Checking

```python
def _check_match_criteria(
    self,
    transaction: Transaction,
    series: Series,
    expected_date: date
) -> MatchCriteria:
    """
    Evaluate all matching criteria.

    Returns MatchCriteria with boolean flags for each criterion.
    """
    # 1. Account match
    account_match = (transaction.account_id == series.account_id)

    # 2. Counterparty match
    counterparty_match = (transaction.counterparty_id == series.counterparty_id)

    # 3. Amount within tolerance
    variance = abs(transaction.amount - series.expected_amount)
    amount_within_tolerance = (variance <= series.tolerance)

    # 4. Date within window
    date_diff = abs((transaction.date - expected_date).days)
    date_within_window = (date_diff <= self.date_window_days)

    # 5. No duplicate (transaction not already linked)
    existing = self.db.execute(
        "SELECT instance_id FROM series_instances WHERE transaction_id = ?",
        (transaction.transaction_id,)
    ).fetchone()
    no_duplicate = (existing is None)

    return MatchCriteria(
        account_match=account_match,
        counterparty_match=counterparty_match,
        amount_within_tolerance=amount_within_tolerance,
        date_within_window=date_within_window,
        no_duplicate=no_duplicate
    )
```

---

## Status State Machine

```
                                ┌──────────┐
                      CREATE    │          │
                     ┌─────────▶│ upcoming │
                     │          │          │
                     │          └──────────┘
                     │                │
                     │                │ expected_date passes
                     │                │ (no transaction)
                     │                ▼
                     │          ┌──────────┐
                     │          │          │
                     │      ┌──▶│ missing  │◀───┐
                     │      │   │          │    │
                     │      │   └──────────┘    │
                     │      │         │         │
                     │      │         │ manual  │
                     │      │         │ link    │
                     │      │         ▼         │
┌──────────────┐    │      │   ┌─────────────────┐
│              │    │      │   │                 │
│ series       │────┘      │   │ matched_manual  │
│ created      │           │   │                 │
│              │           │   └─────────────────┘
└──────────────┘           │
                          │
                          │ user marks
                          │ skipped
                          │
                          ▼
                    ┌──────────┐
                    │          │
                    │ skipped  │
                    │          │
                    └──────────┘


┌──────────┐
│          │
│ upcoming │──transaction arrives──▶┌─────────┐
│          │   (auto-link success)  │ matched │
└──────────┘                        └─────────┘
                                         │
                                         │ amount out
                                         │ of tolerance
                                         ▼
                                   ┌──────────┐
                                   │ variance │
                                   └──────────┘
```

### Status Definitions

| Status | Definition | Trigger | Actions |
|--------|------------|---------|---------|
| **upcoming** | Expected date in future, no match yet | RecurrenceEngine generates instance | Display in "Upcoming" list |
| **matched** | Auto-linked, within tolerance | InstanceTracker.auto_link() success | Mark as paid, update series |
| **matched_manual** | Manually linked by user | User clicks "Link to Series" | Mark as paid, log manual action |
| **variance** | Linked but amount outside tolerance | Force link with variance > tolerance | Alert user, display variance |
| **missing** | Expected date passed, no match | Daily job: detect_missing() | Alert user, mark as overdue |
| **skipped** | User marked as intentionally skipped | User clicks "Mark Skipped" | Hide from missing list |

### State Transitions

```python
def _determine_status(
    self,
    criteria: MatchCriteria,
    link_type: str,
    tolerance: Decimal,
    variance: Decimal
) -> str:
    """
    Determine instance status based on criteria and link type.

    Rules:
    1. link_type="auto" + all criteria met → "matched"
    2. link_type="manual" + within tolerance → "matched_manual"
    3. link_type="forced" + outside tolerance → "variance"
    4. No link + expected_date passed → "missing"
    5. No link + expected_date future → "upcoming"
    """
    if link_type == "auto":
        if all([
            criteria.account_match,
            criteria.counterparty_match,
            criteria.amount_within_tolerance,
            criteria.date_within_window,
            criteria.no_duplicate
        ]):
            return "matched"

    elif link_type == "manual":
        if abs(variance) <= tolerance:
            return "matched_manual"
        else:
            return "variance"

    elif link_type == "forced":
        # Force link always creates "variance" if outside tolerance
        if abs(variance) > tolerance:
            return "variance"
        else:
            return "matched_manual"

    # Default: if expected_date passed → missing, else upcoming
    # (Handled by detect_missing background job)
    return "upcoming"
```

---

## Variance Detection

### Variance Calculation

```python
def _calculate_variance(
    self,
    expected_amount: Decimal,
    actual_amount: Decimal
) -> Decimal:
    """
    Calculate variance: actual - expected.

    Negative variance = spent more (or received less)
    Positive variance = spent less (or received more)

    Examples:
      Expense:
        Expected: -$100, Actual: -$105 → variance = -5 (spent $5 more)
        Expected: -$100, Actual: -$95 → variance = 5 (spent $5 less)

      Income:
        Expected: $1000, Actual: $1050 → variance = 50 (received $50 more)
        Expected: $1000, Actual: $950 → variance = -50 (received $50 less)
    """
    return actual_amount - expected_amount
```

### Variance Alerts

```python
def get_variance_instances(
    self,
    user_id: str,
    min_variance: Decimal = Decimal("0.01")
) -> List[SeriesInstance]:
    """
    Get all instances with variance above threshold.

    Args:
        user_id: Owner
        min_variance: Minimum absolute variance to include

    Returns:
        List of SeriesInstance objects with status="variance" or |variance| > min_variance
    """
    instances = self.db.execute(
        """
        SELECT * FROM series_instances
        WHERE user_id = ?
          AND (status = 'variance' OR ABS(variance) > ?)
        ORDER BY expected_date DESC
        """,
        (user_id, min_variance)
    ).fetchall()

    return [self._hydrate_instance(row) for row in instances]
```

---

## Manual Link Implementation

```python
def manual_link(
    self,
    user_id: str,
    series_id: str,
    transaction_id: str,
    force: bool = False
) -> SeriesInstance:
    """
    Manually link transaction to series.

    Steps:
    1. Verify series and transaction exist and owned by user
    2. Check transaction not already linked
    3. Validate account match (always required, even with force)
    4. Validate amount tolerance (unless force=True)
    5. Find or create expected instance for transaction date
    6. Create/update instance link
    7. Update series next expected date
    """
    # 1. Verify ownership
    series = self._get_series(series_id, user_id)
    if not series:
        raise SeriesNotFoundError(f"Series '{series_id}' not found")

    transaction = self._get_transaction(transaction_id, user_id)
    if not transaction:
        raise TransactionNotFoundError(f"Transaction '{transaction_id}' not found")

    # 2. Check not already linked
    existing = self.db.execute(
        "SELECT instance_id, series_id FROM series_instances WHERE transaction_id = ?",
        (transaction_id,)
    ).fetchone()

    if existing:
        if existing['series_id'] == series_id:
            # Already linked to this series - idempotent
            return self._get_instance(existing['instance_id'])
        else:
            raise TransactionAlreadyLinkedError(
                f"Transaction already linked to series '{existing['series_id']}'"
            )

    # 3. Validate account match (ALWAYS required)
    if transaction.account_id != series.account_id:
        raise AccountMismatchError(
            f"Transaction account '{transaction.account_id}' "
            f"does not match series account '{series.account_id}'"
        )

    # 4. Calculate variance
    variance = self._calculate_variance(series.expected_amount, transaction.amount)

    # 5. Validate amount tolerance (unless force)
    if not force and abs(variance) > series.tolerance:
        raise AmountOutOfToleranceError(
            f"Transaction amount {transaction.amount} exceeds tolerance "
            f"(expected {series.expected_amount} ± {series.tolerance})",
            expected=series.expected_amount,
            actual=transaction.amount,
            tolerance=series.tolerance,
            variance=variance
        )

    # 6. Find or create expected instance near transaction date
    expected_date = self._find_or_create_expected_date(
        series,
        transaction.date
    )

    # 7. Determine status
    link_type = "forced" if force else "manual"
    status = self._determine_status(
        criteria=MatchCriteria(True, True, abs(variance) <= series.tolerance, True, True),
        link_type=link_type,
        tolerance=series.tolerance,
        variance=variance
    )

    # 8. Create instance
    instance = self._create_or_update_instance(
        series_id=series_id,
        user_id=user_id,
        transaction_id=transaction_id,
        expected_date=expected_date,
        actual_date=transaction.date,
        expected_amount=series.expected_amount,
        actual_amount=transaction.amount,
        status=status,
        variance=variance,
        link_type=link_type
    )

    # 9. Update series next expected date
    self._update_series_next_expected(series_id)

    # 10. Log manual link
    logger.info(
        "manual_link",
        user_id=user_id,
        series_id=series_id,
        transaction_id=transaction_id,
        force=force,
        variance=float(variance),
        status=status
    )

    return instance
```

---

## Missing Instance Detection

### Background Job

```python
def detect_missing(
    self,
    user_id: Optional[str] = None,
    as_of_date: Optional[date] = None
) -> int:
    """
    Background job: Detect missing instances.

    For all instances with:
    - expected_date <= as_of_date
    - status = "upcoming"
    - transaction_id IS NULL

    Update status to "missing".

    Runs: Daily cron job at midnight

    Args:
        user_id: Optional user filter (None = all users)
        as_of_date: Check as of this date (None = today)

    Returns:
        Count of instances marked missing
    """
    if as_of_date is None:
        as_of_date = date.today()

    # Build query
    query = """
        UPDATE series_instances
        SET status = 'missing', updated_at = ?
        WHERE status = 'upcoming'
          AND expected_date <= ?
          AND transaction_id IS NULL
    """
    params = [datetime.utcnow(), as_of_date]

    if user_id:
        query += " AND user_id = ?"
        params.append(user_id)

    result = self.db.execute(query, params)
    count = result.rowcount

    logger.info(
        "detect_missing",
        count=count,
        as_of_date=as_of_date.isoformat(),
        user_id=user_id
    )

    return count
```

### Get Missing Instances

```python
def get_missing_instances(
    self,
    user_id: str,
    as_of_date: Optional[date] = None,
    days_overdue_min: int = 0
) -> List[MissingInstance]:
    """
    Get all missing instances for user.

    Returns summary with days overdue calculation.

    Args:
        user_id: Owner
        as_of_date: Calculate overdue as of this date (None = today)
        days_overdue_min: Only return instances overdue >= N days

    Returns:
        List of MissingInstance objects
    """
    if as_of_date is None:
        as_of_date = date.today()

    rows = self.db.execute(
        """
        SELECT
            i.series_id,
            s.name as series_name,
            i.expected_date,
            i.expected_amount,
            s.category,
            (? - i.expected_date) as days_overdue
        FROM series_instances i
        JOIN series s ON i.series_id = s.series_id
        WHERE i.user_id = ?
          AND i.status = 'missing'
          AND (? - i.expected_date) >= ?
        ORDER BY i.expected_date DESC
        """,
        (as_of_date, user_id, as_of_date, days_overdue_min)
    ).fetchall()

    return [
        MissingInstance(
            series_id=row['series_id'],
            series_name=row['series_name'],
            expected_date=row['expected_date'],
            expected_amount=row['expected_amount'],
            days_overdue=row['days_overdue'],
            category=row['category']
        )
        for row in rows
    ]
```

---

## Usage Examples

### Example 1: Auto-Link Success

```python
tracker = InstanceTracker(db_connection)

# Transaction arrives: Netflix $15.99 on Feb 15
result = tracker.auto_link("txn_12345", "user_darwin")

if result.matched:
    print(f"Linked to series: {result.instance.series_id}")
    print(f"Status: {result.instance.status}")  # "matched"
    print(f"Variance: ${result.instance.variance}")  # 0.00
else:
    print(f"No match: {result.reason}")
```

### Example 2: Auto-Link Failure (Amount Out of Tolerance)

```python
# Transaction: Netflix $25.99 (price increased)
# Series: Netflix $15.99 ± $2.00 tolerance
result = tracker.auto_link("txn_67890", "user_darwin")

print(result.matched)  # False
print(result.reason)  # "Amount outside tolerance"
print(result.criteria.amount_within_tolerance)  # False

# User must manually link or update series expected_amount
```

### Example 3: Manual Link

```python
# User manually links rent payment
instance = tracker.manual_link(
    user_id="user_darwin",
    series_id="series_rent_1",
    transaction_id="txn_55555",
    force=False
)

print(instance.status)  # "matched_manual"
print(instance.link_type)  # "manual"
```

### Example 4: Force Link (Variance)

```python
# User force links despite amount variance
try:
    instance = tracker.manual_link(
        user_id="user_darwin",
        series_id="series_rent_1",
        transaction_id="txn_77777",
        force=False  # Will fail
    )
except AmountOutOfToleranceError as e:
    print(f"Error: {e.message}")
    print(f"Variance: ${e.variance}")

    # User confirms: "Yes, this is my rent (landlord raised price)"
    instance = tracker.manual_link(
        user_id="user_darwin",
        series_id="series_rent_1",
        transaction_id="txn_77777",
        force=True  # Override
    )

    print(instance.status)  # "variance"
    print(instance.variance)  # -100.00 (paid $100 more)
```

### Example 5: Unlink

```python
# User realizes they linked wrong transaction
tracker.unlink("user_darwin", "instance_series_netflix_1_202402")

# Instance deleted, transaction remains in system
```

### Example 6: Mark Skipped

```python
# User skipped gym this month (vacation)
instance = tracker.mark_skipped(
    user_id="user_darwin",
    series_id="series_gym_1",
    expected_date=date(2024, 8, 1),
    reason="Out of town"
)

print(instance.status)  # "skipped"
print(instance.transaction_id)  # None
```

### Example 7: Get Missing Instances

```python
# Get all missing payments
missing = tracker.get_missing_instances("user_darwin")

for m in missing:
    print(f"{m.series_name}:")
    print(f"  Expected: {m.expected_date}")
    print(f"  Amount: ${abs(m.expected_amount)}")
    print(f"  Overdue: {m.days_overdue} days")
    print(f"  Category: {m.category}")

# Output:
# Netflix Subscription:
#   Expected: 2024-11-15
#   Amount: $15.99
#   Overdue: 5 days
#   Category: software_saas
```

### Example 8: Detect Missing (Cron Job)

```python
# Daily cron job at midnight
count = tracker.detect_missing(user_id=None)  # All users
logger.info(f"Marked {count} instances as missing")

# Email alerts to users with missing instances
users_with_missing = get_users_with_missing()
for user in users_with_missing:
    send_missing_payment_alert(user)
```

---

## Multi-Domain Examples

### Healthcare: Appointment Tracking

```python
# Expected: Therapy appointment every Tuesday
# Actual: Patient check-in record

result = tracker.auto_link(
    transaction_id="checkin_12345",  # Check-in record
    user_id="patient_123"
)

if result.matched:
    print(f"Appointment attended: {result.instance.series_name}")
else:
    # No check-in → mark as no-show
    tracker.detect_missing(user_id="patient_123")
```

### Legal: Filing Deadline Tracking

```python
# Expected: Monthly court filing deadline
# Actual: Filing submission record

missing = tracker.get_missing_instances(
    user_id="lawyer_456",
    days_overdue_min=1  # Only late filings
)

for deadline in missing:
    print(f"LATE FILING: {deadline.series_name}")
    print(f"  Due: {deadline.expected_date}")
    print(f"  Days late: {deadline.days_overdue}")
    # Alert compliance team
```

### Research: Data Collection Tracking

```python
# Expected: Weekly lab sample collection
# Actual: Sample logged in system

result = tracker.auto_link(
    transaction_id="sample_789",
    user_id="researcher_789"
)

if not result.matched:
    print(f"Sample collection gap detected: {result.reason}")
    # Alert research coordinator
```

---

## Error Handling

### Custom Exceptions

```python
class InstanceTrackerError(Exception):
    """Base exception for InstanceTracker errors."""
    pass

class AmountOutOfToleranceError(InstanceTrackerError):
    def __init__(
        self,
        message: str,
        expected: Decimal,
        actual: Decimal,
        tolerance: Decimal,
        variance: Decimal
    ):
        self.message = message
        self.expected = expected
        self.actual = actual
        self.tolerance = tolerance
        self.variance = variance
        super().__init__(message)

class AccountMismatchError(InstanceTrackerError):
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class TransactionAlreadyLinkedError(InstanceTrackerError):
    def __init__(self, message: str, existing_series_id: str = None):
        self.message = message
        self.existing_series_id = existing_series_id
        super().__init__(message)

class SeriesNotFoundError(InstanceTrackerError):
    pass

class TransactionNotFoundError(InstanceTrackerError):
    pass

class InstanceNotFoundError(InstanceTrackerError):
    pass

class UnauthorizedError(InstanceTrackerError):
    pass
```

---

## Performance Characteristics

### Query Performance

| Operation | Query Type | Expected Latency (p95) | Index Used |
|-----------|------------|------------------------|------------|
| `auto_link()` | SELECT + INSERT | <100ms | idx_series (account+counterparty), idx_transaction |
| `manual_link()` | SELECT + INSERT | <50ms | PRIMARY KEY + idx_transaction |
| `unlink()` | DELETE | <20ms | PRIMARY KEY |
| `get_missing_instances()` | SELECT with JOIN | <100ms | idx_status, idx_user |
| `detect_missing()` | UPDATE | <500ms | idx_status, idx_expected_date |

### Database Indexes

```sql
-- Instance lookups
CREATE INDEX idx_instance_series ON series_instances(series_id, expected_date DESC);
CREATE INDEX idx_instance_transaction ON series_instances(transaction_id);
CREATE INDEX idx_instance_status ON series_instances(user_id, status, expected_date);
CREATE INDEX idx_instance_user_date ON series_instances(user_id, expected_date);

-- Series lookups for auto-link
CREATE INDEX idx_series_account_cpty ON series(user_id, account_id, counterparty_id, is_active);
```

### Optimization: Batch Auto-Link

```python
def auto_link_batch(
    self,
    transaction_ids: List[str],
    user_id: str
) -> Dict[str, MatchResult]:
    """
    Auto-link multiple transactions in batch.

    More efficient than individual auto_link() calls.

    Args:
        transaction_ids: List of transaction IDs
        user_id: Owner

    Returns:
        Dict mapping transaction_id → MatchResult
    """
    # Pre-load all active series for user
    series_list = self._get_active_series(user_id)

    # Pre-load all transactions
    transactions = self._get_transactions_batch(transaction_ids, user_id)

    # Process each transaction
    results = {}
    for txn in transactions:
        result = self._auto_link_internal(txn, series_list)
        results[txn.transaction_id] = result

    return results
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Auto-link
def test_auto_link_success()
def test_auto_link_account_mismatch()
def test_auto_link_counterparty_mismatch()
def test_auto_link_amount_out_of_tolerance()
def test_auto_link_date_outside_window()
def test_auto_link_duplicate_transaction()

# Test: Manual link
def test_manual_link_success()
def test_manual_link_force_variance()
def test_manual_link_account_mismatch()  # Should fail even with force
def test_manual_link_already_linked()
def test_manual_link_idempotent()

# Test: Unlink
def test_unlink_success()
def test_unlink_not_found()
def test_unlink_unauthorized()

# Test: Missing detection
def test_detect_missing_upcoming_to_missing()
def test_detect_missing_no_change_if_matched()
def test_get_missing_instances()
def test_get_missing_instances_days_overdue_filter()

# Test: Status transitions
def test_status_upcoming_to_matched()
def test_status_upcoming_to_missing()
def test_status_missing_to_matched_manual()
def test_status_upcoming_to_skipped()

# Test: Variance
def test_calculate_variance_expense()
def test_calculate_variance_income()
def test_determine_status_variance()
```

---

## Related Primitives

- **SeriesStore** (OL) - Provides series definitions for matching
- **RecurrenceEngine** (OL) - Calculates expected dates
- **TransactionStore** (OL) - Provides transaction data for matching (from 1.3)
- **SeriesManager** (IL) - UI displays instance status
- **TransactionList** (IL) - Shows series badges on transactions (from 2.1)

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 4-5 days
**Dependencies:** Database (PostgreSQL/MySQL), SeriesStore (3.3), RecurrenceEngine (3.3), TransactionStore (1.3), CounterpartyStore (3.2), AccountStore (3.1)
