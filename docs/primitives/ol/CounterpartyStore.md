# CounterpartyStore (OL Primitive)

**Vertical:** 3.2 Counterparty Registry
**Layer:** Objective Layer (OL)
**Type:** Open Registry Store with Aliasing
**Status:** ✅ Specified

---

## Purpose

CRUD operations for counterparties (merchants, vendors, payees) with alias management, fuzzy matching support, and merge capabilities.

**Core Responsibility:** Persist and manage counterparty entities as an open registry (grows dynamically from transactions) with multiple aliases per entity.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY entity with multiple names/aliases:

| Domain | Entity Name | Examples | Aliases |
|--------|-------------|----------|---------|
| **Finance** | Merchant/Vendor | Amazon, Starbucks, Shell Gas | "AMAZON.COM", "Amazon Prime", "AMZ*Marketplace" |
| **Healthcare** | Medical Provider | Dr. Smith Clinic, Memorial Hospital | "Smith Family Practice", "J. Smith MD", "Smith Clinic" |
| **Legal** | Law Firm | Jones & Associates | "Jones Law", "Jones and Associates LLP", "Jones Legal Group" |
| **Research** | Research Institution | MIT, Stanford University | "Massachusetts Inst of Tech", "MIT Cambridge", "MIT.EDU" |
| **Manufacturing** | Supplier | Acme Steel Corp | "ACME STEEL", "Acme Steel Company", "Acme Corp" |
| **Media** | Content Creator | TechReviews Channel | "Tech Reviews", "TechReviews YT", "@techreviews" |

**Pattern:** Open Registry = entities created automatically from observations (transactions), support multiple aliases for fuzzy matching, allow merge when duplicates found.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Set
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Counterparty:
    counterparty_id: str
    user_id: str
    canonical_name: str  # Primary display name
    aliases: Set[str]    # All known aliases (including canonical_name)
    category: Optional[str]  # groceries, gas, dining, etc.
    is_active: bool
    transaction_count: int  # How many transactions reference this counterparty
    created_at: datetime
    updated_at: datetime

@dataclass
class FindOrCreateResult:
    counterparty: Counterparty
    created: bool  # True if new, False if existing
    matched_alias: Optional[str]  # Which alias matched (if found)

@dataclass
class MergeResult:
    primary: Counterparty
    merged_ids: List[str]
    migrated_transaction_count: int
    message: str

class CounterpartyStore:
    """
    Manages CRUD operations for counterparties with aliasing support.

    Enforces:
    - Aliases are unique across ALL counterparties (one alias = one counterparty)
    - Canonical name is always included in aliases
    - Case-insensitive alias matching
    - Soft delete (archive sets is_active = false)
    - Merge operation with transaction migration
    """

    def __init__(self, db_connection):
        self.db = db_connection

    def find_or_create(
        self,
        user_id: str,
        merchant_name: str,
        category: Optional[str] = None
    ) -> FindOrCreateResult:
        """
        Find existing counterparty by alias OR create new one.

        This is the PRIMARY entry point for transaction processing.
        When a transaction arrives with merchant_name="AMZN Mktp",
        system should find existing Amazon counterparty instead of creating duplicate.

        Args:
            user_id: Owner of the counterparty
            merchant_name: Raw merchant name from transaction (will become alias)
            category: Optional category hint (groceries, gas, etc.)

        Returns:
            FindOrCreateResult with counterparty, created flag, matched_alias

        Example:
            # First time seeing "AMAZON.COM"
            result = store.find_or_create("user_123", "AMAZON.COM")
            result.created = True
            result.counterparty.canonical_name = "AMAZON.COM"
            result.counterparty.aliases = {"AMAZON.COM"}

            # Second time seeing "Amazon Prime"
            result = store.find_or_create("user_123", "Amazon Prime")
            result.created = False  # Found existing Amazon
            result.matched_alias = "AMAZON.COM"  # If alias already exists
            # OR result.matched_alias = None + auto-add "Amazon Prime" as new alias
        """

    def create(
        self,
        user_id: str,
        canonical_name: str,
        aliases: Optional[Set[str]] = None,
        category: Optional[str] = None
    ) -> Counterparty:
        """
        Explicitly create new counterparty.

        Args:
            user_id: Owner of the counterparty
            canonical_name: Primary display name
            aliases: Initial set of aliases (canonical_name auto-added)
            category: Optional category (groceries, gas, etc.)

        Returns:
            Created Counterparty object

        Raises:
            DuplicateAliasError: If any alias already exists for another counterparty
            ValidationError: If canonical_name invalid
        """

    def get(self, counterparty_id: str, user_id: str) -> Optional[Counterparty]:
        """
        Get counterparty by ID.

        Args:
            counterparty_id: Counterparty ID to fetch
            user_id: Owner (for authorization)

        Returns:
            Counterparty if found and user owns it, None otherwise

        Raises:
            UnauthorizedError: If counterparty exists but user doesn't own it
        """

    def find_by_alias(
        self,
        user_id: str,
        alias: str
    ) -> Optional[Counterparty]:
        """
        Find counterparty by exact alias match (case-insensitive).

        Args:
            user_id: Owner of the counterparty
            alias: Alias to search for

        Returns:
            Counterparty if alias matches, None otherwise

        Example:
            cp = store.find_by_alias("user_123", "amazon.com")
            # Returns Amazon counterparty (alias match is case-insensitive)
        """

    def list(
        self,
        user_id: str,
        is_active: Optional[bool] = True,
        category: Optional[str] = None,
        min_transaction_count: int = 0
    ) -> List[Counterparty]:
        """
        List counterparties for user.

        Args:
            user_id: Owner of the counterparties
            is_active: Filter by active status (True = active only, False = archived only, None = all)
            category: Filter by category (optional)
            min_transaction_count: Only return counterparties with >= N transactions

        Returns:
            List of Counterparty objects (sorted by transaction_count DESC, then canonical_name ASC)
        """

    def update(
        self,
        counterparty_id: str,
        user_id: str,
        updates: dict
    ) -> Counterparty:
        """
        Update counterparty details.

        Args:
            counterparty_id: Counterparty to update
            user_id: Owner (for authorization)
            updates: Dictionary of fields to update (canonical_name, category)

        Returns:
            Updated Counterparty object

        Raises:
            CounterpartyNotFoundError: If counterparty doesn't exist
            UnauthorizedError: If user doesn't own counterparty
            ValidationError: If updated canonical_name invalid
        """

    def add_alias(
        self,
        counterparty_id: str,
        user_id: str,
        alias: str
    ) -> Counterparty:
        """
        Add new alias to counterparty.

        Args:
            counterparty_id: Counterparty to add alias to
            user_id: Owner (for authorization)
            alias: New alias to add

        Returns:
            Updated Counterparty object

        Raises:
            DuplicateAliasError: If alias already exists for another counterparty
            ValidationError: If alias invalid (empty, too long, etc.)
        """

    def remove_alias(
        self,
        counterparty_id: str,
        user_id: str,
        alias: str
    ) -> Counterparty:
        """
        Remove alias from counterparty.

        Args:
            counterparty_id: Counterparty to remove alias from
            user_id: Owner (for authorization)
            alias: Alias to remove

        Returns:
            Updated Counterparty object

        Raises:
            ValidationError: If trying to remove canonical_name (must keep at least one alias)
            AliasNotFoundError: If alias doesn't exist for this counterparty
        """

    def merge(
        self,
        user_id: str,
        primary_id: str,
        merge_ids: List[str]
    ) -> MergeResult:
        """
        Merge duplicate counterparties into primary.

        Steps:
        1. Validate all counterparties exist and user owns them
        2. Collect all aliases from merge_ids
        3. Update all transactions to reference primary_id
        4. Archive merged counterparties
        5. Add aliases to primary counterparty
        6. Return merge result

        Args:
            user_id: Owner (for authorization)
            primary_id: Keep this counterparty
            merge_ids: List of counterparty IDs to merge into primary

        Returns:
            MergeResult with primary, merged_ids, migrated_transaction_count, message

        Raises:
            CounterpartyNotFoundError: If any ID doesn't exist
            UnauthorizedError: If user doesn't own all counterparties
            ValidationError: If primary_id in merge_ids

        Example:
            # User realizes "Amazon", "AMZN", "Amazon Prime" are all same company
            result = store.merge(
                user_id="user_123",
                primary_id="cp_amazon_1",
                merge_ids=["cp_amzn_2", "cp_amazon_prime_3"]
            )
            # result.primary now has aliases: {"Amazon", "AMZN", "Amazon Prime", ...}
            # result.migrated_transaction_count = 45 (all transactions now point to cp_amazon_1)
        """

    def archive(
        self,
        counterparty_id: str,
        user_id: str
    ) -> Counterparty:
        """
        Archive counterparty (soft delete - set is_active = false).

        Counterparty remains in database to preserve transaction history.

        Args:
            counterparty_id: Counterparty to archive
            user_id: Owner (for authorization)

        Returns:
            Archived Counterparty object

        Raises:
            CounterpartyNotFoundError: If counterparty doesn't exist
            UnauthorizedError: If user doesn't own counterparty
        """

    def unarchive(
        self,
        counterparty_id: str,
        user_id: str
    ) -> Counterparty:
        """
        Unarchive counterparty (set is_active = true).

        Args:
            counterparty_id: Counterparty to unarchive
            user_id: Owner (for authorization)

        Returns:
            Unarchived Counterparty object

        Raises:
            CounterpartyNotFoundError: If counterparty doesn't exist
            UnauthorizedError: If user doesn't own counterparty
        """

    def _generate_counterparty_id(self, canonical_name: str) -> str:
        """
        Generate deterministic counterparty_id from canonical_name.

        Format: cp_{slug}_{sequence}
        Example: "Amazon" → "cp_amazon_1"

        Args:
            canonical_name: Counterparty canonical name

        Returns:
            counterparty_id string
        """
```

---

## Implementation Details

### Database Schema

```sql
CREATE TABLE counterparties (
    counterparty_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,

    -- Core fields
    canonical_name VARCHAR(200) NOT NULL,
    category VARCHAR(50),                   -- Optional category
    transaction_count INT DEFAULT 0,        -- Cached count

    -- Metadata
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    -- Constraints
    INDEX idx_user_active (user_id, is_active),
    INDEX idx_category (category),
    INDEX idx_txn_count (transaction_count),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Aliases table (one-to-many: one counterparty has many aliases)
CREATE TABLE counterparty_aliases (
    alias_id VARCHAR(50) PRIMARY KEY,
    counterparty_id VARCHAR(50) NOT NULL,
    user_id VARCHAR(50) NOT NULL,

    alias VARCHAR(200) NOT NULL,            -- The actual alias text
    is_canonical BOOLEAN DEFAULT FALSE,     -- True for canonical_name

    created_at TIMESTAMP NOT NULL,

    -- Constraints
    UNIQUE(user_id, alias),                 -- One alias = one counterparty (case-insensitive)
    INDEX idx_counterparty (counterparty_id),
    INDEX idx_user (user_id),
    FOREIGN KEY (counterparty_id) REFERENCES counterparties(counterparty_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Change log for audit trail
CREATE TABLE counterparty_change_log (
    log_id VARCHAR(50) PRIMARY KEY,
    counterparty_id VARCHAR(50) NOT NULL,
    user_id VARCHAR(50) NOT NULL,

    operation VARCHAR(20) NOT NULL,         -- CREATE, UPDATE, ADD_ALIAS, REMOVE_ALIAS, MERGE, ARCHIVE
    changes JSONB,                          -- {field: {old: X, new: Y}}
    timestamp TIMESTAMP NOT NULL,

    FOREIGN KEY (counterparty_id) REFERENCES counterparties(counterparty_id),
    INDEX idx_counterparty (counterparty_id),
    INDEX idx_user (user_id)
);
```

### Counterparty ID Generation

```python
def _generate_counterparty_id(self, canonical_name: str) -> str:
    """
    Generate counterparty_id: cp_{slug}_{sequence}

    Examples:
      "Amazon" → "cp_amazon_1"
      "Starbucks Coffee" → "cp_starbucks_coffee_1"
      "Shell Gas Station" → "cp_shell_gas_station_1"
    """
    # Create slug
    slug = canonical_name.lower()
    slug = slug.replace(' ', '_')
    slug = re.sub(r'[^a-z0-9_]', '', slug)  # Remove special chars
    slug = slug[:30]  # Limit slug length

    # Find next sequence number
    existing = self.db.execute(
        "SELECT counterparty_id FROM counterparties WHERE counterparty_id LIKE ? ORDER BY counterparty_id DESC LIMIT 1",
        (f"cp_{slug}_%",)
    ).fetchone()

    if existing:
        # Extract sequence from "cp_amazon_3" → 3
        last_seq = int(existing['counterparty_id'].split('_')[-1])
        seq = last_seq + 1
    else:
        seq = 1

    return f"cp_{slug}_{seq}"
```

### Find or Create Logic

```python
def find_or_create(
    self,
    user_id: str,
    merchant_name: str,
    category: Optional[str] = None
) -> FindOrCreateResult:
    """
    Core logic for transaction processing.

    Steps:
    1. Normalize merchant_name (strip whitespace, uppercase for comparison)
    2. Look for exact alias match
    3. If found, return existing counterparty
    4. If not found, create new counterparty with merchant_name as canonical_name and alias
    """
    # 1. Normalize
    normalized = merchant_name.strip()

    if not normalized:
        raise ValidationError("Merchant name cannot be empty")

    # 2. Find by exact alias match (case-insensitive)
    existing = self.find_by_alias(user_id, normalized)

    if existing:
        # 3. Found existing
        return FindOrCreateResult(
            counterparty=existing,
            created=False,
            matched_alias=normalized
        )

    # 4. Not found - create new
    counterparty = self.create(
        user_id=user_id,
        canonical_name=normalized,
        aliases={normalized},
        category=category
    )

    return FindOrCreateResult(
        counterparty=counterparty,
        created=True,
        matched_alias=None
    )
```

### Alias Management

```python
def add_alias(
    self,
    counterparty_id: str,
    user_id: str,
    alias: str
) -> Counterparty:
    """
    Add new alias to counterparty.

    Steps:
    1. Verify counterparty exists and user owns it
    2. Check alias doesn't already exist for ANOTHER counterparty
    3. Add alias to counterparty_aliases table
    4. Update counterparty.updated_at
    5. Log operation
    """
    # 1. Verify ownership
    counterparty = self.get(counterparty_id, user_id)
    if not counterparty:
        raise CounterpartyNotFoundError(f"Counterparty '{counterparty_id}' not found")

    # 2. Check for duplicate alias in OTHER counterparties
    existing = self.db.execute(
        """
        SELECT counterparty_id FROM counterparty_aliases
        WHERE user_id = ? AND LOWER(alias) = LOWER(?) AND counterparty_id != ?
        """,
        (user_id, alias, counterparty_id)
    ).fetchone()

    if existing:
        raise DuplicateAliasError(
            f"Alias '{alias}' already exists for another counterparty",
            alias=alias,
            existing_counterparty_id=existing['counterparty_id']
        )

    # 3. Check if alias already exists for THIS counterparty (idempotent)
    already_exists = self.db.execute(
        """
        SELECT alias_id FROM counterparty_aliases
        WHERE counterparty_id = ? AND LOWER(alias) = LOWER(?)
        """,
        (counterparty_id, alias)
    ).fetchone()

    if already_exists:
        # Alias already exists for this counterparty - return as-is
        return counterparty

    # 4. Add alias
    alias_id = f"alias_{int(time.time() * 1000)}"
    self.db.execute(
        """
        INSERT INTO counterparty_aliases (alias_id, counterparty_id, user_id, alias, is_canonical, created_at)
        VALUES (?, ?, ?, ?, ?, ?)
        """,
        (alias_id, counterparty_id, user_id, alias, False, datetime.utcnow())
    )

    # 5. Update counterparty timestamp
    self.db.execute(
        "UPDATE counterparties SET updated_at = ? WHERE counterparty_id = ?",
        (datetime.utcnow(), counterparty_id)
    )

    # 6. Log
    self._log_change(
        counterparty_id=counterparty_id,
        user_id=user_id,
        operation="ADD_ALIAS",
        changes={"alias": {"old": None, "new": alias}}
    )

    # 7. Reload and return
    counterparty.aliases.add(alias)
    return counterparty
```

### Merge Operation

```python
def merge(
    self,
    user_id: str,
    primary_id: str,
    merge_ids: List[str]
) -> MergeResult:
    """
    Merge duplicate counterparties.

    Example:
      Primary: cp_amazon_1 (aliases: {"Amazon"})
      Merge: [cp_amzn_2 (aliases: {"AMZN", "AMZ"}), cp_amazon_prime_3 (aliases: {"Amazon Prime"})]

      Result:
      - cp_amazon_1 now has aliases: {"Amazon", "AMZN", "AMZ", "Amazon Prime"}
      - All transactions from cp_amzn_2 and cp_amazon_prime_3 now reference cp_amazon_1
      - cp_amzn_2 and cp_amazon_prime_3 are archived
    """
    # 1. Validate
    if primary_id in merge_ids:
        raise ValidationError("Cannot merge primary counterparty into itself")

    primary = self.get(primary_id, user_id)
    if not primary:
        raise CounterpartyNotFoundError(f"Primary counterparty '{primary_id}' not found")

    merge_counterparties = []
    for merge_id in merge_ids:
        cp = self.get(merge_id, user_id)
        if not cp:
            raise CounterpartyNotFoundError(f"Merge counterparty '{merge_id}' not found")
        merge_counterparties.append(cp)

    # 2. Collect all aliases from merge counterparties
    all_aliases = set()
    for cp in merge_counterparties:
        all_aliases.update(cp.aliases)

    # 3. Start transaction
    with self.db.transaction():
        # 4. Migrate transactions
        total_migrated = 0
        for merge_id in merge_ids:
            result = self.db.execute(
                "UPDATE canonical_transactions SET counterparty_id = ? WHERE counterparty_id = ?",
                (primary_id, merge_id)
            )
            total_migrated += result.rowcount

        # 5. Update primary transaction_count
        self.db.execute(
            "UPDATE counterparties SET transaction_count = transaction_count + ? WHERE counterparty_id = ?",
            (total_migrated, primary_id)
        )

        # 6. Add all aliases to primary (skip duplicates)
        for alias in all_aliases:
            try:
                # Update counterparty_aliases to point to primary
                self.db.execute(
                    """
                    UPDATE counterparty_aliases
                    SET counterparty_id = ?
                    WHERE user_id = ? AND LOWER(alias) = LOWER(?)
                    """,
                    (primary_id, user_id, alias)
                )
            except IntegrityError:
                # Alias already exists for primary - skip
                pass

        # 7. Archive merged counterparties
        for merge_id in merge_ids:
            self.db.execute(
                "UPDATE counterparties SET is_active = false, updated_at = ? WHERE counterparty_id = ?",
                (datetime.utcnow(), merge_id)
            )

        # 8. Log merge operation
        self._log_change(
            counterparty_id=primary_id,
            user_id=user_id,
            operation="MERGE",
            changes={
                "merged_ids": {"old": None, "new": merge_ids},
                "migrated_transactions": {"old": None, "new": total_migrated},
                "added_aliases": {"old": None, "new": list(all_aliases)}
            }
        )

    # 9. Return result
    primary.aliases.update(all_aliases)
    return MergeResult(
        primary=primary,
        merged_ids=merge_ids,
        migrated_transaction_count=total_migrated,
        message=f"Merged {len(merge_ids)} counterparties. Migrated {total_migrated} transactions."
    )
```

---

## Usage Examples

### Example 1: Find or Create (First Time)

```python
store = CounterpartyStore(db_connection)

# Transaction arrives with merchant_name="AMAZON.COM"
result = store.find_or_create(
    user_id="user_darwin",
    merchant_name="AMAZON.COM"
)

print(result.created)  # True (new counterparty)
print(result.counterparty.canonical_name)  # "AMAZON.COM"
print(result.counterparty.aliases)  # {"AMAZON.COM"}
print(result.matched_alias)  # None
```

### Example 2: Find or Create (Subsequent Times)

```python
# Second transaction: "Amazon Prime"
# User manually adds "Amazon Prime" as alias to existing Amazon counterparty

store.add_alias("cp_amazon_1", "user_darwin", "Amazon Prime")

# Third transaction arrives with merchant_name="Amazon Prime"
result = store.find_or_create(
    user_id="user_darwin",
    merchant_name="Amazon Prime"
)

print(result.created)  # False (found existing)
print(result.counterparty.canonical_name)  # "AMAZON.COM"
print(result.counterparty.aliases)  # {"AMAZON.COM", "Amazon Prime"}
print(result.matched_alias)  # "Amazon Prime"
```

### Example 3: Add Multiple Aliases

```python
# User wants to consolidate all Amazon variants
cp = store.get("cp_amazon_1", "user_darwin")

store.add_alias("cp_amazon_1", "user_darwin", "AMZN")
store.add_alias("cp_amazon_1", "user_darwin", "AMZ Mktp")
store.add_alias("cp_amazon_1", "user_darwin", "Amazon Marketplace")

cp = store.get("cp_amazon_1", "user_darwin")
print(cp.aliases)
# {"AMAZON.COM", "Amazon Prime", "AMZN", "AMZ Mktp", "Amazon Marketplace"}
```

### Example 4: Merge Duplicates

```python
# User accidentally created separate counterparties:
# - cp_amazon_1 (canonical: "Amazon")
# - cp_amzn_2 (canonical: "AMZN")
# - cp_amazon_prime_3 (canonical: "Amazon Prime")

result = store.merge(
    user_id="user_darwin",
    primary_id="cp_amazon_1",
    merge_ids=["cp_amzn_2", "cp_amazon_prime_3"]
)

print(result.message)
# "Merged 2 counterparties. Migrated 45 transactions."

print(result.primary.canonical_name)  # "Amazon"
print(result.primary.aliases)
# {"Amazon", "AMZN", "Amazon Prime", ...all other aliases...}

print(result.migrated_transaction_count)  # 45
```

### Example 5: List Top Counterparties

```python
# Get top counterparties by transaction count
counterparties = store.list(
    user_id="user_darwin",
    is_active=True,
    min_transaction_count=5
)

for cp in counterparties:
    print(f"{cp.canonical_name}: {cp.transaction_count} transactions, {len(cp.aliases)} aliases")

# Output:
# Amazon: 145 transactions, 8 aliases
# Starbucks: 87 transactions, 3 aliases
# Shell Gas: 62 transactions, 5 aliases
```

---

## Multi-Domain Examples

### Healthcare: Medical Provider Store

```python
class MedicalProviderStore(OpenRegistryStore):
    """Same pattern for healthcare domain."""

    # Example: Dr. Smith has many name variations in insurance claims
    provider = store.find_or_create(
        user_id="patient_123",
        provider_name="Smith Family Practice"
    )

    # Add aliases for common variations
    store.add_alias(provider.id, "patient_123", "Dr. John Smith")
    store.add_alias(provider.id, "patient_123", "J. Smith MD")
    store.add_alias(provider.id, "patient_123", "Smith Clinic")

    # Merge duplicates
    result = store.merge(
        user_id="patient_123",
        primary_id="mp_smith_1",
        merge_ids=["mp_dr_smith_2", "mp_smith_clinic_3"]
    )
```

### Legal: Law Firm Registry

```python
class LawFirmStore(OpenRegistryStore):
    """Same pattern for legal domain."""

    # Law firm has multiple name variations in billing records
    firm = store.find_or_create(
        user_id="lawyer_456",
        firm_name="Jones & Associates"
    )

    store.add_alias(firm.id, "lawyer_456", "Jones Law")
    store.add_alias(firm.id, "lawyer_456", "Jones and Associates LLP")
    store.add_alias(firm.id, "lawyer_456", "Jones Legal Group")
```

### Research: Institution Registry

```python
class ResearchInstitutionStore(OpenRegistryStore):
    """Same pattern for research domain."""

    # Research institution has many name variations in publications
    institution = store.find_or_create(
        user_id="researcher_789",
        institution_name="MIT"
    )

    store.add_alias(institution.id, "researcher_789", "Massachusetts Institute of Technology")
    store.add_alias(institution.id, "researcher_789", "MIT Cambridge")
    store.add_alias(institution.id, "researcher_789", "MIT.EDU")
```

---

## Error Handling

### Custom Exceptions

```python
class CounterpartyStoreError(Exception):
    """Base exception for CounterpartyStore errors."""
    pass

class DuplicateAliasError(CounterpartyStoreError):
    def __init__(self, message: str, alias: str, existing_counterparty_id: str):
        self.message = message
        self.alias = alias
        self.existing_counterparty_id = existing_counterparty_id
        super().__init__(message)

class CounterpartyNotFoundError(CounterpartyStoreError):
    pass

class UnauthorizedError(CounterpartyStoreError):
    pass

class AliasNotFoundError(CounterpartyStoreError):
    def __init__(self, alias: str):
        self.alias = alias
        self.message = f"Alias '{alias}' not found"
        super().__init__(self.message)

class ValidationError(CounterpartyStoreError):
    pass
```

---

## Performance Characteristics

### Query Performance

| Operation | Query Type | Expected Latency (p95) | Index Used |
|-----------|------------|------------------------|------------|
| `find_or_create()` | SELECT + INSERT | <30ms | idx_user (alias lookup) |
| `find_by_alias()` | SELECT | <10ms | UNIQUE(user_id, alias) |
| `get()` | SELECT by PK | <10ms | PRIMARY KEY |
| `list()` | SELECT by user_id | <50ms | idx_user_active |
| `add_alias()` | INSERT | <20ms | UNIQUE(user_id, alias) |
| `merge()` | UPDATE + INSERT | <200ms | Transaction migration (bulk update) |

### Caching Strategy

```python
# Cache counterparty aliases for fast lookup
cache_key = f"counterparty:aliases:{user_id}"
ttl = 600  # 10 minutes

def find_by_alias_cached(user_id: str, alias: str) -> Optional[Counterparty]:
    """Get counterparty from cache or database."""
    # Build in-memory alias map
    alias_map = redis.get(cache_key)
    if not alias_map:
        alias_map = self._build_alias_map(user_id)
        redis.setex(cache_key, ttl, json.dumps(alias_map))

    counterparty_id = alias_map.get(alias.lower())
    if counterparty_id:
        return self.get(counterparty_id, user_id)

    return None

def _invalidate_cache(user_id: str):
    """Invalidate cache on add_alias, merge, etc."""
    redis.delete(f"counterparty:aliases:{user_id}")
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Find or create
def test_find_or_create_new_counterparty()
def test_find_or_create_existing_by_exact_alias()
def test_find_or_create_case_insensitive()

# Test: Alias management
def test_add_alias_success()
def test_add_alias_duplicate_error()
def test_add_alias_idempotent()
def test_remove_alias_success()
def test_remove_alias_cannot_remove_canonical()

# Test: Merge operation
def test_merge_success()
def test_merge_migrate_transactions()
def test_merge_combine_aliases()
def test_merge_archive_merged_counterparties()
def test_merge_primary_in_merge_list_error()

# Test: List counterparties
def test_list_active_counterparties()
def test_list_by_category()
def test_list_min_transaction_count()
def test_list_sorted_by_transaction_count()

# Test: Authorization
def test_get_counterparty_unauthorized()
def test_add_alias_unauthorized()
def test_merge_unauthorized()
```

---

## Related Primitives

- **CounterpartyMatcher** (OL) - Fuzzy matching to find counterparties from raw names
- **AliasMerger** (OL) - Merge duplicate counterparties with rollback support
- **CounterpartySelector** (IL) - UI component for selecting/creating counterparty
- **CanonicalStore** (from Vertical 1.3) - Transactions reference counterparty_id

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 3-4 days
**Dependencies:** Database (PostgreSQL/MySQL), Transaction table
