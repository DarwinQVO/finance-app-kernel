# ADR-0005: Configurable Normalization Rules (Externalized, Versioned)

**Status**: Accepted
**Date**: 2025-10-23
**Context**: Vertical 1.3 (Normalization) design
**Deciders**: System architects
**Related**: Vertical 1.3, ADR-0004 (Raw-First Extraction)

---

## Context

When normalizing raw observations into canonical records, we must apply domain-specific rules:
- **Date parsing:** Which locale? MM/DD/YYYY or DD/MM/YYYY?
- **Merchant normalization:** How to map "STARBUCKS #1234" → "Starbucks"?
- **Category inference:** Which merchant belongs to which category?
- **Duplicate detection:** What tolerance? ±1 day or exact match?

**Key question:** Should these rules be **hardcoded in normalizer code** or **externalized as configuration**?

---

## Decision

**Normalization rules are externalized as versioned configuration (NormalizationRuleSet), NOT hardcoded in normalizer code.**

Rules are:
1. **Externalized:** Stored outside normalizer code (JSON files, database, etc.)
2. **Versioned:** Each rule set has a version (e.g., "2024.10")
3. **Per source_type:** Different rules for different sources (e.g., BoFA vs Chase)
4. **Tracked in provenance:** Each canonical records which rule set version was used

---

## Rationale

### 1. Rules Change Frequently

**Domain-specific rules evolve:**
- New merchants discovered → add to whitelist
- Better category mappings learned → update rules
- Locales added → support more date formats
- User feedback → refine duplicate detection

**With externalized rules:**
- Update config file, redeploy (no code changes)
- A/B test different rule sets
- Rollback to previous rule version if needed

**With hardcoded rules:**
- Change code, recompile, redeploy (slow)
- Can't rollback without code revert
- Can't A/B test (code branching required)

---

### 2. Multi-Tenant / Multi-Source Flexibility

**Different sources have different rules:**
- **BoFA PDFs:** US locale (MM/DD/YYYY), USD currency
- **Scotia CSVs:** Canadian locale (DD/MM/YYYY), CAD currency
- **Wise JSONs:** International locale (YYYY-MM-DD), multi-currency

**With externalized rules:**
```python
rules_bofa = normalization_rules.get_for_source("bofa_pdf")
rules_scotia = normalization_rules.get_for_source("scotia_csv")

canonical_1 = normalizer.normalize(obs_bofa, rules_bofa)
canonical_2 = normalizer.normalize(obs_scotia, rules_scotia)
```

**With hardcoded rules:**
```python
# ❌ WRONG: Source-specific logic in normalizer code
def normalize(obs):
    if obs.source_type == "bofa_pdf":
        date = parse_date(obs.date, locale="en_US")
    elif obs.source_type == "scotia_csv":
        date = parse_date(obs.date, locale="en_CA")
    # ... 50 more if/elif branches (unmaintainable)
```

---

### 3. Auditability (Which Rules Were Used?)

**Provenance requirement:** Know exactly which rules produced a canonical.

**With versioned rules:**
```python
canonical = normalizer.normalize(obs, rules_v2024_10)
canonical.rule_set_version = "2024.10"  # Tracked in CanonicalTransaction

# Later: Debug incorrect category
# → Check NormalizationLog: "rule_set_version": "2024.10"
# → Retrieve rule set v2024.10 from git/config store
# → See exactly which category rule was applied
```

**With hardcoded rules:**
- No version tracking (normalizer code version != rule version)
- Can't reproduce normalization (code may have changed)
- Can't compare rule versions (diff code commits)

---

### 4. Re-Normalization with Updated Rules

**Scenario:** Discover better category rule, want to re-categorize existing canonicals.

**With externalized rules:**
```python
# 1. Update rule set (add new merchant → category mapping)
rules_v2024_11 = update_rules(rules_v2024_10, new_merchant_mapping)

# 2. Re-normalize observations with new rules
for obs in observation_store.get_all():
    canonical = normalizer.normalize(obs, rules_v2024_11)
    canonical_store.upsert(canonical)  # Idempotent upsert

# 3. All canonicals now have rule_set_version = "2024.11"
```

**With hardcoded rules:**
- Change code, redeploy normalizer
- No way to compare old vs new rules (must diff code)
- Risk: Other unintended changes in code affect normalization

---

### 5. User Customization (Future)

**Potential future requirement:** Let users customize rules per account.

**Example:**
- User A wants "Starbucks" → "Food & Drink"
- User B wants "Starbucks" → "Business Expenses"

**With externalized rules:**
```python
rules_user_a = load_rules(account_id="user_a")
rules_user_b = load_rules(account_id="user_b")

canonical_a = normalizer.normalize(obs, rules_user_a)
canonical_b = normalizer.normalize(obs, rules_user_b)

assert canonical_a.category == "Food & Drink"
assert canonical_b.category == "Business Expenses"
```

**With hardcoded rules:** Not feasible (single global rule set).

---

## Consequences

### Positive

✅ **Flexibility:** Update rules without code changes
✅ **Multi-source support:** Different rules per source_type
✅ **Auditability:** Track which rule version produced each canonical
✅ **Re-normalization:** Apply new rules to existing observations
✅ **A/B testing:** Test new rules on subset of uploads
✅ **User customization:** Per-account rules (future)

### Negative

⚠️ **Complexity:** Need config store, versioning system, deployment pipeline
⚠️ **Validation:** Rules must be validated (invalid config breaks normalization)
⚠️ **Synchronization:** Normalizer code + rules must stay compatible

### Mitigations

**Complexity:**
- Use simple JSON files in git (version controlled)
- Schema validation for rule sets (detect invalid config)
- Automated tests: Run normalizer with all rule versions

**Validation:**
```python
# Rule set schema validation before deployment
rule_set = load_rules("2024.10")
validate_rule_set_schema(rule_set)  # JSON Schema validation

# Compatibility test
normalizer = Normalizer(version="1.0.0")
test_observations = load_golden_data()
for obs in test_observations:
    canonical = normalizer.normalize(obs, rule_set)  # Must not crash
```

**Synchronization:**
- Rule set version references normalizer version (compatibility matrix)
- Example: `rule_set_2024_10` requires `normalizer >= 1.0.0`

---

## Alternatives Considered

### Alternative 1: Hardcoded Rules

**Approach:** Rules embedded in normalizer code.

```python
class Normalizer:
    def normalize(self, obs):
        # Hardcoded locale
        date = parse_date(obs.date, locale="en_US")

        # Hardcoded merchant mapping
        if "STARBUCKS" in obs.description:
            merchant = "Starbucks"
            category = "Food & Drink"

        return CanonicalTransaction(...)
```

**Rejected because:**
- Can't update rules without code changes
- Can't support multiple sources with different rules
- Can't track which rule version produced a canonical
- Can't re-normalize with updated rules easily

---

### Alternative 2: Hardcoded Rules + Config Overrides

**Approach:** Default rules in code, config overrides for specific cases.

```python
class Normalizer:
    def __init__(self, overrides=None):
        self.default_locale = "en_US"
        self.overrides = overrides or {}

    def normalize(self, obs):
        locale = self.overrides.get("locale", self.default_locale)
        date = parse_date(obs.date, locale=locale)
        ...
```

**Rejected because:**
- Still have hardcoded defaults (must change code to change defaults)
- Unclear precedence (which rules are overridden?)
- Not versioned (can't track rule changes)

---

### Alternative 3: Database-Stored Rules

**Approach:** Store rules in database tables.

```sql
CREATE TABLE normalization_rules (
    id SERIAL PRIMARY KEY,
    source_type TEXT,
    date_locale TEXT,
    merchant_whitelist JSONB,
    version TEXT
);
```

**Considered but deferred:**
- More complex (database schema, migrations)
- Harder to version control (git vs database)
- Overkill for MVP (file-based rules sufficient)

**Future possibility:** Migrate to database if rules become very dynamic.

---

## Implementation Notes

### NormalizationRuleSet Structure

```typescript
interface NormalizationRuleSet {
  version: string                    // "2024.10"
  source_type: string                // "bofa_pdf"
  date_locale: string                // "en_US"
  date_formats: string[]             // ["MM/DD/YYYY", "M/D/YYYY"]
  amount_decimal_separator: string   // "."
  merchant_whitelist: Record<string, string>  // {"STARBUCKS": "Starbucks"}
  category_rules: CategoryRule[]
  duplicate_tolerance_days: number   // 1
  large_amount_threshold: number     // 10000.00
}

interface CategoryRule {
  merchant_pattern: string  // "Starbucks"
  category: string          // "Food & Drink"
  confidence: number        // 0.95
}
```

**Storage:** JSON files in `config/normalization-rules/`

**Example:**
```json
// config/normalization-rules/bofa_pdf_2024.10.json
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

### Loading Rules

```python
class NormalizationRuleStore:
    def __init__(self, config_dir: str):
        self.config_dir = config_dir
        self.cache = {}

    def get_for_source(self, source_type: str, version: str = "latest") -> NormalizationRuleSet:
        """Load rule set for source_type."""
        cache_key = f"{source_type}:{version}"
        if cache_key in self.cache:
            return self.cache[cache_key]

        if version == "latest":
            # Find highest version for source_type
            files = glob(f"{self.config_dir}/{source_type}_*.json")
            latest_file = max(files, key=extract_version)
        else:
            latest_file = f"{self.config_dir}/{source_type}_{version}.json"

        with open(latest_file) as f:
            rule_set = NormalizationRuleSet(**json.load(f))

        # Validate schema
        validate_rule_set(rule_set)

        self.cache[cache_key] = rule_set
        return rule_set
```

---

### Versioning Strategy

**Semantic versioning for rule sets:**
- **MAJOR:** Breaking changes (incompatible with old normalizer)
- **MINOR:** New rules added (backwards compatible)
- **PATCH:** Bug fixes in existing rules

**Example:**
- `2024.10` → Initial rule set
- `2024.11` → Added 50 new merchants to whitelist (MINOR)
- `2025.01` → Changed date format parsing (MAJOR, requires normalizer 2.0.0)

**Deployment:**
1. Update rule set JSON file in git
2. Tag commit: `rule-set-2024.11`
3. CI/CD deploys new config to production
4. Normalizer automatically loads new version (hot reload or restart)

---

## Related Decisions

- **ADR-0004:** Raw-First Extraction (observations never modified, can re-normalize)
- **ADR-0003:** Runner/Coordinator split (normalizer is pure function, stateless)
- **Vertical 1.3:** Normalization specification

---

## Review & Updates

- **2025-10-23:** Initial decision (Vertical 1.3 specification)
- Next review: After implementing 5+ source types, assess if file-based rules scale
- Consider: Migrate to database-stored rules if >100 source types

---

**Decision**: Normalization rules are externalized as versioned configuration (NormalizationRuleSet).
**Status**: Accepted and specified in Vertical 1.3.
