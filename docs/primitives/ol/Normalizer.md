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

**🔧 P1 Fix Applied:** Added generic types for type safety across domains

```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic, List
from dataclasses import dataclass

TObservation = TypeVar('TObservation')
TCanonical = TypeVar('TCanonical')

@dataclass
class NormalizationResult(Generic[TCanonical]):
    """
    Container for normalization output with metadata.
    """
    canonical: TCanonical
    confidence: float  # 0.0-1.0
    applied_rules: List[str]  # Rule IDs that were applied
    warnings: List[str]  # Non-fatal validation warnings
    metadata: dict  # Additional context

class Normalizer(ABC, Generic[TObservation, TCanonical]):
    @property
    @abstractmethod
    def version(self) -> str:
        """Semantic version of normalizer (e.g., '1.0.0')"""
        pass

    @abstractmethod
    def normalize(
        self,
        observation: TObservation,
        rules: NormalizationRuleSet
    ) -> NormalizationResult[TCanonical]:
        """
        Transform raw observation into validated canonical.

        Args:
            observation: Raw observation from ObservationStore (type-safe)
            rules: Domain-specific normalization rules

        Returns:
            NormalizationResult containing:
            - canonical: Validated canonical record (type-safe)
            - confidence: Float [0.0, 1.0]
            - applied_rules: List of rule IDs
            - warnings: Non-fatal validation warnings

        Raises:
            ValidationError: If observation data is invalid (fatal)
        """
        pass
```

**Type Safety Example:**

```python
# Finance implementation
class FinanceNormalizer(Normalizer[ObservationTransaction, CanonicalTransaction]):
    def normalize(
        self,
        obs: ObservationTransaction,  # Type-checked at compile time
        rules: NormalizationRuleSet
    ) -> NormalizationResult[CanonicalTransaction]:  # Return type enforced
        canonical = CanonicalTransaction(
            canonical_id=generate_id(obs.upload_id, obs.row_id),
            date=parse_iso_date(obs.raw_data['date']),
            amount=Decimal(obs.raw_data['amount']),
            ...
        )
        return NormalizationResult(
            canonical=canonical,
            confidence=0.95,
            applied_rules=['date_iso', 'amount_decimal'],
            warnings=[],
            metadata={}
        )

# Healthcare implementation
class HealthcareNormalizer(Normalizer[ObservationLabResult, CanonicalLabResult]):
    def normalize(
        self,
        obs: ObservationLabResult,  # Different type, same interface
        rules: NormalizationRuleSet
    ) -> NormalizationResult[CanonicalLabResult]:
        canonical = CanonicalLabResult(
            result_id=generate_id(obs.upload_id, obs.row_id),
            test_date=parse_iso_date(obs.raw_data['date']),
            value=Decimal(obs.raw_data['value']),
            unit=obs.raw_data['unit'],
            ...
        )
        return NormalizationResult(
            canonical=canonical,
            confidence=0.98,
            applied_rules=['date_iso', 'value_decimal', 'unit_loinc'],
            warnings=[],
            metadata={}
        )
```

**Benefits of Generic Types:**
- ✅ Compile-time type checking (catch errors before runtime)
- ✅ Better IDE autocomplete (knows canonical.date exists)
- ✅ Self-documenting code (clear input/output types)
- ✅ Prevents mixing domains (can't pass ObservationTransaction to HealthcareNormalizer)
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
- Make multi-transaction decisions (transfers → vertical 3.5 or 3.9)

---

## Multi-Domain Applicability

**Finance:** ObservationTransaction → CanonicalTransaction (dates, amounts, categories)
**Healthcare:** ObservationLabResult → CanonicalLabResult (numeric values, units, ranges)
**Legal:** ObservationClause → CanonicalClause (clause types, obligations, dates)
**Research (RSRCH - Utilitario):** RawFact → CanonicalFact (entity names, fact claims, source URLs)
**Manufacturing:** ObservationMeasurement → CanonicalMeasurement (sensor values, units)
**Media:** ObservationUtterance → CanonicalUtterance (timestamps, speaker IDs)

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Transform raw bank CSV row into validated canonical transaction
**Example:** Raw observation `{"date": "01/15/2024", "amount": "-5.75", "description": "  STARBUCKS #1234  "}` → Canonical with ISO date "2025-01-15T00:00:00Z", Decimal amount -5.75, merchant "Starbucks", category "Food & Drink"
**Fields validated:** `date` (format: MM/DD/YYYY → ISO 8601), `amount` (string → Decimal), `merchant` (extracted from description), `category` (inferred)
**Rules applied:** date_iso, amount_decimal, merchant_extract, category_infer
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Transform raw lab result text into validated canonical lab result
**Example:** Raw observation `{"date": "2024-03-15", "test": "Glucose", "value": "95", "unit": "mg/dL"}` → Canonical with normalized test code (LOINC 2345-7), validated range (normal: 70-100), flagged abnormal results
**Fields validated:** `test_date` (ISO 8601), `value` (string → Decimal), `unit` (standardized to UCUM), `reference_range` (age/gender-specific)
**Rules applied:** date_iso, value_decimal, unit_ucum, range_validate, loinc_map
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Transform raw contract clause text into structured canonical clause
**Example:** Raw observation `{"clause": "Payment due within 30 days of invoice date", "section": "3.2"}` → Canonical with clause_type "payment_terms", obligation_type "payment", deadline_days 30, enforceability "mandatory"
**Fields validated:** `clause_type` (classified), `obligation_type` (extracted), `deadline_days` (parsed from text), `enforceability` (inferred)
**Rules applied:** clause_classify, obligation_extract, deadline_parse, enforceability_infer
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** Transform raw founder mention into canonical entity record
**Example:** Raw observation `{"text": "@sama invested $375M in OpenAI", "source": "techcrunch.com", "date": "2024-02-25"}` → Canonical with entity_name "Sam Altman", entity_type "person", company "OpenAI", investment_amount 375000000 (parsed), source_type "news"
**Fields validated:** `entity_name` (normalized "@sama" → "Sam Altman"), `investment_amount` (string "$375M" → Decimal 375000000), `source_type` (classified), `fact_date` (ISO 8601)
**Rules applied:** entity_normalize, amount_parse, source_classify, date_iso
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Transform raw product scrape data into validated canonical product record
**Example:** Raw observation `{"title": "  iPhone 15 Pro Max - 256GB  ", "price": "$1,199.99", "category": "Electronics > Phones"}` → Canonical with clean title "iPhone 15 Pro Max 256GB", price Decimal(1199.99), category hierarchy ["Electronics", "Phones"], SKU generated
**Fields validated:** `title` (cleaned, normalized), `price` (string → Decimal), `category` (hierarchy parsed), `SKU` (generated), `availability` (inferred)
**Rules applied:** text_clean, price_decimal, category_parse, sku_generate
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (generic interface with TypeVar[TObservation, TCanonical])
**Reusability:** High (same normalize() method works across all domains, only validation rules differ)

---

## Related Primitives

- **ValidationEngine**: Used by Normalizer for field-level validation
- **NormalizationRuleSet**: Configuration for domain-specific rules
- **ObservationStore**: Source of raw observations
- **CanonicalStore**: Destination for normalized canonicals
- **NormalizationLog**: Records normalization execution details

---

**Last Updated**: 2025-10-27
**Maturity**: Spec complete, ready for implementation
