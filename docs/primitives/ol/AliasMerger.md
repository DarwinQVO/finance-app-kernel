# AliasMerger (OL Primitive)

**Vertical:** 3.2 Counterparty Registry
**Layer:** Objective Layer (OL)
**Type:** Data Consolidation Engine
**Status:** ✅ Specified

---

## Purpose

Safely merges duplicate counterparties by consolidating aliases, migrating foreign key references, and providing rollback on failure.

**Core Responsibility:** Execute multi-step merge operations with transaction safety, data integrity validation, and audit logging.

---

## Multi-Domain Applicability

Merge operations apply to ANY domain with duplicate entities:

| Domain | Entity | Merge Scenario |
|--------|--------|----------------|
| **Finance** | Merchant | Merge "Amazon", "AMZN", "Amazon Prime" into single "Amazon" entity |
| **Healthcare** | Provider | Merge "Dr Smith", "Smith Clinic", "J. Smith MD" into single provider |
| **Legal** | Law Firm | Merge "Jones Law", "Jones & Associates", "Jones Legal" into single firm |
| **Research** | Institution | Merge "MIT", "Massachusetts Inst of Tech", "MIT Cambridge" into single institution |
| **Manufacturing** | Supplier | Merge "Acme Steel", "ACME STEEL CO", "Acme Corp" into single supplier |
| **Media** | Creator | Merge "TechReviews", "Tech Reviews YT", "@techreviews" into single creator |

**Pattern:** Merge Engine = validate inputs, migrate references, consolidate aliases, archive duplicates, rollback on failure.

---

## Interface Contract

### Python Interface

```python
from typing import List, Optional
from dataclasses import dataclass
from datetime import datetime
from enum import Enum

class MergeStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    ROLLED_BACK = "rolled_back"

@dataclass
class MergePreview:
    primary_id: str
    primary_name: str
    merge_ids: List[str]
    merge_names: List[str]
    total_aliases: int                # How many aliases will be consolidated
    total_transactions: int           # How many transactions will be migrated
    potential_conflicts: List[str]    # Any validation warnings
    is_safe: bool                     # False if conflicts detected

@dataclass
class MergeResult:
    merge_id: str
    status: MergeStatus
    primary_id: str
    primary_name: str
    merged_ids: List[str]
    migrated_transaction_count: int
    consolidated_alias_count: int
    execution_time_ms: int
    message: str
    error: Optional[str] = None

@dataclass
class RollbackResult:
    merge_id: str
    status: MergeStatus
    restored_counterparties: List[str]
    restored_transactions: int
    message: str

class AliasMerger:
    """
    Safely merges duplicate counterparties with rollback support.

    Merge Process:
    1. Validation - verify all counterparties exist, user owns them, no circular refs
    2. Preview - calculate impact (aliases, transactions, conflicts)
    3. Execution:
       a. Start database transaction
       b. Create merge record (audit trail)
       c. Migrate transactions (update foreign keys)
       d. Consolidate aliases (move to primary, de-duplicate)
       e. Update transaction counts
       f. Archive merged counterparties
       g. Commit transaction
    4. Rollback (if failure):
       a. Restore archived counterparties
       b. Restore original transaction references
       c. Restore original aliases
       d. Mark merge as ROLLED_BACK

    Safety Guarantees:
    - All-or-nothing: Either full merge succeeds or full rollback
    - No data loss: Archived entities preserved with original data
    - Audit trail: Every merge logged with before/after state
    - Idempotent: Re-running same merge has no additional effect
    """

    def __init__(self, db_connection, counterparty_store):
        self.db = db_connection
        self.store = counterparty_store

    def preview_merge(
        self,
        user_id: str,
        primary_id: str,
        merge_ids: List[str]
    ) -> MergePreview:
        """
        Preview merge impact before execution.

        Shows user what will happen:
        - How many aliases will be consolidated
        - How many transactions will be migrated
        - Any potential conflicts (duplicate aliases, circular refs)

        Args:
            user_id: Owner (for authorization)
            primary_id: Keep this counterparty
            merge_ids: List of counterparty IDs to merge into primary

        Returns:
            MergePreview with impact analysis

        Raises:
            CounterpartyNotFoundError: If any ID doesn't exist
            UnauthorizedError: If user doesn't own all counterparties

        Example:
            preview = merger.preview_merge(
                user_id="user_123",
                primary_id="cp_amazon_1",
                merge_ids=["cp_amzn_2", "cp_amazon_prime_3"]
            )
            print(f"Will consolidate {preview.total_aliases} aliases")
            print(f"Will migrate {preview.total_transactions} transactions")
            if not preview.is_safe:
                print("Conflicts:", preview.potential_conflicts)
        """

    def merge(
        self,
        user_id: str,
        primary_id: str,
        merge_ids: List[str]
    ) -> MergeResult:
        """
        Execute merge operation.

        Steps:
        1. Validate inputs
        2. Start database transaction
        3. Create merge record
        4. Migrate transactions
        5. Consolidate aliases
        6. Update counts
        7. Archive merged counterparties
        8. Commit transaction
        9. Log operation

        Args:
            user_id: Owner (for authorization)
            primary_id: Keep this counterparty
            merge_ids: List of counterparty IDs to merge into primary

        Returns:
            MergeResult with execution details

        Raises:
            CounterpartyNotFoundError: If any ID doesn't exist
            UnauthorizedError: If user doesn't own all counterparties
            ValidationError: If primary_id in merge_ids or other validation fails
            MergeError: If execution fails (automatic rollback triggered)

        Example:
            result = merger.merge(
                user_id="user_123",
                primary_id="cp_amazon_1",
                merge_ids=["cp_amzn_2", "cp_amazon_prime_3"]
            )
            print(result.message)  # "Merged 2 counterparties. Migrated 45 transactions."
            print(result.status)   # MergeStatus.COMPLETED
        """

    def rollback_merge(
        self,
        user_id: str,
        merge_id: str
    ) -> RollbackResult:
        """
        Rollback a completed merge.

        Restores system to pre-merge state:
        - Unarchive merged counterparties
        - Restore original transaction references
        - Restore original aliases

        Args:
            user_id: Owner (for authorization)
            merge_id: Merge operation to rollback

        Returns:
            RollbackResult with restoration details

        Raises:
            MergeNotFoundError: If merge_id doesn't exist
            UnauthorizedError: If user doesn't own merge
            RollbackError: If rollback fails (data corruption risk)

        Example:
            rollback = merger.rollback_merge("user_123", "merge_1234567890")
            print(rollback.message)  # "Rolled back merge. Restored 2 counterparties, 45 transactions."
        """

    def list_merges(
        self,
        user_id: str,
        status: Optional[MergeStatus] = None,
        limit: int = 10
    ) -> List[MergeResult]:
        """
        List merge operations for user.

        Args:
            user_id: Owner of merges
            status: Filter by status (optional)
            limit: Maximum results to return

        Returns:
            List of MergeResult objects (sorted by timestamp DESC)

        Example:
            merges = merger.list_merges("user_123", status=MergeStatus.COMPLETED)
            for m in merges:
                print(f"{m.merge_id}: {m.primary_name} ← {len(m.merged_ids)} merged")
        """

    def _validate_merge_inputs(
        self,
        user_id: str,
        primary_id: str,
        merge_ids: List[str]
    ) -> None:
        """
        Validate merge inputs before execution.

        Checks:
        1. primary_id not in merge_ids
        2. All IDs exist
        3. User owns all counterparties
        4. No circular references

        Raises:
            ValidationError: If validation fails
        """

    def _migrate_transactions(
        self,
        primary_id: str,
        merge_ids: List[str]
    ) -> int:
        """
        Migrate all transactions from merge_ids to primary_id.

        Uses batch UPDATE for performance.

        Args:
            primary_id: Target counterparty
            merge_ids: Source counterparties

        Returns:
            Number of transactions migrated

        Example:
            # UPDATE canonical_transactions
            # SET counterparty_id = 'cp_amazon_1'
            # WHERE counterparty_id IN ('cp_amzn_2', 'cp_amazon_prime_3')
        """

    def _consolidate_aliases(
        self,
        user_id: str,
        primary_id: str,
        merge_ids: List[str]
    ) -> int:
        """
        Consolidate aliases from merge_ids into primary_id.

        Steps:
        1. Get all aliases from merge counterparties
        2. De-duplicate (remove exact duplicates)
        3. Move aliases to primary counterparty
        4. Handle canonical_name conflicts (keep primary's canonical)

        Args:
            user_id: Owner
            primary_id: Target counterparty
            merge_ids: Source counterparties

        Returns:
            Number of aliases consolidated (after de-duplication)

        Example:
            Primary: {"Amazon", "Amazon.com"}
            Merge 1: {"AMZN", "AMZ"}
            Merge 2: {"Amazon Prime", "Amazon.com"}  # Duplicate "Amazon.com"

            Result: {"Amazon", "Amazon.com", "AMZN", "AMZ", "Amazon Prime"}
            # Only 3 new aliases added (de-duplicated)
        """

    def _archive_merged_counterparties(
        self,
        merge_ids: List[str]
    ) -> None:
        """
        Archive merged counterparties.

        Sets is_active = false for all merge_ids.

        Args:
            merge_ids: Counterparties to archive
        """

    def _create_merge_record(
        self,
        user_id: str,
        primary_id: str,
        merge_ids: List[str]
    ) -> str:
        """
        Create merge audit record.

        Stores:
        - merge_id (unique identifier)
        - user_id
        - primary_id
        - merge_ids (JSON array)
        - status
        - before_state (snapshot of data before merge)
        - after_state (snapshot of data after merge)
        - created_at

        Returns:
            merge_id string

        Example:
            merge_id = "merge_1234567890"
        """
```

---

## Implementation Details

### Database Schema

```sql
-- Merge audit log
CREATE TABLE counterparty_merges (
    merge_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,

    -- Merge details
    primary_id VARCHAR(50) NOT NULL,
    merge_ids JSONB NOT NULL,               -- ["cp_amzn_2", "cp_amazon_prime_3"]
    status VARCHAR(20) NOT NULL,            -- PENDING, IN_PROGRESS, COMPLETED, FAILED, ROLLED_BACK

    -- Impact metrics
    migrated_transaction_count INT,
    consolidated_alias_count INT,
    execution_time_ms INT,

    -- State snapshots (for rollback)
    before_state JSONB,                     -- Snapshot before merge
    after_state JSONB,                      -- Snapshot after merge

    -- Timestamps
    created_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    rolled_back_at TIMESTAMP,

    -- Error tracking
    error_message TEXT,

    INDEX idx_user (user_id),
    INDEX idx_primary (primary_id),
    INDEX idx_status (status),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (primary_id) REFERENCES counterparties(counterparty_id)
);
```

### Preview Merge Logic

```python
def preview_merge(
    self,
    user_id: str,
    primary_id: str,
    merge_ids: List[str]
) -> MergePreview:
    """
    Calculate merge impact before execution.
    """
    # 1. Validate inputs
    self._validate_merge_inputs(user_id, primary_id, merge_ids)

    # 2. Get all counterparties
    primary = self.store.get(primary_id, user_id)
    merge_cps = [self.store.get(mid, user_id) for mid in merge_ids]

    # 3. Calculate total aliases (de-duplicated)
    all_aliases = set(primary.aliases)
    for cp in merge_cps:
        all_aliases.update(cp.aliases)
    total_aliases = len(all_aliases)

    # 4. Calculate total transactions
    total_txns = sum(cp.transaction_count for cp in merge_cps)

    # 5. Check for potential conflicts
    conflicts = []

    # Check for circular references (shouldn't happen, but validate)
    if primary_id in merge_ids:
        conflicts.append(f"Primary counterparty '{primary_id}' cannot be in merge list")

    # Check for duplicate canonical names
    all_canonical_names = [cp.canonical_name for cp in merge_cps]
    if len(all_canonical_names) != len(set(all_canonical_names)):
        conflicts.append("Multiple merge counterparties have same canonical name")

    # 6. Return preview
    return MergePreview(
        primary_id=primary_id,
        primary_name=primary.canonical_name,
        merge_ids=merge_ids,
        merge_names=[cp.canonical_name for cp in merge_cps],
        total_aliases=total_aliases,
        total_transactions=total_txns,
        potential_conflicts=conflicts,
        is_safe=len(conflicts) == 0
    )
```

### Merge Execution

```python
import time

def merge(
    self,
    user_id: str,
    primary_id: str,
    merge_ids: List[str]
) -> MergeResult:
    """
    Execute merge with full transaction safety.
    """
    start_time = time.time()

    # 1. Validate
    self._validate_merge_inputs(user_id, primary_id, merge_ids)

    # 2. Create merge record
    merge_id = self._create_merge_record(user_id, primary_id, merge_ids)

    try:
        # 3. Update status to IN_PROGRESS
        self.db.execute(
            "UPDATE counterparty_merges SET status = ? WHERE merge_id = ?",
            (MergeStatus.IN_PROGRESS.value, merge_id)
        )

        # 4. Start database transaction
        with self.db.transaction():
            # 5. Migrate transactions
            migrated_count = self._migrate_transactions(primary_id, merge_ids)

            # 6. Consolidate aliases
            consolidated_count = self._consolidate_aliases(user_id, primary_id, merge_ids)

            # 7. Update primary transaction_count
            self.db.execute(
                "UPDATE counterparties SET transaction_count = transaction_count + ? WHERE counterparty_id = ?",
                (migrated_count, primary_id)
            )

            # 8. Archive merged counterparties
            self._archive_merged_counterparties(merge_ids)

            # 9. Update merge record to COMPLETED
            execution_time = int((time.time() - start_time) * 1000)
            self.db.execute(
                """
                UPDATE counterparty_merges
                SET status = ?, migrated_transaction_count = ?, consolidated_alias_count = ?,
                    execution_time_ms = ?, completed_at = ?
                WHERE merge_id = ?
                """,
                (MergeStatus.COMPLETED.value, migrated_count, consolidated_count,
                 execution_time, datetime.utcnow(), merge_id)
            )

        # 10. Return result
        primary = self.store.get(primary_id, user_id)
        return MergeResult(
            merge_id=merge_id,
            status=MergeStatus.COMPLETED,
            primary_id=primary_id,
            primary_name=primary.canonical_name,
            merged_ids=merge_ids,
            migrated_transaction_count=migrated_count,
            consolidated_alias_count=consolidated_count,
            execution_time_ms=execution_time,
            message=f"Merged {len(merge_ids)} counterparties. Migrated {migrated_count} transactions, consolidated {consolidated_count} aliases."
        )

    except Exception as e:
        # 11. Mark as FAILED
        self.db.execute(
            "UPDATE counterparty_merges SET status = ?, error_message = ? WHERE merge_id = ?",
            (MergeStatus.FAILED.value, str(e), merge_id)
        )

        # 12. Automatic rollback (transaction already rolled back by DB)
        # Log error for debugging
        raise MergeError(f"Merge failed: {str(e)}") from e
```

### Transaction Migration

```python
def _migrate_transactions(
    self,
    primary_id: str,
    merge_ids: List[str]
) -> int:
    """
    Batch update all transactions to reference primary_id.

    Uses single UPDATE query for performance.
    """
    # Build WHERE clause for all merge_ids
    placeholders = ', '.join(['?'] * len(merge_ids))
    query = f"""
        UPDATE canonical_transactions
        SET counterparty_id = ?
        WHERE counterparty_id IN ({placeholders})
    """

    # Execute batch update
    result = self.db.execute(query, [primary_id] + merge_ids)

    return result.rowcount
```

### Alias Consolidation

```python
def _consolidate_aliases(
    self,
    user_id: str,
    primary_id: str,
    merge_ids: List[str]
) -> int:
    """
    Consolidate aliases from merge counterparties into primary.

    Steps:
    1. Get all aliases from merge counterparties
    2. De-duplicate (case-insensitive)
    3. Move aliases to primary counterparty
    """
    consolidated_count = 0

    # 1. Get all aliases to migrate
    placeholders = ', '.join(['?'] * len(merge_ids))
    aliases_to_migrate = self.db.execute(
        f"""
        SELECT alias_id, alias FROM counterparty_aliases
        WHERE counterparty_id IN ({placeholders})
        """,
        merge_ids
    ).fetchall()

    # 2. Get existing primary aliases (for de-duplication)
    existing_aliases = set(
        row['alias'].lower() for row in self.db.execute(
            "SELECT alias FROM counterparty_aliases WHERE counterparty_id = ?",
            (primary_id,)
        ).fetchall()
    )

    # 3. Migrate aliases (skip duplicates)
    for row in aliases_to_migrate:
        alias = row['alias']
        alias_id = row['alias_id']

        # Skip if duplicate (case-insensitive)
        if alias.lower() in existing_aliases:
            # Delete duplicate alias
            self.db.execute("DELETE FROM counterparty_aliases WHERE alias_id = ?", (alias_id,))
            continue

        # Move alias to primary
        self.db.execute(
            "UPDATE counterparty_aliases SET counterparty_id = ?, is_canonical = ? WHERE alias_id = ?",
            (primary_id, False, alias_id)  # Non-canonical (primary keeps its canonical)
        )
        existing_aliases.add(alias.lower())
        consolidated_count += 1

    return consolidated_count
```

### Rollback Logic

```python
def rollback_merge(
    self,
    user_id: str,
    merge_id: str
) -> RollbackResult:
    """
    Rollback a completed merge.

    Uses before_state snapshot to restore original state.
    """
    # 1. Get merge record
    merge_record = self.db.execute(
        "SELECT * FROM counterparty_merges WHERE merge_id = ? AND user_id = ?",
        (merge_id, user_id)
    ).fetchone()

    if not merge_record:
        raise MergeNotFoundError(f"Merge '{merge_id}' not found")

    if merge_record['status'] != MergeStatus.COMPLETED.value:
        raise RollbackError(f"Cannot rollback merge with status '{merge_record['status']}'")

    # 2. Get before_state snapshot
    before_state = json.loads(merge_record['before_state'])

    try:
        with self.db.transaction():
            # 3. Restore transactions
            restored_txns = 0
            for counterparty_id, txn_ids in before_state['transaction_mapping'].items():
                self.db.execute(
                    f"""
                    UPDATE canonical_transactions
                    SET counterparty_id = ?
                    WHERE transaction_id IN ({', '.join(['?'] * len(txn_ids))})
                    """,
                    [counterparty_id] + txn_ids
                )
                restored_txns += len(txn_ids)

            # 4. Restore aliases
            for alias_data in before_state['aliases']:
                self.db.execute(
                    """
                    UPDATE counterparty_aliases
                    SET counterparty_id = ?
                    WHERE alias_id = ?
                    """,
                    (alias_data['counterparty_id'], alias_data['alias_id'])
                )

            # 5. Unarchive merged counterparties
            merge_ids = json.loads(merge_record['merge_ids'])
            placeholders = ', '.join(['?'] * len(merge_ids))
            self.db.execute(
                f"UPDATE counterparties SET is_active = true WHERE counterparty_id IN ({placeholders})",
                merge_ids
            )

            # 6. Update merge record
            self.db.execute(
                "UPDATE counterparty_merges SET status = ?, rolled_back_at = ? WHERE merge_id = ?",
                (MergeStatus.ROLLED_BACK.value, datetime.utcnow(), merge_id)
            )

        # 7. Return result
        return RollbackResult(
            merge_id=merge_id,
            status=MergeStatus.ROLLED_BACK,
            restored_counterparties=merge_ids,
            restored_transactions=restored_txns,
            message=f"Rolled back merge. Restored {len(merge_ids)} counterparties, {restored_txns} transactions."
        )

    except Exception as e:
        raise RollbackError(f"Rollback failed: {str(e)}") from e
```

### State Snapshot (for Rollback)

```python
def _create_before_state_snapshot(
    self,
    primary_id: str,
    merge_ids: List[str]
) -> dict:
    """
    Create snapshot of current state before merge.

    Stores:
    - Transaction mappings (which transactions belong to which counterparty)
    - Alias mappings (which aliases belong to which counterparty)
    - Counterparty metadata

    Used for rollback.
    """
    # 1. Get transaction mappings
    transaction_mapping = {}
    for merge_id in merge_ids:
        txn_ids = [
            row['transaction_id'] for row in self.db.execute(
                "SELECT transaction_id FROM canonical_transactions WHERE counterparty_id = ?",
                (merge_id,)
            ).fetchall()
        ]
        transaction_mapping[merge_id] = txn_ids

    # 2. Get alias mappings
    aliases = []
    for merge_id in merge_ids:
        alias_rows = self.db.execute(
            "SELECT alias_id, alias, counterparty_id, is_canonical FROM counterparty_aliases WHERE counterparty_id = ?",
            (merge_id,)
        ).fetchall()
        aliases.extend([dict(row) for row in alias_rows])

    # 3. Return snapshot
    return {
        "transaction_mapping": transaction_mapping,
        "aliases": aliases,
        "timestamp": datetime.utcnow().isoformat()
    }
```

---

## Usage Examples

### Example 1: Preview Merge

```python
merger = AliasMerger(db_connection, counterparty_store)

# User wants to merge Amazon variants
preview = merger.preview_merge(
    user_id="user_123",
    primary_id="cp_amazon_1",
    merge_ids=["cp_amzn_2", "cp_amazon_prime_3"]
)

print(f"Primary: {preview.primary_name}")
print(f"Merging: {', '.join(preview.merge_names)}")
print(f"Will consolidate {preview.total_aliases} aliases")
print(f"Will migrate {preview.total_transactions} transactions")
print(f"Safe to proceed: {preview.is_safe}")
```

### Example 2: Execute Merge

```python
# User confirms merge
result = merger.merge(
    user_id="user_123",
    primary_id="cp_amazon_1",
    merge_ids=["cp_amzn_2", "cp_amazon_prime_3"]
)

print(result.message)
# "Merged 2 counterparties. Migrated 45 transactions, consolidated 5 aliases."

print(f"Status: {result.status}")  # MergeStatus.COMPLETED
print(f"Execution time: {result.execution_time_ms}ms")
```

### Example 3: Rollback Merge

```python
# User realizes merge was incorrect
rollback = merger.rollback_merge(
    user_id="user_123",
    merge_id="merge_1234567890"
)

print(rollback.message)
# "Rolled back merge. Restored 2 counterparties, 45 transactions."

print(f"Restored: {', '.join(rollback.restored_counterparties)}")
```

### Example 4: List Merge History

```python
# View merge history
merges = merger.list_merges("user_123", status=MergeStatus.COMPLETED, limit=10)

for m in merges:
    print(f"{m.merge_id}: {m.primary_name} ← {len(m.merged_ids)} merged ({m.migrated_transaction_count} txns)")

# Output:
# merge_1234567890: Amazon ← 2 merged (45 txns)
# merge_1234567891: Starbucks ← 3 merged (87 txns)
```

---

## Multi-Domain Examples

### Healthcare: Provider Merge

```python
# Merge duplicate medical providers
preview = merger.preview_merge(
    user_id="patient_123",
    primary_id="mp_smith_1",  # "Smith Family Practice"
    merge_ids=["mp_dr_smith_2", "mp_smith_clinic_3"]
)
# Will consolidate 6 aliases, migrate 34 medical claims
```

### Legal: Law Firm Merge

```python
# Merge duplicate law firms
result = merger.merge(
    user_id="lawyer_456",
    primary_id="lf_jones_1",  # "Jones & Associates"
    merge_ids=["lf_jones_law_2", "lf_jones_legal_3"]
)
# Merged 2 law firms. Migrated 23 billing records.
```

### Research: Institution Merge

```python
# Merge duplicate research institutions
result = merger.merge(
    user_id="researcher_789",
    primary_id="ri_mit_1",  # "MIT"
    merge_ids=["ri_mit_cambridge_2", "ri_mass_inst_tech_3"]
)
# Merged 2 institutions. Migrated 156 publications.
```

---

## Error Handling

### Custom Exceptions

```python
class AliasMergerError(Exception):
    """Base exception for AliasMerger errors."""
    pass

class MergeError(AliasMergerError):
    """Merge execution failed."""
    pass

class RollbackError(AliasMergerError):
    """Rollback failed."""
    pass

class MergeNotFoundError(AliasMergerError):
    """Merge record not found."""
    pass

class ValidationError(AliasMergerError):
    """Merge validation failed."""
    pass
```

---

## Performance Characteristics

### Query Performance

| Operation | Query Type | Expected Latency (p95) | Notes |
|-----------|------------|------------------------|-------|
| `preview_merge()` | SELECT | <50ms | Lightweight calculation |
| `merge()` | UPDATE (batch) | <200ms | Depends on transaction count |
| `rollback_merge()` | UPDATE (batch) | <150ms | Depends on transaction count |
| `list_merges()` | SELECT | <30ms | Paginated results |

### Transaction Safety

```python
# All merge operations use database transactions
with self.db.transaction():
    # Step 1: Migrate transactions
    # Step 2: Consolidate aliases
    # Step 3: Archive counterparties
    # If ANY step fails → automatic rollback
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Preview merge
def test_preview_merge_success()
def test_preview_merge_conflicts()
def test_preview_merge_unauthorized()

# Test: Execute merge
def test_merge_success()
def test_merge_migrate_transactions()
def test_merge_consolidate_aliases()
def test_merge_archive_merged()
def test_merge_idempotent()

# Test: Rollback
def test_rollback_merge_success()
def test_rollback_restore_transactions()
def test_rollback_restore_aliases()
def test_rollback_unarchive_counterparties()

# Test: Error handling
def test_merge_validation_error()
def test_merge_execution_error_triggers_rollback()
def test_rollback_non_existent_merge()

# Test: State snapshots
def test_before_state_snapshot_complete()
def test_rollback_uses_snapshot()
```

---

## Related Primitives

- **CounterpartyStore** (OL) - Provides merge() method that uses AliasMerger internally
- **CounterpartyMatcher** (OL) - Identifies potential duplicates for merge
- **MergeConfirmation** (IL) - UI component that shows merge preview to user

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 3-4 days
**Dependencies:** Database (PostgreSQL/MySQL with transaction support), CounterpartyStore
