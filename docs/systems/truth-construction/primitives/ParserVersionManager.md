# Primitive: ParserVersionManager (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Version Control
> **Vertical:** 3.7 Parser Registry
> **Last Updated:** 2025-10-24

---

## Overview

**ParserVersionManager** manages parser versions with semantic versioning, deprecation tracking, and compatibility matrices. It enables smooth version transitions with breaking change documentation and migration guides.

**Key Responsibilities:**
- Track all versions for each parser (version history)
- Mark versions as deprecated with sunset dates
- Document breaking changes between versions
- Provide migration guide URLs for upgrades
- Track schema compatibility (which schema versions each parser version supports)
- Prevent usage of sunset versions

**Multi-Domain Examples:**
- Finance: BofA parser v1.5.0 → v2.0.0 → v2.1.0 (breaking: merchant → counterparty)
- Healthcare: HL7 v2.3 → v2.5 → v2.8 parser (breaking: field position changes)
- Legal: Court filing parser v1.0 → v2.0 (breaking: jurisdiction codes updated)
- Research (RSRCH - Utilitario): WebPageParser v1.0 → v2.0 (breaking: entity extraction now requires subject_entity field)

---

## API Reference

### `create_version()`
```python
def create_version(
    self,
    parser_id: str,
    version: str,
    breaking_changes: Optional[List[str]] = None
) -> ParserVersion:
    """
    Creates new version record for parser.

    Args:
        parser_id: Unique parser identifier
        version: Semantic version (major.minor.patch)
        breaking_changes: Optional list of breaking changes from previous version

    Returns:
        ParserVersion record

    Raises:
        InvalidVersionError: If version format invalid or already exists
        ParserNotFoundError: If parser_id doesn't exist

    Example:
        version = manager.create_version(
            parser_id="parser_bofa_pdf",
            version="2.0.0",
            breaking_changes=[
                "Merchant field renamed to counterparty",
                "Balance field now required"
            ]
        )
    """
```

### `mark_deprecated()`
```python
def mark_deprecated(
    self,
    parser_id: str,
    version: str,
    sunset_date: date,
    migration_guide_url: Optional[str] = None
) -> None:
    """
    Marks version as deprecated with sunset date.

    Args:
        parser_id: Unique parser identifier
        version: Version to deprecate
        sunset_date: Date after which version removed (must be future, ≥30 days)
        migration_guide_url: Optional URL to migration documentation

    Raises:
        VersionNotFoundError: If version doesn't exist
        InvalidSunsetDateError: If sunset_date < 30 days in future

    Example:
        manager.mark_deprecated(
            parser_id="parser_bofa_pdf",
            version="1.5.0",
            sunset_date=date(2024, 12, 31),
            migration_guide_url="https://docs.example.com/parsers/bofa/v2-migration"
        )
    """
```

### `get_latest_version()`
```python
def get_latest_version(self, parser_id: str) -> Optional[str]:
    """
    Returns latest non-deprecated version.

    Example:
        # Versions: 1.5.0 (deprecated), 2.0.0, 2.1.0
        latest = manager.get_latest_version("parser_bofa_pdf")
        # Returns: "2.1.0"
    """
```

### `get_compatibility_matrix()`
```python
def get_compatibility_matrix(self, parser_id: str) -> Dict[str, List[str]]:
    """
    Returns which schema versions each parser version is compatible with.

    Returns:
        Dict mapping parser version to list of compatible schema versions

    Example:
        matrix = manager.get_compatibility_matrix("parser_bofa_pdf")
        # Returns:
        # {
        #   "1.5.0": ["observation-transaction-v1"],
        #   "2.0.0": ["observation-transaction-v1", "observation-transaction-v2"],
        #   "2.1.0": ["observation-transaction-v2"]
        # }
    """
```

---

## Data Model

### ParserVersion

```python
@dataclass
class ParserVersion:
    parser_id: str
    version: str
    released_at: datetime
    deprecated: bool
    sunset_date: Optional[date]
    breaking_changes: List[str]
    migration_guide_url: Optional[str]
    compatible_with: List[str]  # Schema versions
```

**Example:**
```json
{
  "parser_id": "parser_bofa_pdf",
  "version": "2.1.0",
  "released_at": "2024-11-01T10:00:00Z",
  "deprecated": false,
  "sunset_date": null,
  "breaking_changes": [],
  "migration_guide_url": null,
  "compatible_with": ["observation-transaction-v2"]
}
```

---

## Multi-Domain Examples

### Healthcare: HL7 Version Management

```python
# HL7 v2.3 → v2.5 → v2.8
manager.create_version(
    parser_id="parser_hl7_adt",
    version="2.3.0",
    breaking_changes=[]
)

manager.create_version(
    parser_id="parser_hl7_adt",
    version="2.5.0",
    breaking_changes=[
        "PID segment field positions changed",
        "New required field: PID-39 (Citizenship)"
    ]
)

manager.mark_deprecated(
    parser_id="parser_hl7_adt",
    version="2.3.0",
    sunset_date=date(2025, 6, 30),
    migration_guide_url="https://docs.example.com/hl7/v2.5-migration"
)
```

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Manage BofA PDF parser versions through breaking changes
**Example:** ParserVersionManager tracks BofA parser history → v1.5.0 (extracts "merchant" field) → v2.0.0 released (breaking: "merchant" renamed to "counterparty", "balance" now required) → create_version("parser_bofa_pdf", "2.0.0", breaking_changes=["Merchant → counterparty"]) → mark_deprecated(v1.5.0, sunset_date=2024-12-31, migration_guide_url) → Users have 30+ days to migrate → get_latest_version() returns "2.0.0" → Uploads using v1.5.0 after sunset fail with deprecation error
**Operations:** create_version (track new releases), mark_deprecated (set sunset dates), get_latest_version (non-deprecated), get_compatibility_matrix (schema versions supported)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Manage HL7 parser versions with field position changes
**Example:** HL7 parser v2.3.0 → v2.5.0 (breaking: PID segment field 39 added "Citizenship") → create_version with breaking_changes → mark_deprecated v2.3.0 (sunset 6 months) → EHR systems migrate before sunset → compatibility_matrix shows {v2.3.0: ["hl7-v2.3-schema"], v2.5.0: ["hl7-v2.5-schema"]}
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Manage court filing parser versions with jurisdiction code updates
**Example:** Court parser v1.0 → v2.0 (breaking: jurisdiction codes updated CA-001 → CA-SUPERIOR-001) → create_version with breaking_changes → mark_deprecated v1.0 (90-day sunset) → Law firms update integrations
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Manage web page parser versions with entity extraction schema changes
**Example:** TechCrunchHTMLParser v1.0 (extracts facts without subject_entity field) → v2.0 (breaking: now requires subject_entity for all facts, enables entity-centric queries) → create_version("techcrunch_html_parser", "2.0.0", breaking_changes=["subject_entity field now required for all fact extractions", "Fact type taxonomy updated"]) → mark_deprecated(v1.0, sunset_date 60 days, migration_guide explains new entity model) → Scraper jobs updated to v2.0 before sunset → compatibility_matrix: {v1.0: ["raw-fact-v1"], v2.0: ["raw-fact-v2", "canonical-fact-v1"]}
**Operations:** Version tracking critical for reproducibility (fact extracted with parser v1.5 vs v2.0 may differ), deprecation gives time for scraper job updates, compatibility matrix shows which schema versions each parser supports
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Manage supplier catalog parser versions with SKU format changes
**Example:** SupplierCSVParser v1.0 → v2.0 (breaking: SKU format changed from numeric to alphanumeric) → create_version with breaking_changes → mark_deprecated v1.0 → Merchants update catalog imports
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (semantic versioning, deprecation tracking, compatibility matrices are universal patterns)
**Reusability:** High (same create_version/mark_deprecated/get_latest_version operations work for bank parsers, HL7 parsers, court parsers, web scrapers, catalog parsers; only breaking changes and schema versions differ)

---

## Simplicity Profiles

The same ParserVersionManager interface supports three implementation scales: Personal (no version management), Small Business (YAML version history), Enterprise (database with deprecation + sunset tracking).

### Personal Profile (0 LOC - Not Needed)

**Darwin's Approach:** No parser version management

**Why Darwin doesn't need this:**
- Darwin has exactly 1 parser (BofAPDFParser)
- Parser version never changes (stable BoFA PDF format)
- If BoFA changes PDF format → Darwin updates parser code directly
- No versioning, no deprecation, no compatibility matrix

**What Darwin DOESN'T need:**
- ❌ Version tracking (parser doesn't change)
- ❌ Deprecation dates
- ❌ Breaking change documentation
- ❌ Migration guides
- ❌ Compatibility matrix

**Implementation:**

None! Darwin skips this primitive entirely.

**If Darwin's parser DOES change:**
```python
# Simple approach: Just update the parser code
# Old:
def parse_merchant(line):
    return extract_text(line, pattern="MERCHANT: (.+)")

# New:
def parse_merchant(line):
    # BoFA changed format on 2024-11-01
    return extract_text(line, pattern="Merchant Name: (.+)")
```

**Decision Context:**
- Darwin's PDF format has been stable for 3 years
- If format changes → Darwin updates code once
- No need to support multiple parser versions simultaneously
- YAGNI: Skip ParserVersionManager when you have 1 parser

---

### Small Business Profile (80 LOC)

**Use Case:** Accounting firm with 8 bank parsers, occasional breaking changes

**What they need (vs Personal):**
- ✅ **Version history** (track which parser version was used for each upload)
- ✅ **Breaking change documentation** (notes on what changed between versions)
- ✅ **Simple YAML file** (version metadata without database)
- ❌ No deprecation/sunset dates (manual migration is fine)
- ❌ No compatibility matrix (small scale, changes are rare)

**Implementation:**

**Config File (config/parser-versions.yaml):**
```yaml
parsers:
  bofa_pdf:
    versions:
      - version: "1.0.0"
        released: "2023-01-15"
        description: "Initial BoFA PDF parser"
        breaking_changes: []

      - version: "1.5.0"
        released: "2023-08-01"
        description: "Added merchant normalization"
        breaking_changes: []

      - version: "2.0.0"
        released: "2024-11-01"
        description: "Merchant → counterparty rename"
        breaking_changes:
          - "Renamed 'merchant' field to 'counterparty'"
          - "Balance field now required"
        notes: "Update consumers to use 'counterparty' instead of 'merchant'"

  chase_csv:
    versions:
      - version: "1.0.0"
        released: "2023-02-01"
        description: "Initial Chase CSV parser"
        breaking_changes: []

      - version: "1.1.0"
        released: "2024-05-15"
        description: "Added transaction type field"
        breaking_changes: []
```

**Code Example (80 LOC):**
```python
# parser_version_manager.py (Small Business)

import yaml
from pathlib import Path
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional, Dict

@dataclass
class ParserVersion:
    version: str
    released: str
    description: str
    breaking_changes: List[str]
    notes: Optional[str] = None

class ParserVersionManager:
    """Simple YAML-based parser version tracking."""

    def __init__(self, config_path: str = "config/parser-versions.yaml"):
        self.config_path = Path(config_path)
        self._versions: Dict[str, List[ParserVersion]] = {}
        self._load()

    def _load(self):
        """Load version history from YAML."""
        config = yaml.safe_load(self.config_path.read_text())

        for parser_id, parser_data in config.get("parsers", {}).items():
            versions = []
            for v in parser_data.get("versions", []):
                versions.append(ParserVersion(
                    version=v["version"],
                    released=v["released"],
                    description=v.get("description", ""),
                    breaking_changes=v.get("breaking_changes", []),
                    notes=v.get("notes")
                ))
            self._versions[parser_id] = versions

    def get_latest_version(self, parser_id: str) -> str:
        """Get latest version number."""
        versions = self._versions.get(parser_id, [])
        if not versions:
            return "1.0.0"  # Default

        # Return last version (assumes YAML is in chronological order)
        return versions[-1].version

    def get_version_info(self, parser_id: str, version: str) -> Optional[ParserVersion]:
        """Get metadata for specific version."""
        versions = self._versions.get(parser_id, [])
        for v in versions:
            if v.version == version:
                return v
        return None

    def list_versions(self, parser_id: str) -> List[ParserVersion]:
        """List all versions for parser."""
        return self._versions.get(parser_id, [])

    def get_breaking_changes(self, parser_id: str, from_version: str, to_version: str) -> List[str]:
        """Get all breaking changes between two versions."""
        versions = self._versions.get(parser_id, [])

        # Find version range
        from_idx = None
        to_idx = None
        for i, v in enumerate(versions):
            if v.version == from_version:
                from_idx = i
            if v.version == to_version:
                to_idx = i

        if from_idx is None or to_idx is None:
            return []

        # Collect all breaking changes in range
        all_changes = []
        for v in versions[from_idx + 1 : to_idx + 1]:
            all_changes.extend(v.breaking_changes)

        return all_changes
```

**Usage:**
```python
# Load version manager
vm = ParserVersionManager("config/parser-versions.yaml")

# Get latest version
latest = vm.get_latest_version("bofa_pdf")
# Returns: "2.0.0"

# Check version info
info = vm.get_version_info("bofa_pdf", "2.0.0")
print(f"Released: {info.released}")
print(f"Changes: {info.breaking_changes}")

# Output:
# Released: 2024-11-01
# Changes: ['Renamed merchant to counterparty', 'Balance field now required']

# Migration check
changes = vm.get_breaking_changes("bofa_pdf", "1.5.0", "2.0.0")
if changes:
    print("⚠️ Migration required:")
    for change in changes:
        print(f"  - {change}")
```

**Adding new version (manual YAML edit):**
```yaml
# config/parser-versions.yaml
parsers:
  bofa_pdf:
    versions:
      # ... existing versions ...

      - version: "2.1.0"
        released: "2024-12-01"
        description: "Added transaction type field"
        breaking_changes: []
        notes: "New field: transaction_type (debit/credit)"
```

**Testing:**
- ✅ Unit test: Load YAML, verify versions parsed correctly
- ✅ Unit test: get_breaking_changes returns correct range
- Manual verification: Check version history matches reality

**Monitoring:**
- Startup log: "Loaded version history for 8 parsers"

**Decision Context:**
- Accounting firm releases parser updates 2-3 times/year
- YAML file sufficient for version history (no database needed)
- Breaking changes documented for manual migration planning
- No automated deprecation (admin manually notifies users)

---

### Enterprise Profile (650 LOC)

**Use Case:** Multi-tenant platform with automated deprecation, sunset enforcement, migration guides

**What they need (vs Small Business):**
- ✅ All Small Business features (version history, breaking changes)
- ✅ **Database-backed storage** (PostgreSQL with version lifecycle tracking)
- ✅ **Deprecation management** (mark versions deprecated with sunset dates)
- ✅ **Sunset enforcement** (prevent uploads using sunset versions)
- ✅ **Migration guide URLs** (link to upgrade documentation)
- ✅ **Compatibility matrix** (track which schema versions each parser supports)
- ✅ **Automated notifications** (email tenants 30/14/7 days before sunset)

**Implementation:**

**Database Schema:**
```sql
CREATE TABLE parser_versions (
    id SERIAL PRIMARY KEY,
    parser_id TEXT NOT NULL,
    version TEXT NOT NULL,
    released_at TIMESTAMPTZ NOT NULL,
    deprecated BOOLEAN DEFAULT FALSE,
    sunset_date DATE,
    breaking_changes TEXT[],
    migration_guide_url TEXT,
    compatible_schemas TEXT[],  -- e.g., ["observation-transaction-v1", "observation-transaction-v2"]
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(parser_id, version)
);

CREATE INDEX idx_parser_versions_lookup ON parser_versions(parser_id, version);
CREATE INDEX idx_parser_versions_sunset ON parser_versions(sunset_date) WHERE deprecated = TRUE;

CREATE TABLE version_sunset_notifications (
    parser_id TEXT NOT NULL,
    version TEXT NOT NULL,
    tenant_id TEXT NOT NULL,
    notification_type TEXT NOT NULL,  -- '30_day', '14_day', '7_day'
    sent_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (parser_id, version, tenant_id, notification_type)
);
```

**Code Example (650 LOC - excerpt):**
```python
# parser_version_manager.py (Enterprise)

import psycopg2
from psycopg2.extras import DictCursor
from datetime import datetime, date, timedelta
from typing import List, Optional, Dict
from dataclasses import dataclass

@dataclass
class ParserVersion:
    parser_id: str
    version: str
    released_at: datetime
    deprecated: bool
    sunset_date: Optional[date]
    breaking_changes: List[str]
    migration_guide_url: Optional[str]
    compatible_schemas: List[str]

class InvalidVersionError(Exception):
    pass

class VersionSunsetError(Exception):
    pass

class ParserVersionManager:
    """Enterprise version management with deprecation + sunset tracking."""

    def __init__(self, db_conn_string: str):
        self.db_conn_string = db_conn_string
        self._init_db()

    def _init_db(self):
        """Create tables if not exist."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()
        # ... (schema creation from above)
        conn.commit()
        conn.close()

    def create_version(
        self,
        parser_id: str,
        version: str,
        breaking_changes: Optional[List[str]] = None,
        compatible_schemas: Optional[List[str]] = None
    ) -> ParserVersion:
        """Register a new parser version."""
        # Validate semantic version format
        if not self._is_valid_semver(version):
            raise InvalidVersionError(f"Invalid semantic version: {version}")

        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO parser_versions
            (parser_id, version, released_at, breaking_changes, compatible_schemas)
            VALUES (%s, %s, %s, %s, %s)
        """, (
            parser_id,
            version,
            datetime.now(),
            breaking_changes or [],
            compatible_schemas or []
        ))

        conn.commit()
        conn.close()

        return ParserVersion(
            parser_id=parser_id,
            version=version,
            released_at=datetime.now(),
            deprecated=False,
            sunset_date=None,
            breaking_changes=breaking_changes or [],
            migration_guide_url=None,
            compatible_schemas=compatible_schemas or []
        )

    def mark_deprecated(
        self,
        parser_id: str,
        version: str,
        sunset_date: date,
        migration_guide_url: Optional[str] = None
    ):
        """Mark version as deprecated with sunset date."""
        # Enforce 30-day minimum notice
        if sunset_date < date.today() + timedelta(days=30):
            raise InvalidVersionError("Sunset date must be at least 30 days in future")

        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        cursor.execute("""
            UPDATE parser_versions
            SET deprecated = TRUE,
                sunset_date = %s,
                migration_guide_url = %s
            WHERE parser_id = %s AND version = %s
        """, (sunset_date, migration_guide_url, parser_id, version))

        if cursor.rowcount == 0:
            conn.close()
            raise InvalidVersionError(f"Version not found: {parser_id}:{version}")

        conn.commit()
        conn.close()

        # Trigger notification workflow
        self._schedule_deprecation_notifications(parser_id, version, sunset_date)

    def get_latest_version(self, parser_id: str) -> Optional[str]:
        """Get latest non-deprecated version."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        cursor.execute("""
            SELECT version FROM parser_versions
            WHERE parser_id = %s AND deprecated = FALSE
            ORDER BY released_at DESC
            LIMIT 1
        """, (parser_id,))

        row = cursor.fetchone()
        conn.close()

        return row["version"] if row else None

    def validate_version_usable(self, parser_id: str, version: str):
        """
        Check if version is usable (not sunset).

        Raises:
            VersionSunsetError: If version is sunset
        """
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        cursor.execute("""
            SELECT deprecated, sunset_date, migration_guide_url
            FROM parser_versions
            WHERE parser_id = %s AND version = %s
        """, (parser_id, version))

        row = cursor.fetchone()
        conn.close()

        if not row:
            raise InvalidVersionError(f"Version not found: {parser_id}:{version}")

        if row["deprecated"] and row["sunset_date"] and row["sunset_date"] <= date.today():
            raise VersionSunsetError(
                f"Parser {parser_id} version {version} is sunset as of {row['sunset_date']}. "
                f"Please upgrade to latest version. "
                f"Migration guide: {row['migration_guide_url']}"
            )

    def get_compatibility_matrix(self, parser_id: str) -> Dict[str, List[str]]:
        """Get schema compatibility for all parser versions."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        cursor.execute("""
            SELECT version, compatible_schemas
            FROM parser_versions
            WHERE parser_id = %s
            ORDER BY released_at DESC
        """, (parser_id,))

        rows = cursor.fetchall()
        conn.close()

        return {
            row["version"]: row["compatible_schemas"]
            for row in rows
        }

    def get_breaking_changes_since(
        self,
        parser_id: str,
        from_version: str
    ) -> List[Dict]:
        """Get all breaking changes since a specific version."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        cursor.execute("""
            SELECT version, released_at, breaking_changes
            FROM parser_versions
            WHERE parser_id = %s
                AND released_at > (
                    SELECT released_at FROM parser_versions
                    WHERE parser_id = %s AND version = %s
                )
            ORDER BY released_at ASC
        """, (parser_id, parser_id, from_version))

        rows = cursor.fetchall()
        conn.close()

        changes = []
        for row in rows:
            if row["breaking_changes"]:
                changes.append({
                    "version": row["version"],
                    "released_at": row["released_at"].isoformat(),
                    "breaking_changes": row["breaking_changes"]
                })

        return changes

    def _schedule_deprecation_notifications(
        self,
        parser_id: str,
        version: str,
        sunset_date: date
    ):
        """Schedule notifications at 30/14/7 days before sunset."""
        # This would integrate with a job scheduler (Celery, Airflow, etc.)
        # For now, just log
        print(f"Scheduled sunset notifications for {parser_id}:{version} (sunset: {sunset_date})")

    def send_pending_notifications(self):
        """
        Send deprecation notifications for upcoming sunsets.

        Called daily by cron job.
        """
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        # Find versions with upcoming sunsets (30, 14, 7 days)
        for days_before in [30, 14, 7]:
            target_date = date.today() + timedelta(days=days_before)

            cursor.execute("""
                SELECT parser_id, version, sunset_date, migration_guide_url
                FROM parser_versions
                WHERE deprecated = TRUE
                    AND sunset_date = %s
            """, (target_date,))

            for row in cursor.fetchall():
                self._send_notification(
                    parser_id=row["parser_id"],
                    version=row["version"],
                    sunset_date=row["sunset_date"],
                    migration_guide_url=row["migration_guide_url"],
                    days_before=days_before
                )

        conn.close()

    def _send_notification(
        self,
        parser_id: str,
        version: str,
        sunset_date: date,
        migration_guide_url: Optional[str],
        days_before: int
    ):
        """Send email notification to all tenants using this parser version."""
        # Query for tenants using this parser
        # Send email via SMTP/SendGrid
        # Record notification in version_sunset_notifications table
        print(f"📧 Sending {days_before}-day sunset notice for {parser_id}:{version}")

    def _is_valid_semver(self, version: str) -> bool:
        """Validate semantic version format (major.minor.patch)."""
        import re
        pattern = r'^\d+\.\d+\.\d+$'
        return bool(re.match(pattern, version))
```

**Usage Example (Version Lifecycle):**
```python
vm = ParserVersionManager("postgresql://localhost/parsers")

# Step 1: Create new version
vm.create_version(
    parser_id="bofa_pdf",
    version="2.0.0",
    breaking_changes=[
        "Renamed 'merchant' field to 'counterparty'",
        "Balance field now required"
    ],
    compatible_schemas=["observation-transaction-v2"]
)

# Step 2: Deprecate old version (30-day notice)
vm.mark_deprecated(
    parser_id="bofa_pdf",
    version="1.5.0",
    sunset_date=date(2024, 12, 31),
    migration_guide_url="https://docs.example.com/parsers/bofa/v2-migration"
)

# Step 3: Upload validation (prevents sunset version usage)
try:
    vm.validate_version_usable("bofa_pdf", "1.5.0")
except VersionSunsetError as e:
    print(f"❌ {e}")
    # Reject upload
```

**Automated Notification Schedule:**
```python
# Cron job runs daily
def daily_notification_job():
    vm = ParserVersionManager("postgresql://localhost/parsers")
    vm.send_pending_notifications()

# Sends emails:
# - 30 days before sunset: "Parser version will be sunset in 30 days"
# - 14 days before sunset: "Parser version will be sunset in 14 days"
# - 7 days before sunset: "URGENT: Parser version will be sunset in 7 days"
```

**Testing:**
- ✅ Unit tests for all version operations
- ✅ Integration tests with test database
- ✅ Sunset enforcement validation
- ✅ Notification scheduling logic

**Monitoring:**
- Grafana dashboard: Deprecated versions still in use
- Alert if tenant uploads using sunset version (should be blocked)
- Daily report: Upcoming sunsets

**Decision Context:**
- Platform supports 50+ parsers, each with 3-5 versions
- Breaking changes happen monthly
- Automated deprecation reduces manual coordination
- 30-day notice gives tenants time to upgrade
- Sunset enforcement prevents production errors from old parsers

---

## Summary

ParserVersionManager provides **version lifecycle management** with deprecation tracking, breaking change documentation, and compatibility matrices for smooth version transitions.
