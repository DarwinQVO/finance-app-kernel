# Primitive: ParserCapabilityStore (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Data Store
> **Vertical:** 3.7 Parser Registry
> **Last Updated:** 2025-10-24

---

## Overview

**ParserCapabilityStore** tracks what fields each parser can extract (capabilities) with confidence scores. It enables capability-based parser discovery ("find parsers that can extract 'category' field with 80%+ confidence") and provides transparency into parser strengths/weaknesses.

**Key Responsibilities:**
- Store capability definitions for each parser
- Track extraction confidence per capability (0.0-1.0)
- Mark capabilities as required vs optional
- Query parsers by capability (e.g., "find parsers with 'balance' capability")
- Generate capability matrix (cross-reference parsers × capabilities)
- Validate capabilities against known field types

**Multi-Domain Examples:**
- Finance: Track which parsers extract date, amount, merchant, account, balance
- Healthcare: Track which parsers extract patient_id, diagnosis_code, procedure_code
- Legal: Track which parsers extract case_number, jurisdiction, filing_date
- Research: Track which parsers extract author, title, year, doi, journal

---

## API Reference

### Core Methods

#### `set_capabilities()`
```python
def set_capabilities(
    self,
    parser_id: str,
    capabilities: List[ParserCapability]
) -> None:
    """
    Stores capability definitions for parser.

    Args:
        parser_id: Unique parser identifier (e.g., "parser_bofa_pdf")
        capabilities: List of ParserCapability with confidence scores

    Raises:
        ParserNotFoundError: If parser_id doesn't exist in ParserRegistry
        InvalidCapabilityError: If capability_name not a known field type
        InvalidConfidenceError: If extraction_confidence not in [0.0, 1.0]

    Example:
        store.set_capabilities(
            parser_id="parser_bofa_pdf",
            capabilities=[
                ParserCapability(
                    capability_name="date",
                    field_type="date",
                    extraction_confidence=0.98,
                    required_for_success=True
                ),
                ParserCapability(
                    capability_name="amount",
                    field_type="number",
                    extraction_confidence=0.98,
                    required_for_success=True
                ),
                ParserCapability(
                    capability_name="merchant",
                    field_type="string",
                    extraction_confidence=0.95,
                    required_for_success=False
                )
            ]
        )
    """
```

#### `get_capabilities()`
```python
def get_capabilities(self, parser_id: str) -> List[ParserCapability]:
    """
    Returns all capabilities for parser.

    Args:
        parser_id: Unique parser identifier

    Returns:
        List of ParserCapability sorted by capability_name

    Example:
        capabilities = store.get_capabilities("parser_bofa_pdf")
        for cap in capabilities:
            print(f"{cap.capability_name}: {cap.extraction_confidence:.0%} confidence")
        # Output:
        # date: 98% confidence
        # amount: 98% confidence
        # merchant: 95% confidence
    """
```

#### `find_parsers_with_capability()`
```python
def find_parsers_with_capability(
    self,
    capability_name: str,
    min_confidence: float = 0.7
) -> List[str]:
    """
    Finds parser_ids that support given capability.

    Args:
        capability_name: Name of capability (e.g., "balance", "category")
        min_confidence: Minimum extraction confidence (default: 0.7)

    Returns:
        List of parser_ids sorted by extraction_confidence (descending)

    Example:
        # Find parsers that can extract balance with 80%+ confidence
        parsers = store.find_parsers_with_capability("balance", min_confidence=0.8)
        # Returns: ["parser_bofa_pdf", "parser_chase_pdf"]
        # (Generic PDF parser excluded - balance confidence only 0.4)
    """
```

#### `get_capability_matrix()`
```python
def get_capability_matrix(
    self,
    parser_ids: Optional[List[str]] = None,
    capability_names: Optional[List[str]] = None
) -> CapabilityMatrix:
    """
    Generates capability matrix (cross-reference parsers × capabilities).

    Args:
        parser_ids: Optional filter to specific parsers (default: all)
        capability_names: Optional filter to specific capabilities (default: all)

    Returns:
        CapabilityMatrix with parsers as rows, capabilities as columns

    Example:
        matrix = store.get_capability_matrix(
            parser_ids=["parser_bofa_pdf", "parser_chase_pdf", "parser_generic_pdf"],
            capability_names=["date", "amount", "merchant", "balance"]
        )
        print(matrix.to_table())
        # Output:
        # Parser              | date | amount | merchant | balance
        # --------------------|------|--------|----------|--------
        # parser_bofa_pdf     | 0.98 | 0.98   | 0.95     | 0.95
        # parser_chase_pdf    | 0.97 | 0.97   | 0.92     | N/A
        # parser_generic_pdf  | 0.80 | 0.85   | 0.70     | 0.40
    """
```

#### `update_capability()`
```python
def update_capability(
    self,
    parser_id: str,
    capability_name: str,
    extraction_confidence: Optional[float] = None,
    required_for_success: Optional[bool] = None
) -> ParserCapability:
    """
    Updates specific capability for parser.

    Args:
        parser_id: Unique parser identifier
        capability_name: Name of capability to update
        extraction_confidence: Optional new confidence score
        required_for_success: Optional new required flag

    Returns:
        Updated ParserCapability

    Raises:
        CapabilityNotFoundError: If parser doesn't have this capability

    Example:
        # Improve BofA parser's merchant extraction confidence
        updated = store.update_capability(
            parser_id="parser_bofa_pdf",
            capability_name="merchant",
            extraction_confidence=0.98  # Upgraded from 0.95
        )
    """
```

---

## Data Model

### ParserCapability

```python
@dataclass
class ParserCapability:
    parser_id: str  # Parser this capability belongs to
    capability_name: str  # Name of extractable field (e.g., "date", "amount")
    field_type: str  # Data type (string, number, date, boolean, array)
    extraction_confidence: float  # Confidence score [0.0, 1.0]
    required_for_success: bool  # If True, parser fails if field not extracted
    notes: Optional[str]  # Optional human-readable notes
```

**Example:**
```json
{
  "parser_id": "parser_bofa_pdf",
  "capability_name": "date",
  "field_type": "date",
  "extraction_confidence": 0.98,
  "required_for_success": true,
  "notes": "Extracts transaction date in MM/DD/YYYY format"
}
```

### CapabilityMatrix

```python
@dataclass
class CapabilityMatrix:
    parsers: List[str]  # Parser IDs (rows)
    capabilities: List[str]  # Capability names (columns)
    scores: Dict[str, Dict[str, Optional[float]]]  # {parser_id: {capability: confidence}}

    def to_table(self) -> str:
        """Renders matrix as ASCII table."""

    def get_score(self, parser_id: str, capability_name: str) -> Optional[float]:
        """Get confidence score for parser × capability."""

    def get_best_parser(self, capability_name: str) -> Optional[str]:
        """Returns parser_id with highest confidence for capability."""
```

---

## Multi-Domain Examples

### Finance: Capability Matrix

```
Parser              | date | amount | merchant | account | balance | category
--------------------|------|--------|----------|---------|---------|----------
parser_bofa_pdf     | 0.98 | 0.98   | 0.95     | 0.98    | 0.95    | N/A
parser_chase_pdf    | 0.97 | 0.97   | 0.92     | 0.95    | N/A     | N/A
parser_apple_csv    | 0.99 | 0.99   | 0.98     | N/A     | N/A     | 0.85
parser_generic_pdf  | 0.80 | 0.85   | 0.70     | 0.60    | 0.40    | N/A
```

### Healthcare: HL7 Parser Capabilities

```python
store.set_capabilities(
    parser_id="parser_hl7_v2_5_adt",
    capabilities=[
        ParserCapability(
            capability_name="patient_id",
            field_type="string",
            extraction_confidence=0.99,
            required_for_success=True
        ),
        ParserCapability(
            capability_name="visit_number",
            field_type="string",
            extraction_confidence=0.99,
            required_for_success=True
        ),
        ParserCapability(
            capability_name="admission_date",
            field_type="date",
            extraction_confidence=0.98,
            required_for_success=False
        ),
        ParserCapability(
            capability_name="diagnosis_code",
            field_type="array",
            extraction_confidence=0.90,
            required_for_success=False
        )
    ]
)
```

---

## Validation Rules

### Capability Name
- Must reference known field type (validated against FieldTypeRegistry)
- Length: 1-50 characters
- Pattern: Alphanumeric + underscores (e.g., "patient_id", "diagnosis_code")
- Examples: ✅ "date", "patient_id", ❌ "date!", "patient id" (space)

### Extraction Confidence
- Range: [0.0, 1.0]
- Precision: 2 decimal places (e.g., 0.98)
- Examples: ✅ 0.98, 0.70, 1.0, ❌ 1.5, -0.1

### Field Type
- Allowed values: `string`, `number`, `date`, `boolean`, `array`
- Must match capability_name's expected type
- Examples: ✅ date→date, amount→number, ❌ date→string

---

## Usage Examples

### Example 1: Set Capabilities for Parser

```python
# Define capabilities for BofA PDF parser
store.set_capabilities(
    parser_id="parser_bofa_pdf",
    capabilities=[
        ParserCapability("date", "date", 0.98, True),
        ParserCapability("amount", "number", 0.98, True),
        ParserCapability("merchant", "string", 0.95, False),
        ParserCapability("account", "string", 0.98, True),
        ParserCapability("balance", "number", 0.95, False)
    ]
)
```

### Example 2: Find Parsers by Capability

```python
# Find parsers that extract balance
parsers_with_balance = store.find_parsers_with_capability("balance", min_confidence=0.8)
print(f"Parsers with balance: {parsers_with_balance}")
# Output: ["parser_bofa_pdf"] (chase and generic don't meet 0.8 threshold)

# Find parsers that extract category
parsers_with_category = store.find_parsers_with_capability("category")
print(f"Parsers with category: {parsers_with_category}")
# Output: ["parser_apple_csv", "parser_ml_advanced"]
```

### Example 3: Generate Capability Matrix

```python
# Get capability matrix for PDF parsers
matrix = store.get_capability_matrix(
    parser_ids=["parser_bofa_pdf", "parser_chase_pdf", "parser_generic_pdf"],
    capability_names=["date", "amount", "merchant", "balance"]
)

print(matrix.to_table())
# Outputs formatted table (see above)

# Find best parser for balance extraction
best = matrix.get_best_parser("balance")
print(f"Best parser for balance: {best}")  # "parser_bofa_pdf"
```

---

## Persistence

### Database Schema

```sql
CREATE TABLE parser_capabilities (
    capability_id SERIAL PRIMARY KEY,
    parser_id VARCHAR(100) NOT NULL,
    capability_name VARCHAR(50) NOT NULL,
    field_type VARCHAR(20) NOT NULL,
    extraction_confidence DECIMAL(3,2) NOT NULL CHECK (extraction_confidence >= 0 AND extraction_confidence <= 1),
    required_for_success BOOLEAN DEFAULT FALSE,
    notes TEXT,
    FOREIGN KEY (parser_id) REFERENCES parser_registrations(parser_id) ON DELETE CASCADE,
    UNIQUE(parser_id, capability_name)
);

CREATE INDEX idx_capabilities_parser ON parser_capabilities(parser_id, capability_name);
CREATE INDEX idx_capabilities_name ON parser_capabilities(capability_name, extraction_confidence DESC);
```

---

## Summary

ParserCapabilityStore provides **capability-based parser discovery** with:

✅ **Capability tracking** with confidence scores
✅ **Query by capability** (find parsers that extract specific fields)
✅ **Capability matrix** (compare parsers side-by-side)
✅ **Multi-domain** (Finance, Healthcare, Legal, Research, E-commerce)

**Used By:** ParserSelector (capability-based ranking), ParserCapabilitiesCard (UI display)
