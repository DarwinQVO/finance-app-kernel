# OL Primitive: Normalizer

**Type**: Data Transformation / Validation
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.3 (Normalization)

---

## Purpose

Universal interface for transforming raw observations into validated canonical records. Normalizers apply domain-specific validation rules (dates, amounts, business logic) to produce clean, validated data ready for analysis.

---

## Interface Contract

```python
from abc import ABC, abstractmethod

class Normalizer(ABC):
    @property
    @abstractmethod
    def version(self) -> str:
        """Semantic version of normalizer (e.g., '1.0.0')"""
        pass

    @abstractmethod
    def normalize(
        self,
        observation: Observation,
        rules: NormalizationRuleSet
    ) -> Canonical:
        """
        Transform raw observation into validated canonical.

        Args:
            observation: Raw observation from ObservationStore
            rules: Domain-specific normalization rules

        Returns:
            Validated canonical record

        Raises:
            ValidationError: If observation data is invalid
        """
        pass
```

---

## Behavior

**Transformation example (Finance):**
```python
# Input: Raw observation
observation = ObservationTransaction(
    upload_id="UL_abc123",
    row_id=0,
    raw_data={
        "date": "01/15/2024",        # String (ambiguous format)
        "amount": "-5.75",            # String
        "description": "  STARBUCKS #1234  "  # Dirty
    }
)

# Output: Canonical transaction
canonical = normalizer.normalize(observation, rules)
# canonical.date = "2025-01-15T00:00:00Z" (ISO 8601)
# canonical.amount = -5.75 (Decimal)
# canonical.description = "Starbucks #1234" (cleaned)
# canonical.merchant = "Starbucks" (extracted)
# canonical.category = "Food & Drink" (inferred)
```

---

## Responsibilities

✅ **DOES:**
- Validate field formats (dates, amounts, currencies)
- Transform raw strings → typed values (ISO dates, decimals)
- Clean text (trim spaces, normalize case)
- Extract structured data (merchant from description)
- Infer metadata (category from merchant)
- Apply business rules (duplicate detection)
- Generate confidence scores

❌ **DOES NOT:**
- Modify raw observations (they remain immutable)
- Update UploadRecord.status (Coordinator only)
- Make multi-transaction decisions (transfers → vertical 1.4)

---

## Multi-Domain Applicability

**Finance:** ObservationTransaction → CanonicalTransaction (dates, amounts, categories)
**Healthcare:** ObservationLabResult → CanonicalLabResult (numeric values, units, ranges)
**Legal:** ObservationClause → CanonicalClause (clause types, obligations, dates)
**Research:** ObservationCitation → CanonicalCitation (authors, titles, DOIs)
**Manufacturing:** ObservationMeasurement → CanonicalMeasurement (sensor values, units)
**Media:** ObservationUtterance → CanonicalUtterance (timestamps, speaker IDs)

---

## Related Primitives

- **ValidationEngine**: Used by Normalizer for field-level validation
- **NormalizationRuleSet**: Configuration for domain-specific rules
- **ObservationStore**: Source of raw observations
- **CanonicalStore**: Destination for normalized canonicals
- **NormalizationLog**: Records normalization execution details

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
