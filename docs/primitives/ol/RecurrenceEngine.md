# RecurrenceEngine (OL Primitive)

**Vertical:** 3.3 Series Registry
**Layer:** Objective Layer (OL)
**Type:** Calculation Engine
**Status:** ✅ Specified

---

## Purpose

Calculate next expected dates for recurring events based on frequency patterns with edge case handling for complex calendar scenarios.

**Core Responsibility:** Given a frequency pattern (daily, weekly, monthly, yearly, custom) and a starting date, calculate the next expected occurrence date. Handle calendar edge cases like last-day-of-month, leap years, and daylight saving transitions.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY recurring event calculation:

| Domain | Use Case | Frequency Examples | Edge Cases |
|--------|----------|-------------------|------------|
| **Finance** | Recurring payments, subscriptions, salary | Monthly on 15th, biweekly Friday | Last day of month (Jan 31 → Feb 28) |
| **Healthcare** | Medication schedules, therapy sessions | Every 3 days, weekly Tuesday | Prescription refills spanning months |
| **Legal** | Filing deadlines, retainer payments | Monthly 1st, quarterly last day | Court calendar holidays |
| **Research** | Grant disbursements, data collection | Quarterly 1st, semiannual | Academic calendar alignment |
| **Manufacturing** | Maintenance schedules, inventory orders | Weekly Monday, monthly last Friday | Production calendar shifts |
| **Media** | Content publishing, subscription renewals | Daily 9am, monthly 1st | Time zone handling |

**Pattern:** Recurrence Calculation = take pattern + last occurrence → produce next expected date with calendar-aware edge case handling.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List
from dataclasses import dataclass
from datetime import date, timedelta
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
class ExpectedInstance:
    """Represents a single expected occurrence."""
    expected_date: date
    expected_amount: Decimal
    series_id: str
    series_name: str

class RecurrenceEngine:
    """
    Calculates next expected dates for recurring events.

    Handles:
    - Daily recurrence (every N days)
    - Weekly recurrence (every N weeks on specific day)
    - Monthly recurrence (every N months on specific day, with last-day logic)
    - Yearly recurrence (specific month + day)
    - Custom recurrence (explicit date list)
    - Edge cases: leap years, month-end transitions, invalid dates
    """

    def calculate_next_expected(
        self,
        frequency: FrequencyPattern,
        last_date: date
    ) -> date:
        """
        Calculate next expected date from last occurrence.

        Args:
            frequency: Recurrence pattern
            last_date: Last occurrence date

        Returns:
            Next expected date

        Raises:
            InvalidFrequencyError: If frequency pattern is malformed
            NoMoreOccurrencesError: If custom pattern exhausted

        Example:
            # Monthly on 15th
            engine = RecurrenceEngine()
            frequency = FrequencyPattern(type="monthly", day_of_month=15, interval=1)
            next_date = engine.calculate_next_expected(frequency, date(2024, 1, 15))
            # Returns: date(2024, 2, 15)

            # Last day of month
            frequency = FrequencyPattern(type="monthly", day_of_month=31, interval=1)
            next_date = engine.calculate_next_expected(frequency, date(2024, 1, 31))
            # Returns: date(2024, 2, 29)  # Leap year!
        """

    def generate_expected_instances(
        self,
        series_id: str,
        series_name: str,
        expected_amount: Decimal,
        frequency: FrequencyPattern,
        start_date: date,
        end_date: Optional[date] = None,
        months_ahead: int = 12
    ) -> List[ExpectedInstance]:
        """
        Generate list of expected instances for a series.

        Useful for:
        - Creating initial instances when series is created
        - Regenerating instances after frequency change
        - Forecasting future occurrences

        Args:
            series_id: Series identifier
            series_name: Series display name
            expected_amount: Expected amount for each instance
            frequency: Recurrence pattern
            start_date: First occurrence date
            end_date: Optional end date (None = ongoing)
            months_ahead: How many months to generate (default 12)

        Returns:
            List of ExpectedInstance objects

        Example:
            # Generate 12 months of Netflix payments
            instances = engine.generate_expected_instances(
                series_id="series_netflix_1",
                series_name="Netflix Subscription",
                expected_amount=Decimal("-15.99"),
                frequency=FrequencyPattern(type="monthly", day_of_month=15, interval=1),
                start_date=date(2024, 1, 15),
                months_ahead=12
            )
            # Returns 12 ExpectedInstance objects: Jan 15, Feb 15, Mar 15, ...
        """

    def is_valid_date(self, year: int, month: int, day: int) -> bool:
        """
        Check if date is valid (handles Feb 29, month boundaries).

        Args:
            year: Year
            month: Month (1-12)
            day: Day (1-31)

        Returns:
            True if valid date, False otherwise

        Example:
            engine.is_valid_date(2024, 2, 29)  # True (leap year)
            engine.is_valid_date(2023, 2, 29)  # False (not leap year)
            engine.is_valid_date(2024, 4, 31)  # False (April has 30 days)
        """

    def get_last_day_of_month(self, year: int, month: int) -> int:
        """
        Get last day of month (28, 29, 30, or 31).

        Args:
            year: Year
            month: Month (1-12)

        Returns:
            Last day of month

        Example:
            engine.get_last_day_of_month(2024, 2)  # 29 (leap year)
            engine.get_last_day_of_month(2023, 2)  # 28
            engine.get_last_day_of_month(2024, 4)  # 30
        """

    def adjust_day_for_month(
        self,
        year: int,
        month: int,
        desired_day: int
    ) -> int:
        """
        Adjust day to fit within month boundaries.

        If desired_day > days in month, return last day of month.

        Args:
            year: Year
            month: Month (1-12)
            desired_day: Desired day (1-31)

        Returns:
            Valid day for the given month

        Example:
            # Jan 31 → Feb 28 (non-leap year)
            engine.adjust_day_for_month(2023, 2, 31)  # 28

            # Jan 31 → Feb 29 (leap year)
            engine.adjust_day_for_month(2024, 2, 31)  # 29

            # Jan 31 → Apr 30 (April has 30 days)
            engine.adjust_day_for_month(2024, 4, 31)  # 30
        """
```

---

## Algorithm Details

### Daily Recurrence

**Pattern:** Every N days

```python
def _calculate_daily(self, frequency: FrequencyPattern, last_date: date) -> date:
    """
    Calculate next daily occurrence.

    Example:
      Every 1 day: Jan 1 → Jan 2 → Jan 3 → ...
      Every 3 days: Jan 1 → Jan 4 → Jan 7 → Jan 10 → ...
    """
    interval = frequency.interval or 1
    return last_date + timedelta(days=interval)
```

### Weekly Recurrence

**Pattern:** Every N weeks on specific day of week

```python
def _calculate_weekly(self, frequency: FrequencyPattern, last_date: date) -> date:
    """
    Calculate next weekly occurrence.

    Example:
      Every 1 week on Monday: Jan 1 (Mon) → Jan 8 (Mon) → Jan 15 (Mon) → ...
      Every 2 weeks on Friday: Jan 5 (Fri) → Jan 19 (Fri) → Feb 2 (Fri) → ...

    Algorithm:
    1. Add interval weeks to last_date
    2. If day_of_week specified, adjust to that day
    """
    interval = frequency.interval or 1
    next_date = last_date + timedelta(weeks=interval)

    # Adjust to specific day of week if needed
    if frequency.day_of_week is not None:
        current_day = next_date.weekday()  # 0=Monday, 6=Sunday
        target_day = frequency.day_of_week
        days_ahead = (target_day - current_day) % 7
        if days_ahead > 0:
            next_date += timedelta(days=days_ahead)

    return next_date
```

### Monthly Recurrence (Complex)

**Pattern:** Every N months on specific day of month

**Edge Case: Last Day of Month**
- Series: "Rent - Monthly" on day 31
- Jan 31 → Feb 29 (leap year) or Feb 28 (non-leap year)
- Feb 28 → Mar 31 (back to 31st)
- Key insight: If desired day > days in month, use last day of month

```python
def _calculate_monthly(self, frequency: FrequencyPattern, last_date: date) -> date:
    """
    Calculate next monthly occurrence with last-day-of-month logic.

    Algorithm:
    1. Add interval months to last_date
    2. Adjust day to fit within month (if day 31 but month has 30 days, use 30)
    3. Special case: If original day > 28, track as "end of month" intent

    Example:
      Jan 15 → Feb 15 → Mar 15 → ... (no adjustment needed)
      Jan 31 → Feb 29 → Mar 31 → Apr 30 → May 31 → ...
      Jan 30 → Feb 28 → Mar 30 → Apr 30 → ...
    """
    interval = frequency.interval or 1
    desired_day = frequency.day_of_month

    # Calculate target month
    current_month = last_date.month
    current_year = last_date.year

    target_month = current_month + interval
    target_year = current_year

    # Handle year rollover
    while target_month > 12:
        target_month -= 12
        target_year += 1

    # Adjust day to fit within target month
    adjusted_day = self.adjust_day_for_month(target_year, target_month, desired_day)

    return date(target_year, target_month, adjusted_day)
```

**Last-Day-of-Month Tracking:**

```python
def _is_last_day_intent(self, day_of_month: int) -> bool:
    """
    Determine if user intends "last day of month" pattern.

    If day_of_month >= 29, treat as "end of month" intent.
    This handles: Jan 31 → Feb 28 → Mar 31 pattern.
    """
    return day_of_month >= 29

def adjust_day_for_month(
    self,
    year: int,
    month: int,
    desired_day: int
) -> int:
    """
    Adjust day to fit within month boundaries.

    Examples:
      adjust_day_for_month(2024, 2, 31) → 29 (leap year)
      adjust_day_for_month(2023, 2, 31) → 28 (non-leap year)
      adjust_day_for_month(2024, 4, 31) → 30 (April has 30 days)
      adjust_day_for_month(2024, 1, 31) → 31 (January has 31 days)
    """
    last_day = self.get_last_day_of_month(year, month)
    return min(desired_day, last_day)

def get_last_day_of_month(self, year: int, month: int) -> int:
    """
    Get last day of month.

    Uses calendar trick: next month's 1st minus 1 day = last day of current month.
    """
    if month == 12:
        next_month_date = date(year + 1, 1, 1)
    else:
        next_month_date = date(year, month + 1, 1)

    last_day_date = next_month_date - timedelta(days=1)
    return last_day_date.day
```

### Yearly Recurrence

**Pattern:** Once per year on specific month + day

```python
def _calculate_yearly(self, frequency: FrequencyPattern, last_date: date) -> date:
    """
    Calculate next yearly occurrence.

    Example:
      Birthday: Jan 15 → next year Jan 15
      Tax deadline: Apr 15 → next year Apr 15

    Edge case: Feb 29 (leap year birthday)
      - If next year is NOT leap year, use Feb 28
    """
    target_month = frequency.month
    target_day = frequency.day
    target_year = last_date.year + 1

    # Adjust day if invalid (e.g., Feb 29 in non-leap year)
    adjusted_day = self.adjust_day_for_month(target_year, target_month, target_day)

    return date(target_year, target_month, adjusted_day)
```

### Custom Recurrence

**Pattern:** Explicit list of dates

```python
def _calculate_custom(self, frequency: FrequencyPattern, last_date: date) -> date:
    """
    Calculate next custom occurrence from explicit date list.

    Example:
      Custom dates: ["2024-01-15", "2024-07-15", "2025-01-15"]
      Last: 2024-01-15 → Next: 2024-07-15
      Last: 2024-07-15 → Next: 2025-01-15
      Last: 2025-01-15 → NoMoreOccurrencesError

    Algorithm:
    1. Parse custom dates
    2. Find first date > last_date
    3. If none found, raise NoMoreOccurrencesError
    """
    if not frequency.dates:
        raise InvalidFrequencyError("Custom frequency requires dates list")

    # Parse and sort dates
    custom_dates = sorted([
        datetime.strptime(d, "%Y-%m-%d").date()
        for d in frequency.dates
    ])

    # Find next date after last_date
    for custom_date in custom_dates:
        if custom_date > last_date:
            return custom_date

    # No more occurrences
    raise NoMoreOccurrencesError(f"No custom dates after {last_date}")
```

---

## Edge Cases

### EC1: Last Day of Month Transition

**Scenario:** Monthly series on day 31

```python
# Series: "Rent - Monthly" on day 31
frequency = FrequencyPattern(type="monthly", day_of_month=31, interval=1)

# Calculate sequence
dates = [date(2024, 1, 31)]  # Start: Jan 31
for _ in range(12):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

# Result:
# Jan 31 → Feb 29 (leap year, use last day)
# Feb 29 → Mar 31 (back to 31st)
# Mar 31 → Apr 30 (April has 30 days)
# Apr 30 → May 31 (back to 31st)
# May 31 → Jun 30 (June has 30 days)
# Jun 30 → Jul 31 (back to 31st)
# ...continues pattern
```

**Implementation:**

```python
def _handle_last_day_intent(
    self,
    target_year: int,
    target_month: int,
    desired_day: int
) -> int:
    """
    If desired_day >= 29, treat as "end of month" intent.

    This ensures: Jan 31 → Feb 28/29 → Mar 31 pattern
    (not: Jan 31 → Feb 28 → Feb 28 → Feb 28 forever)
    """
    if desired_day >= 29:
        return self.get_last_day_of_month(target_year, target_month)
    else:
        return min(desired_day, self.get_last_day_of_month(target_year, target_month))
```

### EC2: Leap Year Handling

**Scenario:** Feb 29 birthday (born in leap year)

```python
# Person born Feb 29, 2000
frequency = FrequencyPattern(type="yearly", month=2, day=29)

# Calculate birthdays
dates = [date(2000, 2, 29)]  # Start: Feb 29, 2000 (leap year)
for year in range(2001, 2010):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

# Result:
# 2000-02-29 (leap year) → 2001-02-28 (not leap year, use Feb 28)
# 2001-02-28 → 2002-02-28
# 2002-02-28 → 2003-02-28
# 2003-02-28 → 2004-02-29 (leap year!)
# 2004-02-29 → 2005-02-28
# ...continues pattern
```

### EC3: Week-of-Month Pattern (Advanced)

**Scenario:** "First Friday of every month" (not directly supported, workaround)

```python
# Workaround: Use custom dates for 12 months
first_fridays = [
    "2024-01-05",  # First Friday of Jan
    "2024-02-02",  # First Friday of Feb
    "2024-03-01",  # First Friday of Mar
    # ...generate programmatically
]

frequency = FrequencyPattern(type="custom", dates=first_fridays)

# Alternative: Use monthly on day 1-7, then filter for Friday
# (requires post-processing in InstanceTracker)
```

### EC4: Daylight Saving Time (DST)

**Note:** RecurrenceEngine works with date-only (no time component), so DST is not an issue at this layer. If time-of-day matters, implement in higher layer.

### EC5: Month Increment Edge Cases

**Scenario:** Adding 1 month to Jan 31

```python
# Standard library behavior
from dateutil.relativedelta import relativedelta

date(2024, 1, 31) + relativedelta(months=1)
# Result: 2024-02-29 (dateutil handles this correctly)

# Our implementation matches this behavior
frequency = FrequencyPattern(type="monthly", day_of_month=31, interval=1)
engine.calculate_next_expected(frequency, date(2024, 1, 31))
# Result: date(2024, 2, 29)
```

---

## Implementation Details

### Complete Monthly Algorithm

```python
def _calculate_monthly(self, frequency: FrequencyPattern, last_date: date) -> date:
    """
    Calculate next monthly occurrence with robust edge case handling.

    Steps:
    1. Extract desired day from frequency
    2. Calculate target month (add interval, handle year rollover)
    3. Check if desired day is valid for target month
    4. If not valid (e.g., Feb 31), use last day of month
    5. Return adjusted date
    """
    interval = frequency.interval or 1
    desired_day = frequency.day_of_month

    if desired_day is None or not (1 <= desired_day <= 31):
        raise InvalidFrequencyError("Monthly frequency requires day_of_month (1-31)")

    # Step 1: Calculate target month/year
    current_month = last_date.month
    current_year = last_date.year

    target_month = current_month + interval
    target_year = current_year

    # Handle year rollover
    while target_month > 12:
        target_month -= 12
        target_year += 1

    # Step 2: Adjust day to fit within target month
    # If desired_day > days in target month, use last day
    last_day = self.get_last_day_of_month(target_year, target_month)
    adjusted_day = min(desired_day, last_day)

    # Step 3: Create and return date
    return date(target_year, target_month, adjusted_day)

def get_last_day_of_month(self, year: int, month: int) -> int:
    """
    Get last day of month using calendar math.

    Algorithm:
    - Create date for 1st of next month
    - Subtract 1 day
    - Extract day component

    Handles leap years automatically via Python's date validation.
    """
    if month == 12:
        next_month = date(year + 1, 1, 1)
    else:
        next_month = date(year, month + 1, 1)

    last_day_date = next_month - timedelta(days=1)
    return last_day_date.day
```

### Generate Expected Instances

```python
def generate_expected_instances(
    self,
    series_id: str,
    series_name: str,
    expected_amount: Decimal,
    frequency: FrequencyPattern,
    start_date: date,
    end_date: Optional[date] = None,
    months_ahead: int = 12
) -> List[ExpectedInstance]:
    """
    Generate list of expected instances.

    Algorithm:
    1. Start with start_date
    2. Loop: calculate next expected date
    3. Stop when: date > cutoff OR date > end_date OR max_iterations reached
    4. Create ExpectedInstance for each date

    Safety: Limit to max 1000 iterations (prevents infinite loop)
    """
    instances = []
    current_date = start_date
    cutoff_date = date.today() + timedelta(days=months_ahead * 30)
    max_iterations = 1000
    iteration = 0

    while iteration < max_iterations:
        # Check if past cutoff
        if current_date > cutoff_date:
            break

        # Check if past series end_date
        if end_date and current_date > end_date:
            break

        # Create instance
        instances.append(ExpectedInstance(
            expected_date=current_date,
            expected_amount=expected_amount,
            series_id=series_id,
            series_name=series_name
        ))

        # Calculate next date
        try:
            current_date = self.calculate_next_expected(frequency, current_date)
        except NoMoreOccurrencesError:
            # Custom frequency exhausted
            break

        iteration += 1

    if iteration >= max_iterations:
        logger.warning(f"Max iterations reached for series {series_id}")

    return instances
```

---

## Usage Examples

### Example 1: Monthly on 15th

```python
engine = RecurrenceEngine()

# Netflix subscription on 15th of every month
frequency = FrequencyPattern(type="monthly", day_of_month=15, interval=1)

dates = [date(2024, 1, 15)]
for _ in range(12):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

print(dates)
# [2024-01-15, 2024-02-15, 2024-03-15, 2024-04-15, 2024-05-15, 2024-06-15,
#  2024-07-15, 2024-08-15, 2024-09-15, 2024-10-15, 2024-11-15, 2024-12-15, 2025-01-15]
```

### Example 2: Last Day of Month

```python
# Rent due on last day of month
frequency = FrequencyPattern(type="monthly", day_of_month=31, interval=1)

dates = [date(2024, 1, 31)]
for _ in range(6):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

print(dates)
# [2024-01-31, 2024-02-29, 2024-03-31, 2024-04-30, 2024-05-31, 2024-06-30, 2024-07-31]
# Notice: Feb 29 (leap year), Apr 30 (not 31), Jun 30 (not 31)
```

### Example 3: Biweekly Paycheck

```python
# Paid every 2 weeks on Friday
frequency = FrequencyPattern(type="weekly", day_of_week=4, interval=2)  # 4 = Friday

dates = [date(2024, 1, 5)]  # Friday, Jan 5
for _ in range(10):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

print(dates)
# [2024-01-05, 2024-01-19, 2024-02-02, 2024-02-16, 2024-03-01, ...]
# Every other Friday
```

### Example 4: Quarterly Payment

```python
# Quarterly on the 1st (every 3 months)
frequency = FrequencyPattern(type="monthly", day_of_month=1, interval=3)

dates = [date(2024, 1, 1)]
for _ in range(8):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

print(dates)
# [2024-01-01, 2024-04-01, 2024-07-01, 2024-10-01, 2025-01-01, 2025-04-01, ...]
# Every 3 months
```

### Example 5: Generate Instances

```python
# Generate 12 months of Netflix payments
instances = engine.generate_expected_instances(
    series_id="series_netflix_1",
    series_name="Netflix Subscription",
    expected_amount=Decimal("-15.99"),
    frequency=FrequencyPattern(type="monthly", day_of_month=15, interval=1),
    start_date=date(2024, 1, 15),
    months_ahead=12
)

for inst in instances:
    print(f"{inst.expected_date}: ${abs(inst.expected_amount)}")

# Output:
# 2024-01-15: $15.99
# 2024-02-15: $15.99
# 2024-03-15: $15.99
# ...12 total
```

### Example 6: Custom Dates

```python
# Semiannual payments (Jan 1 and Jul 1)
frequency = FrequencyPattern(
    type="custom",
    dates=["2024-01-01", "2024-07-01", "2025-01-01", "2025-07-01"]
)

current = date(2024, 1, 1)
for _ in range(3):
    next_date = engine.calculate_next_expected(frequency, current)
    print(f"{current} → {next_date}")
    current = next_date

# Output:
# 2024-01-01 → 2024-07-01
# 2024-07-01 → 2025-01-01
# 2025-01-01 → 2025-07-01
```

---

## Multi-Domain Examples

### Healthcare: Prescription Refill

```python
# Refill every 90 days (not monthly, exact day count)
frequency = FrequencyPattern(type="daily", interval=90)

dates = [date(2024, 1, 15)]
for _ in range(4):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

print(dates)
# [2024-01-15, 2024-04-14, 2024-07-13, 2024-10-11, 2025-01-09]
# Exactly 90 days apart
```

### Legal: Quarterly Filing Deadline

```python
# Quarterly on last day of quarter (Mar 31, Jun 30, Sep 30, Dec 31)
frequency = FrequencyPattern(
    type="custom",
    dates=["2024-03-31", "2024-06-30", "2024-09-30", "2024-12-31",
           "2025-03-31", "2025-06-30", "2025-09-30", "2025-12-31"]
)

instances = engine.generate_expected_instances(
    series_id="series_tax_filing_1",
    series_name="Quarterly Tax Filing",
    expected_amount=Decimal("0"),  # No amount, just tracking
    frequency=frequency,
    start_date=date(2024, 3, 31),
    months_ahead=24
)
```

### Research: Biannual Conference

```python
# Conference every 6 months (Jan and Jul)
frequency = FrequencyPattern(type="monthly", day_of_month=15, interval=6)

dates = [date(2024, 1, 15)]
for _ in range(5):
    next_date = engine.calculate_next_expected(frequency, dates[-1])
    dates.append(next_date)

print(dates)
# [2024-01-15, 2024-07-15, 2025-01-15, 2025-07-15, 2026-01-15, 2026-07-15]
```

---

## Error Handling

### Custom Exceptions

```python
class RecurrenceEngineError(Exception):
    """Base exception for RecurrenceEngine errors."""
    pass

class InvalidFrequencyError(RecurrenceEngineError):
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class NoMoreOccurrencesError(RecurrenceEngineError):
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class DateCalculationError(RecurrenceEngineError):
    def __init__(self, message: str, frequency: FrequencyPattern, last_date: date):
        self.message = message
        self.frequency = frequency
        self.last_date = last_date
        super().__init__(message)
```

---

## Performance Characteristics

### Time Complexity

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `calculate_next_expected()` | O(1) | Simple date arithmetic |
| `generate_expected_instances()` | O(n) | n = number of instances generated (typically 12-365) |
| `get_last_day_of_month()` | O(1) | Calendar lookup |
| `adjust_day_for_month()` | O(1) | Min operation |

### Space Complexity

| Operation | Space Complexity | Notes |
|-----------|------------------|-------|
| `generate_expected_instances()` | O(n) | Stores n ExpectedInstance objects |

### Optimization

```python
# Cache last day of month calculations (rarely changes)
from functools import lru_cache

@lru_cache(maxsize=128)
def get_last_day_of_month_cached(self, year: int, month: int) -> int:
    """Cached version for repeated lookups."""
    if month == 12:
        next_month = date(year + 1, 1, 1)
    else:
        next_month = date(year, month + 1, 1)
    last_day_date = next_month - timedelta(days=1)
    return last_day_date.day
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Daily recurrence
def test_daily_every_1_day()
def test_daily_every_7_days()
def test_daily_every_30_days()

# Test: Weekly recurrence
def test_weekly_every_monday()
def test_weekly_every_friday()
def test_weekly_biweekly()
def test_weekly_every_4_weeks()

# Test: Monthly recurrence
def test_monthly_on_15th()
def test_monthly_on_1st()
def test_monthly_on_last_day()
def test_monthly_jan_31_to_feb_29_leap_year()
def test_monthly_jan_31_to_feb_28_non_leap_year()
def test_monthly_quarterly()
def test_monthly_bimonthly()

# Test: Yearly recurrence
def test_yearly_birthday()
def test_yearly_feb_29_leap_year()
def test_yearly_feb_29_non_leap_year()
def test_yearly_anniversary()

# Test: Custom recurrence
def test_custom_explicit_dates()
def test_custom_exhausted()
def test_custom_empty_list()

# Test: Edge cases
def test_last_day_of_month_sequence()
def test_leap_year_handling()
def test_month_boundaries()
def test_year_rollover()

# Test: Instance generation
def test_generate_instances_12_months()
def test_generate_instances_with_end_date()
def test_generate_instances_max_iterations()
```

### Integration Test Examples

```python
def test_full_year_monthly_sequence():
    """Test 12 months of monthly recurrence."""
    engine = RecurrenceEngine()
    frequency = FrequencyPattern(type="monthly", day_of_month=15, interval=1)

    dates = [date(2024, 1, 15)]
    for _ in range(11):  # 11 more to get 12 total
        next_date = engine.calculate_next_expected(frequency, dates[-1])
        dates.append(next_date)

    assert len(dates) == 12
    assert dates[0] == date(2024, 1, 15)
    assert dates[-1] == date(2024, 12, 15)

def test_last_day_pattern_full_year():
    """Test last-day-of-month pattern for full year."""
    engine = RecurrenceEngine()
    frequency = FrequencyPattern(type="monthly", day_of_month=31, interval=1)

    dates = [date(2024, 1, 31)]
    for _ in range(11):
        next_date = engine.calculate_next_expected(frequency, dates[-1])
        dates.append(next_date)

    # Verify specific months
    assert dates[1] == date(2024, 2, 29)  # Feb leap year
    assert dates[3] == date(2024, 4, 30)  # Apr has 30 days
    assert dates[5] == date(2024, 6, 30)  # Jun has 30 days
```

---

## Related Primitives

- **SeriesStore** (OL) - Stores series with frequency patterns
- **InstanceTracker** (OL) - Uses expected dates to match transactions
- **RecurrenceConfigDialog** (IL) - UI for configuring frequency patterns
- **SeriesManager** (IL) - Displays next expected dates

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 2-3 days
**Dependencies:** Python standard library (datetime, timedelta), optional: dateutil for advanced patterns
