# Primitive: ParserSelector (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Service Discovery / Auto-Selection
> **Vertical:** 3.7 Parser Registry
> **Last Updated:** 2025-10-24

---

## Overview

**ParserSelector** implements the auto-selection logic for choosing the best parser based on file metadata (filename, file type, required capabilities) with confidence scoring. It ranks parsers and provides alternatives for user override.

**Key Responsibilities:**
- Auto-select best parser for uploaded file
- Rank parsers by confidence score (0.0-1.0)
- Apply selection criteria (filename pattern, file type, capabilities, user preferences)
- Provide ranked alternatives if confidence < threshold
- Log selection decisions with reasoning

**Selection Algorithm:**
1. Filename pattern match (weight: 0.6)
2. File type match (weight: 0.3)
3. Required capabilities match (weight: 0.1)
4. User preference boost (+0.1)

**Multi-Domain Examples:**
- Finance: "BofA_Statement_Nov2024.pdf" → BofA PDF parser v2.1.0 (confidence: 0.98)
- Healthcare: "ADT_message.hl7" → HL7 ADT parser v2.5.0 (confidence: 0.95)
- Legal: "CA_court_filing_12345.pdf" → CA Court parser v1.2.0 (confidence: 0.92)
- Research: "references.bib" → BibTeX parser v2.0.1 (confidence: 0.99)

---

## API Reference

### `select_parser()`
```python
def select_parser(
    self,
    file_metadata: FileMetadata,
    user_preferences: Optional[UserPreferences] = None
) -> Optional[ParserSelection]:
    """
    Auto-selects best parser based on file metadata.

    Selection criteria (weighted):
    1. Filename pattern match (0.6) - Regex matches filename
    2. File type match (0.3) - Extension matches parser file_types
    3. Required capabilities (0.1) - Has all required caps
    4. User preference boost (+0.1) - Parser in user's preferred list

    Args:
        file_metadata: File metadata (name, type, size, required_capabilities)
        user_preferences: Optional user preferences (preferred_parser_ids, min_confidence)

    Returns:
        ParserSelection with selected parser + alternatives, or None if no match ≥ min_confidence

    Example:
        selection = selector.select_parser(
            file_metadata=FileMetadata(
                file_name="BofA_Statement_Nov2024.pdf",
                file_type="pdf",
                required_capabilities=["date", "amount", "merchant"]
            ),
            user_preferences=UserPreferences(
                preferred_parser_ids=["parser_bofa_pdf"],
                min_confidence=0.7
            )
        )
        # Returns:
        # ParserSelection(
        #   selected_parser=RankedParser(parser_id="parser_bofa_pdf", confidence=0.98),
        #   alternative_parsers=[
        #       RankedParser(parser_id="parser_generic_pdf", confidence=0.40)
        #   ]
        # )
    """
```

### `rank_parsers()`
```python
def rank_parsers(
    self,
    file_metadata: FileMetadata
) -> List[RankedParser]:
    """
    Returns all matching parsers ranked by confidence (descending).

    Args:
        file_metadata: File metadata

    Returns:
        List of RankedParser sorted by confidence (high to low)

    Example:
        ranked = selector.rank_parsers(
            file_metadata=FileMetadata(file_name="statement.pdf", file_type="pdf")
        )
        for parser in ranked:
            print(f"{parser.parser_id}: {parser.confidence:.2f} - {parser.reason}")
        # Output:
        # parser_bofa_pdf: 0.98 - Filename matches '.*BofA.*\.pdf', file type matches
        # parser_chase_pdf: 0.55 - File type matches
        # parser_generic_pdf: 0.40 - File type matches
    """
```

### `suggest_parsers()`
```python
def suggest_parsers(
    self,
    file_metadata: FileMetadata,
    min_confidence: float = 0.5
) -> List[ParserSuggestion]:
    """
    Returns parser suggestions for user to choose from (confidence ≥ min).

    Args:
        file_metadata: File metadata
        min_confidence: Minimum confidence threshold (default: 0.5)

    Returns:
        List of ParserSuggestion with confidence ≥ min_confidence

    Example:
        suggestions = selector.suggest_parsers(
            file_metadata=FileMetadata(file_name="ambiguous.pdf", file_type="pdf"),
            min_confidence=0.5
        )
        # Returns multiple suggestions if confidence < 0.7 (auto-select threshold)
    """
```

---

## Data Model

### FileMetadata

```python
@dataclass
class FileMetadata:
    file_name: str
    file_type: str  # Extension (pdf, csv, xlsx, etc.)
    file_size: Optional[int] = None
    required_capabilities: Optional[List[str]] = None
```

### UserPreferences

```python
@dataclass
class UserPreferences:
    preferred_parser_ids: List[str] = field(default_factory=list)
    min_confidence: float = 0.7
```

### ParserSelection

```python
@dataclass
class ParserSelection:
    selected_parser: RankedParser
    alternative_parsers: List[RankedParser]
    selected_at: datetime
```

### RankedParser

```python
@dataclass
class RankedParser:
    parser_id: str
    version: str
    confidence: float
    reason: str
    capabilities: List[str]
```

---

## Selection Examples

### Example 1: High-Confidence Auto-Selection

```python
# File: "BofA_Statement_Nov2024.pdf"
selection = selector.select_parser(
    file_metadata=FileMetadata(file_name="BofA_Statement_Nov2024.pdf", file_type="pdf")
)

print(f"Selected: {selection.selected_parser.parser_id}")
print(f"Confidence: {selection.selected_parser.confidence:.0%}")
print(f"Reason: {selection.selected_parser.reason}")

# Output:
# Selected: parser_bofa_pdf
# Confidence: 98%
# Reason: Filename matches pattern '.*BofA.*\.pdf' and file type matches
```

### Example 2: Ambiguous File (User Override Needed)

```python
# File: "statement.pdf" (no bank indicator)
selection = selector.select_parser(
    file_metadata=FileMetadata(file_name="statement.pdf", file_type="pdf")
)

if selection and selection.selected_parser.confidence < 0.7:
    print("⚠️ Low confidence - please select parser:")
    print(f"1. {selection.selected_parser.parser_id} ({selection.selected_parser.confidence:.0%})")
    for i, alt in enumerate(selection.alternative_parsers, start=2):
        print(f"{i}. {alt.parser_id} ({alt.confidence:.0%})")

# Output:
# ⚠️ Low confidence - please select parser:
# 1. parser_generic_pdf (55%)
# 2. parser_bofa_pdf (40%)
# 3. parser_chase_pdf (40%)
```

---

## Multi-Domain Examples

### Healthcare: HL7 Message Auto-Selection

```python
selection = selector.select_parser(
    file_metadata=FileMetadata(
        file_name="ADT_A01_20241101.hl7",
        file_type="hl7",
        required_capabilities=["patient_id", "visit_number"]
    )
)
# Selected: parser_hl7_v2_5_adt (confidence: 0.95)
# Reason: Filename matches '.*ADT.*\.hl7', file type matches, has required capabilities
```

### Legal: Court Filing Auto-Selection

```python
selection = selector.select_parser(
    file_metadata=FileMetadata(
        file_name="CA_Superior_Court_Filing_12345.pdf",
        file_type="pdf",
        required_capabilities=["case_number", "jurisdiction"]
    )
)
# Selected: parser_ca_court_filing_pdf (confidence: 0.92)
# Reason: Filename matches '.*CA.*court.*\.pdf', file type matches, has required capabilities
```

---

## Confidence Scoring Algorithm

```python
def calculate_confidence(
    parser: ParserRegistration,
    file_metadata: FileMetadata,
    user_preferences: Optional[UserPreferences]
) -> float:
    score = 0.0

    # Filename pattern match (0.6 weight)
    if matches_filename_pattern(parser.filename_patterns, file_metadata.file_name):
        score += 0.6

    # File type match (0.3 weight)
    if file_metadata.file_type in parser.file_types:
        score += 0.3

    # Required capabilities (0.1 weight)
    if file_metadata.required_capabilities:
        if all(cap in parser.capabilities for cap in file_metadata.required_capabilities):
            score += 0.1

    # User preference boost (+0.1)
    if user_preferences and parser.parser_id in user_preferences.preferred_parser_ids:
        score += 0.1

    return min(score, 1.0)  # Cap at 1.0
```

---

## Summary

ParserSelector provides **intelligent parser auto-selection** with confidence scoring, ranking, and user override support for seamless file processing.
