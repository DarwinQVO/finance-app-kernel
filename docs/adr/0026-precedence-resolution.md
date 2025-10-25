# ADR-0026: Static Precedence Rules

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 4.3 (affects all field value resolution)

---

## Context

When a field has multiple value sources (manual override, normalization rule, extraction, default), we must decide which value to use. This is **precedence resolution**.

**The challenge:**

```
Transaction field "merchant" has 4 sources:
1. Manual override: "Amazon Marketplace" (user corrected on 2025-10-24)
2. Normalization rule: "Amazon" (rule applied on 2025-10-21)
3. Extraction: "AMZN MKTP US*AB123" (parsed from PDF on 2025-10-20)
4. Default: "Unknown Merchant" (fallback if all else fails)

Which value should we show the user?
```

**Key requirements:**
- Predictable (same inputs → same output, always)
- Fast resolution (<100ms for 100 entities)
- User control (manual overrides always win)
- Transparent (user can see all sources + why one won)
- Simple to implement and understand

**Trade-off space:**
- **Static rules** vs **Dynamic rules**: Static = predictable, dynamic = flexible
- **User-configurable** vs **Hard-coded**: User-configurable = powerful, hard-coded = simple
- **AI-determined** vs **Manual**: AI = smart, manual = transparent

---

## Decision

**We use STATIC, HARD-CODED precedence rules:**

```
Priority 1 (Highest): Manual Override  (user explicitly corrected)
Priority 2:           Automated Rule    (normalization rule applied)
Priority 3:           Extraction        (parser/OCR output)
Priority 4 (Lowest):  Default           (fallback value)
```

**Resolution algorithm:**
```typescript
function resolveField(sources: ValueSources): any {
  if (sources.manual_override !== null) return sources.manual_override;
  if (sources.rule_value !== null) return sources.rule_value;
  if (sources.extraction_value !== null) return sources.extraction_value;
  return sources.default_value;
}
```

**Example:**
```typescript
// Merchant field with all 4 sources
const sources = {
  manual_override: "Amazon Marketplace",
  rule_value: "Amazon",
  extraction_value: "AMZN MKTP US*AB123",
  default_value: "Unknown Merchant"
};

const final_value = resolveField(sources);
// → "Amazon Marketplace" (manual override wins)
```

---

## Rationale

### 1. User Control (Manual Override Always Wins)

**Principle:** User knows better than automation.

**Example:**
- Extraction: "AMZN MKTP" (OCR misread)
- Rule: "Amazon" (normalization applied)
- User correction: "Amazon Marketplace" (more specific)

**Resolution:** User correction wins (priority 1).

**Benefit:** User corrections are never ignored by automation.

### 2. Predictable & Deterministic

**Static rules:**
```
✅ Same sources → always same result
✅ No configuration required
✅ Easy to understand (4-level hierarchy)
```

**Dynamic rules:**
```
❌ Different results depending on configuration
❌ Configuration drift (dev vs prod settings)
❌ Debugging nightmare ("why did this resolve differently yesterday?")
```

**Benefit:** No surprises, easy to debug.

### 3. Performance (No Database Lookups)

**Static precedence:**
```typescript
// In-memory resolution (no DB queries)
function resolveField(sources) {
  if (sources.manual_override !== null) return sources.manual_override;
  if (sources.rule_value !== null) return sources.rule_value;
  if (sources.extraction_value !== null) return sources.extraction_value;
  return sources.default_value;
}
```

**Dynamic precedence (rejected):**
```typescript
// Requires DB query to get precedence rules
const rules = await db.query("SELECT * FROM precedence_rules WHERE entity_type = ?");
// Then apply rules
```

**Performance:** Static = <1ms, Dynamic = 10-50ms (DB query overhead).

**Benefit:** Batch resolution of 100 entities in <100ms.

### 4. Transparent (Show All Sources)

**UI shows all sources + which won:**
```json
{
  "field_name": "merchant",
  "final_value": "Amazon Marketplace",
  "source": "manual",
  "sources": {
    "manual_override": "Amazon Marketplace",
    "rule_value": "Amazon",
    "extraction_value": "AMZN MKTP US*AB123",
    "default_value": "Unknown Merchant"
  },
  "metadata": {
    "is_overridden": true,
    "overridden_by": "user_123",
    "overridden_at": "2025-10-24T10:30:00Z"
  }
}
```

**User can see:**
- ✅ What manual override is ("Amazon Marketplace")
- ✅ What rule would have chosen ("Amazon")
- ✅ What extraction found ("AMZN MKTP US*AB123")
- ✅ Why manual override won (priority 1)

**Benefit:** Complete transparency, builds trust.

### 5. Simple Implementation

**Code:**
```typescript
class PrecedenceEngine {
  resolveField(entityId: string, fieldName: string, sources: ValueSources): PrecedenceResolution {
    const final_value =
      sources.manual_override ??
      sources.rule_value ??
      sources.extraction_value ??
      sources.default_value;

    const source =
      sources.manual_override !== null ? 'manual' :
      sources.rule_value !== null ? 'rule' :
      sources.extraction_value !== null ? 'extraction' : 'default';

    return {
      field_name: fieldName,
      final_value,
      source,
      sources,
      metadata: {
        is_overridden: sources.manual_override !== null
      }
    };
  }
}
```

**15 lines of code** vs 100+ lines for dynamic precedence.

**Benefit:** Easy to understand, test, and maintain.

---

## Consequences

### ✅ Positive

- **User control**: Manual overrides always win (user never frustrated by automation ignoring their corrections)
- **Predictable**: Same sources → same result (easy to understand, debug, test)
- **Fast**: <100ms for 100 entities (no DB queries, in-memory resolution)
- **Transparent**: UI shows all sources + which won (builds trust)
- **Simple**: 15 lines of code (easy to maintain)
- **No configuration**: No settings to manage, no dev/prod drift

### ⚠️ Trade-offs

- **Inflexible**: Cannot change precedence per entity type (e.g., "for healthcare, extraction wins over rule")
- **No special cases**: Cannot say "for category field, rule wins over extraction" (all fields use same precedence)

### 🔴 Risks (Mitigated)

- **Risk**: Future requirements need dynamic precedence (e.g., "for finance, rule wins; for healthcare, extraction wins")
  - **Mitigation**: Vertical 4.3 v1 uses static rules. If dynamic precedence needed, create ADR-0027 in v2.
- **Risk**: User accidentally overrides with wrong value (manual override always wins, even if wrong)
  - **Mitigation**: Revert capability (user can delete override, falls back to rule/extraction). Audit trail shows all changes.

---

## Alternatives Considered

### Alternative A: Dynamic/Configurable Precedence (Rejected)

**Approach:** Store precedence rules in database, configurable per entity type/field.

```sql
CREATE TABLE precedence_rules (
  rule_id UUID PRIMARY KEY,
  entity_type VARCHAR,
  field_name VARCHAR,
  precedence_order JSON  -- ["manual", "extraction", "rule", "default"]
);

-- Example: For finance, manual > rule > extraction > default
INSERT INTO precedence_rules VALUES
  ('rule_1', 'transaction', null, '["manual", "rule", "extraction", "default"]');

-- Example: For healthcare, manual > extraction > rule > default
INSERT INTO precedence_rules VALUES
  ('rule_2', 'patient_record', null, '["manual", "extraction", "rule", "default"]');
```

**Pros:**
- ✅ Flexible (different precedence per domain)
- ✅ No code changes for new precedence (just update database)

**Cons:**
- ❌ **Complexity**: DB query per resolution (10-50ms overhead)
- ❌ **Unpredictable**: Different results depending on configuration (debugging nightmare)
- ❌ **Configuration drift**: Dev vs prod settings can differ (bugs in prod)
- ❌ **Overkill**: No current requirement for different precedence per domain
- ❌ **Performance**: Batch resolution slower (need to join precedence_rules table)

**Decision:** ❌ Rejected - Complexity not worth flexibility (no current requirement).

---

### Alternative B: User-Configurable Precedence (Rejected)

**Approach:** Let users choose precedence per field in settings UI.

```
Settings > Corrections > Precedence Rules

Merchant field: [Manual Override] > [Rule] > [Extraction] > [Default]
Category field: [Manual Override] > [Extraction] > [Rule] > [Default]
```

**Pros:**
- ✅ Maximum flexibility (user chooses precedence)

**Cons:**
- ❌ **Confusing**: Most users don't understand precedence (what's the difference between rule and extraction?)
- ❌ **Inconsistent**: Different users configure different precedence (team conflicts)
- ❌ **Support burden**: "Why is my merchant showing different value than my colleague?"
- ❌ **Overkill**: No user request for this feature

**Decision:** ❌ Rejected - Too confusing for users, no clear benefit.

---

### Alternative C: AI-Determined Precedence (Rejected)

**Approach:** ML model predicts which source is most accurate based on historical corrections.

```
Training data: 10,000 corrections
Feature: confidence scores, field type, data quality metrics
Target: which source was correct (manual override, rule, or extraction)

Model predicts: "For this transaction, rule value is 90% likely correct"
```

**Pros:**
- ✅ Smart (learns from data)
- ✅ Adaptive (improves over time)

**Cons:**
- ❌ **Complexity**: Requires ML infrastructure (model training, deployment, monitoring)
- ❌ **Unpredictable**: Different results over time as model learns (hard to debug)
- ❌ **Black box**: Cannot explain why one source won ("the model chose it")
- ❌ **Overkill**: No evidence that static precedence is insufficient
- ❌ **Performance**: Model inference adds 50-200ms latency

**Decision:** ❌ Rejected - Too complex, no clear benefit over simple static rules.

---

### Alternative D: No Precedence (Ambiguity Errors) (Rejected)

**Approach:** If multiple sources exist, throw error and force user to choose.

```
Error: Multiple values for merchant field:
- Rule: "Amazon"
- Extraction: "AMZN MKTP US*AB123"

Please select which value to use.
```

**Pros:**
- ✅ No assumptions (user always chooses)

**Cons:**
- ❌ **User friction**: User must resolve every conflict manually (annoying)
- ❌ **Cannot automate**: System cannot show value until user chooses
- ❌ **Poor UX**: Constant popups/errors

**Decision:** ❌ Rejected - Too much user friction.

---

## Implementation Notes

### PrecedenceEngine

```typescript
interface ValueSources {
  manual_override?: any;
  rule_value?: any;
  extraction_value?: any;
  default_value?: any;
}

interface PrecedenceResolution {
  field_name: string;
  final_value: any;
  source: 'manual' | 'rule' | 'extraction' | 'default';
  sources: ValueSources;
  metadata: {
    is_overridden: boolean;
    overridden_by?: string;
    overridden_at?: string;
  };
}

class PrecedenceEngine {
  resolveField(fieldName: string, sources: ValueSources): PrecedenceResolution {
    // Apply precedence rules (manual > rule > extraction > default)
    const final_value =
      sources.manual_override ??
      sources.rule_value ??
      sources.extraction_value ??
      sources.default_value;

    const source =
      sources.manual_override !== null ? 'manual' :
      sources.rule_value !== null ? 'rule' :
      sources.extraction_value !== null ? 'extraction' : 'default';

    return {
      field_name: fieldName,
      final_value,
      source,
      sources,
      metadata: {
        is_overridden: sources.manual_override !== null
      }
    };
  }

  async resolveAllBatch(entityIds: string[]): Promise<Map<string, Record<string, PrecedenceResolution>>> {
    // 1. Fetch all overrides in one query
    const overrides = await OverrideStore.query({ entity_id: { $in: entityIds } });

    // 2. Group by entity_id
    const overridesByEntity = groupBy(overrides, 'entity_id');

    // 3. Resolve in-memory (no more DB queries)
    const results = new Map();
    for (const entityId of entityIds) {
      const entity = await EntityStore.getById(entityId);
      const entityOverrides = overridesByEntity[entityId] || [];

      const resolved: Record<string, PrecedenceResolution> = {};
      for (const field of Object.keys(entity)) {
        const override = entityOverrides.find(o => o.field_name === field);
        const sources: ValueSources = {
          manual_override: override?.override_value,
          extraction_value: entity[field]
        };

        resolved[field] = this.resolveField(field, sources);
      }

      results.set(entityId, resolved);
    }

    return results;
  }
}
```

### Performance Benchmarks

| Operation | Latency (p95) |
|-----------|---------------|
| Resolve 1 field (in-memory) | <1ms |
| Resolve 10 fields (in-memory) | <5ms |
| Resolve 100 entities (batch) | <100ms |

### Null Handling

```typescript
// If source is null/undefined, skip to next priority
sources.manual_override ?? sources.rule_value ?? sources.extraction_value ?? sources.default_value

// Examples:
{ manual_override: null, rule_value: "Amazon", extraction_value: "AMZN" }
→ "Amazon" (rule wins, manual is null)

{ manual_override: null, rule_value: null, extraction_value: "AMZN" }
→ "AMZN" (extraction wins, manual and rule are null)

{ manual_override: null, rule_value: null, extraction_value: null, default_value: "Unknown" }
→ "Unknown" (default wins, all others are null)
```

---

## Multi-Domain Applicability

### Finance
**Merchant field:**
- Manual override: "Amazon Marketplace" (user corrected)
- Rule: "Amazon" (normalization)
- Extraction: "AMZN MKTP US*AB123" (parser)
→ Final: "Amazon Marketplace" (manual wins)

### Healthcare
**Diagnosis code:**
- Manual override: "E11.65" (nurse corrected)
- Extraction: "E11.9" (OCR)
→ Final: "E11.65" (manual wins, no rule)

### Legal
**Case number:**
- Extraction: "2025-CV-1234" (OCR)
- Default: "UNKNOWN"
→ Final: "2025-CV-1234" (extraction wins, no manual/rule)

### Research
**Author name:**
- Manual override: "Smith, John A." (curator corrected)
- Rule: "Smith, J." (normalization)
- Extraction: "J. Smith" (PDF parser)
→ Final: "Smith, John A." (manual wins)

### E-commerce
**Product category:**
- Manual override: "Home Appliances" (catalog manager corrected)
- Rule: "Electronics" (auto-classification)
→ Final: "Home Appliances" (manual wins)

### SaaS
**MRR:**
- Manual override: $500 (operations analyst corrected)
- Extraction: $50 (billing system misread)
→ Final: $500 (manual wins)

---

## Related Decisions

- **ADR-0024**: Field-Level Overrides (manual overrides stored per field)
- **ADR-0025**: Audit Storage Strategy (audit log shows which source won)

---

**References:**
- Vertical 4.3: [docs/verticals/4.3-corrections-flow.md](../verticals/4.3-corrections-flow.md)
- PrecedenceEngine: [docs/primitives/ol/PrecedenceEngine.md](../primitives/ol/PrecedenceEngine.md)
- Schema: [docs/schemas/precedence-resolution.schema.json](../schemas/precedence-resolution.schema.json)
