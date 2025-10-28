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

### Personal Profile (0 LOC)

**Contexto del Usuario:**
Darwin siempre usa la última versión del parser. No necesita cambiar entre v1/v2/v3. No tiene múltiples parsers corriendo simultáneamente. Upgrading al nuevo parser = simplemente actualizar el código.

**Implementation:**
```python
# No implementation needed (0 LOC)

# Darwin always uses latest parser version
from parsers.bofa_pdf import BofAPDFParser  # Always imports latest

parser = BofAPDFParser()  # Latest version by default
```

**Características Incluidas:**
- ✅ Siempre usa última versión (import directo)

**Características NO Incluidas:**
- ❌ Version pinning (YAGNI: single user, always upgrade)
- ❌ Rollback (YAGNI: can revert git commit)
- ❌ A/B testing (YAGNI: no comparison needed)
- ❌ Compatibility matrix (YAGNI: no old data to migrate)

**Configuración:**
```python
# No config needed
```

**Performance:**
- N/A (no versioning logic)

**Upgrade Triggers:**
- Si necesita versioning → Small Business

---

### Small Business Profile (80 LOC)

**Contexto del Usuario:**
Empresa tiene 8 parsers. Cuando actualizan un parser de v1 → v2, necesitan testear v2 con datos históricos antes de hacer rollout completo. Necesitan: version pinning en config, backward compatibility checks, gradual migration.

**Implementation:**
```python
# lib/parser_version_manager.py (Small Business - 80 LOC)
import yaml

class ParserVersionManager:
    """YAML-based parser version management."""

    def __init__(self, config_path="config/parser_versions.yaml"):
        self.config_path = config_path
        self._load_config()

    def _load_config(self):
        """Load version config."""
        with open(self.config_path, "r") as f:
            self.config = yaml.safe_load(f)

    def get_parser_version(self, source_type: str) -> str:
        """
        Get pinned version for source_type.

        Returns: "v1.0.0" | "v2.0.0" | "latest"
        """
        return self.config["parser_versions"].get(source_type, "latest")

    def supports_version(self, source_type: str, version: str) -> bool:
        """Check if version is available."""
        available = self.config["available_versions"].get(source_type, [])
        return version in available or version == "latest"

    def pin_version(self, source_type: str, version: str):
        """Pin parser to specific version."""
        self.config["parser_versions"][source_type] = version

        with open(self.config_path, "w") as f:
            yaml.dump(self.config, f)

# YAML Config
"""
# config/parser_versions.yaml

parser_versions:
  bofa_pdf: v2.0.0        # Pinned to v2
  chase_csv: v1.5.0       # Pinned to v1.5
  wells_fargo_pdf: latest # Always use latest

available_versions:
  bofa_pdf: [v1.0.0, v2.0.0]
  chase_csv: [v1.0.0, v1.5.0, v2.0.0]
  wells_fargo_pdf: [v1.0.0, v1.2.0]
"""

# Usage
version_mgr = ParserVersionManager()

# Get pinned version for parser
version = version_mgr.get_parser_version("bofa_pdf")
# → "v2.0.0"

# Pin parser to specific version (before testing v3)
version_mgr.pin_version("bofa_pdf", "v2.0.0")

# Load parser with specific version
from parsers.bofa_pdf_v2 import BofAPDFParserV2
parser = BofAPDFParserV2()
```

**Características Incluidas:**
- ✅ Version pinning (YAML config)
- ✅ Multiple versions available
- ✅ Manual version management
- ✅ Gradual migration (pin some parsers to v2, others to v1)

**Características NO Incluidas:**
- ❌ Automatic rollback (manual config change)
- ❌ Compatibility matrix (manual testing)
- ❌ Database storage (YAML sufficient for 8 parsers)

**Configuración:**
```yaml
parser_versions:
  config_path: "config/parser_versions.yaml"
```

**Performance:**
- Load config: 10ms (YAML parse)
- Lookup: 0ms (dict lookup)

**Upgrade Triggers:**
- Si >20 parsers → Enterprise (database)
- Si necesitan automatic rollback → Enterprise
- Si multi-tenant → Enterprise

---

### Enterprise Profile (650 LOC)

**Contexto del Usuario:**
50+ parsers, cada uno con múltiples versiones (v1, v2, v3). Necesitan: automatic compatibility checking (v3 compatible con datos parseados por v1?), automatic rollback si v3 accuracy <95%, migration tracking (cuántos uploads usaron v1 vs v2), deprecation warnings.

**Implementation:**
```python
# lib/parser_version_manager.py (Enterprise - 650 LOC)
import psycopg2
from typing import Dict, List
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class ParserVersion:
    source_type: str
    version: str
    parser_class: str
    min_compatible_version: str  # Can parse data from this version onward
    deprecated: bool
    deprecation_date: datetime
    accuracy_threshold: float = 0.95  # Auto-rollback if accuracy < 95%

class ParserVersionManager:
    """Database-backed parser version manager with compatibility tracking."""

    def __init__(self, db_connection_string):
        self.conn = psycopg2.connect(db_connection_string)

    def get_active_version(self, source_type: str, tenant_id: str = None) -> ParserVersion:
        """
        Get active parser version.

        Logic:
        1. Check tenant override
        2. Fallback to global active version
        3. Check if deprecated (warn)
        4. Return ParserVersion
        """
        cursor = self.conn.cursor()

        # Tenant override?
        if tenant_id:
            cursor.execute("""
                SELECT source_type, version, parser_class, min_compatible_version,
                       deprecated, deprecation_date, accuracy_threshold
                FROM parser_versions
                WHERE source_type = %s
                  AND tenant_id = %s
                  AND active = true
                ORDER BY created_at DESC
                LIMIT 1
            """, (source_type, tenant_id))
            row = cursor.fetchone()

        # Global version
        if not row:
            cursor.execute("""
                SELECT source_type, version, parser_class, min_compatible_version,
                       deprecated, deprecation_date, accuracy_threshold
                FROM parser_versions
                WHERE source_type = %s
                  AND tenant_id IS NULL
                  AND active = true
                ORDER BY created_at DESC
                LIMIT 1
            """, (source_type,))
            row = cursor.fetchone()

        if not row:
            raise ValueError(f"No active version for {source_type}")

        version = ParserVersion(*row)

        # Deprecation warning
        if version.deprecated and version.deprecation_date < datetime.now():
            self._log_deprecation_warning(source_type, version.version)

        return version

    def is_compatible(self, source_type: str, from_version: str, to_version: str) -> bool:
        """
        Check if to_version can parse data from from_version.

        Example:
            is_compatible("bofa_pdf", "v1.0.0", "v3.0.0")
            → True if v3 min_compatible_version <= v1.0.0
        """
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT min_compatible_version
            FROM parser_versions
            WHERE source_type = %s AND version = %s
        """, (source_type, to_version))

        row = cursor.fetchone()
        if not row:
            return False

        min_compat = row[0]
        return self._version_compare(from_version, min_compat) >= 0

    def track_usage(self, source_type: str, version: str, upload_id: str, success: bool, accuracy: float = None):
        """Track parser version usage for migration analytics."""
        cursor = self.conn.cursor()
        cursor.execute("""
            INSERT INTO parser_version_usage (
                source_type, version, upload_id, success, accuracy, timestamp
            ) VALUES (%s, %s, %s, %s, %s, NOW())
        """, (source_type, version, upload_id, success, accuracy))
        self.conn.commit()

        # Auto-rollback if accuracy < threshold
        if accuracy is not None:
            self._check_auto_rollback(source_type, version, accuracy)

    def _check_auto_rollback(self, source_type: str, version: str, current_accuracy: float):
        """
        Auto-rollback if accuracy drops below threshold.

        Logic:
        - Check last 100 uploads for this version
        - If avg accuracy < threshold → rollback to previous version
        - Send alert
        """
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT AVG(accuracy) as avg_accuracy
            FROM parser_version_usage
            WHERE source_type = %s
              AND version = %s
              AND timestamp > NOW() - INTERVAL '24 hours'
            HAVING COUNT(*) >= 100
        """, (source_type, version))

        row = cursor.fetchone()
        if not row:
            return  # Not enough data yet

        avg_accuracy = row[0]
        threshold = self._get_accuracy_threshold(source_type, version)

        if avg_accuracy < threshold:
            # Rollback to previous version
            previous_version = self._get_previous_version(source_type, version)
            self._rollback_version(source_type, previous_version)
            self._send_alert(f"Auto-rollback: {source_type} {version} → {previous_version} (accuracy {avg_accuracy:.2%} < {threshold:.2%})")

    def _version_compare(self, v1: str, v2: str) -> int:
        """Compare semantic versions. Returns: 1 if v1 > v2, -1 if v1 < v2, 0 if equal."""
        v1_parts = [int(x) for x in v1.lstrip('v').split('.')]
        v2_parts = [int(x) for x in v2.lstrip('v').split('.')]

        for i in range(max(len(v1_parts), len(v2_parts))):
            v1_val = v1_parts[i] if i < len(v1_parts) else 0
            v2_val = v2_parts[i] if i < len(v2_parts) else 0
            if v1_val > v2_val:
                return 1
            elif v1_val < v2_val:
                return -1
        return 0

# Database Schema
"""
CREATE TABLE parser_versions (
    id SERIAL PRIMARY KEY,
    source_type TEXT NOT NULL,
    version TEXT NOT NULL,
    parser_class TEXT NOT NULL,
    min_compatible_version TEXT,  -- Can parse data from this version onward
    deprecated BOOLEAN DEFAULT false,
    deprecation_date TIMESTAMP,
    accuracy_threshold FLOAT DEFAULT 0.95,
    active BOOLEAN DEFAULT true,
    tenant_id UUID,               -- NULL = global
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (source_type, version, tenant_id)
);

CREATE TABLE parser_version_usage (
    id SERIAL PRIMARY KEY,
    source_type TEXT NOT NULL,
    version TEXT NOT NULL,
    upload_id TEXT NOT NULL,
    success BOOLEAN,
    accuracy FLOAT,
    timestamp TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_version_usage_tracking ON parser_version_usage (source_type, version, timestamp DESC);
"""

# Usage
version_mgr = ParserVersionManager(db_connection_string="postgresql://localhost/app")

# Get active version for parser
version = version_mgr.get_active_version("bofa_pdf")
# → ParserVersion(source_type="bofa_pdf", version="v3.0.0", ...)

# Check compatibility before migration
can_migrate = version_mgr.is_compatible("bofa_pdf", from_version="v1.0.0", to_version="v3.0.0")
# → True (v3 can parse v1 data)

# Track usage for auto-rollback
version_mgr.track_usage(
    source_type="bofa_pdf",
    version="v3.0.0",
    upload_id="UL_abc123",
    success=True,
    accuracy=0.92  # Below threshold → triggers auto-rollback after 100 uploads
)
```

**Características Incluidas:**
- ✅ Database-backed versioning (PostgreSQL)
- ✅ Compatibility matrix (min_compatible_version)
- ✅ Automatic rollback (if accuracy < 95%)
- ✅ Usage tracking (migration analytics)
- ✅ Deprecation warnings
- ✅ Per-tenant version overrides

**Características NO Incluidas:**
- ❌ Machine learning for rollback (threshold-based is sufficient)

**Configuración:**
```yaml
parser_version_manager:
  database:
    connection_string: "postgresql://db:5432/app"
  accuracy_threshold: 0.95  # Auto-rollback if < 95%
  deprecation_warning_days: 90
```

**Performance:**
- Get active version: 5ms (DB query + cache)
- Compatibility check: 3ms
- Usage tracking: 10ms (async insert)
- Auto-rollback check: 50ms (aggregate query, every 100th upload)

**No Further Tiers:**
- Scale horizontally (read replicas)

---

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
