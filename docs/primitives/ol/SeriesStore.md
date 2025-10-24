# SeriesStore (OL Primitive)

**Vertical:** 3.3 Series Registry
**Layer:** Objective Layer (OL)
**Type:** Closed Registry Store
**Status:** ✅ Specified

---

## Purpose

CRUD operations for recurring payment series with validation, uniqueness enforcement, immutable field protection, and soft delete pattern.

**Core Responsibility:** Persist and manage user-defined recurring payment templates (subscriptions, bills, salaries) as a closed registry (manually curated list). Each series represents an expected pattern (e.g., "Netflix $15.99 monthly") that the system tracks over time.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY closed registry of recurring templates:

| Domain | Entity Name | Examples | Attributes |
|--------|-------------|----------|------------|
| **Finance** | Recurring Payment | Netflix subscription, rent, salary, insurance premium | expected_amount, frequency, tolerance |
| **Healthcare** | Recurring Treatment | Monthly therapy sessions, prescription refills, insurance premiums | expected_amount, frequency, provider |
| **Legal** | Recurring Obligation | Monthly retainer, lease payments, filing deadlines | expected_amount, frequency, jurisdiction |
| **Research** | Recurring Funding | Quarterly grant disbursements, annual subscriptions, recurring donations | expected_amount, frequency, funding_source |
| **Manufacturing** | Recurring Maintenance | Equipment service schedules, recurring supplier orders | expected_cost, frequency, vendor |
| **Media** | Recurring Revenue | Monthly licensing fees, hosting fees, API subscriptions | expected_amount, frequency, platform |

**Pattern:** Closed Registry = user manually creates each recurring template, enforces unique names, tracks expected vs actual occurrences, allows archive (soft delete).

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List
from dataclasses import dataclass
from datetime import datetime, date
from decimal import Decimal

@dataclass
class FrequencyPattern:
    """Represents a recurrence pattern."""
    type: str  # "daily", "weekly", "monthly", "yearly", "custom"
    interval: Optional[int] = None  # For daily, weekly, monthly (every N periods)
    day_of_week: Optional[int] = None  # For weekly (0=Monday, 6=Sunday)
    day_of_month: Optional[int] = None  # For monthly (1-31)
    month: Optional[int] = None  # For yearly (1-12)
    day: Optional[int] = None  # For yearly (1-31)
    dates: Optional[List[str]] = None  # For custom (explicit date list)

@dataclass
class Series:
    series_id: str
    user_id: str
    name: str
    account_id: str  # IMMUTABLE - references Account from 3.1
    counterparty_id: str  # IMMUTABLE - references Counterparty from 3.2
    expected_amount: Decimal
    tolerance: Decimal
    frequency: FrequencyPattern
    start_date: date
    end_date: Optional[date]  # None = ongoing, date = archived
    category: Optional[str]  # software_saas, utilities, salary, etc.
    is_active: bool
    next_expected_date: Optional[date]  # Calculated by RecurrenceEngine
    created_at: datetime
    updated_at: datetime

@dataclass
class ArchiveResult:
    series: Series
    instance_count: int  # How many instances exist for this series
    message: str

class SeriesStore:
    """
    Manages CRUD operations for recurring payment series.

    Enforces:
    - Unique names per user (case-insensitive)
    - Immutable account_id and counterparty_id (cannot change after creation)
    - Soft delete (archive sets is_active = false, end_date = today)
    - Valid frequency patterns (validated against schema)
    """

    def __init__(self, db_connection):
        self.db = db_connection

    def create(
        self,
        user_id: str,
        name: str,
        account_id: str,
        counterparty_id: str,
        expected_amount: Decimal,
        tolerance: Decimal,
        frequency: FrequencyPattern,
        start_date: date,
        category: Optional[str] = None
    ) -> Series:
        """
        Create new recurring payment series.

        Args:
            user_id: Owner of the series
            name: Display name (e.g., "Netflix Subscription")
            account_id: Account from which payment occurs (from 3.1 Account Registry)
            counterparty_id: Recipient of payment (from 3.2 Counterparty Registry)
            expected_amount: Expected payment amount (negative for expenses, positive for income)
            tolerance: Acceptable variance (e.g., 2.00 = ±$2)
            frequency: Recurrence pattern (daily, weekly, monthly, yearly, custom)
            start_date: Date series begins (cannot be future)
            category: Optional category (software_saas, utilities, salary, etc.)

        Returns:
            Created Series object

        Raises:
            DuplicateSeriesNameError: If series with same name exists (case-insensitive)
            InvalidAccountError: If account_id doesn't exist or not owned by user
            InvalidCounterpartyError: If counterparty_id doesn't exist or not owned by user
            InvalidFrequencyError: If frequency pattern is malformed
            InvalidDateError: If start_date is in the future
            ValidationError: If name too long, empty, or contains invalid characters

        Example:
            series = store.create(
                user_id="user_darwin",
                name="Netflix Subscription",
                account_id="acc_chase_credit_1",
                counterparty_id="cpty_netflix_1",
                expected_amount=Decimal("-15.99"),
                tolerance=Decimal("2.00"),
                frequency=FrequencyPattern(type="monthly", day_of_month=15, interval=1),
                start_date=date(2024, 1, 15),
                category="software_saas"
            )
        """

    def get(self, series_id: str, user_id: str) -> Optional[Series]:
        """
        Get series by ID.

        Args:
            series_id: Series ID to fetch
            user_id: Owner of the series (for authorization)

        Returns:
            Series if found and user owns it, None otherwise

        Raises:
            UnauthorizedError: If series exists but user doesn't own it
        """

    def list(
        self,
        user_id: str,
        is_active: Optional[bool] = True,
        account_id: Optional[str] = None,
        counterparty_id: Optional[str] = None,
        category: Optional[str] = None
    ) -> List[Series]:
        """
        List series for user.

        Args:
            user_id: Owner of the series
            is_active: Filter by active status (True = active only, False = archived only, None = all)
            account_id: Filter by account (optional)
            counterparty_id: Filter by counterparty (optional)
            category: Filter by category (optional)

        Returns:
            List of Series objects (sorted by name ASC)

        Example:
            # Get all active series for user
            active_series = store.list(user_id="user_darwin", is_active=True)

            # Get all series for specific account
            account_series = store.list(
                user_id="user_darwin",
                account_id="acc_chase_credit_1"
            )
        """

    def update(
        self,
        series_id: str,
        user_id: str,
        updates: dict
    ) -> Series:
        """
        Update series details.

        IMMUTABLE FIELDS (cannot be updated):
        - account_id: Changing account would break instance links
        - counterparty_id: Changing counterparty would break matching logic

        MUTABLE FIELDS (can be updated):
        - name: Display name (must remain unique)
        - expected_amount: Expected payment amount
        - tolerance: Acceptable variance
        - frequency: Recurrence pattern
        - category: Category tag

        Args:
            series_id: Series to update
            user_id: Owner of the series (for authorization)
            updates: Dictionary of fields to update

        Returns:
            Updated Series object

        Raises:
            SeriesNotFoundError: If series doesn't exist
            UnauthorizedError: If user doesn't own series
            DuplicateSeriesNameError: If new name conflicts with existing series
            ImmutableFieldError: If trying to update account_id or counterparty_id
            ValidationError: If updated values invalid

        Example:
            # User wants to increase tolerance due to price changes
            series = store.update(
                series_id="series_netflix_1",
                user_id="user_darwin",
                updates={
                    "expected_amount": Decimal("-17.99"),  # Price increased
                    "tolerance": Decimal("3.00")  # Wider tolerance
                }
            )
        """

    def archive(
        self,
        series_id: str,
        user_id: str,
        end_date: Optional[date] = None
    ) -> ArchiveResult:
        """
        Archive series (soft delete - set is_active = false, end_date = today).

        Series remains in database to preserve instance history.
        After archiving, RecurrenceEngine stops generating new expected instances.

        Args:
            series_id: Series to archive
            user_id: Owner of the series (for authorization)
            end_date: Optional end date (defaults to today)

        Returns:
            ArchiveResult with series, instance_count, message

        Raises:
            SeriesNotFoundError: If series doesn't exist
            UnauthorizedError: If user doesn't own series

        Example:
            # User canceled their Netflix subscription
            result = store.archive(
                series_id="series_netflix_1",
                user_id="user_darwin"
            )
            print(result.message)
            # "Series archived. 24 historical instances remain."
        """

    def unarchive(
        self,
        series_id: str,
        user_id: str
    ) -> Series:
        """
        Unarchive series (set is_active = true, clear end_date).

        Args:
            series_id: Series to unarchive
            user_id: Owner of the series (for authorization)

        Returns:
            Unarchived Series object

        Raises:
            SeriesNotFoundError: If series doesn't exist
            UnauthorizedError: If user doesn't own series

        Example:
            # User reactivated their gym membership
            series = store.unarchive(
                series_id="series_gym_membership_1",
                user_id="user_darwin"
            )
        """

    def update_next_expected_date(
        self,
        series_id: str,
        next_date: date
    ) -> None:
        """
        Update next expected date for series.

        Called by RecurrenceEngine after calculating next occurrence.
        Internal operation - no authorization check needed (system call).

        Args:
            series_id: Series to update
            next_date: Next expected date
        """

    def _generate_series_id(self, name: str) -> str:
        """
        Generate deterministic series_id from name.

        Format: series_{slug}_{sequence}
        Example: "Netflix Subscription" → "series_netflix_subscription_1"

        Args:
            name: Series name

        Returns:
            series_id string
        """

    def _validate_frequency(self, frequency: FrequencyPattern) -> None:
        """
        Validate frequency pattern structure.

        Raises:
            InvalidFrequencyError: If pattern is malformed
        """
```

---

## Implementation Details

### Database Schema

```sql
CREATE TABLE series (
    series_id VARCHAR(100) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,

    -- Core fields
    name VARCHAR(100) NOT NULL,
    account_id VARCHAR(50) NOT NULL,           -- IMMUTABLE - references accounts table
    counterparty_id VARCHAR(50) NOT NULL,      -- IMMUTABLE - references counterparties table
    expected_amount DECIMAL(12,2) NOT NULL,
    tolerance DECIMAL(12,2) NOT NULL DEFAULT 0,
    frequency JSONB NOT NULL,                  -- FrequencyPattern as JSON
    start_date DATE NOT NULL,
    end_date DATE,                             -- NULL = active, DATE = archived
    category VARCHAR(50),

    -- Metadata
    is_active BOOLEAN DEFAULT TRUE,
    next_expected_date DATE,                   -- Calculated by RecurrenceEngine
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    -- Constraints
    UNIQUE(user_id, LOWER(name)),              -- Unique name per user (case-insensitive)
    INDEX idx_user_active (user_id, is_active),
    INDEX idx_account (account_id),
    INDEX idx_counterparty (counterparty_id),
    INDEX idx_category (category),
    INDEX idx_next_expected (next_expected_date),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    FOREIGN KEY (counterparty_id) REFERENCES counterparties(counterparty_id)
);

-- Change log for audit trail
CREATE TABLE series_change_log (
    log_id VARCHAR(50) PRIMARY KEY,
    series_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(50) NOT NULL,

    operation VARCHAR(20) NOT NULL,            -- CREATE, UPDATE, ARCHIVE, UNARCHIVE
    changes JSONB,                             -- {field: {old: X, new: Y}}
    timestamp TIMESTAMP NOT NULL,

    FOREIGN KEY (series_id) REFERENCES series(series_id),
    INDEX idx_series (series_id),
    INDEX idx_user (user_id)
);
```

### Series ID Generation

```python
def _generate_series_id(self, name: str) -> str:
    """
    Generate series_id: series_{slug}_{sequence}

    Examples:
      "Netflix Subscription" → "series_netflix_subscription_1"
      "Rent - Monthly" → "series_rent_monthly_1"
      "OpenAI ChatGPT Plus" → "series_openai_chatgpt_plus_1"
    """
    # Create slug
    slug = name.lower()
    slug = slug.replace(' ', '_')
    slug = re.sub(r'[^a-z0-9_]', '', slug)  # Remove special chars
    slug = slug[:50]  # Limit slug length

    # Find next sequence number
    existing = self.db.execute(
        "SELECT series_id FROM series WHERE series_id LIKE ? ORDER BY series_id DESC LIMIT 1",
        (f"series_{slug}_%",)
    ).fetchone()

    if existing:
        # Extract sequence from "series_netflix_subscription_3" → 3
        last_seq = int(existing['series_id'].split('_')[-1])
        seq = last_seq + 1
    else:
        seq = 1

    return f"series_{slug}_{seq}"
```

### Uniqueness Validation (Case-Insensitive)

```python
def _check_duplicate_name(
    self,
    user_id: str,
    name: str,
    exclude_series_id: Optional[str] = None
) -> None:
    """
    Raise DuplicateSeriesNameError if name already exists (case-insensitive).

    Args:
        user_id: User to check
        name: Name to check
        exclude_series_id: Exclude this series from check (for updates)

    Raises:
        DuplicateSeriesNameError
    """
    query = """
        SELECT series_id FROM series
        WHERE user_id = ? AND LOWER(name) = LOWER(?) AND is_active = true
    """
    params = [user_id, name]

    if exclude_series_id:
        query += " AND series_id != ?"
        params.append(exclude_series_id)

    existing = self.db.execute(query, params).fetchone()

    if existing:
        raise DuplicateSeriesNameError(
            f"Series with name '{name}' already exists",
            field="name",
            existing_series_id=existing['series_id']
        )
```

### Immutable Field Protection

```python
def _check_immutable_fields(self, updates: dict) -> None:
    """
    Raise ImmutableFieldError if trying to update immutable fields.

    Immutable fields:
    - account_id: Changing account would break instance matching
    - counterparty_id: Changing counterparty would break instance matching

    Args:
        updates: Dictionary of fields to update

    Raises:
        ImmutableFieldError
    """
    immutable_fields = {'account_id', 'counterparty_id'}
    attempted_immutable = immutable_fields.intersection(updates.keys())

    if attempted_immutable:
        raise ImmutableFieldError(
            f"Cannot update immutable fields: {', '.join(attempted_immutable)}",
            fields=list(attempted_immutable)
        )
```

### Archive Operation (Soft Delete)

```python
def archive(
    self,
    series_id: str,
    user_id: str,
    end_date: Optional[date] = None
) -> ArchiveResult:
    """
    Archive series (soft delete).

    Steps:
    1. Verify series exists and user owns it
    2. Count linked instances
    3. Set is_active = false, end_date = today (or provided date)
    4. Log operation to series_change_log
    5. Return result with instance count
    """
    # 1. Verify ownership
    series = self.get(series_id, user_id)
    if not series:
        raise SeriesNotFoundError(f"Series '{series_id}' not found")

    # 2. Count linked instances
    instance_count = self.db.execute(
        "SELECT COUNT(*) as count FROM series_instances WHERE series_id = ?",
        (series_id,)
    ).fetchone()['count']

    # 3. Set end_date
    if end_date is None:
        end_date = date.today()

    # 4. Archive
    self.db.execute(
        """
        UPDATE series
        SET is_active = false, end_date = ?, updated_at = ?
        WHERE series_id = ? AND user_id = ?
        """,
        (end_date, datetime.utcnow(), series_id, user_id)
    )

    # 5. Log
    self._log_change(
        series_id=series_id,
        user_id=user_id,
        operation="ARCHIVE",
        changes={
            "is_active": {"old": True, "new": False},
            "end_date": {"old": None, "new": end_date.isoformat()}
        }
    )

    # 6. Return
    series.is_active = False
    series.end_date = end_date
    return ArchiveResult(
        series=series,
        instance_count=instance_count,
        message=f"Series archived. {instance_count} historical instances remain."
    )
```

### Frequency Pattern Validation

```python
def _validate_frequency(self, frequency: FrequencyPattern) -> None:
    """
    Validate frequency pattern structure.

    Valid patterns:
    - Daily: {type: "daily", interval: 1+}
    - Weekly: {type: "weekly", day_of_week: 0-6, interval: 1+}
    - Monthly: {type: "monthly", day_of_month: 1-31, interval: 1+}
    - Yearly: {type: "yearly", month: 1-12, day: 1-31}
    - Custom: {type: "custom", dates: ["YYYY-MM-DD", ...]}

    Raises:
        InvalidFrequencyError
    """
    freq_type = frequency.type

    if freq_type == "daily":
        if not frequency.interval or frequency.interval < 1:
            raise InvalidFrequencyError("Daily frequency requires interval >= 1")

    elif freq_type == "weekly":
        if not frequency.interval or frequency.interval < 1:
            raise InvalidFrequencyError("Weekly frequency requires interval >= 1")
        if frequency.day_of_week is None or not (0 <= frequency.day_of_week <= 6):
            raise InvalidFrequencyError("Weekly frequency requires day_of_week (0-6)")

    elif freq_type == "monthly":
        if not frequency.interval or frequency.interval < 1:
            raise InvalidFrequencyError("Monthly frequency requires interval >= 1")
        if frequency.day_of_month is None or not (1 <= frequency.day_of_month <= 31):
            raise InvalidFrequencyError("Monthly frequency requires day_of_month (1-31)")

    elif freq_type == "yearly":
        if frequency.month is None or not (1 <= frequency.month <= 12):
            raise InvalidFrequencyError("Yearly frequency requires month (1-12)")
        if frequency.day is None or not (1 <= frequency.day <= 31):
            raise InvalidFrequencyError("Yearly frequency requires day (1-31)")

    elif freq_type == "custom":
        if not frequency.dates or len(frequency.dates) == 0:
            raise InvalidFrequencyError("Custom frequency requires dates list")
        # Validate date format
        for date_str in frequency.dates:
            try:
                datetime.strptime(date_str, "%Y-%m-%d")
            except ValueError:
                raise InvalidFrequencyError(f"Invalid date format: {date_str}")

    else:
        raise InvalidFrequencyError(f"Invalid frequency type: {freq_type}")
```

---

## Usage Examples

### Example 1: Create Monthly Subscription

```python
store = SeriesStore(db_connection)

# Create Netflix subscription series
series = store.create(
    user_id="user_darwin",
    name="Netflix Subscription",
    account_id="acc_chase_credit_1",
    counterparty_id="cpty_netflix_1",
    expected_amount=Decimal("-15.99"),
    tolerance=Decimal("2.00"),  # Allow $13.99 - $17.99
    frequency=FrequencyPattern(
        type="monthly",
        day_of_month=15,
        interval=1  # Every 1 month
    ),
    start_date=date(2024, 1, 15),
    category="software_saas"
)

print(series.series_id)  # "series_netflix_subscription_1"
print(series.name)  # "Netflix Subscription"
print(series.is_active)  # True
```

### Example 2: Create Weekly Grocery Payment

```python
# Weekly grocery budget
series = store.create(
    user_id="user_darwin",
    name="Weekly Groceries",
    account_id="acc_chase_credit_1",
    counterparty_id="cpty_whole_foods_1",
    expected_amount=Decimal("-150.00"),
    tolerance=Decimal("50.00"),  # Allow $100 - $200
    frequency=FrequencyPattern(
        type="weekly",
        day_of_week=6,  # Sunday
        interval=1
    ),
    start_date=date(2024, 1, 7),
    category="groceries"
)
```

### Example 3: Create Biweekly Salary

```python
# Biweekly paycheck (positive amount = income)
series = store.create(
    user_id="user_darwin",
    name="Salary - Biweekly",
    account_id="acc_chase_checking_1",
    counterparty_id="cpty_employer_1",
    expected_amount=Decimal("3500.00"),  # Positive = income
    tolerance=Decimal("100.00"),
    frequency=FrequencyPattern(
        type="weekly",
        day_of_week=4,  # Friday
        interval=2  # Every 2 weeks
    ),
    start_date=date(2024, 1, 5),
    category="salary"
)
```

### Example 4: Update Series Tolerance

```python
# Netflix increased price, update expected amount
series = store.update(
    series_id="series_netflix_subscription_1",
    user_id="user_darwin",
    updates={
        "expected_amount": Decimal("-17.99"),
        "tolerance": Decimal("3.00")  # Wider tolerance
    }
)

print(series.expected_amount)  # Decimal("-17.99")
print(series.tolerance)  # Decimal("3.00")
```

### Example 5: List Active Series

```python
# Get all active series for user
active_series = store.list(user_id="user_darwin", is_active=True)

for s in active_series:
    print(f"{s.name}: ${abs(s.expected_amount)} {s.frequency.type}")

# Output:
# Netflix Subscription: $17.99 monthly
# Weekly Groceries: $150.00 weekly
# Salary - Biweekly: $3500.00 weekly
```

### Example 6: Archive Canceled Subscription

```python
# User canceled Netflix
result = store.archive(
    series_id="series_netflix_subscription_1",
    user_id="user_darwin"
)

print(result.message)  # "Series archived. 24 historical instances remain."
print(result.instance_count)  # 24
print(result.series.is_active)  # False
print(result.series.end_date)  # today's date
```

### Example 7: Handle Duplicate Name Error

```python
try:
    store.create(
        user_id="user_darwin",
        name="Netflix Subscription",  # Already exists
        account_id="acc_chase_credit_1",
        counterparty_id="cpty_netflix_1",
        expected_amount=Decimal("-15.99"),
        tolerance=Decimal("2.00"),
        frequency=FrequencyPattern(type="monthly", day_of_month=15, interval=1),
        start_date=date(2024, 1, 15)
    )
except DuplicateSeriesNameError as e:
    print(e.message)  # "Series with name 'Netflix Subscription' already exists"
    print(e.existing_series_id)  # "series_netflix_subscription_1"
```

### Example 8: Handle Immutable Field Error

```python
try:
    # Try to change account (not allowed)
    store.update(
        series_id="series_netflix_subscription_1",
        user_id="user_darwin",
        updates={"account_id": "acc_different_account_1"}
    )
except ImmutableFieldError as e:
    print(e.message)  # "Cannot update immutable fields: account_id"
    print(e.fields)  # ["account_id"]
```

---

## Multi-Domain Examples

### Healthcare: Insurance Premium Series

```python
# Monthly health insurance premium
series = store.create(
    user_id="patient_123",
    name="Health Insurance Premium",
    account_id="acc_checking_1",
    counterparty_id="cpty_blue_cross_1",
    expected_amount=Decimal("-450.00"),
    tolerance=Decimal("50.00"),
    frequency=FrequencyPattern(type="monthly", day_of_month=1, interval=1),
    start_date=date(2024, 1, 1),
    category="health_insurance"
)

# Quarterly prescription refill
refill_series = store.create(
    user_id="patient_123",
    name="Prescription Refill - Medication X",
    account_id="acc_hsa_1",
    counterparty_id="cpty_pharmacy_1",
    expected_amount=Decimal("-85.00"),
    tolerance=Decimal("10.00"),
    frequency=FrequencyPattern(type="monthly", day_of_month=15, interval=3),
    start_date=date(2024, 1, 15),
    category="prescriptions"
)
```

### Legal: Monthly Retainer

```python
# Law firm monthly retainer
series = store.create(
    user_id="lawyer_456",
    name="Client Trust Retainer - Smith Case",
    account_id="acc_trust_account_1",
    counterparty_id="cpty_law_firm_1",
    expected_amount=Decimal("-2500.00"),
    tolerance=Decimal("0.00"),  # No variance allowed
    frequency=FrequencyPattern(type="monthly", day_of_month=1, interval=1),
    start_date=date(2024, 1, 1),
    category="legal_retainer"
)
```

### Research: Grant Disbursement

```python
# Quarterly NSF grant disbursement
series = store.create(
    user_id="researcher_789",
    name="NSF Grant R01-ABC123 - Quarterly",
    account_id="acc_grant_account_1",
    counterparty_id="cpty_nsf_1",
    expected_amount=Decimal("125000.00"),  # Positive = income
    tolerance=Decimal("5000.00"),
    frequency=FrequencyPattern(type="monthly", day_of_month=1, interval=3),
    start_date=date(2024, 1, 1),
    category="research_grants"
)
```

### Manufacturing: Equipment Maintenance

```python
# Annual equipment maintenance
series = store.create(
    user_id="plant_manager_101",
    name="CNC Machine Maintenance - Annual",
    account_id="acc_operations_1",
    counterparty_id="cpty_maintenance_vendor_1",
    expected_amount=Decimal("-15000.00"),
    tolerance=Decimal("2000.00"),
    frequency=FrequencyPattern(type="yearly", month=6, day=15),
    start_date=date(2024, 6, 15),
    category="equipment_maintenance"
)
```

---

## Error Handling

### Custom Exceptions

```python
class SeriesStoreError(Exception):
    """Base exception for SeriesStore errors."""
    pass

class DuplicateSeriesNameError(SeriesStoreError):
    def __init__(self, message: str, field: str = "name", existing_series_id: str = None):
        self.message = message
        self.field = field
        self.existing_series_id = existing_series_id
        super().__init__(message)

class InvalidAccountError(SeriesStoreError):
    def __init__(self, message: str, account_id: str):
        self.message = message
        self.account_id = account_id
        super().__init__(message)

class InvalidCounterpartyError(SeriesStoreError):
    def __init__(self, message: str, counterparty_id: str):
        self.message = message
        self.counterparty_id = counterparty_id
        super().__init__(message)

class InvalidFrequencyError(SeriesStoreError):
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class InvalidDateError(SeriesStoreError):
    def __init__(self, message: str, date_value: date):
        self.message = message
        self.date_value = date_value
        super().__init__(message)

class SeriesNotFoundError(SeriesStoreError):
    pass

class UnauthorizedError(SeriesStoreError):
    pass

class ImmutableFieldError(SeriesStoreError):
    def __init__(self, message: str, fields: List[str]):
        self.message = message
        self.fields = fields
        super().__init__(message)

class ValidationError(SeriesStoreError):
    pass
```

### Error Response Mapping (HTTP)

```python
ERROR_STATUS_CODES = {
    DuplicateSeriesNameError: 409,  # Conflict
    InvalidAccountError: 400,       # Bad Request
    InvalidCounterpartyError: 400,  # Bad Request
    InvalidFrequencyError: 400,     # Bad Request
    InvalidDateError: 400,          # Bad Request
    ValidationError: 400,           # Bad Request
    SeriesNotFoundError: 404,       # Not Found
    UnauthorizedError: 403,         # Forbidden
    ImmutableFieldError: 400        # Bad Request
}
```

---

## Performance Characteristics

### Query Performance

| Operation | Query Type | Expected Latency (p95) | Index Used |
|-----------|------------|------------------------|------------|
| `create()` | INSERT | <30ms | N/A |
| `get()` | SELECT by PK | <10ms | PRIMARY KEY |
| `list()` | SELECT by user_id | <50ms | idx_user_active |
| `update()` | UPDATE by PK | <20ms | PRIMARY KEY |
| `archive()` | UPDATE + SELECT | <100ms | PRIMARY KEY + series_instances index |

### Caching Strategy

```python
# Cache active series per user
cache_key = f"series:{user_id}:active"
ttl = 300  # 5 minutes

def list_cached(user_id: str) -> List[Series]:
    """Get series from cache or database."""
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    series = self.list(user_id=user_id, is_active=True)
    redis.setex(cache_key, ttl, json.dumps([s.__dict__ for s in series]))
    return series

def _invalidate_cache(user_id: str):
    """Invalidate cache on create/update/archive."""
    redis.delete(f"series:{user_id}:active")
```

---

## Audit Trail

Every series operation is logged to `series_change_log` table:

```python
def _log_change(
    self,
    series_id: str,
    user_id: str,
    operation: str,  # CREATE, UPDATE, ARCHIVE, UNARCHIVE
    changes: dict    # {field: {old: X, new: Y}}
):
    """Log series change for audit trail."""
    log_id = f"scl_{int(time.time() * 1000)}"

    self.db.execute(
        """
        INSERT INTO series_change_log (log_id, series_id, user_id, operation, changes, timestamp)
        VALUES (?, ?, ?, ?, ?, ?)
        """,
        (log_id, series_id, user_id, operation, json.dumps(changes), datetime.utcnow())
    )
```

**Example Log Entries:**

```json
// CREATE operation
{
  "log_id": "scl_1234567890",
  "series_id": "series_netflix_subscription_1",
  "user_id": "user_darwin",
  "operation": "CREATE",
  "changes": {
    "name": {"old": null, "new": "Netflix Subscription"},
    "expected_amount": {"old": null, "new": -15.99},
    "frequency": {"old": null, "new": {"type": "monthly", "day_of_month": 15}}
  },
  "timestamp": "2025-05-15T10:00:00Z"
}

// UPDATE operation
{
  "log_id": "scl_1234567891",
  "series_id": "series_netflix_subscription_1",
  "user_id": "user_darwin",
  "operation": "UPDATE",
  "changes": {
    "expected_amount": {"old": -15.99, "new": -17.99},
    "tolerance": {"old": 2.00, "new": 3.00}
  },
  "timestamp": "2025-06-01T14:30:00Z"
}

// ARCHIVE operation
{
  "log_id": "scl_1234567892",
  "series_id": "series_netflix_subscription_1",
  "user_id": "user_darwin",
  "operation": "ARCHIVE",
  "changes": {
    "is_active": {"old": true, "new": false},
    "end_date": {"old": null, "new": "2025-12-31"}
  },
  "timestamp": "2025-12-31T09:00:00Z"
}
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Create series
def test_create_series_success()
def test_create_series_duplicate_name()
def test_create_series_invalid_account()
def test_create_series_invalid_counterparty()
def test_create_series_invalid_frequency()
def test_create_series_future_start_date()  # Should fail

# Test: Update series
def test_update_series_name()
def test_update_series_expected_amount()
def test_update_series_tolerance()
def test_update_series_frequency()
def test_update_series_immutable_account()  # Should fail
def test_update_series_immutable_counterparty()  # Should fail
def test_update_series_duplicate_name()

# Test: Archive series
def test_archive_series_success()
def test_archive_series_with_instances()
def test_archive_series_not_found()
def test_unarchive_series()

# Test: List series
def test_list_active_series()
def test_list_archived_series()
def test_list_all_series()
def test_list_by_account()
def test_list_by_category()

# Test: Authorization
def test_get_series_unauthorized()
def test_update_series_unauthorized()
def test_archive_series_unauthorized()

# Test: Frequency validation
def test_validate_daily_frequency()
def test_validate_weekly_frequency()
def test_validate_monthly_frequency()
def test_validate_yearly_frequency()
def test_validate_custom_frequency()
def test_validate_invalid_frequency()
```

---

## Related Primitives

- **RecurrenceEngine** (OL) - Calculates next expected dates from frequency patterns
- **InstanceTracker** (OL) - Links transactions to series instances
- **AccountStore** (OL) - Series references account_id (from 3.1)
- **CounterpartyStore** (OL) - Series references counterparty_id (from 3.2)
- **SeriesManager** (IL) - UI component for series CRUD operations
- **SeriesSelector** (IL) - Dropdown component for selecting series

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 2-3 days
**Dependencies:** Database (PostgreSQL/MySQL), AccountStore (3.1), CounterpartyStore (3.2), Validation library
