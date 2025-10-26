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

## Summary

ParserVersionManager provides **version lifecycle management** with deprecation tracking, breaking change documentation, and compatibility matrices for smooth version transitions.
