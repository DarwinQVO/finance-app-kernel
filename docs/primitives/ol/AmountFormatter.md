# Primitive: AmountFormatter (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Display Formatter
> **Vertical:** 3.6 Unit (Currency & Date Normalization)
> **Last Updated:** 2025-10-24

---

## Overview

**AmountFormatter** is an Objective Layer primitive that formats monetary amounts for display with locale-aware rules. It handles currency symbols, thousands separators, decimal symbols, and precision rules across 150+ locales.

**Core Responsibilities:**
- Format amounts with locale-specific rules (en-US, es-MX, de-DE, etc.)
- Apply currency symbols ($ vs € vs £)
- Apply thousands separators (, vs . vs space)
- Apply decimal symbols (. vs ,)
- Handle negative amounts (parentheses vs minus sign)
- Support precision rules (2 for fiat, 4-8 for crypto)

**Finance Use Case:**
Display $1,234.56 as "$1,234.56" (US) or "$1.234,56" (Spain).

**Multi-Domain Applicability:**
- Healthcare: Format medical claim amounts in local currency
- E-commerce: Display product prices in customer's locale
- Legal: Format billing amounts in jurisdiction format
- Manufacturing: Format measurement values with locale conventions
- Travel: Display expense amounts in traveler's home currency format

---

## API Contract

### Methods

```python
class AmountFormatter:
    """
    Formats monetary amounts with locale-aware display rules.

    Attributes:
        locale_db: Babel locale database (loaded at initialization)
    """

    def __init__(self):
        """
        Initialize formatter with Babel locale database.

        Loads locale-specific number formatting rules for 150+ locales.
        """
        from babel import Locale
        self.locale_db = Locale

    def format(
        self,
        amount: Decimal,
        currency: str,
        locale: str
    ) -> str:
        """
        Format amount with locale-aware rules.

        Args:
            amount: Decimal amount (can be negative)
            currency: ISO 4217 currency code (e.g., "USD", "EUR")
            locale: BCP 47 locale (e.g., "en-US", "es-MX", "de-DE")

        Returns:
            Formatted string (e.g., "$1,234.56", "1.234,56 €")

        Raises:
            UnsupportedLocaleError: If locale not in BCP 47
            UnsupportedCurrencyError: If currency not in ISO 4217

        Example:
            >>> formatter = AmountFormatter()
            >>> formatted = formatter.format(Decimal("1234.56"), "USD", "en-US")
            >>> print(formatted)
            "$1,234.56"

            >>> formatted = formatter.format(Decimal("1234.56"), "EUR", "es-ES")
            >>> print(formatted)
            "1.234,56 €"
        """

    def parse(
        self,
        formatted_string: str,
        locale: str
    ) -> Decimal:
        """
        Parse formatted string back to Decimal (inverse of format).

        Args:
            formatted_string: Locale-formatted string (e.g., "$1,234.56")
            locale: BCP 47 locale used for formatting

        Returns:
            Decimal amount

        Raises:
            InvalidFormatError: If string cannot be parsed

        Example:
            >>> amount = formatter.parse("$1,234.56", "en-US")
            >>> print(amount)
            1234.56
        """

    def get_locale_rules(self, locale: str) -> LocaleRules:
        """
        Get formatting rules for locale.

        Args:
            locale: BCP 47 locale string

        Returns:
            LocaleRules with thousands separator, decimal symbol, etc.

        Example:
            >>> rules = formatter.get_locale_rules("en-US")
            >>> print(rules.thousands_separator)
            ","
            >>> print(rules.decimal_symbol)
            "."

            >>> rules = formatter.get_locale_rules("de-DE")
            >>> print(rules.thousands_separator)
            "."
            >>> print(rules.decimal_symbol)
            ","
        """

    def format_compact(
        self,
        amount: Decimal,
        currency: str,
        locale: str
    ) -> str:
        """
        Format amount in compact notation (K, M, B).

        Args:
            amount: Decimal amount
            currency: ISO 4217 currency code
            locale: BCP 47 locale

        Returns:
            Compact formatted string (e.g., "$1.2K", "$1.5M")

        Example:
            >>> formatter.format_compact(Decimal("1234.56"), "USD", "en-US")
            "$1.2K"

            >>> formatter.format_compact(Decimal("1234567.89"), "USD", "en-US")
            "$1.2M"
        """

    def validate_locale(self, locale: str) -> bool:
        """
        Validate locale string against BCP 47 standard.

        Args:
            locale: Locale string to validate

        Returns:
            True if valid, False otherwise

        Example:
            >>> formatter.validate_locale("en-US")
            True
            >>> formatter.validate_locale("invalid")
            False
        """
```

---

## Data Structures

### Input Types

```python
# Amount (use Decimal to avoid floating-point errors)
Amount = Decimal

# Currency code (ISO 4217)
Currency = str  # Pattern: ^[A-Z]{3}$

# Locale code (BCP 47)
Locale = str  # Examples: "en-US", "es-MX", "de-DE", "fr-FR"
```

### Output Types

```python
@dataclass
class LocaleRules:
    """
    Formatting rules for a specific locale.

    Attributes:
        thousands_separator: Character for thousands (e.g., ",", ".", " ")
        decimal_symbol: Character for decimal point (e.g., ".", ",")
        currency_symbol_position: "prefix" or "suffix"
        negative_format: "minus" or "parentheses"
        grouping: Tuple of grouping sizes (e.g., (3,) for 1,234,567)
    """
    thousands_separator: str
    decimal_symbol: str
    currency_symbol_position: str  # "prefix" or "suffix"
    negative_format: str  # "minus" or "parentheses"
    grouping: tuple  # (3,) for 1,234,567 or (3, 2) for 1,23,45,678 (India)

# Formatted display string
FormattedString = str  # Example: "$1,234.56"
```

---

## Behavior Specifications

### US Locale (en-US)

**Rules:**
- Thousands separator: `,`
- Decimal symbol: `.`
- Currency symbol: Prefix (`$1,234.56`)
- Negative format: Minus sign (`-$1,234.56`)

```python
formatter = AmountFormatter()

# Positive amount
result = formatter.format(Decimal("1234.56"), "USD", "en-US")
# → "$1,234.56"

# Negative amount
result = formatter.format(Decimal("-1234.56"), "USD", "en-US")
# → "-$1,234.56"

# Large amount
result = formatter.format(Decimal("1234567.89"), "USD", "en-US")
# → "$1,234,567.89"
```

---

### Spanish (Spain) Locale (es-ES)

**Rules:**
- Thousands separator: `.`
- Decimal symbol: `,`
- Currency symbol: Suffix (`1.234,56 €`)
- Negative format: Minus sign (`-1.234,56 €`)

```python
# Positive amount
result = formatter.format(Decimal("1234.56"), "EUR", "es-ES")
# → "1.234,56 €"

# Negative amount
result = formatter.format(Decimal("-1234.56"), "EUR", "es-ES")
# → "-1.234,56 €"
```

---

### German Locale (de-DE)

**Rules:**
- Thousands separator: `.`
- Decimal symbol: `,`
- Currency symbol: Suffix (`1.234,56 €`)
- Negative format: Minus sign (`-1.234,56 €`)

```python
result = formatter.format(Decimal("1234.56"), "EUR", "de-DE")
# → "1.234,56 €"
```

---

### French Locale (fr-FR)

**Rules:**
- Thousands separator: (space)
- Decimal symbol: `,`
- Currency symbol: Suffix (`1 234,56 €`)
- Negative format: Minus sign (`-1 234,56 €`)

```python
result = formatter.format(Decimal("1234.56"), "EUR", "fr-FR")
# → "1 234,56 €"
```

---

### Mexican Spanish Locale (es-MX)

**Rules:**
- Thousands separator: `,`
- Decimal symbol: `.`
- Currency symbol: Prefix (`$1,234.56`)
- Negative format: Minus sign (`-$1,234.56`)

```python
result = formatter.format(Decimal("1234.56"), "MXN", "es-MX")
# → "$1,234.56"
```

---

### Japanese Locale (ja-JP)

**Rules:**
- Thousands separator: `,`
- Decimal symbol: `.`
- Currency symbol: Prefix (`¥1,234`)
- Negative format: Minus sign (`-¥1,234`)
- Note: JPY has 0 decimal places (no subunits)

```python
result = formatter.format(Decimal("1234.56"), "JPY", "ja-JP")
# → "¥1,235"  # Rounded to nearest yen (no decimals)
```

---

### Crypto Currency (Bitcoin)

**Rules:**
- Precision: 8 decimals
- Symbol: BTC (suffix)

```python
result = formatter.format(Decimal("0.12345678"), "BTC", "en-US")
# → "0.12345678 BTC"

# Large BTC amount
result = formatter.format(Decimal("1.23456789"), "BTC", "en-US")
# → "1.23456789 BTC"
```

---

### Negative Amount with Parentheses (Accounting Format)

**Use Case:** Some locales/contexts use parentheses for negative amounts

```python
# Custom format (not default for any locale)
result = formatter.format(
    Decimal("-1234.56"),
    "USD",
    "en-US",
    negative_format="parentheses"
)
# → "($1,234.56)"
```

---

### Compact Notation

**Use Case:** Display large amounts concisely

```python
# $1,234.56 → $1.2K
result = formatter.format_compact(Decimal("1234.56"), "USD", "en-US")
# → "$1.2K"

# $1,234,567.89 → $1.2M
result = formatter.format_compact(Decimal("1234567.89"), "USD", "en-US")
# → "$1.2M"

# $1,234,567,890.12 → $1.2B
result = formatter.format_compact(Decimal("1234567890.12"), "USD", "en-US")
# → "$1.2B"
```

---

## Error Handling

### UnsupportedLocaleError

**Trigger:** Locale not in BCP 47 standard

```python
try:
    formatter.format(Decimal("1234.56"), "USD", "invalid-locale")
except UnsupportedLocaleError as e:
    print(e)
    # "Locale 'invalid-locale' is not supported. Must be BCP 47 locale."
```

**Valid Locales:** 150+ BCP 47 locales (en-US, es-MX, de-DE, fr-FR, etc.)

---

### UnsupportedCurrencyError

**Trigger:** Currency not in ISO 4217

```python
try:
    formatter.format(Decimal("1234.56"), "FAKE", "en-US")
except UnsupportedCurrencyError as e:
    print(e)
    # "Currency 'FAKE' is not supported. Must be ISO 4217 code."
```

---

### InvalidFormatError

**Trigger:** Cannot parse formatted string

```python
try:
    formatter.parse("invalid-amount", "en-US")
except InvalidFormatError as e:
    print(e)
    # "Cannot parse 'invalid-amount' as amount in locale 'en-US'"
```

---

## Performance Characteristics

**Latency Targets (p95):**
- Format amount: <5ms
- Parse amount: <5ms
- Get locale rules: <2ms (cached)

**Throughput:**
- ~200,000 formats/second (single thread)

**Optimization Techniques:**
1. **Locale Rules Caching:** Load rules once per locale
2. **Symbol Caching:** Cache currency symbols
3. **Format String Caching:** Pre-compile format strings

---

## Dependencies

**Internal:**
- `babel` (Python internationalization library)
- `Decimal` (Python standard library)

**External:**
- None

**Data:**
- Babel locale database (~10 MB, 150+ locales)
- ISO 4217 currency symbols (~50 KB)

---

## Testing Strategy

### Unit Tests

```python
def test_format_usd_en_us():
    """Test US dollar formatting in US locale."""
    formatter = AmountFormatter()
    result = formatter.format(Decimal("1234.56"), "USD", "en-US")
    assert result == "$1,234.56"

def test_format_eur_es_es():
    """Test Euro formatting in Spanish locale."""
    formatter = AmountFormatter()
    result = formatter.format(Decimal("1234.56"), "EUR", "es-ES")
    assert result == "1.234,56 €"

def test_format_negative_amount():
    """Test negative amount formatting."""
    formatter = AmountFormatter()
    result = formatter.format(Decimal("-1234.56"), "USD", "en-US")
    assert result == "-$1,234.56"

def test_format_jpy_no_decimals():
    """Test JPY formatting (0 decimals)."""
    formatter = AmountFormatter()
    result = formatter.format(Decimal("1234.56"), "JPY", "ja-JP")
    assert result == "¥1,235"  # Rounded

def test_format_btc_high_precision():
    """Test Bitcoin formatting (8 decimals)."""
    formatter = AmountFormatter()
    result = formatter.format(Decimal("0.12345678"), "BTC", "en-US")
    assert result == "0.12345678 BTC"

def test_format_compact():
    """Test compact notation."""
    formatter = AmountFormatter()

    assert formatter.format_compact(Decimal("1234"), "USD", "en-US") == "$1.2K"
    assert formatter.format_compact(Decimal("1234567"), "USD", "en-US") == "$1.2M"
    assert formatter.format_compact(Decimal("1234567890"), "USD", "en-US") == "$1.2B"

def test_parse_usd_en_us():
    """Test parsing US formatted string."""
    formatter = AmountFormatter()
    result = formatter.parse("$1,234.56", "en-US")
    assert result == Decimal("1234.56")

def test_parse_eur_es_es():
    """Test parsing Spanish formatted string."""
    formatter = AmountFormatter()
    result = formatter.parse("1.234,56 €", "es-ES")
    assert result == Decimal("1234.56")

def test_get_locale_rules():
    """Test locale rules retrieval."""
    formatter = AmountFormatter()

    rules_us = formatter.get_locale_rules("en-US")
    assert rules_us.thousands_separator == ","
    assert rules_us.decimal_symbol == "."

    rules_de = formatter.get_locale_rules("de-DE")
    assert rules_de.thousands_separator == "."
    assert rules_de.decimal_symbol == ","

def test_unsupported_locale_error():
    """Test error on invalid locale."""
    formatter = AmountFormatter()
    with pytest.raises(UnsupportedLocaleError):
        formatter.format(Decimal("1234.56"), "USD", "invalid-locale")
```

---

## Multi-Domain Examples

### Finance: Multi-Currency Display

**Use Case:** Display transaction amounts in user's locale

```python
formatter = AmountFormatter()

# US user sees:
usd_display = formatter.format(Decimal("1234.56"), "USD", "en-US")
# → "$1,234.56"

# Mexican user sees:
mxn_display = formatter.format(Decimal("1234.56"), "MXN", "es-MX")
# → "$1,234.56"

# German user sees:
eur_display = formatter.format(Decimal("1234.56"), "EUR", "de-DE")
# → "1.234,56 €"
```

---

### Healthcare: Medical Bill Display

**Use Case:** Display medical bill in patient's locale

```python
# US patient
bill_usd = formatter.format(Decimal("5000.00"), "USD", "en-US")
# → "$5,000.00"

# French patient
bill_eur = formatter.format(Decimal("4500.00"), "EUR", "fr-FR")
# → "4 500,00 €"
```

---

### E-commerce: Product Pricing

**Use Case:** Display product price in customer's currency and locale

```python
# Product costs $99.99 USD
product_price = Decimal("99.99")

# Display to US customer
display_us = formatter.format(product_price, "USD", "en-US")
# → "$99.99"

# Display to Spanish customer (assume converted to EUR)
price_eur = Decimal("92.50")  # After conversion
display_es = formatter.format(price_eur, "EUR", "es-ES")
# → "92,50 €"
```

---

### Travel: Expense Reporting

**Use Case:** Display expense in employee's home currency format

```python
# Employee based in Germany
expense_eur = Decimal("250.75")

display = formatter.format(expense_eur, "EUR", "de-DE")
# → "250,75 €"
```

---

## Observability

### Metrics

```python
# Formatting metrics
amount.format.count (counter, labels: locale, currency)
amount.format.latency (histogram)
amount.parse.count (counter, labels: locale)
amount.parse.latency (histogram)

# Error metrics
amount.errors.unsupported_locale (counter, labels: locale)
amount.errors.unsupported_currency (counter, labels: currency)
amount.errors.invalid_format (counter)
```

### Structured Logs

```json
{
  "timestamp": "2024-11-01T10:00:00Z",
  "level": "INFO",
  "event": "amount_formatted",
  "amount": 1234.56,
  "currency": "USD",
  "locale": "en-US",
  "formatted": "$1,234.56",
  "latency_ms": 2
}
```

---

## Security Considerations

**Input Validation:**
- Locale: Must be in BCP 47 (prevent arbitrary strings)
- Currency: Must be in ISO 4217 (prevent injection)
- Amount: Must be finite decimal (prevent overflow)

**Format String Safety:**
- Use whitelisted format strings only
- Sanitize locale-specific separators (prevent injection)

---

## Configuration

```yaml
# config/amount_formatter.yaml
default_locale: en-US

locale_overrides:
  # Force specific rules for certain locales
  en-US:
    thousands_separator: ","
    decimal_symbol: "."
    currency_symbol_position: "prefix"

  es-ES:
    thousands_separator: "."
    decimal_symbol: ","
    currency_symbol_position: "suffix"

compact_notation:
  enabled: true
  thresholds:
    K: 1000        # $1,000 → $1K
    M: 1000000     # $1,000,000 → $1M
    B: 1000000000  # $1,000,000,000 → $1B
```

---

## Changelog

**v1.0.0 (2024-11-01):**
- Initial implementation
- Support for 150+ BCP 47 locales
- Compact notation formatting
- Crypto currency support

**v1.1.0 (Future):**
- Custom locale rules support
- Accounting format (parentheses for negatives)
- Indian numbering system (1,23,45,678)

---

## References

- BCP 47: Language tags for identifying languages
- Babel documentation: https://babel.pocoo.org/
- ISO 4217: Currency codes
- Unicode CLDR: Common Locale Data Repository

---

**Total Lines:** ~870
