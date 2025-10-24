# OL Primitive: FXConverter

**Type**: Calculation / Currency Conversion
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 3.5 (Relationships)

---

## Purpose

Calculate exchange rates from transaction amounts, fetch historical market rates, and track FX gain/loss for multi-currency operations. Enables accurate currency conversion tracking with optional variance analysis against market rates.

**Core Responsibility:** Derive exchange rates from paired transactions (calculated source), optionally compare to external market rates (API source), and calculate FX gain/loss variance. Supports multi-currency analytics and historical rate preservation.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. The pattern of **Currency/Unit Conversion + Variance Tracking** applies universally:

| Domain | Use Case | From | To | Variance Tracking |
|--------|----------|------|-----|-------------------|
| **Finance** | FX conversion | $1,000 USD | $18,500 MXN | Market rate vs actual rate |
| **Healthcare** | International billing | $500 USD | €450 EUR | Standard rate vs actual rate |
| **Research** | Grant conversions | £10,000 GBP | $13,200 USD | Budget rate vs actual rate |
| **Manufacturing** | Unit conversion | 100 kg | 220.46 lbs | Standard factor vs measured |
| **Energy** | Currency + units | $100/MWh | €85/MWh | Exchange rate variance |
| **Legal** | Cross-border payments | ¥500,000 JPY | $4,500 USD | Contracted rate vs actual |
| **Supply Chain** | Multi-currency invoicing | CHF 1,000 | $1,100 USD | Spot rate vs invoice rate |
| **Real Estate** | International transactions | CAD 500,000 | USD 370,000 | Appraisal rate vs closing rate |

**Pattern:** Convert value from one unit/currency to another, calculate conversion rate, optionally compare to reference rate, track variance for gain/loss analysis.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, Dict, Any
from dataclasses import dataclass
from datetime import date
from decimal import Decimal

@dataclass
class ExchangeRate:
    """Represents a calculated or fetched exchange rate."""
    from_currency: str  # ISO 4217 code (e.g., "USD")
    to_currency: str    # ISO 4217 code (e.g., "MXN")
    rate: Decimal       # Exchange rate (4 decimal places)
    source: str         # "calculated", "api", "manual"
    date: Optional[date] = None
    provider: Optional[str] = None  # API provider if source="api"

@dataclass
class FXDetails:
    """FX conversion details stored in relationship.fx_details JSONB."""
    from_currency: str
    to_currency: str
    from_amount: Decimal
    to_amount: Decimal
    exchange_rate: Decimal
    rate_source: str  # "calculated", "api", "manual"
    market_rate: Optional[Decimal] = None
    fx_gain_loss: Optional[Decimal] = None
    fx_gain_loss_pct: Optional[Decimal] = None
    calculation_date: Optional[date] = None

@dataclass
class FXGainLoss:
    """FX gain/loss calculation result."""
    expected_amount: Decimal  # What you should have received at market rate
    actual_amount: Decimal    # What you actually received
    gain_loss: Decimal        # Difference (positive = gain, negative = loss)
    gain_loss_pct: Decimal    # Percentage variance
    market_rate: Decimal      # Market rate used for comparison
    actual_rate: Decimal      # Actual rate from transaction

class FXConverter:
    """
    Calculate exchange rates and track FX conversions.

    Responsibilities:
    - Calculate rate from transaction amounts (primary source)
    - Fetch historical market rates from external API (optional)
    - Calculate FX gain/loss (actual vs market rate)
    - Support multi-currency analytics
    - Preserve historical rates for audit trail
    """

    def __init__(self, market_rate_provider: Optional['MarketRateProvider'] = None):
        """
        Initialize FXConverter.

        Args:
            market_rate_provider: Optional external API provider for market rates
        """
        self.market_rate_provider = market_rate_provider

    def calculate_rate(
        self,
        from_amount: Decimal,
        from_currency: str,
        to_amount: Decimal,
        to_currency: str
    ) -> ExchangeRate:
        """
        Calculate exchange rate from transaction amounts.

        Primary method for deriving exchange rate from observed data.

        Args:
            from_amount: Source amount (e.g., 1000.00)
            from_currency: Source currency ISO code (e.g., "USD")
            to_amount: Destination amount (e.g., 18500.00)
            to_currency: Destination currency ISO code (e.g., "MXN")

        Returns:
            ExchangeRate with calculated rate

        Raises:
            ValueError: If from_amount is zero
            ValueError: If currencies are invalid ISO 4217 codes
            ValueError: If amounts have wrong sign (both positive or both negative)

        Formula:
            rate = |to_amount| / |from_amount|

        Example:
            rate = converter.calculate_rate(
                from_amount=Decimal("1000.00"),
                from_currency="USD",
                to_amount=Decimal("18500.00"),
                to_currency="MXN"
            )
            # Returns: ExchangeRate(rate=18.5000, source="calculated")
        """

    def get_market_rate(
        self,
        conversion_date: date,
        from_currency: str,
        to_currency: str
    ) -> Optional[ExchangeRate]:
        """
        Fetch historical market rate from external API.

        Optional method for comparing actual rate to market rate.

        Args:
            conversion_date: Date of conversion
            from_currency: Source currency ISO code
            to_currency: Destination currency ISO code

        Returns:
            ExchangeRate with market rate, or None if unavailable

        Raises:
            APIError: If external API request fails
            ValueError: If currencies are invalid

        Example:
            market_rate = converter.get_market_rate(
                conversion_date=date(2025, 10, 16),
                from_currency="USD",
                to_currency="MXN"
            )
            # Returns: ExchangeRate(rate=18.3000, source="api", provider="exchangerate.host")
        """

    def calculate_fx_gain_loss(
        self,
        actual_rate: Decimal,
        market_rate: Decimal,
        from_amount: Decimal,
        from_currency: str,
        to_currency: str
    ) -> FXGainLoss:
        """
        Calculate FX gain/loss by comparing actual rate to market rate.

        Determines whether conversion was favorable (gain) or unfavorable (loss).

        Args:
            actual_rate: Rate actually received from transaction
            market_rate: Market rate on conversion date
            from_amount: Source amount converted
            from_currency: Source currency
            to_currency: Destination currency

        Returns:
            FXGainLoss with variance calculation

        Formula:
            expected_amount = |from_amount| * market_rate
            actual_amount = |from_amount| * actual_rate
            gain_loss = actual_amount - expected_amount
            gain_loss_pct = (actual_rate - market_rate) / market_rate * 100

        Example:
            gain_loss = converter.calculate_fx_gain_loss(
                actual_rate=Decimal("18.5"),
                market_rate=Decimal("18.3"),
                from_amount=Decimal("1000.00"),
                from_currency="USD",
                to_currency="MXN"
            )
            # Returns: FXGainLoss(
            #   expected_amount=18300.00,
            #   actual_amount=18500.00,
            #   gain_loss=+200.00,  # Gained 200 MXN
            #   gain_loss_pct=+1.09%
            # )
        """

    def build_fx_details(
        self,
        from_amount: Decimal,
        from_currency: str,
        to_amount: Decimal,
        to_currency: str,
        conversion_date: Optional[date] = None,
        include_market_comparison: bool = True
    ) -> FXDetails:
        """
        Build complete FX details for storage in relationship.fx_details.

        Convenience method that combines rate calculation, market rate lookup,
        and gain/loss calculation.

        Args:
            from_amount: Source amount
            from_currency: Source currency
            to_amount: Destination amount
            to_currency: Destination currency
            conversion_date: Date of conversion (for market rate lookup)
            include_market_comparison: Whether to fetch and compare market rate

        Returns:
            FXDetails ready for JSONB storage

        Example:
            fx_details = converter.build_fx_details(
                from_amount=Decimal("1000.00"),
                from_currency="USD",
                to_amount=Decimal("18500.00"),
                to_currency="MXN",
                conversion_date=date(2025, 10, 16),
                include_market_comparison=True
            )
            # Returns: FXDetails with all fields populated
        """

    def convert_amount(
        self,
        amount: Decimal,
        from_currency: str,
        to_currency: str,
        rate: Decimal
    ) -> Decimal:
        """
        Convert amount using given exchange rate.

        Utility method for applying rate to amount.

        Args:
            amount: Amount to convert
            from_currency: Source currency
            to_currency: Destination currency
            rate: Exchange rate to apply

        Returns:
            Converted amount (2 decimal places for currencies)

        Example:
            converted = converter.convert_amount(
                amount=Decimal("1000.00"),
                from_currency="USD",
                to_currency="MXN",
                rate=Decimal("18.5")
            )
            # Returns: Decimal("18500.00")
        """

    def validate_currency_code(self, code: str) -> bool:
        """
        Validate ISO 4217 currency code.

        Args:
            code: Currency code to validate (e.g., "USD", "MXN")

        Returns:
            True if valid ISO 4217 code, False otherwise

        Example:
            assert converter.validate_currency_code("USD") == True
            assert converter.validate_currency_code("XXX") == False
        """
```

---

## Implementation Details

### Rate Calculation Algorithm

```python
def calculate_rate(
    self,
    from_amount: Decimal,
    from_currency: str,
    to_amount: Decimal,
    to_currency: str
) -> ExchangeRate:
    """Calculate exchange rate from transaction amounts."""

    # Validation
    if from_amount == 0:
        raise ValueError("From amount cannot be zero")

    if not self.validate_currency_code(from_currency):
        raise ValueError(f"Invalid currency code: {from_currency}")

    if not self.validate_currency_code(to_currency):
        raise ValueError(f"Invalid currency code: {to_currency}")

    # Both amounts should have opposite signs (one negative, one positive)
    # or both should be absolute values
    if (from_amount > 0 and to_amount < 0) or (from_amount < 0 and to_amount > 0):
        # This is correct for paired transactions
        pass
    elif from_amount * to_amount < 0:
        # One positive, one negative - warning but allow
        logger.warning(
            f"Unusual sign combination for FX conversion: "
            f"{from_amount} {from_currency} -> {to_amount} {to_currency}"
        )

    # Calculate rate using absolute values
    rate = abs(to_amount) / abs(from_amount)

    # Round to 4 decimal places (standard FX precision)
    rate = rate.quantize(Decimal('0.0001'), rounding=ROUND_HALF_UP)

    return ExchangeRate(
        from_currency=from_currency,
        to_currency=to_currency,
        rate=rate,
        source="calculated"
    )
```

**Precision Handling:**
- Exchange rates: 4 decimal places (e.g., 18.5000 MXN/USD)
- Currency amounts: 2 decimal places (e.g., 1000.00 USD)
- Gain/loss percentages: 2 decimal places (e.g., 1.09%)

**Rate Direction:**
- Rate represents "units of to_currency per unit of from_currency"
- Example: 18.5 MXN/USD means 1 USD = 18.5 MXN
- Inverse rate: 1/18.5 = 0.0541 USD/MXN

---

### Market Rate Integration

```python
from abc import ABC, abstractmethod
from typing import Optional

class MarketRateProvider(ABC):
    """Abstract interface for market rate providers."""

    @abstractmethod
    def get_rate(
        self,
        date: date,
        from_currency: str,
        to_currency: str
    ) -> Optional[Decimal]:
        """Fetch historical exchange rate."""
        pass

class ExchangeRateHostProvider(MarketRateProvider):
    """Implementation using exchangerate.host API."""

    def __init__(self, api_key: Optional[str] = None):
        self.api_key = api_key
        self.base_url = "https://api.exchangerate.host"
        self.cache_ttl = 86400  # 24 hours

    def get_rate(
        self,
        date: date,
        from_currency: str,
        to_currency: str
    ) -> Optional[Decimal]:
        """
        Fetch rate from exchangerate.host.

        API: GET /convert?from={from}&to={to}&date={date}&amount=1
        """

        # Check cache first
        cache_key = f"fx_rate:{from_currency}:{to_currency}:{date.isoformat()}"
        cached = cache.get(cache_key)
        if cached:
            return Decimal(cached)

        try:
            response = requests.get(
                f"{self.base_url}/convert",
                params={
                    "from": from_currency,
                    "to": to_currency,
                    "date": date.isoformat(),
                    "amount": "1"
                },
                timeout=5
            )
            response.raise_for_status()

            data = response.json()
            if data.get("success"):
                rate = Decimal(str(data["result"]))
                rate = rate.quantize(Decimal('0.0001'))

                # Cache result
                cache.set(cache_key, str(rate), ttl=self.cache_ttl)

                return rate
            else:
                logger.warning(
                    f"Market rate API returned error: {data.get('error')}"
                )
                return None

        except requests.RequestException as e:
            logger.error(f"Failed to fetch market rate: {e}")
            return None

        except (KeyError, ValueError) as e:
            logger.error(f"Failed to parse market rate response: {e}")
            return None

class FallbackRateProvider(MarketRateProvider):
    """Chain multiple providers with fallback."""

    def __init__(self, providers: list[MarketRateProvider]):
        self.providers = providers

    def get_rate(
        self,
        date: date,
        from_currency: str,
        to_currency: str
    ) -> Optional[Decimal]:
        """Try providers in order until one succeeds."""

        for provider in self.providers:
            try:
                rate = provider.get_rate(date, from_currency, to_currency)
                if rate is not None:
                    return rate
            except Exception as e:
                logger.warning(
                    f"Provider {provider.__class__.__name__} failed: {e}"
                )
                continue

        logger.error(
            f"All market rate providers failed for "
            f"{from_currency}/{to_currency} on {date}"
        )
        return None
```

**API Providers (in order of preference):**

1. **exchangerate.host** (Free, reliable, historical data)
2. **fixer.io** (Backup, requires API key)
3. **currencylayer.com** (Fallback, rate limits)

**Caching Strategy:**
- Cache historical rates for 24 hours (they don't change)
- No cache for real-time rates (use with caution)
- Store in Redis with key: `fx_rate:{from}:{to}:{date}`

**Graceful Degradation:**
- If all APIs fail, continue without market comparison
- Store `market_rate: null` in fx_details
- User sees calculated rate only (no gain/loss)

---

### FX Gain/Loss Calculation

```python
def calculate_fx_gain_loss(
    self,
    actual_rate: Decimal,
    market_rate: Decimal,
    from_amount: Decimal,
    from_currency: str,
    to_currency: str
) -> FXGainLoss:
    """Calculate FX gain/loss variance."""

    # Use absolute amount for calculation
    abs_from_amount = abs(from_amount)

    # Calculate expected amount at market rate
    expected_amount = abs_from_amount * market_rate
    expected_amount = expected_amount.quantize(Decimal('0.01'))

    # Calculate actual amount at actual rate
    actual_amount = abs_from_amount * actual_rate
    actual_amount = actual_amount.quantize(Decimal('0.01'))

    # Calculate gain/loss
    gain_loss = actual_amount - expected_amount
    gain_loss = gain_loss.quantize(Decimal('0.01'))

    # Calculate percentage variance
    rate_diff = actual_rate - market_rate
    gain_loss_pct = (rate_diff / market_rate) * 100
    gain_loss_pct = gain_loss_pct.quantize(Decimal('0.01'))

    return FXGainLoss(
        expected_amount=expected_amount,
        actual_amount=actual_amount,
        gain_loss=gain_loss,
        gain_loss_pct=gain_loss_pct,
        market_rate=market_rate,
        actual_rate=actual_rate
    )
```

**Example Scenarios:**

**Scenario 1: Favorable Rate (Gain)**
```
Convert: $1,000 USD → MXN
Actual rate:  18.5000 MXN/USD
Market rate:  18.3000 MXN/USD

Expected:  1,000 * 18.3 = 18,300.00 MXN
Actual:    1,000 * 18.5 = 18,500.00 MXN
Gain:      +200.00 MXN
Gain %:    +1.09%

Interpretation: You got a better rate than market (saved ~$10.93 USD)
```

**Scenario 2: Unfavorable Rate (Loss)**
```
Convert: $1,000 USD → MXN
Actual rate:  18.0000 MXN/USD
Market rate:  18.3000 MXN/USD

Expected:  1,000 * 18.3 = 18,300.00 MXN
Actual:    1,000 * 18.0 = 18,000.00 MXN
Loss:      -300.00 MXN
Loss %:    -1.64%

Interpretation: You got a worse rate than market (lost ~$16.39 USD)
```

**Scenario 3: No Market Rate Available**
```
Convert: $1,000 USD → MXN
Actual rate:  18.5000 MXN/USD
Market rate:  null (API failed)

Result: Store actual rate only, no gain/loss calculation
UI shows: "Exchange rate: 18.5000 MXN/USD (market rate unavailable)"
```

---

### Multi-Currency Support

**ISO 4217 Currency Codes:**

```python
# Supported currency codes (ISO 4217)
SUPPORTED_CURRENCIES = {
    # Major currencies
    "USD": "US Dollar",
    "EUR": "Euro",
    "GBP": "British Pound",
    "JPY": "Japanese Yen",
    "CHF": "Swiss Franc",
    "CAD": "Canadian Dollar",
    "AUD": "Australian Dollar",
    "NZD": "New Zealand Dollar",

    # Latin America
    "MXN": "Mexican Peso",
    "BRL": "Brazilian Real",
    "ARS": "Argentine Peso",
    "CLP": "Chilean Peso",
    "COP": "Colombian Peso",
    "PEN": "Peruvian Sol",

    # Asia
    "CNY": "Chinese Yuan",
    "INR": "Indian Rupee",
    "KRW": "South Korean Won",
    "SGD": "Singapore Dollar",
    "HKD": "Hong Kong Dollar",
    "THB": "Thai Baht",

    # Europe
    "PLN": "Polish Zloty",
    "SEK": "Swedish Krona",
    "NOK": "Norwegian Krone",
    "DKK": "Danish Krone",
    "CZK": "Czech Koruna",
    "HUF": "Hungarian Forint",

    # Middle East / Africa
    "AED": "UAE Dirham",
    "SAR": "Saudi Riyal",
    "ZAR": "South African Rand",
    "EGP": "Egyptian Pound",
    "ILS": "Israeli Shekel",

    # Crypto (experimental)
    "BTC": "Bitcoin",
    "ETH": "Ethereum",
    "USDT": "Tether"
}

def validate_currency_code(self, code: str) -> bool:
    """Validate currency code against ISO 4217."""
    return code.upper() in SUPPORTED_CURRENCIES
```

**Currency Pair Handling:**

Some currency pairs have special considerations:

| Pair | Notes | Precision |
|------|-------|-----------|
| USD/MXN | High volume, stable APIs | 4 decimals |
| USD/JPY | No decimals in JPY amounts | 2 decimals for rate |
| USD/EUR | Most liquid pair | 4 decimals |
| BTC/USD | High volatility, 8 decimal precision | 8 decimals |
| USD/CLP | Large numbers (1 USD ≈ 900 CLP) | 2 decimals |

**Handling Zero-Decimal Currencies:**

```python
# Currencies with no decimal places
ZERO_DECIMAL_CURRENCIES = {"JPY", "KRW", "VND", "CLP", "ISK"}

def format_amount(self, amount: Decimal, currency: str) -> str:
    """Format amount according to currency conventions."""

    if currency in ZERO_DECIMAL_CURRENCIES:
        return f"{amount.quantize(Decimal('1'))} {currency}"
    else:
        return f"{amount.quantize(Decimal('0.01'))} {currency}"

# Example:
# format_amount(Decimal("100000"), "JPY") -> "100000 JPY"
# format_amount(Decimal("100000"), "USD") -> "100000.00 USD"
```

---

## Storage Format

**JSONB Structure for relationship.fx_details:**

```json
{
  "from_currency": "USD",
  "to_currency": "MXN",
  "from_amount": 1000.00,
  "to_amount": 18500.00,
  "exchange_rate": 18.5000,
  "rate_source": "calculated",
  "market_rate": 18.3000,
  "fx_gain_loss": 200.00,
  "fx_gain_loss_pct": 1.09,
  "calculation_date": "2025-10-16",
  "market_rate_provider": "exchangerate.host",
  "market_rate_fetched_at": "2025-10-16T09:15:00Z"
}
```

**Field Definitions:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from_currency` | string | Yes | ISO 4217 code |
| `to_currency` | string | Yes | ISO 4217 code |
| `from_amount` | decimal | Yes | Source amount (absolute value) |
| `to_amount` | decimal | Yes | Destination amount (absolute value) |
| `exchange_rate` | decimal | Yes | Calculated rate (4 decimals) |
| `rate_source` | string | Yes | "calculated", "api", "manual" |
| `market_rate` | decimal | No | Market rate if available |
| `fx_gain_loss` | decimal | No | Gain/loss in to_currency |
| `fx_gain_loss_pct` | decimal | No | Gain/loss percentage |
| `calculation_date` | string | No | ISO date for market rate lookup |
| `market_rate_provider` | string | No | API provider name |
| `market_rate_fetched_at` | string | No | ISO timestamp |

**Querying FX Details:**

```sql
-- Find all conversions with positive FX gain
SELECT
    r.relationship_id,
    r.txn_1_id,
    r.txn_2_id,
    r.fx_details->>'from_currency' as from_currency,
    r.fx_details->>'to_currency' as to_currency,
    (r.fx_details->>'fx_gain_loss')::decimal as gain_loss
FROM relationships r
WHERE r.type = 'fx_conversion'
  AND r.deleted_at IS NULL
  AND (r.fx_details->>'fx_gain_loss')::decimal > 0
ORDER BY (r.fx_details->>'fx_gain_loss')::decimal DESC;

-- Find conversions with rate above market
SELECT
    r.relationship_id,
    r.fx_details->>'exchange_rate' as actual_rate,
    r.fx_details->>'market_rate' as market_rate,
    (r.fx_details->>'exchange_rate')::decimal - (r.fx_details->>'market_rate')::decimal as rate_diff
FROM relationships r
WHERE r.type = 'fx_conversion'
  AND r.deleted_at IS NULL
  AND r.fx_details->>'market_rate' IS NOT NULL
  AND (r.fx_details->>'exchange_rate')::decimal > (r.fx_details->>'market_rate')::decimal;
```

---

## Error Handling

### Validation Errors

```python
class FXConverterError(Exception):
    """Base exception for FXConverter errors."""
    pass

class InvalidCurrencyError(FXConverterError):
    """Invalid ISO 4217 currency code."""
    pass

class ZeroAmountError(FXConverterError):
    """Cannot calculate rate from zero amount."""
    pass

class MarketRateUnavailableError(FXConverterError):
    """Market rate API failed or data unavailable."""
    pass

class RateCalculationError(FXConverterError):
    """Error calculating exchange rate."""
    pass

# Usage:
try:
    rate = converter.calculate_rate(
        from_amount=Decimal("0"),
        from_currency="USD",
        to_amount=Decimal("18500"),
        to_currency="MXN"
    )
except ZeroAmountError:
    logger.error("Cannot calculate rate from zero amount")
    # Fallback: prompt user for manual rate entry
```

### API Failure Handling

```python
def get_market_rate_safe(
    self,
    conversion_date: date,
    from_currency: str,
    to_currency: str
) -> Optional[ExchangeRate]:
    """
    Fetch market rate with comprehensive error handling.

    Returns None if ANY step fails (API down, invalid response, etc.)
    """
    try:
        if not self.market_rate_provider:
            logger.info("No market rate provider configured")
            return None

        rate = self.market_rate_provider.get_rate(
            conversion_date,
            from_currency,
            to_currency
        )

        if rate is None:
            logger.warning(
                f"Market rate unavailable for {from_currency}/{to_currency} "
                f"on {conversion_date}"
            )
            return None

        return ExchangeRate(
            from_currency=from_currency,
            to_currency=to_currency,
            rate=rate,
            source="api",
            date=conversion_date,
            provider=self.market_rate_provider.__class__.__name__
        )

    except Exception as e:
        logger.error(
            f"Failed to fetch market rate for {from_currency}/{to_currency}: {e}",
            exc_info=True
        )
        return None
```

**Error Scenarios:**

| Scenario | Behavior | User Impact |
|----------|----------|-------------|
| Zero from_amount | Raise ZeroAmountError | Show error, prompt manual rate |
| Invalid currency | Raise InvalidCurrencyError | Show error, suggest valid codes |
| API timeout | Return None | Continue without market rate |
| API rate limit | Return None | Continue without market rate |
| Malformed API response | Return None | Continue without market rate |
| Network error | Return None | Continue without market rate |

**Logging Strategy:**

```python
# INFO: Normal operations
logger.info(
    f"Calculated FX rate: {rate.rate} {rate.from_currency}/{rate.to_currency}"
)

# WARNING: Non-critical issues
logger.warning(
    f"Market rate unavailable, continuing with calculated rate only"
)

# ERROR: Critical issues (but non-blocking)
logger.error(
    f"Failed to fetch market rate: {e}",
    extra={
        "from_currency": from_currency,
        "to_currency": to_currency,
        "date": conversion_date,
        "provider": provider_name
    }
)
```

---

## Multi-Domain Examples

### Finance: FX Conversion Tracking

```python
# User converts USD to MXN in Wise
txn_1 = Transaction(
    txn_id="txn_001",
    amount=Decimal("-1000.00"),
    currency="USD",
    date=date(2025, 10, 16)
)

txn_2 = Transaction(
    txn_id="txn_002",
    amount=Decimal("18500.00"),
    currency="MXN",
    date=date(2025, 10, 16)
)

# Calculate FX details
converter = FXConverter(market_rate_provider=ExchangeRateHostProvider())

fx_details = converter.build_fx_details(
    from_amount=txn_1.amount,
    from_currency=txn_1.currency,
    to_amount=txn_2.amount,
    to_currency=txn_2.currency,
    conversion_date=txn_1.date,
    include_market_comparison=True
)

# Store in relationship
relationship = RelationshipStore.create(
    txn_1_id=txn_1.txn_id,
    txn_2_id=txn_2.txn_id,
    type="fx_conversion",
    fx_details=asdict(fx_details)
)

# Result:
# fx_details = {
#   "exchange_rate": 18.5000,
#   "market_rate": 18.3000,
#   "fx_gain_loss": 200.00,
#   "fx_gain_loss_pct": 1.09
# }
```

### Healthcare: International Billing

```python
# Medical procedure billed in USD, patient pays in EUR
procedure = BillingRecord(
    amount=Decimal("500.00"),
    currency="USD",
    date=date(2025, 10, 20)
)

payment = PaymentRecord(
    amount=Decimal("450.00"),
    currency="EUR",
    date=date(2025, 10, 20)
)

# Calculate conversion rate and variance
converter = FXConverter()

rate = converter.calculate_rate(
    from_amount=procedure.amount,
    from_currency=procedure.currency,
    to_amount=payment.amount,
    to_currency=payment.currency
)

# Result: rate = 0.9000 EUR/USD (1 USD = 0.90 EUR)

# Compare to standard hospital rate (0.92 EUR/USD)
gain_loss = converter.calculate_fx_gain_loss(
    actual_rate=rate.rate,
    market_rate=Decimal("0.92"),
    from_amount=procedure.amount,
    from_currency=procedure.currency,
    to_currency=payment.currency
)

# Result: Patient saved 10 EUR by using favorable rate
```

### Research: Grant Currency Conversion

```python
# Grant awarded in GBP, converted to USD for local expenses
grant = GrantAward(
    amount=Decimal("10000.00"),
    currency="GBP",
    award_date=date(2025, 9, 1)
)

disbursement = Disbursement(
    amount=Decimal("13200.00"),
    currency="USD",
    disbursement_date=date(2025, 10, 1)
)

# Calculate rate and compare to budget rate
converter = FXConverter(market_rate_provider=ExchangeRateHostProvider())

fx_details = converter.build_fx_details(
    from_amount=grant.amount,
    from_currency=grant.currency,
    to_amount=disbursement.amount,
    to_currency=disbursement.currency,
    conversion_date=disbursement.disbursement_date,
    include_market_comparison=True
)

# Budget assumed 1.35 USD/GBP
# Actual rate: 1.32 USD/GBP
# Loss: -300 USD (lower than budgeted)

# Track variance for grant reporting
grant_variance_report = {
    "grant_id": grant.grant_id,
    "budgeted_rate": Decimal("1.35"),
    "actual_rate": fx_details.exchange_rate,
    "variance_usd": fx_details.fx_gain_loss,
    "impact": "Reduced purchasing power by $300"
}
```

### Manufacturing: Unit Conversion with Tolerance

```python
# Weight measurement conversion: kg → lbs
# (Demonstrates FXConverter pattern for unit conversion)

measurement_kg = Measurement(
    value=Decimal("100.00"),
    unit="kg",
    date=date(2025, 10, 15)
)

measurement_lbs = Measurement(
    value=Decimal("220.46"),
    unit="lbs",
    date=date(2025, 10, 15)
)

# Calculate conversion factor
converter = FXConverter()  # Same primitive, different domain

factor = converter.calculate_rate(
    from_amount=measurement_kg.value,
    from_currency="kg",  # Treating units as "currency"
    to_amount=measurement_lbs.value,
    to_currency="lbs"
)

# Result: factor = 2.2046 lbs/kg

# Compare to standard conversion factor (2.2046)
standard_factor = Decimal("2.2046")

gain_loss = converter.calculate_fx_gain_loss(
    actual_rate=factor.rate,
    market_rate=standard_factor,
    from_amount=measurement_kg.value,
    from_currency="kg",
    to_currency="lbs"
)

# If gain_loss > tolerance (e.g., 0.1 lbs):
#   Flag measurement for recalibration
```

---

## Performance Optimization

### Caching Strategy

```python
import redis
from typing import Optional

class CachedMarketRateProvider:
    """Wrap market rate provider with Redis cache."""

    def __init__(
        self,
        provider: MarketRateProvider,
        redis_client: redis.Redis,
        ttl_seconds: int = 86400  # 24 hours
    ):
        self.provider = provider
        self.redis = redis_client
        self.ttl = ttl_seconds

    def get_rate(
        self,
        date: date,
        from_currency: str,
        to_currency: str
    ) -> Optional[Decimal]:
        """Get rate with caching."""

        # Build cache key
        cache_key = f"fx_rate:{from_currency}:{to_currency}:{date.isoformat()}"

        # Check cache
        cached = self.redis.get(cache_key)
        if cached:
            logger.debug(f"Cache HIT for {cache_key}")
            return Decimal(cached.decode('utf-8'))

        logger.debug(f"Cache MISS for {cache_key}")

        # Fetch from provider
        rate = self.provider.get_rate(date, from_currency, to_currency)

        # Cache result (even if None, to avoid repeated API calls)
        if rate is not None:
            self.redis.setex(cache_key, self.ttl, str(rate))
        else:
            # Cache "not found" for shorter period (1 hour)
            self.redis.setex(cache_key, 3600, "null")

        return rate
```

**Cache Metrics:**

```
fx_cache_hits (counter)
fx_cache_misses (counter)
fx_cache_hit_rate (gauge) = hits / (hits + misses)
fx_api_calls (counter)
fx_api_call_duration_ms (histogram)
```

### Batch Processing

```python
def calculate_rates_batch(
    self,
    conversions: list[tuple[Decimal, str, Decimal, str]]
) -> list[ExchangeRate]:
    """
    Calculate multiple rates in batch.

    Useful for processing historical data or bulk imports.
    """
    results = []

    for from_amount, from_currency, to_amount, to_currency in conversions:
        try:
            rate = self.calculate_rate(
                from_amount, from_currency,
                to_amount, to_currency
            )
            results.append(rate)
        except Exception as e:
            logger.error(
                f"Failed to calculate rate for "
                f"{from_currency}/{to_currency}: {e}"
            )
            results.append(None)

    return results

# Usage:
conversions = [
    (Decimal("1000"), "USD", Decimal("18500"), "MXN"),
    (Decimal("500"), "EUR", Decimal("550"), "USD"),
    (Decimal("10000"), "GBP", Decimal("13200"), "USD")
]

rates = converter.calculate_rates_batch(conversions)
```

### Query Optimization

**Index for FX Details Queries:**

```sql
-- Index for JSONB queries on fx_details
CREATE INDEX idx_relationships_fx_details_gin
ON relationships USING gin(fx_details)
WHERE type = 'fx_conversion' AND deleted_at IS NULL;

-- Index for specific FX fields
CREATE INDEX idx_relationships_fx_currencies
ON relationships ((fx_details->>'from_currency'), (fx_details->>'to_currency'))
WHERE type = 'fx_conversion' AND deleted_at IS NULL;

-- Index for gain/loss queries
CREATE INDEX idx_relationships_fx_gain_loss
ON relationships (((fx_details->>'fx_gain_loss')::decimal))
WHERE type = 'fx_conversion'
  AND deleted_at IS NULL
  AND fx_details->>'fx_gain_loss' IS NOT NULL;
```

**Performance Targets:**

| Operation | Target Latency (p95) |
|-----------|----------------------|
| Calculate rate | <5ms |
| Fetch market rate (cached) | <10ms |
| Fetch market rate (uncached) | <500ms |
| Calculate FX gain/loss | <5ms |
| Build complete FX details | <50ms (with market rate) |
| Batch calculate 100 rates | <100ms |

---

## Testing Guidance

### Unit Tests

```python
import pytest
from decimal import Decimal
from datetime import date

def test_calculate_rate_basic():
    """Test basic rate calculation."""
    converter = FXConverter()

    rate = converter.calculate_rate(
        from_amount=Decimal("1000.00"),
        from_currency="USD",
        to_amount=Decimal("18500.00"),
        to_currency="MXN"
    )

    assert rate.rate == Decimal("18.5000")
    assert rate.source == "calculated"
    assert rate.from_currency == "USD"
    assert rate.to_currency == "MXN"

def test_calculate_rate_zero_amount():
    """Test error handling for zero amount."""
    converter = FXConverter()

    with pytest.raises(ValueError, match="cannot be zero"):
        converter.calculate_rate(
            from_amount=Decimal("0"),
            from_currency="USD",
            to_amount=Decimal("18500"),
            to_currency="MXN"
        )

def test_calculate_rate_invalid_currency():
    """Test error handling for invalid currency."""
    converter = FXConverter()

    with pytest.raises(ValueError, match="Invalid currency"):
        converter.calculate_rate(
            from_amount=Decimal("1000"),
            from_currency="XXX",
            to_amount=Decimal("18500"),
            to_currency="MXN"
        )

def test_calculate_rate_precision():
    """Test rate precision (4 decimal places)."""
    converter = FXConverter()

    rate = converter.calculate_rate(
        from_amount=Decimal("1000.00"),
        from_currency="USD",
        to_amount=Decimal("18533.33"),
        to_currency="MXN"
    )

    # Should round to 4 decimals
    assert rate.rate == Decimal("18.5333")

def test_calculate_fx_gain_loss_gain():
    """Test FX gain calculation."""
    converter = FXConverter()

    result = converter.calculate_fx_gain_loss(
        actual_rate=Decimal("18.5"),
        market_rate=Decimal("18.3"),
        from_amount=Decimal("1000.00"),
        from_currency="USD",
        to_currency="MXN"
    )

    assert result.expected_amount == Decimal("18300.00")
    assert result.actual_amount == Decimal("18500.00")
    assert result.gain_loss == Decimal("200.00")
    assert result.gain_loss_pct == Decimal("1.09")

def test_calculate_fx_gain_loss_loss():
    """Test FX loss calculation."""
    converter = FXConverter()

    result = converter.calculate_fx_gain_loss(
        actual_rate=Decimal("18.0"),
        market_rate=Decimal("18.3"),
        from_amount=Decimal("1000.00"),
        from_currency="USD",
        to_currency="MXN"
    )

    assert result.gain_loss == Decimal("-300.00")
    assert result.gain_loss_pct == Decimal("-1.64")

def test_build_fx_details_complete():
    """Test building complete FX details."""
    mock_provider = MockMarketRateProvider(
        rate=Decimal("18.3")
    )
    converter = FXConverter(market_rate_provider=mock_provider)

    fx_details = converter.build_fx_details(
        from_amount=Decimal("1000.00"),
        from_currency="USD",
        to_amount=Decimal("18500.00"),
        to_currency="MXN",
        conversion_date=date(2025, 10, 16),
        include_market_comparison=True
    )

    assert fx_details.exchange_rate == Decimal("18.5000")
    assert fx_details.market_rate == Decimal("18.3000")
    assert fx_details.fx_gain_loss == Decimal("200.00")
    assert fx_details.rate_source == "calculated"

def test_build_fx_details_no_market_rate():
    """Test FX details without market rate."""
    converter = FXConverter()  # No provider

    fx_details = converter.build_fx_details(
        from_amount=Decimal("1000.00"),
        from_currency="USD",
        to_amount=Decimal("18500.00"),
        to_currency="MXN",
        conversion_date=date(2025, 10, 16),
        include_market_comparison=False
    )

    assert fx_details.exchange_rate == Decimal("18.5000")
    assert fx_details.market_rate is None
    assert fx_details.fx_gain_loss is None
```

### Integration Tests

```python
def test_end_to_end_fx_conversion():
    """Test complete FX conversion flow."""

    # Setup: Create two transactions
    txn_1 = create_transaction(
        amount=Decimal("-1000.00"),
        currency="USD",
        date=date(2025, 10, 16)
    )

    txn_2 = create_transaction(
        amount=Decimal("18500.00"),
        currency="MXN",
        date=date(2025, 10, 16)
    )

    # Calculate FX details
    converter = FXConverter(
        market_rate_provider=ExchangeRateHostProvider()
    )

    fx_details = converter.build_fx_details(
        from_amount=txn_1.amount,
        from_currency=txn_1.currency,
        to_amount=txn_2.amount,
        to_currency=txn_2.currency,
        conversion_date=txn_1.date,
        include_market_comparison=True
    )

    # Create relationship
    relationship = RelationshipStore.create(
        txn_1_id=txn_1.txn_id,
        txn_2_id=txn_2.txn_id,
        type="fx_conversion",
        fx_details=asdict(fx_details)
    )

    # Verify
    assert relationship.type == "fx_conversion"
    assert relationship.fx_details["exchange_rate"] == "18.5000"
    assert relationship.fx_details["from_currency"] == "USD"
    assert relationship.fx_details["to_currency"] == "MXN"

@pytest.mark.external_api
def test_market_rate_api_integration():
    """Test real API integration (requires network)."""

    provider = ExchangeRateHostProvider()

    rate = provider.get_rate(
        date=date(2025, 1, 15),  # Historical date
        from_currency="USD",
        to_currency="EUR"
    )

    assert rate is not None
    assert isinstance(rate, Decimal)
    assert Decimal("0.80") < rate < Decimal("1.20")  # Reasonable range
```

### Property-Based Tests

```python
from hypothesis import given, strategies as st

@given(
    from_amount=st.decimals(min_value=0.01, max_value=1000000, places=2),
    to_amount=st.decimals(min_value=0.01, max_value=1000000, places=2)
)
def test_rate_calculation_properties(from_amount, to_amount):
    """Test rate calculation properties with random inputs."""
    converter = FXConverter()

    rate = converter.calculate_rate(
        from_amount=from_amount,
        from_currency="USD",
        to_amount=to_amount,
        to_currency="MXN"
    )

    # Property 1: Rate is always positive
    assert rate.rate > 0

    # Property 2: Rate precision is 4 decimals
    assert len(str(rate.rate).split('.')[-1]) <= 4

    # Property 3: Inverse calculation returns original ratio
    inverse_rate = to_amount / from_amount
    assert abs(rate.rate - inverse_rate) < Decimal("0.0001")
```

---

## Configuration

```yaml
# config/fx_converter.yml

fx_converter:
  # Market rate provider
  market_rate_provider:
    enabled: true
    primary: "exchangerate.host"
    fallback: ["fixer.io", "currencylayer.com"]
    api_keys:
      fixer: "${FIXER_API_KEY}"
      currencylayer: "${CURRENCYLAYER_API_KEY}"
    timeout_seconds: 5
    retry_attempts: 2

  # Caching
  cache:
    enabled: true
    backend: "redis"
    ttl_hours: 24
    cache_null_results: true
    null_result_ttl_hours: 1

  # Precision
  precision:
    exchange_rate_decimals: 4
    currency_amount_decimals: 2
    percentage_decimals: 2

  # Supported currencies
  supported_currencies:
    - USD
    - EUR
    - GBP
    - MXN
    - CAD
    # ... (full list)

  # Zero-decimal currencies
  zero_decimal_currencies:
    - JPY
    - KRW
    - VND
    - CLP

  # Performance
  performance:
    batch_size: 100
    max_concurrent_api_calls: 5
```

---

## Related Primitives

### Within Vertical 3.5:
- **RelationshipStore**: Stores FX conversion relationships with fx_details JSONB
- **TransferDetector**: Detects FX conversions (different currencies, opposite signs)
- **RelationshipMatcher**: Matches transaction pairs for FX conversion linking

### From Other Verticals:
- **ValidationEngine** (1.3): Validates currency codes (ISO 4217)
- **CanonicalStore** (1.3): Provides transaction data for rate calculation
- **ProvenanceLedger** (1.1): Logs FX calculation operations

---

**Last Updated**: 2025-10-24
**Maturity**: Specification complete, ready for implementation
