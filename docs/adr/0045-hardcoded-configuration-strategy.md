# ADR-0045: Hardcoded Configuration Strategy

**Status**: Accepted
**Date**: 2025-10-27
**Context**: Phase 2 (Personal Profile implementation patterns)
**Deciders**: System architects
**Related**: ADR-0042 (Simplicity Profiles), ADR-0044 (YAGNI), Phase 3 (Darwin hardcodes 20 merchants, 2 accounts, 5 categories)

---

## Context

When designing the Personal Profile, we must choose between:

1. **Hardcoded** (in-code dicts/lists):
   ```python
   ACCOUNTS = {
       "****1234": "BoFA Checking",
       "****5678": "Apple Card"
   }
   ```

2. **Database** (table-driven):
   ```sql
   CREATE TABLE accounts (
       account_id TEXT PRIMARY KEY,
       account_name TEXT
   );
   ```

3. **Configuration file** (YAML/JSON):
   ```yaml
   accounts:
     - id: "****1234"
       name: "BoFA Checking"
   ```

**Traditional wisdom:** "Never hardcode, always use database/config."

**The Question:** When is hardcoding actually the BETTER choice?

---

## Decision

**Personal Profile uses hardcoding when ALL of:**

1. ✅ **Data is small** (<20 items)
2. ✅ **Data is stable** (changes <4×/year)
3. ✅ **No user UI needed** (user doesn't add/edit via UI)
4. ✅ **Type safety valuable** (benefit from IDE autocomplete, compile-time checks)

**Upgrade to database when ANY of:**

1. ❌ Data grows >20 items
2. ❌ Changes >4×/year
3. ❌ Need UI for user to add/edit
4. ❌ Need multi-user data isolation

---

## Rationale

### 1. Hardcoding Is Simpler for Small, Stable Data

**Example: Account Registry**

**Darwin's context:**
- 2 accounts (BoFA Checking, Apple Card)
- Accounts change <1×/year (Darwin rarely switches banks)
- No need to add accounts via UI (Darwin edits code directly)

**Hardcoded implementation (15 LOC):**
```python
# accounts.py
ACCOUNTS = {
    "****1234": {"name": "BoFA Checking", "type": "checking"},
    "****5678": {"name": "Apple Card", "type": "credit"}
}

def get_account(account_id: str) -> dict:
    return ACCOUNTS.get(account_id, {"name": "Unknown", "type": "unknown"})
```

**Database implementation (100+ LOC):**
```python
# models/account.py
class Account(Base):
    __tablename__ = 'accounts'
    account_id = Column(String, primary_key=True)
    name = Column(String, nullable=False)
    type = Column(String, nullable=False)

# migrations/001_create_accounts.sql
CREATE TABLE accounts (...);

# repositories/account_repository.py
class AccountRepository:
    def get_account(self, account_id: str) -> Account: ...
    def add_account(self, account: Account): ...
    def update_account(self, account: Account): ...
    def delete_account(self, account_id: str): ...

# ui/account_manager.tsx
<AccountForm onSubmit={handleAddAccount} />
```

**Comparison:**

| Aspect | Hardcoded (15 LOC) | Database (100+ LOC) |
|--------|-------------------|---------------------|
| **Complexity** | Dict lookup | ORM, migrations, UI |
| **Dependencies** | None | SQLAlchemy, Alembic, React |
| **Latency** | 0ms (in-memory) | 5ms (DB query) |
| **Type safety** | ✅ Python types | ⚠️ Runtime validation |
| **IDE support** | ✅ Autocomplete | ❌ String keys |
| **Testing** | ✅ No DB setup needed | ❌ Requires test DB |

**Decision:** Hardcode for Personal (2 accounts, stable).
**Upgrade trigger:** If Darwin adds >10 accounts OR needs multi-user → Database.

---

### 2. Hardcoding Provides Type Safety

**Example: Categorization Rules**

**Hardcoded (40 LOC):**
```python
CATEGORY_RULES = {
    "WHOLE FOODS": "Groceries",
    "SAFEWAY": "Groceries",
    "NETFLIX": "Entertainment",
    "SPOTIFY": "Entertainment",
    "SHELL": "Transportation",
    # ... 15 more
}

def categorize(merchant: str) -> str:
    for pattern, category in CATEGORY_RULES.items():
        if pattern in merchant.upper():
            return category  # Type: str (IDE knows this)
    return "Uncategorized"
```

**Benefits:**
- IDE autocomplete shows all categories
- Typo in category name caught at "compile time" (static analysis)
- Refactoring: rename "Entertainment" → "Streaming" updates all usages

**YAML config (no type safety):**
```yaml
rules:
  - pattern: "WHOLE FOODS"
    category: "Grocereis"  # Typo not caught until runtime
```

**Decision:** Hardcode rules for Personal (20 merchants, static analysis benefits).

---

### 3. Hardcoding Avoids Over-Engineering

**Example: Darwin's actual implementation**

**Hardcoded elements:**
- ✅ **2 accounts** (BoFA, Apple Card) → 15 LOC dict
- ✅ **20 merchants** (Whole Foods, Safeway, Netflix, ...) → 40 LOC rules
- ✅ **5 categories** (Groceries, Dining, Gas, Utilities, Subscriptions) → Dict keys
- ✅ **1 parser** (BoFA PDF) → Direct import
- ✅ **1 normalization strategy** (US locale, MM/DD/YYYY) → Constants

**Total hardcoded LOC:** ~100 LOC

**Alternative (database/config):**
- AccountStore table + UI → 400 LOC
- MerchantRegistry table + UI → 500 LOC
- CategoryManager table + UI → 300 LOC
- ParserRegistry table + UI → 400 LOC
- NormalizationConfig YAML + parser → 200 LOC

**Total database LOC:** ~1,800 LOC

**Avoided complexity:** 1,700 LOC (18× more code for same functionality)

**Rationale:**
- Darwin's data is small (2 accounts, 20 merchants, 5 categories)
- Changes are rare (<4×/year)
- UI for managing accounts/categories is overkill (Darwin edits code directly)
- **Hardcoding is the SIMPLER solution**

---

### 4. Clear Upgrade Path When Data Grows

**Each hardcoded element documents upgrade trigger:**

```python
# accounts.py

# HARDCODED (Personal Profile)
# Upgrade trigger: If accounts >10 OR multi-user needed → AccountStore (Small Business)
ACCOUNTS = {
    "****1234": {"name": "BoFA Checking", "type": "checking"},
    "****5678": {"name": "Apple Card", "type": "credit"}
}
```

**Upgrade path clear:**
1. Create `accounts` table
2. Migrate dict → SQL: `INSERT INTO accounts VALUES (...)`
3. Replace `ACCOUNTS.get()` → `account_repo.get_account()`
4. Add UI for account management

**No premature database setup.**

---

## Consequences

### Positive

✅ **Radical simplicity**: Dict lookup vs database queries (100× simpler)
✅ **Zero dependencies**: No ORM, migrations, UI framework for account management
✅ **Type safety**: IDE autocomplete, static analysis, refactoring support
✅ **Fast**: 0ms (in-memory) vs 5ms (DB query)
✅ **Easy testing**: No test database setup
✅ **Version controlled**: Changes tracked in git (vs SQL migrations)
✅ **Honest about scale**: Code explicitly designed for 2 accounts, not 1000

### Negative

⚠️ **Limited scalability**: Hardcoded dict doesn't scale to 100+ accounts
⚠️ **No UI for non-technical users**: Darwin must edit code to add accounts
⚠️ **Merge conflicts**: Multiple developers editing same dict = conflicts

### Mitigations

**Limited scalability:**
- Acceptable for Personal Profile (by definition: single user, small data)
- Upgrade trigger documented (>10 accounts → database)

**No UI:**
- Target audience: technical users comfortable editing code
- UI added in Small Business profile (when multi-user needed)

**Merge conflicts:**
- Non-issue for Personal (single user: Darwin)
- Database required for Small Business (multi-user = need isolation)

---

## Decision Matrix: Hardcode or Database?

### Hardcode if ALL TRUE:
- ✅ Data size: <20 items
- ✅ Change frequency: <4×/year
- ✅ Users: 1-3 (all technical, comfortable editing code)
- ✅ Type safety valuable: IDE autocomplete, refactoring
- ✅ Simplicity > Flexibility

### Database if ANY TRUE:
- ❌ Data size: >20 items (or expected to grow)
- ❌ Change frequency: >4×/year (monthly or more frequent)
- ❌ Users: >3 OR non-technical (need UI)
- ❌ Multi-user: Need data isolation per user
- ❌ Flexibility > Simplicity

### Examples:

**Hardcode: Account Registry (Personal)**
- 2 accounts (BoFA, Apple Card)
- Changes <1×/year
- 1 user (Darwin, technical)
- **Decision:** Dict (15 LOC)

**Database: Account Registry (Small Business)**
- 50 accounts (multiple clients)
- Changes ~5×/month (new clients)
- 10 users (need UI)
- **Decision:** PostgreSQL table (400 LOC)

---

## Alternatives Considered

### Alternative 1: Always Use Database

**Approach:** "Best practice" says always use database for persistence.

```sql
CREATE TABLE accounts (
    account_id TEXT PRIMARY KEY,
    account_name TEXT
);
```

**Rejected for Personal because:**
- Overkill for 2 accounts (100 LOC vs 15 LOC)
- Adds dependencies (SQLAlchemy, Alembic)
- Requires migrations
- Slower (5ms query vs 0ms dict lookup)
- No type safety (string keys vs Python dict)

**Current approach:** Hardcode for Personal, database for Small Business/Enterprise.

---

### Alternative 2: Configuration Files (YAML/JSON)

**Approach:** Externalize data to config files.

```yaml
# accounts.yaml
accounts:
  - id: "****1234"
    name: "BoFA Checking"
    type: "checking"
  - id: "****5678"
    name: "Apple Card"
    type: "credit"
```

**Rejected for Personal because:**
- No type safety (typos not caught)
- Extra file to manage (vs single .py file)
- Requires YAML parser (dependency)
- No IDE autocomplete
- Harder to refactor (rename category across files)

**Current approach:** Hardcode in Python (type safety, simplicity).

---

### Alternative 3: Hybrid (Hardcode with Override File)

**Approach:** Hardcode defaults, allow optional YAML override.

```python
DEFAULT_ACCOUNTS = {"****1234": "BoFA Checking"}
OVERRIDE_ACCOUNTS = load_yaml("accounts.yaml") if exists("accounts.yaml") else {}
ACCOUNTS = {**DEFAULT_ACCOUNTS, **OVERRIDE_ACCOUNTS}
```

**Rejected because:**
- Adds complexity (merge logic)
- Two sources of truth (code + YAML)
- Unclear precedence (which wins?)

**Current approach:** Single source of truth (hardcoded dict for Personal).

---

## Implementation Notes

### Hardcoding Template

```python
# {resource}_registry.py

# HARDCODED (Personal Profile)
# Use when: Data is small (<20 items), stable (<4 changes/year), single user
# Upgrade trigger: If data >20 items OR changes >4×/year OR multi-user → Database
{RESOURCE}_REGISTRY = {
    "key1": {"field": "value"},
    "key2": {"field": "value"},
    # ... <20 items total
}

def get_{resource}(key: str) -> dict:
    return {RESOURCE}_REGISTRY.get(key, DEFAULT)
```

### Real-World Example: Darwin's Hardcoded Registries

**1. Account Registry (15 LOC):**
```python
ACCOUNTS = {
    "****1234": {"name": "BoFA Checking", "type": "checking"},
    "****5678": {"name": "Apple Card", "type": "credit"}
}
```

**2. Merchant Rules (40 LOC):**
```python
CATEGORY_RULES = {
    "WHOLE FOODS": "Groceries",
    "SAFEWAY": "Groceries",
    "NETFLIX": "Entertainment",
    "SPOTIFY": "Entertainment",
    "SHELL": "Transportation",
    # ... 15 more
}
```

**3. Parser Registry (5 LOC):**
```python
PARSERS = {"bofa_pdf": BofaParser()}
```

**Total:** 60 LOC hardcoded (vs 1,300 LOC database equivalent)

---

## Related Decisions

- **ADR-0042:** Simplicity Profiles Pattern (defines Personal Profile characteristics)
- **ADR-0043:** Three-Tier Scaling Strategy (Personal tier = small, stable data)
- **ADR-0044:** YAGNI Principle (skip features → less data to manage → hardcoding viable)

---

## Review & Updates

- **2025-10-27:** Initial decision (based on Phase 3 Darwin implementation analysis)
- Next review: After 10+ Personal profile implementations

---

**Decision**: Hardcode small (<20 items), stable (<4 changes/year) data in Personal Profile.
**Status**: Accepted and implemented in Phase 3 (Darwin hardcodes accounts, merchants, categories).
