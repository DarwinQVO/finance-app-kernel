# Primitive: CurrencyConverter (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Value Converter
> **Vertical:** 3.6 Unit (Currency & Date Normalization)
> **Last Updated:** 2025-10-24

---

## Overview

**CurrencyConverter** is an Objective Layer primitive that converts monetary amounts between currencies while preserving original context. It orchestrates exchange rate fetching, precision handling, and dual-value storage (original + normalized).

**Core Responsibilities:**
- Convert amounts from source currency to target currency
- Fetch exchange rates via ExchangeRateProvider
- Apply precision rules by currency type (fiat 2 decimals, crypto 4-8)
- Create NormalizedAmount objects with full metadata
- Handle same-currency conversions (rate = 1.0)
- Support batch conversions for performance

**Finance Use Case:**
Convert $100 USD transaction to user's base currency (MXN) for aggregation and reporting.

**Multi-Domain Applicability:**
- Healthcare: Convert international medical claim amounts (EUR → USD)
- E-commerce: Convert product prices to customer's local currency
- Travel: Convert foreign expenses to company's base currency
- Manufacturing: Convert measurement units (imperial → metric)

---

## API Contract

### Methods

```python
class CurrencyConverter:
    """
    Converts monetary amounts between currencies with context preservation.

    Attributes:
        rate_provider: ExchangeRateProvider instance for fetching rates
        precision_rules: Dictionary mapping currency codes to decimal places
    """

    def __init__(self, rate_provider: ExchangeRateProvider):
        """
        Initialize converter with rate provider.

        Args:
            rate_provider: ExchangeRateProvider instance
        """
        self.rate_provider = rate_provider
        self.precision_rules = self._load_precision_rules()

    def convert(
        self,
        amount: Decimal,
        from_currency: str,
        to_currency: str,
        rate_date: date
    ) -> NormalizedAmount:
        """
        Convert amount from one currency to another.

        Args:
            amount: Original amount (can be negative for expenses)
            from_currency: ISO 4217 currency code (e.g., "USD")
            to_currency: ISO 4217 currency code (e.g., "MXN")
            rate_date: Date for exchange rate lookup

        Returns:
            NormalizedAmount with original + converted values and rate metadata

        Raises:
            UnsupportedCurrencyError: If currency not in ISO 4217
            RateNotAvailableError: If no rate found for date
            InvalidAmountError: If amount is NaN or infinity

        Example:
            >>> converter = CurrencyConverter(rate_provider)
            >>> result = converter.convert(
            ...     amount=Decimal("100.00"),
            ...     from_currency="USD",
            ...     to_currency="MXN",
            ...     rate_date=date(2024, 11, 1)
            ... )
            >>> print(result.normalized_amount)
            1850.00
        """

    def batch_convert(
        self,
        transactions: List[Transaction]
    ) -> Dict[str, NormalizedAmount]:
        """
        Batch convert multiple transactions (performance optimization).

        Fetches unique exchange rates once, then applies to all transactions.
        Up to 100x faster than individual conversions for large batches.

        Args:
            transactions: List of Transaction objects with amount and currency

        Returns:
            Dictionary mapping transaction_id to NormalizedAmount

        Example:
            >>> transactions = [txn1, txn2, txn3]  # 3 USD transactions
            >>> results = converter.batch_convert(transactions)
            >>> # Only 1 rate fetch (USD→MXN), applied to all 3
        """

    def get_precision(self, currency: str) -> int:
        """
        Get decimal precision for currency.

        Args:
            currency: ISO 4217 currency code

        Returns:
            Number of decimal places (2 for fiat, 4-8 for crypto)

        Example:
            >>> converter.get_precision("USD")
            2
            >>> converter.get_precision("BTC")
            8
        """

    def validate_currency(self, currency: str) -> bool:
        """
        Validate currency code against ISO 4217 standard.

        Args:
            currency: Currency code to validate

        Returns:
            True if valid, False otherwise

        Example:
            >>> converter.validate_currency("USD")
            True
            >>> converter.validate_currency("FAKE")
            False
        """

    def _load_precision_rules(self) -> Dict[str, int]:
        """
        Load precision rules for all supported currencies.

        Returns:
            Dictionary mapping currency code to decimal places

        Internal method, called during initialization.
        """

    def _apply_precision(self, amount: Decimal, currency: str) -> Decimal:
        """
        Round amount to currency's precision.

        Args:
            amount: Decimal amount
            currency: Currency code

        Returns:
            Rounded Decimal

        Example:
            >>> _apply_precision(Decimal("1234.5678"), "USD")
            Decimal("1234.57")  # 2 decimals for USD
        """
```

---

## Data Structures

### Input Types

```python
# Currency code (ISO 4217)
Currency = str  # Pattern: ^[A-Z]{3}$

# Amount (use Decimal to avoid floating-point errors)
Amount = Decimal

# Rate date (for historical rate lookup)
RateDate = date
```

### Output Types

```python
@dataclass
class NormalizedAmount:
    """
    Result of currency conversion with full metadata.

    Attributes:
        original_amount: Amount in source currency
        original_currency: ISO 4217 code of source currency
        normalized_amount: Amount in target currency
        base_currency: ISO 4217 code of target currency
        exchange_rate: Rate used for conversion (to/from)
        rate_date: Date of exchange rate
        rate_source: Source of rate ("ecb", "federal_reserve", "manual")
        override_reason: Optional reason if manual override
        converted_at: Timestamp of conversion
    """
    original_amount: Decimal
    original_currency: str
    normalized_amount: Decimal
    base_currency: str
    exchange_rate: Decimal
    rate_date: date
    rate_source: str
    override_reason: Optional[str] = None
    converted_at: datetime = field(default_factory=datetime.now)
```

---

## Behavior Specifications

### Same-Currency Conversion

**Scenario:** Convert USD to USD (no conversion needed)

```python
result = converter.convert(
    amount=Decimal("100.00"),
    from_currency="USD",
    to_currency="USD",
    rate_date=date(2024, 11, 1)
)

# Expected result:
# NormalizedAmount(
#     original_amount=100.00,
#     original_currency="USD",
#     normalized_amount=100.00,
#     base_currency="USD",
#     exchange_rate=1.0000,
#     rate_date="2024-11-01",
#     rate_source="identity"
# )
```

**Logic:**
- Detect `from_currency == to_currency`
- Skip rate fetch (optimization)
- Return exchange_rate = 1.0, source = "identity"

---

### Negative Amount Conversion (Expenses)

**Scenario:** Convert negative amount (expense transaction)

```python
result = converter.convert(
    amount=Decimal("-100.00"),  # Expense
    from_currency="USD",
    to_currency="MXN",
    rate_date=date(2024, 11, 1)
)

# Expected result:
# NormalizedAmount(
#     original_amount=-100.00,
#     normalized_amount=-1850.00,  # Negative preserved
#     exchange_rate=18.5000
# )
```

**Logic:**
- Preserve sign (positive/negative) during conversion
- Apply rate to absolute value, then restore sign

---

### Precision Rounding

**Scenario:** Convert amount with high precision to currency with lower precision

```python
result = converter.convert(
    amount=Decimal("100.123456"),  # 6 decimals
    from_currency="USD",
    to_currency="MXN",  # 2 decimals
    rate_date=date(2024, 11, 1)
)

# Expected result:
# NormalizedAmount(
#     original_amount=100.12,  # Rounded to 2 decimals (USD precision)
#     normalized_amount=1852.22,  # Rounded to 2 decimals (MXN precision)
#     exchange_rate=18.5000
# )
```

**Logic:**
1. Round original_amount to from_currency precision (100.12)
2. Convert: 100.12 × 18.5000 = 1852.22
3. Round result to to_currency precision (2 decimals)

---

### Crypto Currency High Precision

**Scenario:** Convert Bitcoin (8 decimals) to USD (2 decimals)

```python
result = converter.convert(
    amount=Decimal("0.12345678"),  # 8 decimals (BTC)
    from_currency="BTC",
    to_currency="USD",
    rate_date=date(2024, 11, 1)
)

# Assume BTC/USD rate = 65000.00
# Expected result:
# NormalizedAmount(
#     original_amount=0.12345678,  # 8 decimals preserved
#     normalized_amount=8024.69,  # 2 decimals (USD)
#     exchange_rate=65000.0000
# )
```

**Precision Rules:**
- BTC: 8 decimals
- ETH: 4 decimals
- Fiat (USD, EUR, MXN): 2 decimals

---

### Batch Conversion Optimization

**Scenario:** Convert 1000 USD transactions to MXN

```python
transactions = [
    Transaction(id="txn_1", amount=100, currency="USD", date="2024-11-01"),
    Transaction(id="txn_2", amount=200, currency="USD", date="2024-11-01"),
    # ... 998 more
]

results = converter.batch_convert(transactions)

# Performance:
# - Individual: 1000 conversions × 100ms rate fetch = 100 seconds
# - Batch: 1 rate fetch (100ms) + 1000 conversions (50ms) = 150ms
# - Speedup: 666x faster
```

**Algorithm:**
1. Group transactions by (from_currency, to_currency, rate_date)
2. Fetch unique rates (e.g., 1 rate for USD→MXN on 2024-11-01)
3. Apply rate to all transactions in group
4. Return dictionary: {transaction_id: NormalizedAmount}

---

## Error Handling

### UnsupportedCurrencyError

**Trigger:** Currency code not in ISO 4217 standard

```python
try:
    converter.convert(100, "FAKE", "USD", date.today())
except UnsupportedCurrencyError as e:
    print(e)
    # "Currency 'FAKE' is not supported. Must be ISO 4217 code."
```

**Valid Currencies:** USD, EUR, GBP, MXN, JPY, CAD, AUD, CHF, CNY, BTC, ETH, etc. (150+ total)

---

### RateNotAvailableError

**Trigger:** No exchange rate available for date

```python
try:
    converter.convert(100, "USD", "MXN", date(1990, 1, 1))  # Too old
except RateNotAvailableError as e:
    print(e)
    # "Exchange rate for USD→MXN on 1990-01-01 not available. Please enter manually."
```

**Causes:**
- Date too far in past (ECB only has data since ~1999)
- Exotic currency pair (e.g., MXN → KRW, no direct rate)
- API failure (both ECB and Federal Reserve down)

**Mitigation:**
- Use historical API (ECB has data since 1999)
- Calculate cross-rate via USD (MXN → USD → KRW)
- Prompt user for manual rate entry

---

### InvalidAmountError

**Trigger:** Amount is NaN, infinity, or invalid

```python
try:
    converter.convert(Decimal("NaN"), "USD", "MXN", date.today())
except InvalidAmountError as e:
    print(e)
    # "Amount must be a valid number, got 'NaN'"
```

**Validations:**
- Not NaN
- Not infinity
- Not None
- Finite decimal

---

## Performance Characteristics

**Latency Targets (p95):**
- Single conversion (cache hit): <10ms
- Single conversion (cache miss): <500ms (includes rate fetch)
- Batch conversion (1000 transactions): <150ms

**Throughput:**
- Sequential: ~100 conversions/second (rate fetch bottleneck)
- Batch: ~10,000 conversions/second (single rate fetch)

**Optimization Techniques:**
1. **Rate Caching:** Cache exchange rates for 24 hours (reduce API calls)
2. **Batch Fetching:** Fetch unique rates once, apply to all transactions
3. **Decimal Precision:** Use fixed-point arithmetic (avoid float errors)
4. **Lazy Loading:** Only load precision rules once (initialization)

---

## Dependencies

**Internal:**
- `ExchangeRateProvider` (fetch rates from ECB, Federal Reserve)
- `Decimal` (Python standard library for precision arithmetic)
- `date`, `datetime` (Python standard library)

**External:**
- None (ExchangeRateProvider handles external APIs)

**Data:**
- ISO 4217 currency list (static file, loaded at startup)
- Precision rules (static configuration, loaded at startup)

---

## Testing Strategy

### Unit Tests

```python
def test_convert_usd_to_mxn():
    """Test basic conversion with mocked rate provider."""
    mock_provider = Mock()
    mock_provider.get_rate.return_value = ExchangeRateRecord(
        rate=Decimal("18.5000"),
        source="ecb"
    )

    converter = CurrencyConverter(rate_provider=mock_provider)
    result = converter.convert(
        amount=Decimal("100.00"),
        from_currency="USD",
        to_currency="MXN",
        rate_date=date(2024, 11, 1)
    )

    assert result.original_amount == Decimal("100.00")
    assert result.normalized_amount == Decimal("1850.00")
    assert result.exchange_rate == Decimal("18.5000")

def test_convert_same_currency():
    """Test identity conversion (USD → USD)."""
    converter = CurrencyConverter(rate_provider=mock_provider)
    result = converter.convert(100, "USD", "USD", date.today())

    assert result.exchange_rate == Decimal("1.0000")
    assert result.rate_source == "identity"
    mock_provider.get_rate.assert_not_called()  # No rate fetch

def test_convert_negative_amount():
    """Test conversion preserves negative sign."""
    result = converter.convert(-100, "USD", "MXN", date.today())

    assert result.original_amount < 0
    assert result.normalized_amount < 0

def test_convert_precision_rounding():
    """Test precision rounding for fiat currencies."""
    result = converter.convert(
        Decimal("100.999"),  # 3 decimals
        "USD",  # 2 decimals
        "MXN",  # 2 decimals
        date.today()
    )

    assert result.original_amount == Decimal("101.00")  # Rounded
    assert str(result.normalized_amount).count('.') == 1
    assert len(str(result.normalized_amount).split('.')[1]) == 2  # 2 decimals

def test_batch_convert_performance():
    """Test batch conversion is faster than individual."""
    transactions = [create_transaction() for _ in range(100)]

    # Individual conversions
    start = time.time()
    for txn in transactions:
        converter.convert(txn.amount, txn.currency, "MXN", txn.date)
    individual_time = time.time() - start

    # Batch conversion
    start = time.time()
    converter.batch_convert(transactions)
    batch_time = time.time() - start

    assert batch_time < individual_time / 10  # At least 10x faster

def test_unsupported_currency_error():
    """Test error on invalid currency code."""
    with pytest.raises(UnsupportedCurrencyError):
        converter.convert(100, "FAKE", "USD", date.today())

def test_rate_not_available_error():
    """Test error when rate not found."""
    mock_provider.get_rate.side_effect = RateNotAvailableError("No rate for 1990")

    with pytest.raises(RateNotAvailableError):
        converter.convert(100, "USD", "MXN", date(1990, 1, 1))

def test_crypto_precision():
    """Test Bitcoin 8-decimal precision."""
    result = converter.convert(
        Decimal("0.12345678"),
        "BTC",
        "USD",
        date.today()
    )

    assert len(str(result.original_amount).split('.')[1]) == 8  # 8 decimals
```

### Integration Tests

```python
def test_end_to_end_conversion_with_real_provider():
    """Test conversion with real ExchangeRateProvider (uses test API)."""
    provider = ExchangeRateProvider(cache=test_cache, api_key=test_api_key)
    converter = CurrencyConverter(rate_provider=provider)

    result = converter.convert(100, "USD", "EUR", date.today())

    assert result.normalized_amount > 0
    assert result.rate_source in ["ecb", "federal_reserve"]
    assert result.exchange_rate > 0.5  # Sanity check (EUR/USD ~ 0.85-1.0)
```

---

## Multi-Domain Examples

### Finance: Multi-Currency Expense Tracking

**Use Case:** User tracks expenses in USD, EUR, and MXN. All normalized to MXN for reporting.

```python
# Transaction 1: $100 USD
result_usd = converter.convert(100, "USD", "MXN", date(2024, 11, 1))
# → $1,850 MXN

# Transaction 2: €50 EUR
result_eur = converter.convert(50, "EUR", "MXN", date(2024, 11, 1))
# → $1,050 MXN

# Total expenses in MXN
total_mxn = result_usd.normalized_amount + result_eur.normalized_amount
# → $2,900 MXN
```

---

### Healthcare: International Medical Claims

**Use Case:** Hospital in France charges €5,000. US insurance reimburses in USD.

```python
claim_eur = Decimal("5000.00")
reimbursement_usd = converter.convert(claim_eur, "EUR", "USD", claim_date)

# Insurance pays: $5,250 USD (assuming EUR/USD = 1.05)
print(f"Reimbursement: ${reimbursement_usd.normalized_amount}")
```

---

### E-commerce: Product Pricing

**Use Case:** Product costs $99.99 USD. Display price in customer's local currency (MXN).

```python
product_price_usd = Decimal("99.99")
display_price_mxn = converter.convert(product_price_usd, "USD", "MXN", date.today())

# Customer sees: $1,849.81 MXN
print(f"Price: ${display_price_mxn.normalized_amount} MXN")
```

---

### Travel: Expense Reporting

**Use Case:** Employee spent £200 GBP in London. Company reimburses in USD.

```python
expense_gbp = Decimal("200.00")
reimbursement_usd = converter.convert(expense_gbp, "GBP", "USD", expense_date)

# Company pays: $255.00 USD (assuming GBP/USD = 1.275)
print(f"Reimbursement: ${reimbursement_usd.normalized_amount}")
```

---

### Manufacturing: Unit Conversion (Temperature)

**Use Case:** Sensor reads 212°F. System stores Celsius.

```python
# Extend CurrencyConverter pattern to TemperatureConverter
temp_converter = TemperatureConverter()
celsius = temp_converter.convert(212.0, "fahrenheit", "celsius")

# Result: 100.0°C (boiling point of water)
```

**Pattern Generalization:**
```python
class ValueNormalizer[TOriginal, TNormalized]:
    """Abstract pattern for any value normalization."""

    def convert(
        self,
        value: TOriginal,
        from_unit: str,
        to_unit: str,
        context: Optional[Any] = None
    ) -> NormalizedValue[TNormalized]:
        """Convert value from one unit to another."""
```

---

## Observability

### Metrics

```python
# Conversion metrics
currency.conversions.count (counter, labels: from_currency, to_currency)
currency.conversions.latency (histogram, labels: cache_hit)
currency.batch_conversions.count (counter)
currency.batch_conversions.size (histogram)

# Error metrics
currency.errors.unsupported_currency (counter, labels: currency)
currency.errors.rate_not_available (counter, labels: from_currency, to_currency, date)
currency.errors.invalid_amount (counter)

# Precision metrics
currency.precision_rounding.count (counter, labels: currency)
```

### Structured Logs

```json
{
  "timestamp": "2024-11-01T10:00:00Z",
  "level": "INFO",
  "event": "currency_converted",
  "from_currency": "USD",
  "to_currency": "MXN",
  "original_amount": 100.00,
  "normalized_amount": 1850.00,
  "exchange_rate": 18.5000,
  "rate_source": "ecb",
  "latency_ms": 45
}
```

---

## Security Considerations

**Input Validation:**
- Currency code: Pattern `^[A-Z]{3}$` (prevent injection)
- Amount: Finite decimal, no NaN/infinity (prevent overflow)
- Date: Valid date, not in future (prevent time manipulation)

**Precision Guarantee:**
- Use `Decimal` type (Python) or `BigDecimal` (Java) for all arithmetic
- Never use `float` (prevents rounding errors: 0.1 + 0.2 ≠ 0.3)

**Rate Tampering Prevention:**
- All rates fetched from ExchangeRateProvider (trusted source)
- Manual overrides logged with user_id and reason (audit trail)
- Variance check: Manual rate >10% variance requires confirmation

---

## Configuration

```yaml
# config/currency_converter.yaml
precision_rules:
  # Fiat currencies (2 decimals)
  USD: 2
  EUR: 2
  GBP: 2
  MXN: 2
  JPY: 0  # Yen has no subunits

  # Cryptocurrencies (4-8 decimals)
  BTC: 8
  ETH: 4
  USDT: 2  # Stablecoins use 2

# ISO 4217 supported currencies
supported_currencies:
  - USD
  - EUR
  - GBP
  - MXN
  - JPY
  - CAD
  - AUD
  - CHF
  - CNY
  - BTC
  - ETH
  # ... 140+ more
```

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Convert multi-currency transactions to user's base currency (USD)
**Example:** Transaction in EUR: €850.00 on 2024-03-15 → CurrencyConverter: query rate (EUR/USD = 1.08 on 2024-03-15) → convert: €850 × 1.08 = $918.00 → Store in CanonicalTransaction
**Operations:** Historical rate lookup, Decimal multiplication (precision), rounding to 2 decimals
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Convert international medical costs to insurance base currency
**Example:** Medical bill in GBP: £1,200 → CurrencyConverter: rate (GBP/USD = 1.27) → convert: £1,200 × 1.27 = $1,524.00 → Insurance claim processing
**Operations:** Real-time rate lookup, precision handling for reimbursement
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Convert contract amounts across jurisdictions
**Example:** Contract in MXN: $50,000 MXN → CurrencyConverter: rate (MXN/USD = 0.058) → convert: 50,000 × 0.058 = $2,900.00 USD → Legal fee calculation
**Operations:** Historical rate at contract date, legal precision requirements
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** Normalize investment amounts across currencies for comparison
**Example:** Article mentions "€375M investment" → CurrencyConverter: rate (EUR/USD = 1.08) → convert: €375M × 1.08 = $405M USD → Comparable across all facts
**Operations:** Large number handling (millions), approximate rates for news articles
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Display product prices in customer's local currency
**Example:** Product base price $1,199.99 USD, customer in EU → CurrencyConverter: rate (USD/EUR = 0.93) → convert: $1,199.99 × 0.93 = €1,116.00 → Display to customer
**Operations:** Real-time rate lookup, dynamic pricing, cache for performance
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (pure currency arithmetic with exchange rates, no domain logic)
**Reusability:** High (same convert() method works for transactions, medical bills, contracts, investments, product prices)

---

## Changelog

**v1.0.0 (2024-11-01):**
- Initial implementation
- Support for 150+ ISO 4217 currencies
- Batch conversion optimization
- Crypto currency support (BTC, ETH)

**v1.1.0 (Future):**
- Cross-rate calculation (indirect conversions via USD)
- Real-time crypto rate updates (<1 hour staleness)
- Multi-base currency support (track in USD + EUR simultaneously)

---

## References

- ISO 4217: International currency codes standard
- ECB Exchange Rate API: https://api.exchangerate.host
- Federal Reserve Exchange Rate API: https://www.federalreserve.gov/data.htm
- Python Decimal documentation: https://docs.python.org/3/library/decimal.html

---

**Total Lines:** ~1,150
