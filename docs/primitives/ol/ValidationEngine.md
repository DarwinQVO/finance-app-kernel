# OL Primitive: ValidationEngine

**Type**: Validation / Business Rules
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.3 (Normalization)

---

## Purpose

Rule-based validation for individual fields during normalization. Provides reusable validators (dates, amounts, currencies, etc.) that work across domains. Separates validation logic from transformation logic.

---

## Interface Contract

```python
class ValidationEngine:
    def validate_date(self, raw: str, locale: str) -> datetime:
        """
        Parse and validate date from various formats.
        Raises ValidationError if unparseable or out of range.
        """
        pass

    def validate_amount(self, raw: str) -> Decimal:
        """
        Parse and validate numeric amount.
        Handles: "-50.00", "(50.00)", "1,234.56"
        Raises ValidationError if unparseable.
        """
        pass

    def validate_currency(self, code: str) -> str:
        """
        Validate ISO 4217 currency code.
        Raises ValidationError if invalid.
        """
        pass

    def clean_text(self, raw: str) -> str:
        """
        Clean and normalize text.
        - Strip whitespace
        - Collapse multiple spaces
        - Optional: Titlecase
        """
        pass
```

---

## Behavior Examples

**Date validation:**
```python
engine = ValidationEngine()

# Success cases
assert engine.validate_date("01/15/2024", "en_US") == datetime(2025, 1, 15)
assert engine.validate_date("15/01/2024", "es_MX") == datetime(2025, 1, 15)
assert engine.validate_date("2024-01-15", "ISO") == datetime(2025, 1, 15)

# Error cases
engine.validate_date("invalid", "en_US")  # Raises ValidationError
engine.validate_date("01/32/2024", "en_US")  # Raises ValidationError (invalid day)
engine.validate_date("01/01/2050", "en_US")  # Raises ValidationError (too far in future)
```

**Amount validation:**
```python
assert engine.validate_amount("-50.00") == Decimal("-50.00")
assert engine.validate_amount("(50.00)") == Decimal("-50.00")  # Accounting notation
assert engine.validate_amount("1,234.56") == Decimal("1234.56")

engine.validate_amount("N/A")  # Raises ValidationError
```

---

## Multi-Domain Applicability

**Finance:** Validate dates, amounts, currency codes
**Healthcare:** Validate numeric test values, unit formats, date ranges
**Legal:** Validate clause dates, monetary amounts in contracts
**Research:** Validate publication dates, DOIs, ISSNs
**Manufacturing:** Validate sensor values, measurement units, timestamps
**Media:** Validate timestamps, duration formats, speaker IDs

---

## Related Primitives

- **Normalizer**: Uses ValidationEngine for field-level validation
- **NormalizationRuleSet**: Provides locale and format configurations

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
