# OL Primitive: Parser

**Type**: Data Extraction / Transform
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.2 (Extraction)

---

## Purpose

Universal interface for extracting structured observations from unstructured/semi-structured source documents. Parsers are domain-specific (e.g., `bofa_pdf_parser`, `lab_results_parser`) but follow a universal contract that enables pluggable extraction across ANY domain.

---

## Interface Contract

### Core Interface

```python
from abc import ABC, abstractmethod
from typing import List

class Parser(ABC):
    @property
    @abstractmethod
    def parser_id(self) -> str:
        """Unique identifier for this parser. Example: 'bofa_pdf_parser'"""
        pass

    @property
    @abstractmethod
    def version(self) -> str:
        """Semantic version (e.g., '1.2.0')"""
        pass

    @property
    @abstractmethod
    def supported_source_types(self) -> List[str]:
        """List of source_type values this parser handles. Example: ['bofa_pdf']"""
        pass

    @abstractmethod
    def parse(self, content: bytes) -> List[Observation]:
        """
        Extract raw observations from source document.

        Args:
            content: Raw file bytes

        Returns:
            List of raw observations (unvalidated, AS-IS from source)

        Raises:
            ParseError: If document is corrupted or unparseable
        """
        pass

    @abstractmethod
    def can_parse(self, content: bytes) -> bool:
        """
        Quick check if this parser can handle the content.

        Returns:
            True if parser recognizes format, False otherwise
        """
        pass
```

### Types

```python
from dataclasses import dataclass
from typing import Any, Dict, List

@dataclass
class Observation:
    """
    Raw extracted data (AS-IS from parser, no validation).
    Domain-specific schema (ObservationTransaction, ObservationLabResult, etc.)
    """
    raw_data: Dict[str, Any]  # Domain-specific fields
    row_id: int                # Position in source file (0-indexed)
    warnings: List[str] = []   # Row-level warnings (not errors)

@dataclass
class ParseResult:
    """Result of parse operation"""
    success: bool
    observations: List[Observation]
    errors: List[str] = []
    warnings: List[str] = []
```

---

## Behavior Specifications

### 1. Extract AS-IS (No Validation)

**Property**: Parser extracts text exactly as it appears, WITHOUT interpretation or validation.

**Example:**
```python
# Parser sees "01/02/2024" → Extract as string "01/02/2024"
# DO NOT interpret as date (could be Jan 2 or Feb 1)
# Normalization stage (vertical 1.3) handles interpretation

observation = Observation(
    raw_data={
        "date": "01/02/2024",  # AS-IS (string)
        "amount": "(50.00)",   # AS-IS (string, not -50.00)
        "description": "  STARBUCKS #1234  "  # AS-IS (keep spaces)
    },
    row_id=0
)
```

**Rationale** (from ADR-0004):
- Preserve original data (lossless extraction)
- Enable re-normalization without re-parsing
- Simpler parser implementation (no domain logic)

---

### 2. Deterministic Row Ordering

**Property**: Same file → same row_id assignments (stable)

**Implementation:**
```python
def parse(self, content: bytes) -> List[Observation]:
    observations = []

    for row_id, raw_row in enumerate(self.extract_rows(content)):
        observations.append(Observation(
            raw_data=self.extract_fields(raw_row),
            row_id=row_id  # 0-indexed, deterministic
        ))

    return observations
```

**Guarantee**: `row_id` is stable across re-parsing (same file, same row_ids).

---

### 3. Zero Rows is Success

**Property**: Empty file (0 observations) is valid parse success, NOT error.

**Example:**
```python
# Empty bank statement (no transactions this month)
result = parser.parse(empty_statement_pdf)

assert result.success == True
assert len(result.observations) == 0
```

**Rationale**: Empty ≠ Error. Corrupted file = error, empty file = success with 0 rows.

---

### 4. Warnings vs Errors

**Property**: Warnings allow parse to continue, errors fail entire parse.

```python
# WARNING: Suspicious but parseable
observation = Observation(
    raw_data={"date": "01/02/2024", "amount": "-5.75"},
    row_id=0,
    warnings=["Date format ambiguous (MM/DD or DD/MM?)"]
)

# ERROR: Corrupted, unparseable
raise ParseError("PDF structure invalid, cannot extract rows")
```

**Principle**: Use warnings generously, errors sparingly.

---

## Error Handling

| Error | When | Response |
|-------|------|----------|
| `ParseError` | Document is corrupted, unparseable | Raise error, log details, mark upload as `error` |
| `UnsupportedFormatError` | Parser doesn't recognize format | Raise error, try next parser in registry |
| `PartialParseError` | Some rows extracted, then corruption | Return partial results, log error |
| `EmptyDocumentWarning` | 0 rows extracted (not an error) | Return success with empty list |

---

## Configuration

```yaml
parser:
  parser_id: "bofa_pdf_parser"
  version: "1.2.0"
  supported_source_types:
    - "bofa_pdf"
  settings:
    encoding: "utf-8"
    strict_mode: false  # If true, fail on warnings
    max_rows: 10000     # Prevent memory exhaustion
```

---

## Multi-Domain Applicability

This primitive constructs verifiable truth about **raw data extraction** - a universal concept across ALL domains:

**Finance Domain:**
- Parser: `bofa_pdf_parser`
- Input: Bank statement PDF
- Output: `ObservationTransaction[]` (date, amount, description, account)
- Example: Extract 42 transactions from PDF table

**Healthcare Domain:**
- Parser: `lab_results_parser`
- Input: Lab report PDF
- Output: `ObservationLabResult[]` (test_name, value, unit, normal_range)
- Example: Extract glucose, cholesterol, etc. from lab PDF

**Legal Domain:**
- Parser: `contract_parser`
- Input: Contract PDF
- Output: `ObservationClause[]` (clause_number, text, page_number)
- Example: Extract clauses from legal contract

**Research Domain (RSRCH - Utilitario):**
- Parser: `web_page_fact_parser`
- Input: TechCrunch HTML page
- Output: `RawFact[]` (subject_entity, claim, fact_type, confidence)
- Example: Extract founder investment facts from TechCrunch article

**Manufacturing Domain:**
- Parser: `sensor_log_parser`
- Input: CSV from sensor
- Output: `ObservationMeasurement[]` (timestamp, sensor_id, value, unit)
- Example: Extract temperature readings from production line

**Media Domain:**
- Parser: `transcript_parser`
- Input: Video transcript JSON
- Output: `ObservationUtterance[]` (timestamp, speaker, text)
- Example: Extract subtitles from video file

---

## Implementation Example

### Finance: Bank Statement Parser

```python
class BofAPDFParser(Parser):
    parser_id = "bofa_pdf_parser"
    version = "1.2.0"
    supported_source_types = ["bofa_pdf"]

    def can_parse(self, content: bytes) -> bool:
        """Check if PDF has BoFA signature"""
        text = extract_text_from_pdf(content)
        return "BANK OF AMERICA" in text

    def parse(self, content: bytes) -> List[Observation]:
        """Extract transaction rows from PDF table"""
        pdf = PyPDF2.PdfReader(BytesIO(content))
        observations = []

        for page in pdf.pages:
            text = page.extract_text()
            table = self._extract_table(text)

            for row_id, row in enumerate(table):
                observations.append(Observation(
                    raw_data={
                        "date": row[0],         # AS-IS: "01/15/2024"
                        "description": row[1],  # AS-IS: "  STARBUCKS #1234  "
                        "amount": row[2],       # AS-IS: "-5.75"
                        "currency": "USD",      # Inferred from source_type
                        "account": "bofa_debit" # Inferred from source_type
                    },
                    row_id=row_id,
                    warnings=self._check_warnings(row)
                ))

        return observations

    def _extract_table(self, text: str) -> List[List[str]]:
        """Extract table from PDF text (domain-specific logic)"""
        # ... implementation details ...
        pass
```

---

## Performance Characteristics

| Operation | Time Complexity | Notes |
|-----------|-----------------|-------|
| `parse()` | O(n) where n = file size | Dominated by PDF/CSV parsing |
| `can_parse()` | O(1) | Quick signature check |

**Benchmarks (PDF parser, 100-row statement):**
- Parse latency: <5s (p95)
- Memory usage: <50MB
- Throughput: ~500 rows/second

---

## Testing Strategy

### Unit Tests

```python
def test_parse_valid_statement():
    parser = BofAPDFParser()
    content = read_file("test_data/bofa_jan_2024.pdf")

    observations = parser.parse(content)

    assert len(observations) == 42
    assert observations[0].row_id == 0
    assert observations[0].raw_data["date"] == "01/15/2024"

def test_parse_empty_statement():
    parser = BofAPDFParser()
    content = read_file("test_data/bofa_empty.pdf")

    observations = parser.parse(content)

    assert len(observations) == 0  # Success, not error

def test_parse_corrupted_pdf():
    parser = BofAPDFParser()
    content = b"corrupted data"

    with pytest.raises(ParseError):
        parser.parse(content)
```

### Golden Data Tests

```python
def test_parse_golden_data():
    """Parse known file, verify exact output"""
    parser = BofAPDFParser()
    content = read_file("golden/bofa_statement.pdf")

    observations = parser.parse(content)

    # Load expected output
    expected = json.load(open("golden/bofa_expected.json"))

    assert len(observations) == len(expected)
    for obs, exp in zip(observations, expected):
        assert obs.raw_data == exp
```

---

## Security Considerations

### 1. Resource Limits

```python
def parse(self, content: bytes) -> List[Observation]:
    # Limit file size
    if len(content) > 50 * 1024 * 1024:  # 50MB
        raise ParseError("File too large")

    # Limit rows extracted
    observations = []
    for row_id, row in enumerate(self.extract_rows(content)):
        if row_id >= self.config['max_rows']:
            raise ParseError(f"Too many rows (>{self.config['max_rows']})")

        observations.append(...)

    return observations
```

### 2. Sandboxing

```python
# Run parser in isolated process
def run_parser_sandboxed(parser: Parser, content: bytes) -> List[Observation]:
    with multiprocessing.Pool(1) as pool:
        future = pool.apply_async(parser.parse, (content,))

        try:
            return future.get(timeout=60)  # 60s timeout
        except TimeoutError:
            raise ParseError("Parser timeout (>60s)")
```

---

## Observability

### Metrics

```python
parser_operations_total{parser_id="bofa_pdf_parser", status="success"} 1234
parser_operations_total{parser_id="bofa_pdf_parser", status="error"} 5
parser_rows_extracted{parser_id="bofa_pdf_parser"} 52420
parser_latency_seconds{parser_id="bofa_pdf_parser", quantile="0.95"} 4.2
```

### Logs

```json
{
  "timestamp": "2025-10-23T14:35:00Z",
  "level": "INFO",
  "event": "parse.started",
  "upload_id": "UL_abc123",
  "parser_id": "bofa_pdf_parser",
  "parser_version": "1.2.0",
  "file_size_bytes": 245760
}
```

---

## Related Primitives

- **ParserRegistry**: Discovers and selects appropriate parser for `source_type`
- **ObservationStore**: Stores raw observations extracted by parser
- **ParseLog**: Records parser execution details
- **Normalizer** (Vertical 1.3): Validates and transforms parser output

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
