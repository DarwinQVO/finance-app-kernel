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

## Summary

ParserVersionManager provides **version lifecycle management** with deprecation tracking, breaking change documentation, and compatibility matrices for smooth version transitions.
