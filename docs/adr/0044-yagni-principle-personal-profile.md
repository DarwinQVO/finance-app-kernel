# ADR-0044: YAGNI Principle in Personal Profile

**Status**: Accepted
**Date**: 2025-10-27
**Context**: Phase 2 (Personal Profile design patterns)
**Deciders**: System architects
**Related**: ADR-0042 (Simplicity Profiles), ADR-0043 (Three-Tier Scaling), Phase 3 (Darwin uses 3.8% of Enterprise code)

---

## Context

When designing the Personal Profile, we face a fundamental question: **Should we implement features that might be useful, or only features that are demonstrably necessary?**

**Example scenarios:**
1. **Duplicate detection**: Darwin sees <5 duplicates per year
2. **FX fee linking**: Darwin travels internationally once per year
3. **Series forecasting**: Darwin has 3 recurring subscriptions (predictable)
4. **Fuzzy matching**: Darwin has 20 merchants (string matching works)

**Traditional approach:**
Implement all features at minimal complexity, "just in case."

**Alternative:**
Skip features entirely (0 LOC) unless proven pain point.

**The Question:**
Is it acceptable to have primitives with **0 LOC implementations** (documented "not needed" decision)?

---

## Decision

**Personal Profile applies YAGNI (You Aren't Gonna Need It) aggressively:**

- **Features used < 12 times/year → 0 LOC** (document as "SKIPPED")
- **Pain point must be observed** (not hypothetical)
- **Upgrade path documented** (how to add feature when needed)

**Key principle:** 0 LOC is a valid implementation. It explicitly documents the "not needed" decision.

---

## Rationale

### 1. Ruthless Simplicity

**Darwin's actual usage (Phase 3 findings):**

| Feature | Enterprise LOC | Personal LOC | Usage/Year | Decision |
|---------|----------------|--------------|------------|----------|
| **Upload** | 4,200 | 300 | 12× | ✅ Implement (monthly statements) |
| **Parsing** | 6,000 | 350 | 12× | ✅ Implement (needed for every upload) |
| **View transactions** | 800 | 80 | 100× | ✅ Implement (core feature) |
| **Normalization** | 3,500 | 280 | 12× | ✅ Implement (cleanup needed) |
| **Duplicates** | 1,500 | 20 | 5× | ⚠️ Manual (click "mark as duplicate") |
| **Account registry** | 800 | 15 | 0× | ⚠️ Hardcoded dict (2 accounts: BoFA, Apple Card) |
| **Counterparty resolution** | 2,000 | 0 | 0× | ❌ **SKIPPED** (use merchant string directly) |
| **Clustering** | 3,000 | 40 | 12× | ⚠️ Hardcoded rules (5 categories) |
| **Corrections** | 1,800 | 10 | 20× | ⚠️ Simple UPDATE (no version history) |
| **Series detection** | 2,500 | 0 | 0× | ❌ **SKIPPED** (manual "subscription" tags) |
| **Tax categories** | 4,000 | 60 | 1× | ⚠️ On-demand calculation (no precomputation) |
| **FX fee linking** | 1,200 | 0 | 1× | ❌ **SKIPPED** (manual search for "Foreign") |

**Key findings:**
- **Darwin uses 8/12 features** (67% feature coverage)
- **4 features skipped entirely** (0 LOC = explicit decision)
- **Total LOC: 1,365** (3.8% of Enterprise's 35,700 LOC)

**Rationale for skips:**
- **Counterparty resolution**: 20 merchants only → `GROUP BY merchant` works fine
- **Series detection**: 3 subscriptions (Netflix, Spotify, Apple) → manual tags sufficient
- **FX fee linking**: Travels 1×/year → manual search acceptable (<5 min/year burden)

### 2. Cost-Benefit Analysis

**Example: Duplicate Detection**

**Cost to implement:**
- Development: 150 LOC (fuzzy matching, confidence scores)
- Testing: 20 test cases
- Maintenance: Handle false positives
- User burden: Review "Possible duplicates" notifications

**Benefit:**
- Saves 5 manual clicks/year (2 minutes total)

**YAGNI decision:** 0 LOC. Manual "mark as duplicate" button (20 LOC) sufficient.

**Rationale:**
- 150 LOC for 2 minutes/year savings = Poor ROI
- False positives cost MORE time than manual review
- Feature complexity > problem complexity

---

### 3. Upgrade Path Must Be Clear

**YAGNI doesn't mean "never implement."**

Each 0 LOC decision documents **upgrade triggers:**

```markdown
## DuplicateDetector (Personal Profile)

**LOC:** 0 (Not implemented)

**Rationale:** Darwin sees <5 duplicates/year. Manual review (click button) takes 2 min/year.

**Upgrade trigger:** If manual review > 15 min/month (indicates volume growth) → Implement fuzzy matching (Small Business profile: 150 LOC).

**Upgrade path:**
1. Add `duplicate_confidence` column to transactions table
2. Implement FuzzyMatcher (Levenshtein distance, date proximity)
3. Add "Review duplicates" UI with auto-link suggestions
```

**Benefits:**
- User knows feature exists in other profiles
- Clear pain point threshold for upgrade
- Implementation guidance provided

---

### 4. Prevents Feature Creep

**Without YAGNI:**
- "Maybe Darwin will need forecasting someday" → Implement 500 LOC
- "Counterparty resolution might be useful" → Implement 2,000 LOC
- "Let's add ML clustering just in case" → Implement 3,000 LOC
- **Result:** 5,500 LOC of unused features (4× actual usage)

**With YAGNI:**
- Feature must solve observed pain point (not hypothetical)
- 0 LOC explicitly documents "not needed yet"
- **Result:** 1,365 LOC total (only what's used)

---

## Consequences

### Positive

✅ **Radically simple codebase**: Personal profile = 1,365 LOC (3.8% of Enterprise)
✅ **No unused code**: Every line serves observed use case
✅ **Fast development**: Build 8 features, skip 4 → 2× faster to v1
✅ **Easy maintenance**: 1,365 LOC to maintain vs 35,700 LOC
✅ **Clear upgrade path**: User knows when to add features (pain point thresholds)
✅ **Honest documentation**: 0 LOC documents explicit "not needed" decision

### Negative

⚠️ **Missing features**: Personal users lack 4/12 features (duplicates, counterparty, series, FX linking)
⚠️ **Manual workflows**: User must manually tag subscriptions, search for FX fees
⚠️ **Perception risk**: "Incomplete" product perception

### Mitigations

**Missing features:**
- Documented as intentional (not bugs)
- Upgrade path clearly described
- Manual workarounds provided (20 LOC helpers)

**Manual workflows:**
- Only apply to <12×/year tasks (total burden <1 hour/year)
- Automated workflows reserved for >12×/year tasks

**Perception risk:**
- Personal profile explicitly marketed as "minimal viable" not "feature-complete"
- Target audience: single users who value simplicity over features

---

## Decision Matrix: Implement or Skip?

### Implement if ANY of:
- ✅ Feature used ≥12×/year (monthly or more frequent)
- ✅ Manual alternative takes >1 hour/year total
- ✅ Feature blocks core user journey (upload, parse, view)

### Skip (0 LOC) if ALL of:
- ❌ Feature used <12×/year
- ❌ Manual alternative takes <1 hour/year
- ❌ Feature is "nice to have" not "must have"
- ❌ Complexity >> problem size

### Examples:

**Implement: Transaction normalization (✅)**
- Used 12×/year (every statement upload)
- Manual alternative: fix 42 transactions × 12 = 504 fixes/year = 8 hours
- Core user journey dependency
- **Decision:** Implement (280 LOC)

**Skip: Series forecasting (❌)**
- Used 0×/year (Darwin doesn't check forecasts)
- Manual alternative: mental math ("Netflix is $15/month")
- Nice to have, not must have
- **Decision:** Skip (0 LOC, document upgrade path)

---

## Alternatives Considered

### Alternative 1: Implement All Features at Minimal Complexity

**Approach:** Every feature has Personal implementation (even if simple).

```python
# Minimal duplicate detection
def detect_duplicates(transactions):
    # Exact match only (same date, same amount, same merchant)
    return [t for t in transactions if exact_match(t)]
```

**Rejected because:**
- Still requires testing, maintenance
- "Minimal" implementation often 50-100 LOC per feature
- False positives waste user time (reviewing non-duplicates)
- Accumulated cruft: 12 minimal features = 600 LOC of rarely-used code

**YAGNI approach:** 0 LOC, document manual workaround.

---

### Alternative 2: Build Features Proactively Based on "Common Sense"

**Approach:** Implement features that "every finance app has."

**Rejected because:**
- "Common sense" often wrong (Darwin doesn't need forecasting)
- Builds features for hypothetical users, not real user (Darwin)
- Results in feature bloat (20+ features, Darwin uses 8)

**YAGNI approach:** Only build for observed pain points.

---

### Alternative 3: Implement Everything, Disable via Config

**Approach:** Single codebase, disable features via flags.

```yaml
personal_config:
  enable_duplicate_detection: false
  enable_series_forecasting: false
  enable_fx_linking: false
```

**Rejected because:**
- Code still exists (must be tested, maintained)
- Disabled features accumulate technical debt
- No clear "this feature doesn't exist" boundary

**YAGNI approach:** Feature literally doesn't exist (0 LOC).

---

## Implementation Notes

### Documenting 0 LOC Decisions

**Template for skipped features:**

```markdown
## {FeatureName} (Personal Profile)

**LOC:** 0 (Not implemented)

**Rationale:** {Why it's not needed for Personal scale}

**Manual alternative:** {How user achieves goal without feature}

**Time cost:** {How much time manual approach takes per year}

**Upgrade trigger:** {Pain point threshold that signals need for feature}

**Upgrade path:** {Steps to implement when triggered}
```

**Example:**

```markdown
## SeriesDetector (Personal Profile)

**LOC:** 0 (Not implemented)

**Rationale:** Darwin has 3 recurring subscriptions (Netflix, Spotify, Apple). Manual tagging takes 30 seconds/month = 6 min/year.

**Manual alternative:** Add "subscription" tag to transactions manually.

**Time cost:** 6 min/year

**Upgrade trigger:** If recurring transactions >20 OR manual tagging >30 min/year → Implement series detection (Small Business profile: 200 LOC).

**Upgrade path:**
1. Add `series_id` column to transactions table
2. Implement pattern detection (amount+merchant+frequency)
3. Add "Suggested series" UI with auto-link
```

---

## Real-World Example: Darwin's Implementation

**Features implemented (8/12):**
- Upload Document (300 LOC) - Used 12×/year
- Parsing (350 LOC) - Used 12×/year
- View OL/RL (360 LOC combined) - Used 100×/year
- Normalization (280 LOC) - Used 12×/year
- Clustering (40 LOC hardcoded) - Used 12×/year
- Corrections (10 LOC simple UPDATE) - Used 20×/year
- Tax Categories (60 LOC on-demand) - Used 1×/year

**Features skipped (4/12):**
- Duplicates (0 LOC) - Manual button click
- Counterparty (0 LOC) - Use merchant string
- Series (0 LOC) - Manual tags
- FX Linking (0 LOC) - Manual search

**Total:** 1,365 LOC (vs Enterprise 35,700 LOC)

**Time saved by YAGNI:** ~2 months development (estimated 4,000 LOC avoided)

---

## Related Decisions

- **ADR-0042:** Simplicity Profiles Pattern (defines profile structure)
- **ADR-0043:** Three-Tier Scaling Strategy (Personal tier characteristics)
- **ADR-0045:** Hardcoded Configuration Strategy (complements YAGNI - when to hardcode vs configure)

---

## Review & Updates

- **2025-10-27:** Initial decision (based on Phase 3 Darwin implementation analysis)
- Next review: After 10+ Personal profile implementations in production

---

**Decision**: Apply YAGNI aggressively in Personal Profile. 0 LOC is valid for features used <12×/year.
**Status**: Accepted and implemented in Phase 2/3 (Darwin uses 3.8% of Enterprise code).
