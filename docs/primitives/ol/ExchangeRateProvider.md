# Primitive: ExchangeRateProvider (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Data Provider
> **Vertical:** 3.6 Unit (Currency & Date Normalization)
> **Last Updated:** 2025-10-24

---

## Overview

**ExchangeRateProvider** is an Objective Layer primitive that fetches, caches, and manages exchange rates from multiple sources. It implements multi-source fallback (ECB → Federal Reserve → manual), staleness detection, and historical rate support.

**Core Responsibilities:**
- Fetch exchange rates from ECB (European Central Bank) API
- Fallback to Federal Reserve API if ECB fails
- Cache rates with 24-hour TTL (1-hour for crypto)
- Detect stale rates (> staleness threshold)
- Support historical rate queries (date range)
- Handle manual rate overrides with audit trail
- Calculate inverse rates (USD→EUR = 1 / EUR→USD)

**Finance Use Case:**
Fetch USD/MXN exchange rate for Nov 1, 2024, cache for 24 hours.

**Multi-Domain Applicability:**
- Healthcare: Currency conversion for international claims
- E-commerce: Real-time product pricing conversion
- Travel: Expense conversion with historical rates
- Manufacturing: Raw material cost conversion (international suppliers)
- Payroll: Multi-currency salary payments

---

## API Contract

### Methods

```python
class ExchangeRateProvider:
    """
    Fetches and caches exchange rates from multiple sources.

    Attributes:
        cache: Redis cache instance
        ecb_client: HTTP client for ECB API
        fed_client: HTTP client for Federal Reserve API
        staleness_threshold: Duration before rate marked stale (24h default)
    """

    def __init__(
        self,
        cache: CacheClient,
        ecb_api_url: str,
        fed_api_url: str,
        fed_api_key: str,
        staleness_threshold: timedelta = timedelta(hours=24)
    ):
        """
        Initialize provider with cache and API clients.

        Args:
            cache: Redis cache instance
            ecb_api_url: ECB API endpoint
            fed_api_url: Federal Reserve API endpoint
            fed_api_key: Federal Reserve API key (from env)
            staleness_threshold: Duration before rate marked stale
        """
        self.cache = cache
        self.ecb_client = HTTPClient(ecb_api_url)
        self.fed_client = HTTPClient(fed_api_url, api_key=fed_api_key)
        self.staleness_threshold = staleness_threshold

    def get_rate(
        self,
        from_currency: str,
        to_currency: str,
        rate_date: date
    ) -> ExchangeRateRecord:
        """
        Get exchange rate from cache or fetch from source.

        Rate Source Priority:
        1. Cache (if fresh, < staleness_threshold)
        2. ECB API (European Central Bank)
        3. Federal Reserve API
        4. Error (all sources failed)

        Args:
            from_currency: ISO 4217 currency code (e.g., "USD")
            to_currency: ISO 4217 currency code (e.g., "MXN")
            rate_date: Date for exchange rate lookup

        Returns:
            ExchangeRateRecord with rate and metadata

        Raises:
            RateNotAvailableError: If all sources fail

        Example:
            >>> provider = ExchangeRateProvider(cache, ecb_url, fed_url, fed_key)
            >>> rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))
            >>> print(rate.rate)
            18.5000
            >>> print(rate.source)
            "ecb"
        """

    def refresh_cache(
        self,
        from_currency: str,
        to_currency: str
    ) -> ExchangeRateRecord:
        """
        Force cache invalidation and fetch fresh rate.

        Bypasses cache, always fetches from ECB/Fed API.

        Args:
            from_currency: ISO 4217 currency code
            to_currency: ISO 4217 currency code

        Returns:
            ExchangeRateRecord with fresh rate

        Example:
            >>> rate = provider.refresh_cache("USD", "MXN")
            >>> print(rate.is_stale)
            False
        """

    def get_historical(
        self,
        from_currency: str,
        to_currency: str,
        start_date: date,
        end_date: date
    ) -> List[ExchangeRateRecord]:
        """
        Fetch historical rates for date range.

        Args:
            from_currency: ISO 4217 currency code
            to_currency: ISO 4217 currency code
            start_date: Start of date range (inclusive)
            end_date: End of date range (inclusive)

        Returns:
            List of ExchangeRateRecord, one per day

        Example:
            >>> rates = provider.get_historical(
            ...     "USD", "EUR",
            ...     date(2024, 10, 1),
            ...     date(2024, 10, 31)
            ... )
            >>> print(len(rates))
            31  # One rate per day
        """

    def fetch_from_ecb(
        self,
        from_currency: str,
        to_currency: str,
        rate_date: date
    ) -> Decimal:
        """
        Fetch rate from ECB API (primary source).

        ECB API:
        - Updates daily at 16:00 CET
        - Covers 150+ currencies
        - Base currency: EUR (convert via EUR if needed)
        - Free (no API key)

        Args:
            from_currency: ISO 4217 currency code
            to_currency: ISO 4217 currency code
            rate_date: Date for rate

        Returns:
            Exchange rate (Decimal, 6 decimals)

        Raises:
            ECBAPIError: If API call fails

        Example:
            >>> rate = provider.fetch_from_ecb("USD", "MXN", date(2024, 11, 1))
            >>> print(rate)
            18.5000
        """

    def fetch_from_fed(
        self,
        from_currency: str,
        to_currency: str,
        rate_date: date
    ) -> Decimal:
        """
        Fetch rate from Federal Reserve API (fallback source).

        Federal Reserve API:
        - Updates daily
        - Covers major currencies only
        - Base currency: USD
        - Requires API key

        Args:
            from_currency: ISO 4217 currency code
            to_currency: ISO 4217 currency code
            rate_date: Date for rate

        Returns:
            Exchange rate (Decimal, 6 decimals)

        Raises:
            FedAPIError: If API call fails

        Example:
            >>> rate = provider.fetch_from_fed("USD", "MXN", date(2024, 11, 1))
            >>> print(rate)
            18.5000
        """

    def create_manual_override(
        self,
        from_currency: str,
        to_currency: str,
        rate: Decimal,
        rate_date: date,
        reason: str,
        user_id: str
    ) -> ExchangeRateRecord:
        """
        Create manual rate override (user-provided).

        Args:
            from_currency: ISO 4217 currency code
            to_currency: ISO 4217 currency code
            rate: Manual exchange rate
            rate_date: Date for rate
            reason: User explanation for override
            user_id: ID of user creating override

        Returns:
            ExchangeRateRecord with source="manual"

        Example:
            >>> rate = provider.create_manual_override(
            ...     "USD", "MXN", Decimal("19.0000"),
            ...     date(2024, 11, 1),
            ...     "Bank charged higher rate",
            ...     "user_123"
            ... )
            >>> print(rate.source)
            "manual"
        """

    def _check_cache(
        self,
        from_currency: str,
        to_currency: str,
        rate_date: date
    ) -> Optional[ExchangeRateRecord]:
        """
        Check cache for existing rate.

        Args:
            from_currency: Currency code
            to_currency: Currency code
            rate_date: Date

        Returns:
            ExchangeRateRecord if found, None otherwise

        Internal method, called by get_rate().
        """

    def _cache_rate(
        self,
        record: ExchangeRateRecord
    ) -> None:
        """
        Store rate in cache.

        Args:
            record: ExchangeRateRecord to cache

        Internal method, called after successful fetch.
        """

    def _calculate_inverse(
        self,
        rate: Decimal
    ) -> Decimal:
        """
        Calculate inverse rate (EUR→USD = 1 / USD→EUR).

        Args:
            rate: Original rate

        Returns:
            Inverse rate (1 / rate)

        Example:
            >>> inverse = provider._calculate_inverse(Decimal("1.20"))
            >>> print(inverse)
            0.833333  # 1 / 1.20
        """
```

---

## Data Structures

### Output Types

```python
@dataclass
class ExchangeRateRecord:
    """
    Exchange rate record with metadata.

    Attributes:
        rate_id: Unique identifier (e.g., "rate_usd_mxn_20241101")
        from_currency: Source currency (ISO 4217)
        to_currency: Target currency (ISO 4217)
        rate: Exchange rate (to/from, 6 decimals)
        rate_date: Date of rate
        source: Rate source ("ecb", "federal_reserve", "manual")
        fetched_at: Timestamp of fetch
        expires_at: Timestamp when rate becomes stale
        is_stale: True if past expiration, False otherwise
        override_reason: Optional reason if manual override
        user_id: Optional user ID if manual override
    """
    rate_id: str
    from_currency: str
    to_currency: str
    rate: Decimal
    rate_date: date
    source: str  # "ecb", "federal_reserve", "manual"
    fetched_at: datetime
    expires_at: datetime
    is_stale: bool
    override_reason: Optional[str] = None
    user_id: Optional[str] = None
```

---

## Behavior Specifications

### Cache Hit (Fresh Rate)

**Scenario:** Rate in cache, fetched <24h ago

```python
provider = ExchangeRateProvider(cache, ecb_url, fed_url, fed_key)

# First call: Cache miss, fetch from ECB
rate1 = provider.get_rate("USD", "MXN", date(2024, 11, 1))
# → Fetches from ECB, caches with 24h TTL

# Second call: Cache hit
rate2 = provider.get_rate("USD", "MXN", date(2024, 11, 1))
# → Returns cached rate, no API call

assert rate1.rate == rate2.rate
assert rate2.source == "ecb"
assert rate2.is_stale == False
```

**Performance:**
- Cache hit: <10ms (Redis lookup)
- Cache miss: ~500ms (API call + cache write)

---

### Cache Miss: Fetch from ECB

**Scenario:** Rate not in cache, fetch from ECB API

```python
rate = provider.get_rate("USD", "EUR", date(2024, 11, 1))

# Expected flow:
# 1. Check cache → MISS
# 2. Fetch from ECB API → SUCCESS
# 3. Cache rate with 24h TTL
# 4. Return rate

assert rate.source == "ecb"
assert rate.rate > 0
```

**ECB API Response Example:**
```json
{
  "success": true,
  "timestamp": 1698796800,
  "base": "EUR",
  "date": "2024-11-01",
  "rates": {
    "USD": 1.0600,
    "MXN": 19.6100,
    "GBP": 0.8700
  }
}
```

**Rate Calculation (USD → MXN):**
- ECB base: EUR
- EUR → USD = 1.0600
- EUR → MXN = 19.6100
- USD → MXN = 19.6100 / 1.0600 = 18.5000

---

### Fallback: ECB Fail → Federal Reserve

**Scenario:** ECB API times out, fallback to Federal Reserve

```python
# Mock ECB failure
with mock.patch.object(provider, 'fetch_from_ecb') as mock_ecb:
    mock_ecb.side_effect = ECBAPIError("Timeout")

    rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))

    # Expected flow:
    # 1. Try ECB → FAIL (timeout)
    # 2. Try Federal Reserve → SUCCESS
    # 3. Cache rate
    # 4. Return rate

    assert rate.source == "federal_reserve"
    assert rate.rate > 0
```

**Federal Reserve API Response Example:**
```json
{
  "observations": [
    {
      "date": "2024-11-01",
      "value": "18.5000"
    }
  ]
}
```

---

### Stale Rate Detection

**Scenario:** Rate fetched >24h ago, marked as stale

```python
# Fetch rate
rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))
# fetched_at: 2024-11-01 08:00:00 UTC
# expires_at: 2024-11-02 08:00:00 UTC (24h later)

# 25 hours later...
# is_stale = NOW() > expires_at = True

assert rate.is_stale == False  # Initially fresh

# Simulate time passing (25 hours)
with freeze_time("2024-11-02 09:00:00"):
    stale_rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))
    assert stale_rate.is_stale == True
```

**Staleness Threshold:**
- Fiat currencies: 24 hours
- Crypto currencies: 1 hour (higher volatility)

---

### Historical Rate Query

**Scenario:** Fetch rates for date range (Oct 1-31, 2024)

```python
rates = provider.get_historical(
    "USD", "EUR",
    start_date=date(2024, 10, 1),
    end_date=date(2024, 10, 31)
)

assert len(rates) == 31  # One rate per day
assert all(r.source == "ecb" for r in rates)
```

**Use Cases:**
- Historical analysis (trend charting)
- Retroactive conversions (import old transactions)
- Audit trail (what rate was used when)

---

### Manual Rate Override

**Scenario:** User provides custom rate (bank charged different rate)

```python
manual_rate = provider.create_manual_override(
    from_currency="USD",
    to_currency="MXN",
    rate=Decimal("19.0000"),  # System rate: 18.5000 (2.7% variance)
    rate_date=date(2024, 11, 1),
    reason="Bank charged higher rate due to foreign transaction fee",
    user_id="user_123"
)

assert manual_rate.source == "manual"
assert manual_rate.override_reason == "Bank charged higher rate..."
assert manual_rate.user_id == "user_123"

# Variance logged for audit
variance = abs(19.0000 - 18.5000) / 18.5000 * 100
# → 2.7%
```

**Variance Validation:**
- If variance <10%, allow automatically
- If variance ≥10%, require user confirmation
- Log all manual overrides for audit

---

### Cross-Rate Calculation (Indirect Conversion)

**Scenario:** No direct rate for MXN→JPY, calculate via USD

```python
# MXN → USD → JPY (two-step conversion)

# Step 1: MXN → USD
rate_mxn_usd = provider.get_rate("MXN", "USD", date(2024, 11, 1))
# → 0.0541 (1 / 18.5000)

# Step 2: USD → JPY
rate_usd_jpy = provider.get_rate("USD", "JPY", date(2024, 11, 1))
# → 150.0000

# Cross-rate: MXN → JPY
rate_mxn_jpy = rate_mxn_usd * rate_usd_jpy
# → 0.0541 × 150 = 8.1150
```

**When to Use:**
- Exotic currency pairs (MXN ↔ KRW, THB ↔ AUD)
- ECB/Fed don't provide direct rates

---

## Error Handling

### RateNotAvailableError

**Trigger:** All sources fail (ECB + Federal Reserve down)

```python
try:
    rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))
except RateNotAvailableError as e:
    print(e)
    # "Exchange rate for USD→MXN on 2024-11-01 not available.
    #  All sources failed. Please enter manually."
```

**Causes:**
- ECB API timeout/error
- Federal Reserve API timeout/error
- Network connectivity issue
- Currency pair not supported by any source

---

### ECBAPIError

**Trigger:** ECB API call fails

```python
try:
    rate = provider.fetch_from_ecb("USD", "MXN", date(2024, 11, 1))
except ECBAPIError as e:
    print(e)
    # "ECB API error: Request timeout after 5 seconds"
```

**Retry Logic:**
- Retry 3 times with exponential backoff (1s, 2s, 4s)
- If all retries fail, fallback to Federal Reserve

---

### FedAPIError

**Trigger:** Federal Reserve API call fails

```python
try:
    rate = provider.fetch_from_fed("USD", "MXN", date(2024, 11, 1))
except FedAPIError as e:
    print(e)
    # "Federal Reserve API error: Invalid API key"
```

---

## Performance Characteristics

**Latency Targets (p95):**
- `get_rate()` (cache hit): <10ms
- `get_rate()` (cache miss, ECB): <500ms
- `get_rate()` (cache miss, Fed fallback): <1000ms
- `get_historical()` (30 days): <2000ms (batch fetch)

**Throughput:**
- Cache hits: ~10,000 requests/second
- Cache misses: ~100 requests/second (API rate limit)

**Cache Strategy:**
- Key: `rate_{from}_{to}_{date}` (e.g., `rate_usd_mxn_20241101`)
- TTL: 24 hours (fiat), 1 hour (crypto)
- Storage: Redis (distributed, fast)
- Eviction: LRU (Least Recently Used)

---

## Dependencies

**Internal:**
- `CacheClient` (Redis)
- `HTTPClient` (requests library)
- `Decimal` (Python standard library)

**External APIs:**
- **ECB API:** https://api.exchangerate.host/latest (free, no API key)
- **Federal Reserve API:** https://api.federalreserve.gov/data (API key required)

**Data:**
- ISO 4217 currency codes (~50 KB)

---

## Testing Strategy

### Unit Tests

```python
def test_get_rate_cache_hit():
    """Test rate retrieval from cache."""
    mock_cache = Mock()
    mock_cache.get.return_value = {
        "rate": 18.5000,
        "source": "ecb",
        "fetched_at": "2024-11-01T08:00:00Z"
    }

    provider = ExchangeRateProvider(mock_cache, ecb_url, fed_url, fed_key)
    rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))

    assert rate.rate == Decimal("18.5000")
    assert rate.source == "ecb"
    assert rate.is_stale == False
    mock_cache.get.assert_called_once()

def test_get_rate_cache_miss_fetch_ecb():
    """Test rate fetch from ECB on cache miss."""
    mock_cache = Mock()
    mock_cache.get.return_value = None  # Cache miss

    provider = ExchangeRateProvider(mock_cache, ecb_url, fed_url, fed_key)

    with mock.patch.object(provider, 'fetch_from_ecb') as mock_ecb:
        mock_ecb.return_value = Decimal("18.5000")

        rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))

        assert rate.rate == Decimal("18.5000")
        assert rate.source == "ecb"
        mock_ecb.assert_called_once()
        mock_cache.set.assert_called_once()  # Rate cached

def test_get_rate_ecb_fail_fallback_fed():
    """Test fallback to Federal Reserve on ECB failure."""
    mock_cache = Mock()
    mock_cache.get.return_value = None

    provider = ExchangeRateProvider(mock_cache, ecb_url, fed_url, fed_key)

    with mock.patch.object(provider, 'fetch_from_ecb') as mock_ecb, \
         mock.patch.object(provider, 'fetch_from_fed') as mock_fed:

        mock_ecb.side_effect = ECBAPIError("Timeout")
        mock_fed.return_value = Decimal("18.6000")

        rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))

        assert rate.source == "federal_reserve"
        assert rate.rate == Decimal("18.6000")

def test_refresh_cache():
    """Test forced cache refresh."""
    mock_cache = Mock()
    provider = ExchangeRateProvider(mock_cache, ecb_url, fed_url, fed_key)

    with mock.patch.object(provider, 'fetch_from_ecb') as mock_ecb:
        mock_ecb.return_value = Decimal("18.7500")

        rate = provider.refresh_cache("USD", "MXN")

        assert rate.rate == Decimal("18.7500")
        mock_cache.delete.assert_called_once()  # Cache invalidated
        mock_cache.set.assert_called_once()  # New rate cached

def test_get_historical():
    """Test historical rate query."""
    provider = ExchangeRateProvider(mock_cache, ecb_url, fed_url, fed_key)

    with mock.patch.object(provider, 'fetch_from_ecb') as mock_ecb:
        mock_ecb.return_value = Decimal("18.5000")

        rates = provider.get_historical(
            "USD", "EUR",
            date(2024, 10, 1),
            date(2024, 10, 31)
        )

        assert len(rates) == 31
        assert all(r.source == "ecb" for r in rates)

def test_create_manual_override():
    """Test manual rate override."""
    provider = ExchangeRateProvider(mock_cache, ecb_url, fed_url, fed_key)

    rate = provider.create_manual_override(
        "USD", "MXN", Decimal("19.0000"),
        date(2024, 11, 1),
        "Bank charged higher rate",
        "user_123"
    )

    assert rate.source == "manual"
    assert rate.override_reason == "Bank charged higher rate"
    assert rate.user_id == "user_123"

def test_stale_rate_detection():
    """Test stale rate marking."""
    provider = ExchangeRateProvider(
        mock_cache, ecb_url, fed_url, fed_key,
        staleness_threshold=timedelta(hours=24)
    )

    # Fetch rate
    rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))
    assert rate.is_stale == False

    # Simulate 25 hours passing
    with freeze_time(rate.expires_at + timedelta(hours=1)):
        stale_rate = provider.get_rate("USD", "MXN", date(2024, 11, 1))
        assert stale_rate.is_stale == True
```

### Integration Tests

```python
def test_end_to_end_ecb_fetch():
    """Test actual ECB API call (uses test API key)."""
    provider = ExchangeRateProvider(
        test_cache,
        ecb_api_url="https://api.exchangerate.host/latest",
        fed_api_url=test_fed_url,
        fed_api_key=test_fed_key
    )

    rate = provider.get_rate("USD", "EUR", date.today())

    assert rate.rate > 0.5  # Sanity check (EUR/USD ~ 0.85-1.0)
    assert rate.source == "ecb"
    assert rate.is_stale == False
```

---

## Multi-Domain Examples

### Finance: Daily Rate Refresh

**Use Case:** Fetch rates for popular pairs every morning

```python
provider = ExchangeRateProvider(cache, ecb_url, fed_url, fed_key)

# Popular pairs
pairs = [("USD", "EUR"), ("USD", "MXN"), ("USD", "GBP"), ("EUR", "GBP")]

for from_curr, to_curr in pairs:
    rate = provider.get_rate(from_curr, to_curr, date.today())
    print(f"{from_curr}/{to_curr}: {rate.rate}")
```

---

### Healthcare: Historical Claim Conversion

**Use Case:** Convert international claims using historical rates

```python
# Claim from 6 months ago
claim_date = date(2024, 5, 1)
claim_eur = Decimal("5000.00")

rate = provider.get_rate("EUR", "USD", claim_date)
claim_usd = claim_eur * rate.rate

print(f"Claim: €{claim_eur} = ${claim_usd} (rate: {rate.rate})")
```

---

### E-commerce: Real-Time Pricing

**Use Case:** Display product price in customer's currency

```python
# Product priced in USD
product_price_usd = Decimal("99.99")

# Customer's currency: EUR
rate = provider.get_rate("USD", "EUR", date.today())
product_price_eur = product_price_usd * rate.rate

print(f"Price: ${product_price_usd} USD = €{product_price_eur} EUR")
```

---

## Observability

### Metrics

```python
# Fetch metrics
exchange_rates.fetched.count (counter, labels: source, from_currency, to_currency)
exchange_rates.cache_hit.count (counter)
exchange_rates.cache_miss.count (counter)
exchange_rates.fetch.latency (histogram, labels: source)

# Error metrics
exchange_rates.errors.ecb_api (counter)
exchange_rates.errors.fed_api (counter)
exchange_rates.errors.rate_not_available (counter)

# Staleness metrics
exchange_rates.stale.count (gauge)
exchange_rates.manual_overrides.count (counter)
```

### Structured Logs

```json
{
  "timestamp": "2024-11-01T08:00:00Z",
  "level": "INFO",
  "event": "exchange_rate_fetched",
  "from_currency": "USD",
  "to_currency": "MXN",
  "rate": 18.5000,
  "source": "ecb",
  "latency_ms": 345,
  "cache_miss": true
}
```

---

## Security Considerations

**API Key Protection:**
- Federal Reserve API key stored in environment variable (not code)
- Never log API keys
- Rotate API keys quarterly

**Rate Limiting:**
- ECB: 1000 requests/day (free tier)
- Federal Reserve: 10,000 requests/day
- Implement circuit breaker (disable source after 3 failures)

**Input Validation:**
- Currency codes: Must be ISO 4217 (prevent injection)
- Dates: Must be valid dates, not in future (prevent manipulation)

---

## Configuration

```yaml
# config/exchange_rate_provider.yaml
ecb_api_url: https://api.exchangerate.host/latest
fed_api_url: https://api.federalreserve.gov/data
fed_api_key: ${FED_API_KEY}  # From environment

staleness_threshold:
  fiat: 24h  # 24 hours
  crypto: 1h  # 1 hour

cache:
  redis_url: redis://localhost:6379/0
  ttl:
    fiat: 86400  # 24 hours in seconds
    crypto: 3600  # 1 hour in seconds

retry:
  max_retries: 3
  backoff: exponential  # 1s, 2s, 4s
  timeout: 5s
```

---

## Changelog

**v1.0.0 (2024-11-01):**
- Initial implementation
- ECB + Federal Reserve multi-source
- 24h cache with staleness detection
- Historical rate support

**v1.1.0 (Future):**
- Real-time crypto rates (<1 minute refresh)
- Additional sources (Bloomberg, Reuters)
- Cross-rate calculation for exotic pairs

---

## References

- ECB Exchange Rate API: https://api.exchangerate.host/
- Federal Reserve API: https://www.federalreserve.gov/data.htm
- ISO 4217 Currency Codes: https://www.iso.org/iso-4217-currency-codes.html

---

**Total Lines:** ~1,080
