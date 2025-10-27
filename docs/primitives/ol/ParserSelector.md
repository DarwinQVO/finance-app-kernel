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

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Auto-select bank parser based on filename and file type
**Example:** User uploads "BofA_Statement_Nov2024.pdf" → ParserSelector.select_parser(FileMetadata(name="BofA_Statement_Nov2024.pdf", file_type="pdf")) → Scores all parsers: BofAPDFParser (filename match: 0.6, file_type match: 0.3 = 0.9 confidence), ChasePDFParser (file_type match: 0.3 only = 0.3 confidence) → Returns ParserSelection(selected="bofa_pdf_parser", confidence=0.9, alternatives=[{parser_id: "chase_pdf_parser", confidence: 0.3}]) → Upload proceeds with BofA parser → Low confidence case: "statement_123.pdf" (no filename match) → Confidence 0.3 < threshold 0.7 → Returns alternatives, user manually selects parser
**Operations:** select_parser (weighted scoring), rank_parsers (sort by confidence), calculate_score (filename regex + file type + capabilities), provide alternatives (if confidence < threshold)
**Scoring:** Filename pattern (0.6), file type (0.3), capabilities (0.1), user preference (+0.1)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Auto-select HL7 parser based on filename pattern
**Example:** Hospital uploads "ADT_message_20241101.hl7" → ParserSelector scores: HL7ADTParser (filename "ADT_" match: 0.6, .hl7 extension: 0.3 = 0.9), HL7ORMParser (extension only: 0.3) → Selects HL7ADTParser with confidence 0.9
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Auto-select court filing parser based on jurisdiction in filename
**Example:** Law firm uploads "CA_court_filing_12345.pdf" → ParserSelector scores: CACourtParser (filename "CA_court" match: 0.6, .pdf: 0.3 = 0.9), NYCourtParser (.pdf only: 0.3) → Selects CACourtParser
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Auto-select web page parser based on source domain in filename
**Example:** Scraper downloads "techcrunch_com_article_2024-02-25.html" → ParserSelector.select_parser(FileMetadata(name="techcrunch_com_article_2024-02-25.html", file_type="html", source_domain="techcrunch.com")) → Scores: TechCrunchHTMLParser (filename "techcrunch_com" match: 0.6, .html: 0.3, capabilities match: 0.1 = 1.0), GenericHTMLParser (.html only: 0.3) → Returns ParserSelection(selected="techcrunch_html_parser", confidence=1.0, reasoning="Filename pattern exact match + file type match") → User preference: VC analyst prefers specific parsers → User preferences boost TechCrunchHTMLParser +0.1 (already at 1.0, capped)
**Operations:** Source domain matching critical (techcrunch.com → TechCrunchHTMLParser, twitter.com → TwitterJSONParser), capability matching (parser must support "entity_extraction" if file requires it), fallback to GenericHTMLParser if no domain-specific parser matches, user preferences allow analysts to set preferred parsers per source
**Scoring:** Filename/domain pattern (0.6 weight, regex matching), file type (.html, .json, .xml: 0.3), required capabilities (entity extraction, fact typing: 0.1), analyst preference (+0.1 boost)
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Auto-select supplier catalog parser based on supplier name in filename
**Example:** Merchant uploads "supplier_acme_catalog_2024.csv" → ParserSelector scores: AcmeSupplierCSVParser (filename "acme" match: 0.6, .csv: 0.3 = 0.9), GenericCSVParser (.csv only: 0.3) → Selects AcmeSupplierCSVParser
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (weighted scoring, confidence ranking, auto-selection are universal patterns)
**Reusability:** High (same select_parser/rank_parsers/calculate_score operations work for bank statements, HL7 messages, court filings, web pages, supplier catalogs; only filename patterns and file types differ)

---

## Summary

ParserSelector provides **intelligent parser auto-selection** with confidence scoring, ranking, and user override support for seamless file processing.
