# Primitive: DateNormalizer (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Timezone Converter
> **Vertical:** 3.6 Unit (Currency & Date Normalization)
> **Last Updated:** 2025-10-24

---

## Overview

**DateNormalizer** is an Objective Layer primitive that converts dates and times between user's local timezone and UTC storage format. It handles timezone conversions, daylight saving time (DST) transitions, and ISO 8601 formatting.

**Core Responsibilities:**
- Convert user timezone datetime to UTC for storage
- Convert UTC datetime to user timezone for display
- Handle DST transitions (spring forward, fall back)
- Parse and validate ISO 8601 datetime strings
- Format datetimes for human-readable display
- Support IANA timezone database (400+ timezones)

**Finance Use Case:**
Store transaction dates in UTC, display in user's local timezone (e.g., America/Mexico_City).

**Multi-Domain Applicability:**
- Healthcare: Normalize appointment times across global clinical trials
- E-commerce: Display order timestamps in customer's local time
- Legal: Convert court filing deadlines to jurisdiction timezone
- Travel: Show flight times in departure/arrival timezone
- Manufacturing: Log production timestamps in factory's local time

---

## API Contract

### Methods

```python
class DateNormalizer:
    """
    Converts dates/times between timezones with DST awareness.

    Attributes:
        timezone_db: IANA timezone database (loaded at initialization)
    """

    def __init__(self):
        """
        Initialize normalizer with IANA timezone database.

        Loads all 400+ timezones from pytz/tzinfo.
        """
        self.timezone_db = pytz.all_timezones

    def normalize(
        self,
        user_date: datetime,
        user_tz: str
    ) -> datetime:
        """
        Convert user's local datetime to UTC.

        Args:
            user_date: Datetime in user's local timezone (naive or aware)
            user_tz: IANA timezone string (e.g., "America/Mexico_City")

        Returns:
            UTC datetime (timezone-aware, tzinfo=pytz.UTC)

        Raises:
            InvalidTimezoneError: If user_tz not in IANA database
            AmbiguousTimeError: If datetime falls in DST transition (fall back)
            NonExistentTimeError: If datetime doesn't exist (spring forward)

        Example:
            >>> normalizer = DateNormalizer()
            >>> user_date = datetime(2024, 11, 1, 10, 0, 0)  # 10 AM Mexico City
            >>> utc = normalizer.normalize(user_date, "America/Mexico_City")
            >>> print(utc)
            2024-11-01 16:00:00+00:00  # 4 PM UTC (GMT-6)
        """

    def display(
        self,
        utc_datetime: datetime,
        user_tz: str,
        format_string: str = "%b %d, %Y %I:%M %p"
    ) -> str:
        """
        Convert UTC datetime to user's local timezone and format for display.

        Args:
            utc_datetime: Timezone-aware datetime in UTC
            user_tz: IANA timezone string for display
            format_string: strftime format string (default: "Nov 1, 2024 10:00 AM")

        Returns:
            Formatted datetime string in user's timezone

        Example:
            >>> utc = datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)
            >>> display = normalizer.display(utc, "America/Mexico_City")
            >>> print(display)
            "Nov 1, 2024 10:00 AM"  # Back to Mexico City time
        """

    def parse_iso8601(self, iso_string: str) -> datetime:
        """
        Parse ISO 8601 datetime string to timezone-aware datetime.

        Args:
            iso_string: ISO 8601 formatted string (e.g., "2024-11-01T16:00:00Z")

        Returns:
            Timezone-aware datetime (UTC)

        Raises:
            InvalidDateFormatError: If string doesn't match ISO 8601

        Example:
            >>> dt = normalizer.parse_iso8601("2024-11-01T16:00:00Z")
            >>> print(dt.tzinfo)
            UTC
        """

    def format_iso8601(self, dt: datetime) -> str:
        """
        Format datetime as ISO 8601 string.

        Args:
            dt: Timezone-aware datetime

        Returns:
            ISO 8601 formatted string (e.g., "2024-11-01T16:00:00Z")

        Example:
            >>> dt = datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)
            >>> iso = normalizer.format_iso8601(dt)
            >>> print(iso)
            "2024-11-01T16:00:00Z"
        """

    def validate_timezone(self, tz: str) -> bool:
        """
        Validate timezone string against IANA database.

        Args:
            tz: Timezone string to validate

        Returns:
            True if valid, False otherwise

        Example:
            >>> normalizer.validate_timezone("America/Mexico_City")
            True
            >>> normalizer.validate_timezone("Invalid/Timezone")
            False
        """

    def get_timezone_offset(self, tz: str, dt: datetime) -> timedelta:
        """
        Get UTC offset for timezone at specific datetime.

        Args:
            tz: IANA timezone string
            dt: Datetime to check offset for (needed for DST)

        Returns:
            timedelta representing UTC offset

        Example:
            >>> offset = normalizer.get_timezone_offset(
            ...     "America/New_York",
            ...     datetime(2024, 7, 1)  # Summer (DST)
            ... )
            >>> print(offset)
            -4:00:00  # EDT (GMT-4)

            >>> offset = normalizer.get_timezone_offset(
            ...     "America/New_York",
            ...     datetime(2024, 12, 1)  # Winter (Standard)
            ... )
            >>> print(offset)
            -5:00:00  # EST (GMT-5)
        """

    def list_timezones(self, region: Optional[str] = None) -> List[str]:
        """
        List all available timezones, optionally filtered by region.

        Args:
            region: Optional region filter (e.g., "America", "Europe", "Asia")

        Returns:
            List of IANA timezone strings

        Example:
            >>> normalizer.list_timezones("America")
            ["America/New_York", "America/Chicago", "America/Mexico_City", ...]
        """
```

---

## Data Structures

### Input Types

```python
# Timezone string (IANA)
Timezone = str  # Examples: "America/Mexico_City", "Europe/London", "Asia/Tokyo"

# Naive datetime (no timezone info)
NaiveDatetime = datetime  # tzinfo=None

# Timezone-aware datetime
AwareDatetime = datetime  # tzinfo set (e.g., pytz.UTC)
```

### Output Types

```python
# UTC datetime (always timezone-aware)
UTCDatetime = datetime  # tzinfo=pytz.UTC

# Formatted display string
DisplayString = str  # Example: "Nov 1, 2024 10:00 AM"

# ISO 8601 string
ISO8601String = str  # Example: "2024-11-01T16:00:00Z"
```

---

## Behavior Specifications

### Basic Timezone Conversion

**Scenario:** Convert 10:00 AM Mexico City to UTC

```python
user_date = datetime(2024, 11, 1, 10, 0, 0)  # Naive datetime
utc = normalizer.normalize(user_date, "America/Mexico_City")

# Expected: 2024-11-01 16:00:00+00:00 (4 PM UTC)
# Mexico City is GMT-6 in winter (CST)
```

**Logic:**
1. Localize naive datetime to user's timezone
2. Convert to UTC
3. Return timezone-aware datetime

---

### DST Transition: Spring Forward (Non-Existent Time)

**Scenario:** Convert 2:30 AM on March 10, 2024 (DST transition in US)

```python
# In US, clocks "spring forward" from 2:00 AM to 3:00 AM
# 2:30 AM doesn't exist!

user_date = datetime(2024, 3, 10, 2, 30, 0)

try:
    utc = normalizer.normalize(user_date, "America/New_York")
except NonExistentTimeError as e:
    # Handle: Use 3:30 AM (post-transition) or 1:30 AM (pre-transition)
    utc = normalizer.normalize(user_date, "America/New_York", is_dst=False)
    # Returns: 2024-03-10 07:30:00+00:00 (assumes 3:30 AM EDT)
```

**Default Behavior:**
- If `is_dst=None` (default): Raise `NonExistentTimeError`
- If `is_dst=False`: Use post-transition time (3:30 AM EDT)
- If `is_dst=True`: Use pre-transition time (1:30 AM EST) - not applicable here

---

### DST Transition: Fall Back (Ambiguous Time)

**Scenario:** Convert 1:30 AM on November 3, 2024 (DST ends in US)

```python
# In US, clocks "fall back" from 2:00 AM to 1:00 AM
# 1:30 AM occurs twice!

user_date = datetime(2024, 11, 3, 1, 30, 0)

try:
    utc = normalizer.normalize(user_date, "America/New_York")
except AmbiguousTimeError as e:
    # Handle: Specify which 1:30 AM (first or second occurrence)
    utc_first = normalizer.normalize(user_date, "America/New_York", is_dst=True)
    # Returns: 2024-11-03 05:30:00+00:00 (1:30 AM EDT, before transition)

    utc_second = normalizer.normalize(user_date, "America/New_York", is_dst=False)
    # Returns: 2024-11-03 06:30:00+00:00 (1:30 AM EST, after transition)
```

**Default Behavior:**
- If `is_dst=None` (default): Raise `AmbiguousTimeError`
- If `is_dst=True`: Use first occurrence (before "fall back")
- If `is_dst=False`: Use second occurrence (after "fall back")

---

### Display Formatting

**Scenario:** Display UTC datetime in user's timezone

```python
utc = datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)

# Format 1: Default (Nov 1, 2024 10:00 AM)
display = normalizer.display(utc, "America/Mexico_City")
# → "Nov 1, 2024 10:00 AM"

# Format 2: ISO 8601 (2024-11-01T10:00:00-06:00)
display = normalizer.display(utc, "America/Mexico_City", "%Y-%m-%dT%H:%M:%S%z")
# → "2024-11-01T10:00:00-0600"

# Format 3: Full date (Friday, November 1, 2024)
display = normalizer.display(utc, "America/Mexico_City", "%A, %B %d, %Y")
# → "Friday, November 1, 2024"
```

**Common Format Strings:**
- `%b %d, %Y %I:%M %p` → Nov 1, 2024 10:00 AM (default)
- `%Y-%m-%d %H:%M:%S` → 2024-11-01 10:00:00
- `%A, %B %d, %Y` → Friday, November 1, 2024
- `%I:%M %p` → 10:00 AM (time only)

---

### ISO 8601 Parsing and Formatting

**Scenario:** Parse API response datetime

```python
# API returns ISO 8601 string
iso_string = "2024-11-01T16:00:00Z"

dt = normalizer.parse_iso8601(iso_string)
# Returns: datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)

# Format back to ISO 8601
iso_output = normalizer.format_iso8601(dt)
# Returns: "2024-11-01T16:00:00Z"
```

**Supported ISO 8601 Formats:**
- `2024-11-01T16:00:00Z` (UTC, "Z" suffix)
- `2024-11-01T10:00:00-06:00` (Mexico City, with offset)
- `2024-11-01T16:00:00+00:00` (UTC, explicit offset)

---

## Error Handling

### InvalidTimezoneError

**Trigger:** Timezone string not in IANA database

```python
try:
    normalizer.normalize(datetime.now(), "Invalid/Timezone")
except InvalidTimezoneError as e:
    print(e)
    # "Timezone 'Invalid/Timezone' is not valid. Must be IANA timezone."
```

**Valid Timezones:** 400+ IANA timezones (America/New_York, Europe/London, Asia/Tokyo, etc.)

---

### NonExistentTimeError

**Trigger:** Datetime falls in DST "spring forward" gap

```python
try:
    # March 10, 2024, 2:30 AM doesn't exist in New York (DST starts)
    normalizer.normalize(datetime(2024, 3, 10, 2, 30), "America/New_York")
except NonExistentTimeError as e:
    print(e)
    # "Time 2024-03-10 02:30:00 doesn't exist in America/New_York (DST transition)"
```

**Mitigation:**
- Prompt user to confirm intended time (before or after transition)
- Use `is_dst=False` to assume post-transition time
- Add 1 hour to user's input (2:30 AM → 3:30 AM)

---

### AmbiguousTimeError

**Trigger:** Datetime falls in DST "fall back" overlap

```python
try:
    # November 3, 2024, 1:30 AM occurs twice in New York (DST ends)
    normalizer.normalize(datetime(2024, 11, 3, 1, 30), "America/New_York")
except AmbiguousTimeError as e:
    print(e)
    # "Time 2024-11-03 01:30:00 is ambiguous in America/New_York (DST transition)"
```

**Mitigation:**
- Prompt user to specify first or second occurrence
- Default to first occurrence (`is_dst=True`)
- Show both options: "1:30 AM EDT" or "1:30 AM EST"

---

### InvalidDateFormatError

**Trigger:** ISO 8601 string malformed

```python
try:
    normalizer.parse_iso8601("invalid-date-string")
except InvalidDateFormatError as e:
    print(e)
    # "Invalid ISO 8601 format: 'invalid-date-string'"
```

**Valid Formats:** ISO 8601 compliant strings only

---

## Performance Characteristics

**Latency Targets (p95):**
- Normalize (user → UTC): <5ms
- Display (UTC → user): <5ms
- Parse ISO 8601: <2ms
- Format ISO 8601: <2ms

**Throughput:**
- ~200,000 conversions/second (single thread)

**Optimization Techniques:**
1. **Timezone Caching:** Load IANA database once at initialization
2. **Offset Caching:** Cache UTC offsets per timezone per date (DST changes)
3. **Format Caching:** Cache format strings (avoid re-parsing)

---

## Dependencies

**Internal:**
- `pytz` (Python IANA timezone database)
- `datetime` (Python standard library)

**External:**
- None (all dependencies bundled)

**Data:**
- IANA timezone database (updated quarterly, ~5 MB)

---

## Testing Strategy

### Unit Tests

```python
def test_normalize_basic():
    """Test basic timezone conversion."""
    normalizer = DateNormalizer()
    user_date = datetime(2024, 11, 1, 10, 0, 0)

    utc = normalizer.normalize(user_date, "America/Mexico_City")

    assert utc.tzinfo == pytz.UTC
    assert utc.hour == 16  # 10 AM CST = 4 PM UTC (GMT-6)

def test_normalize_dst_spring_forward():
    """Test non-existent time during spring forward."""
    normalizer = DateNormalizer()
    user_date = datetime(2024, 3, 10, 2, 30, 0)  # Doesn't exist

    with pytest.raises(NonExistentTimeError):
        normalizer.normalize(user_date, "America/New_York")

    # With is_dst=False (assume post-transition)
    utc = normalizer.normalize(user_date, "America/New_York", is_dst=False)
    assert utc.hour == 7  # 3:30 AM EDT = 7:30 AM UTC (GMT-4)

def test_normalize_dst_fall_back():
    """Test ambiguous time during fall back."""
    normalizer = DateNormalizer()
    user_date = datetime(2024, 11, 3, 1, 30, 0)  # Occurs twice

    with pytest.raises(AmbiguousTimeError):
        normalizer.normalize(user_date, "America/New_York")

    # First occurrence (is_dst=True, before fall back)
    utc_first = normalizer.normalize(user_date, "America/New_York", is_dst=True)
    assert utc_first.hour == 5  # 1:30 AM EDT = 5:30 AM UTC (GMT-4)

    # Second occurrence (is_dst=False, after fall back)
    utc_second = normalizer.normalize(user_date, "America/New_York", is_dst=False)
    assert utc_second.hour == 6  # 1:30 AM EST = 6:30 AM UTC (GMT-5)

def test_display_basic():
    """Test display formatting in user timezone."""
    normalizer = DateNormalizer()
    utc = datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)

    display = normalizer.display(utc, "America/Mexico_City")

    assert "Nov 1, 2024" in display
    assert "10:00 AM" in display  # Back to Mexico City time

def test_parse_iso8601():
    """Test ISO 8601 parsing."""
    normalizer = DateNormalizer()
    iso_string = "2024-11-01T16:00:00Z"

    dt = normalizer.parse_iso8601(iso_string)

    assert dt.year == 2024
    assert dt.month == 11
    assert dt.day == 1
    assert dt.hour == 16
    assert dt.tzinfo == pytz.UTC

def test_format_iso8601():
    """Test ISO 8601 formatting."""
    normalizer = DateNormalizer()
    dt = datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)

    iso = normalizer.format_iso8601(dt)

    assert iso == "2024-11-01T16:00:00Z"

def test_validate_timezone():
    """Test timezone validation."""
    normalizer = DateNormalizer()

    assert normalizer.validate_timezone("America/Mexico_City") == True
    assert normalizer.validate_timezone("Invalid/Timezone") == False

def test_get_timezone_offset():
    """Test UTC offset calculation."""
    normalizer = DateNormalizer()

    # Summer (EDT, GMT-4)
    offset_summer = normalizer.get_timezone_offset(
        "America/New_York",
        datetime(2024, 7, 1)
    )
    assert offset_summer == timedelta(hours=-4)

    # Winter (EST, GMT-5)
    offset_winter = normalizer.get_timezone_offset(
        "America/New_York",
        datetime(2024, 12, 1)
    )
    assert offset_winter == timedelta(hours=-5)

def test_list_timezones():
    """Test timezone listing."""
    normalizer = DateNormalizer()

    all_tz = normalizer.list_timezones()
    assert len(all_tz) > 400  # IANA has 400+ timezones

    america_tz = normalizer.list_timezones("America")
    assert "America/New_York" in america_tz
    assert "America/Mexico_City" in america_tz
    assert "Europe/London" not in america_tz
```

### Integration Tests

```python
def test_end_to_end_normalize_display():
    """Test round-trip: normalize → display."""
    normalizer = DateNormalizer()

    # User enters 10:00 AM Mexico City
    user_date = datetime(2024, 11, 1, 10, 0, 0)

    # Normalize to UTC for storage
    utc = normalizer.normalize(user_date, "America/Mexico_City")

    # Display back in Mexico City time
    display = normalizer.display(utc, "America/Mexico_City")

    assert "10:00 AM" in display  # Round-trip success

def test_cross_timezone_display():
    """Test displaying UTC datetime in different timezones."""
    normalizer = DateNormalizer()
    utc = datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)  # 4 PM UTC

    # Display in Mexico City (GMT-6)
    display_mx = normalizer.display(utc, "America/Mexico_City")
    assert "10:00 AM" in display_mx

    # Display in New York (GMT-5)
    display_ny = normalizer.display(utc, "America/New_York")
    assert "11:00 AM" in display_ny

    # Display in London (GMT+0)
    display_london = normalizer.display(utc, "Europe/London")
    assert "4:00 PM" in display_london

    # Display in Tokyo (GMT+9)
    display_tokyo = normalizer.display(utc, "Asia/Tokyo")
    assert "1:00 AM" in display_tokyo  # Next day!
```

---

## Multi-Domain Examples

### Finance: Transaction Timestamps

**Use Case:** Store transaction time in UTC, display in user's timezone.

```python
# User creates transaction at 10:00 AM Mexico City
user_time = datetime(2024, 11, 1, 10, 0, 0)
utc_storage = normalizer.normalize(user_time, "America/Mexico_City")
# → Store: 2024-11-01 16:00:00+00:00 (UTC)

# Display to user
display = normalizer.display(utc_storage, "America/Mexico_City")
# → Show: "Nov 1, 2024 10:00 AM"
```

---

### Healthcare: Global Clinical Trial

**Use Case:** Schedule patient appointments across timezones.

```python
# Trial coordinator in London schedules appointment for patient in Mexico City
london_time = datetime(2024, 11, 1, 14, 0, 0)  # 2 PM London
utc = normalizer.normalize(london_time, "Europe/London")

# Patient sees appointment in their timezone
patient_display = normalizer.display(utc, "America/Mexico_City")
# → "Nov 1, 2024 8:00 AM" (patient's local time)
```

---

### E-commerce: Order Timestamps

**Use Case:** Display order time in customer's timezone.

```python
# Order placed at 16:00 UTC
order_utc = datetime(2024, 11, 1, 16, 0, 0, tzinfo=pytz.UTC)

# Customer in Tokyo sees:
display_tokyo = normalizer.display(order_utc, "Asia/Tokyo")
# → "Nov 2, 2024 1:00 AM" (next day!)

# Customer in Los Angeles sees:
display_la = normalizer.display(order_utc, "America/Los_Angeles")
# → "Nov 1, 2024 9:00 AM"
```

---

### Legal: Court Filing Deadlines

**Use Case:** File document by 11:59 PM in court's jurisdiction.

```python
# Deadline: 11:59 PM EST (court in New York)
deadline_local = datetime(2024, 11, 1, 23, 59, 59)
deadline_utc = normalizer.normalize(deadline_local, "America/New_York")

# Lawyer in California checks deadline:
deadline_display = normalizer.display(deadline_utc, "America/Los_Angeles")
# → "Nov 1, 2024 8:59 PM" (3 hours earlier)
```

---

### Travel: Flight Departure Times

**Use Case:** Display flight departure in airport's local time.

```python
# Flight departs Mexico City at 10:00 AM local time
departure_local = datetime(2024, 11, 1, 10, 0, 0)
departure_utc = normalizer.normalize(departure_local, "America/Mexico_City")

# Traveler in New York checks departure time:
departure_display_ny = normalizer.display(departure_utc, "America/New_York")
# → "Nov 1, 2024 11:00 AM" (New York time)

# But also show original local time:
departure_display_mx = normalizer.display(departure_utc, "America/Mexico_City")
# → "Nov 1, 2024 10:00 AM" (Mexico City time)
```

---

## Observability

### Metrics

```python
# Conversion metrics
date.normalize.count (counter, labels: timezone)
date.display.count (counter, labels: timezone)
date.normalize.latency (histogram)
date.display.latency (histogram)

# Error metrics
date.errors.invalid_timezone (counter, labels: timezone)
date.errors.nonexistent_time (counter, labels: timezone)
date.errors.ambiguous_time (counter, labels: timezone)

# DST metrics
date.dst_transitions.count (counter, labels: timezone, transition_type)
```

### Structured Logs

```json
{
  "timestamp": "2024-11-01T10:00:00Z",
  "level": "INFO",
  "event": "date_normalized",
  "user_date": "2024-11-01T10:00:00",
  "user_timezone": "America/Mexico_City",
  "utc_date": "2024-11-01T16:00:00Z",
  "latency_ms": 3
}
```

---

## Security Considerations

**Input Validation:**
- Timezone: Must be in IANA database (prevent arbitrary strings)
- Datetime: Must be valid date (no invalid dates like Feb 30)
- Format string: Sanitize to prevent injection (limit to whitelisted formats)

**Timezone Database Updates:**
- Update pytz library quarterly (IANA releases updates)
- Test after updates (DST rules change, new timezones added)

---

## Configuration

```yaml
# config/date_normalizer.yaml
default_timezone: UTC
default_format: "%b %d, %Y %I:%M %p"  # Nov 1, 2024 10:00 AM

supported_regions:
  - America
  - Europe
  - Asia
  - Africa
  - Pacific

dst_handling:
  spring_forward: raise_error  # or "skip_to_next_hour"
  fall_back: raise_error  # or "use_first_occurrence"
```

---

## Changelog

**v1.0.0 (2024-11-01):**
- Initial implementation
- Support for 400+ IANA timezones
- DST transition handling
- ISO 8601 parsing and formatting

**v1.1.0 (Future):**
- Automatic timezone detection from IP address
- Timezone abbreviation support (PST, EST, etc.)
- Historical timezone rule support (pre-2000)

---

## References

- IANA Timezone Database: https://www.iana.org/time-zones
- Python pytz documentation: https://pypi.org/project/pytz/
- ISO 8601 standard: https://en.wikipedia.org/wiki/ISO_8601
- DST rules by country: https://www.timeanddate.com/time/dst/

---

**Total Lines:** ~930
