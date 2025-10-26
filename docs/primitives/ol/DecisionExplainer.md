# DecisionExplainer (OL Primitive)

## Definition

**DecisionExplainer** explains how normalization decisions were made for each field in a canonical record. It shows which rules matched, confidence scores, alternatives considered, and whether manual overrides occurred.

**Problem it solves:**
- Users don't understand WHY a transaction was categorized as "Business" vs "Personal"
- Debugging normalization is hard (which rule matched? what was the pattern?)
- Confidence scores are opaque (why 0.87 vs 0.95?)
- Manual corrections are not traceable (was this auto-decided or manually fixed?)

**Solution:**
- Human-readable decision explanations
- Rule pattern display (e.g., "merchant='uber' → Food")
- Confidence score breakdown
- Alternative options that were considered but rejected
- Manual override tracking

---

## Interface Contract

```python
from typing import List, Optional
from dataclasses import dataclass

@dataclass
class DecisionExplanation:
    """Explanation of a single field's normalization decision"""
    field: str  # Field name (e.g., "category", "merchant", "amount_usd")
    decision: str  # Human-readable explanation
    rule_applied: Optional[str]  # Rule ID (e.g., "category_merchant_match_v1.2")
    rule_pattern: Optional[str]  # Pattern that matched (e.g., "merchant='uber' → Food")
    confidence: float  # 0-1 score
    alternatives_considered: List[str]  # Other options that were rejected
    manual_override: bool  # True if manually corrected, False if auto-decided
    metadata: dict  # Additional context (exchange rates, data sources, etc.)

class DecisionExplainer:
    """
    Explains normalization decisions for canonical records.

    Domain-agnostic - works on ANY normalization rule set.
    """

    def __init__(
        self,
        normalization_log_store,
        rule_registry
    ):
        """
        Initialize with normalization logs and rule registry.

        Args:
            normalization_log_store: Source of normalization execution logs
            rule_registry: Source of normalization rules (for pattern lookup)
        """

    def explain_all(
        self,
        canonical_id: str
    ) -> List[DecisionExplanation]:
        """
        Explains ALL field decisions for a canonical record.

        Args:
            canonical_id: Canonical record ID

        Returns:
            List of decision explanations (one per field that was normalized)

        Example:
            [
                DecisionExplanation(field="merchant", decision="Normalized 'UBER EATS' → 'Uber Eats'", ...),
                DecisionExplanation(field="category", decision="Categorized as Food", ...),
                DecisionExplanation(field="amount_usd", decision="Converted 640 MXN → $32.82", ...)
            ]
        """

    def explain_field(
        self,
        canonical_id: str,
        field_name: str
    ) -> DecisionExplanation:
        """
        Explains decision for a SPECIFIC field.

        Args:
            canonical_id: Canonical record ID
            field_name: Field to explain (e.g., "category")

        Returns:
            Single decision explanation

        Raises:
            FieldNotFoundError: If field doesn't exist in canonical
            NoDecisionFoundError: If field was not normalized (copied as-is)
        """

    def get_rule_pattern(
        self,
        rule_id: str
    ) -> str:
        """
        Retrieves human-readable pattern for a rule.

        Args:
            rule_id: Rule ID (e.g., "category_merchant_match_v1.2")

        Returns:
            Human-readable pattern (e.g., "merchant='uber' → Food")
        """
```

---

## Multi-Domain Applicability

**Finance:**
```python
explainer = DecisionExplainer(...)
explanation = explainer.explain_field("can_bofa_001", "category")
# Returns: "Categorized as 'Business > Software' because merchant='openai' matches pattern"
```

**Healthcare:**
```python
explainer = DecisionExplainer(...)
explanation = explainer.explain_field("can_lab_456", "abnormal_flag")
# Returns: "Flagged as abnormal because value=180 > threshold=140 (high blood pressure)"
```

**Legal:**
```python
explainer = DecisionExplainer(...)
explanation = explainer.explain_field("can_clause_789", "risk_level")
# Returns: "Classified as high-risk because clause contains keyword: 'unlimited liability'"
```

**Research (RSRCH - Utilitario):**
```python
explainer = DecisionExplainer(...)
explanation = explainer.explain_field("fact_sama_helion_001", "source_credibility")
# Returns: "Marked as high credibility (0.95) because source is TechCrunch (0.9) + confirmed in Lex Fridman podcast (0.98)"
```

**Manufacturing:**
```python
explainer = DecisionExplainer(...)
explanation = explainer.explain_field("can_qc_567", "pass_fail")
# Returns: "Marked as FAIL because measurement=12.3 outside tolerance range 10.0-12.0"
```

**Media:**
```python
explainer = DecisionExplainer(...)
explanation = explainer.explain_field("can_transcript_999", "confidence")
# Returns: "Low confidence (0.45) because audio quality < 0.5 (background noise detected)"
```

**Domain-agnostic nature:** Same interface across all domains. Only field names and rule patterns change.

---

## Responsibilities

**DecisionExplainer IS responsible for:**
- ✅ Explaining normalization decisions in human-readable form
- ✅ Showing which rules matched and their patterns
- ✅ Displaying confidence scores
- ✅ Listing alternatives that were considered
- ✅ Tracking manual overrides vs auto-decisions

**DecisionExplainer is NOT responsible for:**
- ❌ Executing normalization (that's Normalizer)
- ❌ Storing normalization logs (that's NormalizationLog)
- ❌ Rendering UI explanations (that's IL layer)
- ❌ Modifying decisions (that's Corrections Flow 4.3)
- ❌ Creating new rules (that's Rule Registry 3.8)

---

## Implementation Notes

### Storage Pattern

**No persistent storage** - DecisionExplainer reads from existing logs and rules.

It reads from:
- NormalizationLog (execution logs with rules applied)
- RuleRegistry (rule definitions with patterns)
- CanonicalStore (canonical record to verify fields)

### Decision Reconstruction

```python
def explain_field(canonical_id, field_name):
    # 1. Load normalization log for this canonical
    log = normalization_log_store.get(canonical_id)

    # 2. Find field-specific entry in log
    field_log = log["fields"][field_name]

    # 3. Look up rule pattern from registry
    rule_pattern = rule_registry.get_pattern(field_log["rule_id"])

    # 4. Assemble human-readable explanation
    return DecisionExplanation(
        field=field_name,
        decision=f"Normalized '{field_log['input']}' → '{field_log['output']}'",
        rule_applied=field_log["rule_id"],
        rule_pattern=rule_pattern,
        confidence=field_log["confidence"],
        alternatives_considered=field_log["alternatives"],
        manual_override=field_log.get("manual", False),
        metadata=field_log.get("metadata", {})
    )
```

### Confidence Score Breakdown

```python
# Example: Explain why confidence is 0.87 for category decision
{
  "field": "category",
  "decision": "Categorized as 'Personal > Travel > Flights'",
  "rule_applied": "category_merchant_match",
  "confidence": 0.87,
  "confidence_breakdown": {
    "merchant_match": 0.95,  # Strong merchant pattern match
    "amount_range": 0.80,    # Amount within typical range for flights
    "date_pattern": 0.85     # Transaction date aligns with travel season
  },
  "confidence_formula": "min(merchant_match, amount_range, date_pattern) = 0.80 * 1.1 (boost) = 0.87"
}
```

### Alternative Display

```python
# Show alternatives that were considered but rejected
{
  "field": "category",
  "decision": "Categorized as 'Business > Software'",
  "alternatives_considered": [
    "Personal > Entertainment (rejected: confidence 0.45)",
    "Business > Other (rejected: too generic)"
  ]
}
```

### Manual Override Tracking

```python
# If user manually corrected this field (via Corrections Flow 4.3)
{
  "field": "category",
  "decision": "Categorized as 'Business > Travel' (manual override)",
  "rule_applied": None,  # No rule applied (manual)
  "confidence": 1.0,  # Manual = 100% confidence
  "manual_override": True,
  "metadata": {
    "corrected_by": "eugenio@example.com",
    "corrected_at": "2025-05-23T14:30:00Z",
    "reason": "This was a client meeting, not personal travel"
  }
}
```

---

## Related Primitives

**Upstream dependencies:**
- **NormalizationLog** (1.3) - Source of normalization execution data
- **RuleRegistry** (3.8) - Source of rule patterns and definitions
- **CanonicalStore** (1.3) - Source of canonical records

**Downstream consumers:**
- **ProvenanceTracer** (2.2) - Uses DecisionExplainer to assemble drill-down data
- **DrillDownPanel** (IL) - Renders decision explanations in UI
- **RuleExplanationCard** (IL) - Visual card component for decision display

**Composition pattern:**
```python
# DecisionExplainer is used by ProvenanceTracer
class ProvenanceTracer:
    def __init__(self, ..., decision_explainer):
        self.explainer = decision_explainer

    def trace_lineage(self, canonical_id, user_id):
        ...
        # Get decision explanations
        decisions = self.explainer.explain_all(canonical_id)

        return DrillDownData(
            canonical=...,
            decisions=decisions,  # Includes all field explanations
            ...
        )
```

---

## Example Usage

### Finance Domain

```python
from primitives.ol import DecisionExplainer

# Initialize
explainer = DecisionExplainer(
    normalization_log_store=normalization_log_store,
    rule_registry=rule_registry
)

# Explain all fields
explanations = explainer.explain_all("can_bofa_20250426_001")

for exp in explanations:
    print(f"{exp.field}: {exp.decision}")
    print(f"  Rule: {exp.rule_pattern}")
    print(f"  Confidence: {exp.confidence}")
    print()

# Output:
# merchant: Normalized 'AEROMEXICO MEX' → 'Aeromexico'
#   Rule: merchant_normalization_v1.2
#   Confidence: 0.95
#
# category: Categorized as 'Personal > Travel > Flights'
#   Rule: merchant='aeromexico' → Travel > Flights
#   Confidence: 0.87
#
# amount_usd: Converted 4,516 MXN → $231.78 USD
#   Rule: currency_conversion_v1.0
#   Confidence: 1.0
```

### Explain Specific Field

```python
# User asks: "Why was this categorized as Personal?"
exp = explainer.explain_field("can_bofa_20250426_001", "category")

print(exp.decision)
# "Categorized as 'Personal > Travel > Flights'"

print(exp.rule_pattern)
# "merchant='aeromexico' → Travel > Flights"

print(exp.alternatives_considered)
# ["Business > Travel (rejected: no business keywords in merchant)"]

print(exp.manual_override)
# False (auto-decided, not manually corrected)
```

### Healthcare Example

```python
# Same explainer, different domain
exp = explainer.explain_field("can_lab_result_456", "abnormal_flag")

print(exp.decision)
# "Flagged as abnormal because value=180 > threshold=140 (high blood pressure)"

print(exp.confidence)
# 1.0 (threshold-based rules are 100% confident)

print(exp.metadata)
# {"threshold": 140, "value": 180, "unit": "mg/dL", "reference_range": "70-140"}
```

**Key insight:** Same DecisionExplainer, different domains. It doesn't know it's explaining transaction categories vs lab abnormalities - it just reads logs and rules.
