# ADR-0015: Rule Precedence System

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 3.8 Cluster Rules

---

## Context

After raw extraction, merchant names are messy and inconsistent:

```
Raw extractions:
  - "AMAZON.COM*AB12CD"
  - "Amazon.com - Marketplace"
  - "AMZN Mktp US*AB12CD"
  - "amazon web services"
```

**Normalization rules** clean these into canonical forms:
```
Normalized:
  - "Amazon.com" (for retail purchases)
  - "Amazon Web Services" (for AWS charges)
```

**The problem:**
- Multiple rules can match the same raw string
- Execution order determines final result
- Without deterministic precedence, results are unpredictable

**Example conflict:**
```python
rules = [
    Rule(type="regex", pattern="AMAZON.*", canonical="Amazon.com"),
    Rule(type="exact", pattern="AMAZON.COM*AB12CD", canonical="Amazon Prime"),
    Rule(type="fuzzy", pattern="amazon", canonical="Amazon Retail")
]

# Which rule wins for "AMAZON.COM*AB12CD"?
# All three match! Need precedence system.
```

---

## Decision

**We use priority-based precedence with rule type fallback.**

### Execution Order

```python
def normalize(raw_merchant: str, rules: List[Rule]) -> str:
    # Step 1: Sort rules by priority (descending), then type precedence
    sorted_rules = sorted(
        rules,
        key=lambda r: (
            -r.priority,  # Higher priority first (descending)
            RULE_TYPE_PRECEDENCE[r.type]  # Type precedence
        )
    )

    # Step 2: Apply first matching rule (early stop)
    for rule in sorted_rules:
        if rule.matches(raw_merchant):
            return rule.canonical_name

    # Step 3: No match → return raw (unchanged)
    return raw_merchant
```

### Priority System (0-100)

- **100** — Highest priority (executes first)
- **0** — Lowest priority (executes last)
- **Default priority by type:**
  - `exact`: 100 (most specific)
  - `regex`: 90
  - `fuzzy`: 70
  - `soundex`: 50 (least specific)

### Type Precedence (for same priority)

```python
RULE_TYPE_PRECEDENCE = {
    "exact": 1,    # Highest specificity
    "regex": 2,
    "fuzzy": 3,
    "soundex": 4   # Lowest specificity
}
```

### Early Stop Strategy

- **Stop after first match** (no rule chaining)
- Prevents cascading transformations (e.g., `A → B → C`)
- Ensures predictable output

---

## Rationale

### 1. Deterministic Behavior

Same input + same rules = same output (always)

```python
# Guaranteed deterministic
normalize("AMAZON.COM*AB12CD", rules)  # → "Amazon Prime"
normalize("AMAZON.COM*AB12CD", rules)  # → "Amazon Prime" (same)
```

Without precedence, order depends on:
- Database query order (nondeterministic)
- Parallel execution timing (nondeterministic)

### 2. User Control for Power Users

Users can override default precedence:

```json
{
  "type": "fuzzy",
  "pattern": "amazon",
  "canonical": "Amazon Retail",
  "priority": 95  // Override default (70) to execute before regex (90)
}
```

### 3. Sensible Defaults

- **Exact match** most specific → highest priority
- **Soundex** least specific → lowest priority
- Matches user intuition ("specific rules override generic rules")

### 4. Performance Optimization

Early stop prevents redundant checks:

```python
# Without early stop (100 rules)
for rule in rules:
    if rule.matches(merchant):
        results.append(rule.canonical)  # All matches collected

# With early stop (100 rules)
for rule in sorted_rules:
    if rule.matches(merchant):
        return rule.canonical  # Stop immediately
```

Average case: 5 checks vs 100 checks (20x faster)

---

## Consequences

### ✅ Positive

- **Predictable**: Same input always produces same output
- **User control**: Power users can tune priority
- **Performance**: Early stop reduces redundant checks
- **Intuitive defaults**: Exact > regex > fuzzy > soundex

### ⚠️ Trade-offs

- **Learning curve**: Users must understand priority system
- **Conflict resolution**: Manual priority adjustment needed for conflicts
- **No chaining**: Can't apply multiple transformations (intentional)

### 🔴 Risks (Mitigated)

- **Risk**: Two rules same priority + type → arbitrary tie-break
  - **Mitigation**: Use `rule_id` (lexicographic sort) as final tie-breaker
  - **Mitigation**: UI warns user about conflicts

- **Risk**: User sets wrong priority → unexpected results
  - **Mitigation**: Rule testing interface shows execution order preview
  - **Mitigation**: Validation warns if high-priority rule shadows low-priority

---

## Alternatives Considered

### Alternative A: FIFO Order (Creation Time) (Rejected)

**Approach:** Execute rules in order they were created

**Pros:**
- Simple to implement
- No priority metadata needed

**Cons:**
- **Unpredictable**: User has no control over execution order
- **Hard to reason about**: "Which rule was created first?" (not memorable)
- **Fragile**: Deleting/recreating rule changes order

**Why rejected:** Not user-controllable.

---

### Alternative B: Specificity-Based (Rejected)

**Approach:** Calculate "specificity score" for each rule

```python
specificity = (
    len(pattern) * 0.5 +           # Longer pattern = more specific
    (1.0 if type == "exact" else 0.5) +
    regex_complexity(pattern) * 0.3
)
```

**Pros:**
- Automatic (no user configuration)
- Intuitive (longer patterns more specific)

**Cons:**
- **Hard to define objectively**: Is `"AMAZON.*"` more specific than `"amzn"`?
- **Opaque**: User doesn't understand why rule executed first
- **Can't override**: User has no control

**Why rejected:** Too complex, not explainable.

---

### Alternative C: User Manual Ordering (Drag & Drop) (Rejected)

**Approach:** UI lets user drag rules to reorder execution

**Pros:**
- Visual control
- No priority numbers needed

**Cons:**
- **Tedious**: 100+ rules = painful to order manually
- **No bulk operations**: Can't set priority for all "exact" rules
- **Mobile unfriendly**: Drag-and-drop doesn't work on mobile

**Why rejected:** Doesn't scale to large rulesets.

---

### Alternative D: Apply All Rules (Chaining) (Rejected)

**Approach:** Apply every matching rule in sequence

```python
result = raw_merchant
for rule in sorted_rules:
    if rule.matches(result):
        result = rule.canonical  # Transform in place
return result
```

**Example:**
```
"AMAZON.COM*AB12CD"
  → "Amazon.com" (Rule 1: strip suffix)
  → "Amazon" (Rule 2: remove domain)
  → "AMZN" (Rule 3: abbreviate)
```

**Pros:**
- Compositional (complex transformations from simple rules)

**Cons:**
- **Unpredictable**: Cascading effects hard to debug
- **Order-dependent**: Rule order critically important
- **Performance**: Must check all rules (can't early stop)

**Why rejected:** Too complex, hard to debug.

---

## Implementation Notes

### Rule Type Definitions

```python
class NormalizationRule:
    type: Literal["exact", "regex", "fuzzy", "soundex"]
    pattern: str
    canonical: str
    priority: int = None  # Auto-set if not provided

    def __post_init__(self):
        # Auto-set priority based on type
        if self.priority is None:
            self.priority = {
                "exact": 100,
                "regex": 90,
                "fuzzy": 70,
                "soundex": 50
            }[self.type]
```

### Fuzzy Match Example (Levenshtein)

```python
def fuzzy_match(pattern: str, text: str, threshold: float = 0.8) -> bool:
    distance = levenshtein(pattern.lower(), text.lower())
    max_len = max(len(pattern), len(text))
    similarity = 1 - (distance / max_len)
    return similarity >= threshold
```

### Soundex Example

```python
import soundex

def soundex_match(pattern: str, text: str) -> bool:
    return soundex.encode(pattern) == soundex.encode(text)

# Example:
soundex_match("Amazon", "Amazn")  # True (A525 = A525)
```

---

## Related Decisions

- **ADR-0005**: Configurable normalization rules (this ADR defines execution order)
- **ADR-0008**: Cluster merge strategies (rules produce clusters, then merge logic applies)
- **Future ADR**: Rule conflict detection UI (warn users about shadowing)

---

**References:**
- Vertical 3.8: [docs/verticals/3.8-cluster-rules.md](../verticals/3.8-cluster-rules.md)
- NormalizationRuleStore: [docs/primitives/ol/NormalizationRuleStore.md](../primitives/ol/NormalizationRuleStore.md)
- Schema: [docs/schemas/normalization-rule.schema.json](../schemas/normalization-rule.schema.json)
