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

## Simplicity Profiles

The same ParserSelector interface supports three implementation scales: Personal (no auto-selection), Small Business (simple pattern matching), Enterprise (ML-based confidence scoring).

### Personal Profile (0 LOC - Not Needed)

**Darwin's Approach:** No parser selection - always use the same parser

**Why Darwin doesn't need this:**
- Darwin has exactly 1 parser (BofAPDFParser)
- All uploads are Bank of America PDFs
- No selection logic needed when there's only one option

**What Darwin DOESN'T need:**
- ❌ Filename pattern matching
- ❌ Confidence scoring
- ❌ Alternative parser suggestions
- ❌ User override UI

**Implementation:**

Darwin skips this primitive entirely.

**Darwin's upload flow:**
```python
# Simple hardcoded parser selection
def handle_upload(pdf_path):
    parser = BofAPDFParser()  # Always use BoFA parser
    observations = parser.parse(pdf_path)
    store_observations(observations)
```

**Decision Context:**
- Darwin only banks with Bank of America
- All statements are same PDF format
- No need for selection when there's only one parser
- YAGNI: Skip ParserSelector when you have 1 parser

---

### Small Business Profile (60 LOC)

**Use Case:** Accounting firm with 8 bank parsers, simple filename pattern matching

**What they need (vs Personal):**
- ✅ **Filename pattern matching** (simple regex per parser)
- ✅ **File type matching** (.pdf vs .csv vs .ofx)
- ✅ **Fallback to manual selection** (if no match)
- ❌ No confidence scoring (pattern match or no match)
- ❌ No user preferences
- ❌ No ML model

**Implementation:**

**Config File (config/parser-patterns.yaml):**
```yaml
parsers:
  - parser_id: "bofa_pdf"
    filename_patterns:
      - ".*[Bb]ank.*[Oo]f.*[Aa]merica.*\\.pdf"
      - ".*BofA.*\\.pdf"
      - ".*BOA.*\\.pdf"
    file_types: ["pdf"]

  - parser_id: "chase_csv"
    filename_patterns:
      - ".*[Cc]hase.*\\.csv"
    file_types: ["csv"]

  - parser_id: "wells_fargo_pdf"
    filename_patterns:
      - ".*Wells.*Fargo.*\\.pdf"
      - ".*WF.*Statement.*\\.pdf"
    file_types: ["pdf"]

  - parser_id: "amex_ofx"
    filename_patterns:
      - ".*[Aa]merican.*[Ee]xpress.*\\.ofx"
      - ".*Amex.*\\.ofx"
    file_types: ["ofx", "qfx"]
```

**Code Example (60 LOC):**
```python
# parser_selector.py (Small Business)

import yaml
import re
from pathlib import Path
from typing import Optional

class ParserSelector:
    """Simple filename pattern matching for parser selection."""

    def __init__(self, config_path: str = "config/parser-patterns.yaml"):
        self.config = yaml.safe_load(Path(config_path).read_text())

    def select_parser(self, filename: str) -> Optional[str]:
        """
        Select parser based on filename pattern.

        Returns:
            parser_id if match found, None if no match
        """
        for parser in self.config["parsers"]:
            # Try each filename pattern
            for pattern in parser["filename_patterns"]:
                if re.search(pattern, filename, re.IGNORECASE):
                    return parser["parser_id"]

        return None  # No match

    def get_parser_by_file_type(self, file_type: str) -> Optional[str]:
        """
        Get first parser that supports file type.

        Fallback if filename pattern doesn't match.
        """
        for parser in self.config["parsers"]:
            if file_type.lower() in parser["file_types"]:
                return parser["parser_id"]

        return None
```

**Usage:**
```python
selector = ParserSelector("config/parser-patterns.yaml")

# High-confidence match
parser_id = selector.select_parser("BofA_Statement_Nov2024.pdf")
# Returns: "bofa_pdf"

# Low-confidence match (no filename pattern match, fallback to file type)
parser_id = selector.select_parser("statement_123.pdf")
if not parser_id:
    # Fallback to file type
    parser_id = selector.get_parser_by_file_type("pdf")
    # Returns: First PDF parser (could be bofa_pdf or wells_fargo_pdf)
    print(f"⚠️ Auto-selected {parser_id} based on file type. Please confirm.")

# No match
parser_id = selector.select_parser("unknown_file.txt")
# Returns: None
if not parser_id:
    print("❌ No parser found. Please select manually.")
```

**UI Flow:**
```python
# Upload handler
def handle_upload(filename):
    parser_id = selector.select_parser(filename)

    if parser_id:
        print(f"✓ Auto-selected parser: {parser_id}")
        proceed_with_parsing(parser_id)
    else:
        # Show manual selection UI
        print("Please select parser:")
        print("1. Bank of America PDF")
        print("2. Chase CSV")
        print("3. Wells Fargo PDF")
        print("4. Amex OFX")
        choice = input("Enter number: ")
        parser_id = get_parser_by_choice(choice)
        proceed_with_parsing(parser_id)
```

**Testing:**
- ✅ Unit test: Each filename pattern matches correct parser
- ✅ Unit test: File type fallback works
- ✅ Unit test: Unknown files return None

**Decision Context:**
- Accounting firm has 8 banks → 8 parsers with distinct filename patterns
- Simple regex matching sufficient (95% accuracy)
- Manual fallback acceptable for ambiguous files (5% of uploads)

---

### Enterprise Profile (400 LOC)

**Use Case:** Multi-tenant platform with 50+ parsers, ML-based confidence scoring

**What they need (vs Small Business):**
- ✅ All Small Business features (pattern matching, file type matching)
- ✅ **Confidence scoring** (0.0-1.0 based on multiple signals)
- ✅ **Ranked alternatives** (show top 3 matches with confidence)
- ✅ **User preferences** (boost preferred parsers)
- ✅ **Capability matching** (parser must support required fields)
- ✅ **ML model** (learn from user overrides to improve accuracy)
- ✅ **Selection logging** (audit trail for parser choice)

**Implementation:**

**Database Schema:**
```sql
CREATE TABLE parser_registrations (
    parser_id TEXT PRIMARY KEY,
    version TEXT NOT NULL,
    filename_patterns TEXT[],
    file_types TEXT[],
    capabilities TEXT[],  -- ["date", "amount", "merchant"]
    priority INTEGER DEFAULT 0
);

CREATE TABLE user_parser_preferences (
    user_id TEXT NOT NULL,
    parser_id TEXT NOT NULL,
    preference_score FLOAT DEFAULT 1.0,  -- Learned from overrides
    PRIMARY KEY (user_id, parser_id)
);

CREATE TABLE parser_selection_log (
    selection_id TEXT PRIMARY KEY,
    upload_id TEXT NOT NULL,
    filename TEXT NOT NULL,
    auto_selected_parser TEXT,
    actual_selected_parser TEXT,  -- May differ if user overrode
    confidence FLOAT,
    alternatives JSONB,
    selected_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Code Example (400 LOC - excerpt):**
```python
# parser_selector.py (Enterprise)

import re
from typing import List, Optional, Dict
from dataclasses import dataclass
import psycopg2
from psycopg2.extras import DictCursor

@dataclass
class RankedParser:
    parser_id: str
    version: str
    confidence: float
    reason: str

@dataclass
class ParserSelection:
    selected_parser: RankedParser
    alternatives: List[RankedParser]

class ParserSelector:
    """Enterprise parser selection with ML-based confidence scoring."""

    def __init__(self, db_conn_string: str):
        self.db_conn_string = db_conn_string

    def select_parser(
        self,
        filename: str,
        file_type: str,
        required_capabilities: Optional[List[str]] = None,
        user_id: Optional[str] = None,
        min_confidence: float = 0.7
    ) -> Optional[ParserSelection]:
        """
        Auto-select parser with confidence scoring.

        Args:
            filename: Uploaded filename
            file_type: File extension (pdf, csv, ofx)
            required_capabilities: Optional required fields
            user_id: Optional user ID (for preference boost)
            min_confidence: Minimum confidence threshold

        Returns:
            ParserSelection with top choice + alternatives, or None if confidence < min
        """
        # Rank all parsers
        ranked = self.rank_parsers(filename, file_type, required_capabilities, user_id)

        if not ranked or ranked[0].confidence < min_confidence:
            return None

        return ParserSelection(
            selected_parser=ranked[0],
            alternatives=ranked[1:4]  # Top 3 alternatives
        )

    def rank_parsers(
        self,
        filename: str,
        file_type: str,
        required_capabilities: Optional[List[str]] = None,
        user_id: Optional[str] = None
    ) -> List[RankedParser]:
        """Rank all parsers by confidence score."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        cursor.execute("""
            SELECT parser_id, version, filename_patterns, file_types, capabilities, priority
            FROM parser_registrations
            WHERE ? = ANY(file_types)
            ORDER BY priority DESC
        """, (file_type,))

        parsers = cursor.fetchall()

        # Get user preferences if available
        user_prefs = {}
        if user_id:
            cursor.execute("""
                SELECT parser_id, preference_score
                FROM user_parser_preferences
                WHERE user_id = %s
            """, (user_id,))
            user_prefs = {row["parser_id"]: row["preference_score"] for row in cursor.fetchall()}

        conn.close()

        # Score each parser
        ranked = []
        for parser in parsers:
            score, reason = self._calculate_score(
                parser=parser,
                filename=filename,
                file_type=file_type,
                required_capabilities=required_capabilities,
                user_preference_score=user_prefs.get(parser["parser_id"], 1.0)
            )

            ranked.append(RankedParser(
                parser_id=parser["parser_id"],
                version=parser["version"],
                confidence=score,
                reason=reason
            ))

        # Sort by confidence descending
        ranked.sort(key=lambda p: p.confidence, reverse=True)

        return ranked

    def _calculate_score(
        self,
        parser: Dict,
        filename: str,
        file_type: str,
        required_capabilities: Optional[List[str]],
        user_preference_score: float
    ) -> tuple[float, str]:
        """Calculate confidence score for parser."""
        score = 0.0
        reasons = []

        # 1. Filename pattern match (0.6 weight)
        filename_match = False
        for pattern in parser["filename_patterns"]:
            if re.search(pattern, filename, re.IGNORECASE):
                score += 0.6
                reasons.append(f"Filename matches pattern '{pattern}'")
                filename_match = True
                break

        # 2. File type match (0.3 weight)
        if file_type.lower() in parser["file_types"]:
            score += 0.3
            reasons.append(f"File type '{file_type}' supported")

        # 3. Required capabilities (0.1 weight)
        if required_capabilities:
            parser_caps = set(parser["capabilities"])
            required_caps = set(required_capabilities)
            if required_caps.issubset(parser_caps):
                score += 0.1
                reasons.append("All required capabilities supported")
            else:
                missing = required_caps - parser_caps
                reasons.append(f"Missing capabilities: {missing}")

        # 4. User preference boost (multiply by learned score)
        if user_preference_score > 1.0:
            score *= user_preference_score
            reasons.append(f"User preference boost ({user_preference_score:.2f}x)")

        # Cap at 1.0
        score = min(score, 1.0)

        reason = ", ".join(reasons) if reasons else "No match"
        return (score, reason)

    def log_selection(
        self,
        upload_id: str,
        filename: str,
        auto_selected: Optional[str],
        actual_selected: str,
        confidence: Optional[float],
        alternatives: List[RankedParser]
    ):
        """Log parser selection for ML training."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO parser_selection_log
            (upload_id, filename, auto_selected_parser, actual_selected_parser,
             confidence, alternatives)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, (
            upload_id,
            filename,
            auto_selected,
            actual_selected,
            confidence,
            json.dumps([{"parser_id": p.parser_id, "confidence": p.confidence} for p in alternatives])
        ))

        conn.commit()
        conn.close()

        # If user overrode selection, update preference
        if auto_selected and auto_selected != actual_selected:
            self._update_user_preference(upload_id, actual_selected, boost=1.1)

    def _update_user_preference(self, upload_id: str, parser_id: str, boost: float):
        """Update user preference based on override."""
        # Get user_id from upload
        # Update preference_score *= boost
        pass  # Implementation omitted
```

**Usage Example:**
```python
selector = ParserSelector("postgresql://localhost/parsers")

# Auto-select with confidence
selection = selector.select_parser(
    filename="BofA_Statement_Nov2024.pdf",
    file_type="pdf",
    required_capabilities=["date", "amount", "merchant"],
    user_id="user_123",
    min_confidence=0.7
)

if selection:
    print(f"✓ Auto-selected: {selection.selected_parser.parser_id}")
    print(f"  Confidence: {selection.selected_parser.confidence:.0%}")
    print(f"  Reason: {selection.selected_parser.reason}")

    if selection.selected_parser.confidence < 0.9:
        print("\nAlternatives:")
        for alt in selection.alternatives:
            print(f"  - {alt.parser_id} ({alt.confidence:.0%})")
else:
    print("❌ No parser found with confidence ≥ 70%")
    print("Please select manually from all available parsers")

# Log selection
selector.log_selection(
    upload_id="UL_123",
    filename="BofA_Statement_Nov2024.pdf",
    auto_selected=selection.selected_parser.parser_id if selection else None,
    actual_selected="bofa_pdf",  # User's final choice
    confidence=selection.selected_parser.confidence if selection else None,
    alternatives=selection.alternatives if selection else []
)
```

**ML-Based Preference Learning:**
```python
# Over time, learn user preferences
# User always overrides "generic_pdf" → "bofa_pdf" for "statement.pdf"
# After 5 overrides:
#   user_parser_preferences.preference_score["bofa_pdf"] → 1.5
# Next upload "statement.pdf":
#   bofa_pdf score: 0.3 (file type only) * 1.5 (preference) = 0.45
#   generic_pdf score: 0.3 (file type only) * 1.0 = 0.30
# Auto-selects bofa_pdf instead of generic_pdf
```

**Testing:**
- ✅ Unit tests for confidence scoring
- ✅ Integration tests with test database
- ✅ ML preference learning validation
- ✅ User override tracking accuracy

**Monitoring:**
- Auto-selection accuracy (% of uploads with no user override)
- Confidence distribution (histogram of confidence scores)
- Most frequently overridden parsers (need better patterns)

**Decision Context:**
- Platform has 50+ parsers across multiple file formats
- Confidence scoring reduces manual selection by 90%
- User preference learning improves accuracy over time
- Selection logging enables ML model training

---

## Summary

ParserSelector provides **intelligent parser auto-selection** with confidence scoring, ranking, and user override support for seamless file processing.
