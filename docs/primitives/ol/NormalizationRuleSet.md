# OL Primitive: NormalizationRuleSet

**Type**: Configuration / Business Rules
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.3 (Normalization)
**Related ADR**: ADR-0005 (Configurable Normalization Rules)

---

## Purpose

Externalized, versioned configuration for domain-specific normalization rules. Enables updating rules without code changes, supports per-source-type customization, and provides auditability (which rules produced which canonicals).

---

## Schema

```python
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class NormalizationRuleSet:
    version: str                    # "2024.10"
    source_type: str                # "bofa_pdf"
    date_locale: str                # "en_US", "es_MX", "fr_FR"
    date_formats: List[str]         # ["MM/DD/YYYY", "M/D/YYYY"]
    amount_decimal_separator: str   # "." or ","
    merchant_whitelist: Dict[str, str]  # {"STARBUCKS": "Starbucks"}
    category_rules: List[CategoryRule]
    duplicate_tolerance_days: int   # 1 (allow ±1 day for duplicates)
    large_amount_threshold: float   # 10000.00
```

---

## Example (Finance Domain)

```json
{
  "version": "2024.10",
  "source_type": "bofa_pdf",
  "date_locale": "en_US",
  "date_formats": ["MM/DD/YYYY"],
  "amount_decimal_separator": ".",
  "merchant_whitelist": {
    "STARBUCKS": "Starbucks",
    "AMAZON.COM": "Amazon"
  },
  "category_rules": [
    {"merchant_pattern": "Starbucks", "category": "Food & Drink", "confidence": 0.95},
    {"merchant_pattern": "Amazon", "category": "Shopping", "confidence": 0.90}
  ],
  "duplicate_tolerance_days": 1,
  "large_amount_threshold": 10000.00
}
```

---

## Storage

**Path:** `config/normalization-rules/{source_type}_{version}.json`
**Versioning:** Git-tracked, semantic versioning (YYYY.MM format)
**Deployment:** CI/CD deploys updated configs, normalizer hot-reloads

---

## Multi-Domain Applicability

**Finance:** Date locales, merchant whitelists, category taxonomies
**Healthcare:** Unit conversions, normal ranges by age/gender, test name normalization
**Legal:** Clause taxonomies, obligation patterns, jurisdiction-specific rules
**Research:** Citation formats (APA, MLA, Chicago), field extraction patterns
**Manufacturing:** Sensor calibration factors, measurement thresholds, unit conversions

---

## Related Primitives

- **Normalizer**: Loads and applies rules from RuleSet
- **ValidationEngine**: Uses locale/format config from RuleSet
- **CanonicalStore**: Canonicals reference `rule_set_version` used

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
